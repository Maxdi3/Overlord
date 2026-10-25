# 11 — Diagnostics, crash evidence, testing, build, release, acceptance

Test *recipes* (how to build the harnesses) are in `16-test-harness-recipes.md`. This file is
the policy layer: what to instrument, what evidence to gate on, how to build, package and
accept. A VR mod hooks the renderer and the XR frame loop, so crashes, stalls and "black
headset" reports are frequent and the user cannot read counters. Build the evidence
machinery early (phase 3, with the first solid-color frame), not after the first bug report.

Overlord files: `src/client/component/vr/diagnostics.cpp` (coordinator),
`vr/diagnostics/*` (status sections, `support_bundle.cpp`, `support_archive.cpp`,
`renderer_evidence.*`, `wait_timing.hpp`, `screen_display.hpp`, `post_aa.hpp`,
`input_status.hpp`, `steamvr_input_status.hpp`, `input_manifest.hpp`, `report_paths.hpp`,
`runtime_code_snapshot.*`, `native_flare_probe.*`, `README.md` = module boundaries),
`vr/vr_component.cpp` (watchdog thread), `vr/region_capture.cpp`, `vr/debug_options.hpp`,
`component/exception.cpp`, `src/common/exception/{minidump,dump_evidence}.*`,
`component/d3d11/texture_diagnostics.cpp`, `tools/{read_crash_evidence,analyze_region_capture,
package_client,deploy_client_pair,capture_equipment_visibility}.py`, `premake5.lua`,
`premake/vr_tests.lua`, `.github/workflows/build.yml`.
Docs: `docs/vr-diagnostics-workflow.md`, `development.md`, `releasing.md`,
`release-validation.md`, `vr-region-capture.md`, `vr-equipment-visibility-capture.md`,
`vr-interaction-diagnostics.md`, `source-provenance.md`.

---

## 1. Diagnostics architecture

### 1.1 One coordinator, pure append sections, one lock around format-and-replace

- **One coordinator owns all mutable diagnostic state and synchronization** (trace ring,
  callback timings, watchdog snapshots, snapshot collection, report persistence).
  Overlord: `vr/diagnostics.cpp`.
- **Each report section is a pure `append_X(std::ostringstream&, const snapshot&...)`
  function in its own file** that takes already-collected const snapshots by reference. A
  section never resamples, never owns runtime state, never copies a large report, and
  appends to one shared stream in a fixed order (it save/restores `flags()/precision()`
  around float dumps so the stream's numeric state is not corrupted). Forward-declared
  snapshot structs (`status_sections.hpp`) keep the report layer decoupled from every
  engine-module header. Overlord splits eleven sections by subject (session, environment,
  runtime, frontend, execution, scene, history, effect, owner, census, resource).
- **The status-file lock covers the whole format-and-replace transaction.** Do not
  parallelize sections or format outside the lock: an older snapshot could replace a newer
  one, or stream state could mix. This file lock is separate from producer locks.
- **Producers (render, Present, runtime, game threads) never format, hash, compress or do
  file I/O.** They bump atomics and publish fixed-size records; the diagnostics control
  thread does the heavy work. Heavyweight probes are explicit commands or startup-frozen
  options and are never enabled by the support bundle.
- **Report format is grep-friendly `key=value`** with stable prefixes, a `scope=` label on
  every counter group, `age_ms=-1` meaning "never observed", and explicit
  `dropped= / overflow= / snapshot_busy= / truncated=` on every bounded group, so missing
  data is visible instead of silently absent. If deep formatting fails, write
  `report_quality=core_only` and keep the core report rather than leaving an old file.
- Write core summaries *before* the deep ones in packages, so an oversized deep GPU census
  cannot consume the space reserved for identity/runtime/input evidence.
- Atomic writes: temp file + `FILE_FLAG_WRITE_THROUGH` + `MoveFileEx(REPLACE|WRITE_THROUGH)`
  (`utils::io::write_file_atomic`). The live-trace file never truncates the last complete
  snapshot in place (`diagnostics.cpp` ~1292: a kernel bugcheck can leave a zero-filled file).
  Report directory fallback: `<cwd>/diagnose` -> `%LOCALAPPDATA%/<Mod>/diagnose` ->
  `%TEMP%/<Mod>/diagnose`; report names reject `/ \ :` and `.`/`..`.
- Warm every status getter once on the loader thread before the watchdog thread exists
  (first-use static initialisation must not happen inside a stall).
- Long status text: formatters that go through a fixed CRT buffer crash on long output
  (`vr_carry_status` > 4 KiB overflowed a secure-CRT formatter). Size with copied varargs
  (up to 1 MiB) and chunk console output (`split_text_chunks`); test 4095/4096/8192/65536.

### 1.2 Non-blocking producers: try-lock buckets, first/latest failure, values only

Pattern in `wait_timing.hpp::bucket`, `screen_display.hpp::trace`, `post_aa.hpp::history`:

```cpp
// producer: never waits
void record(const sample& v) noexcept {
    if (busy_.test_and_set(std::memory_order_acquire)) {      // contended
        dropped_.fetch_add(1, std::memory_order_relaxed);     // visible loss
        return;
    }
    update_counters_and_first_latest(v);
    busy_.clear(std::memory_order_release);
}
// reader: also try-lock; if busy print snapshot_busy=1 and atomics only.
// Copy data out under the lock, format outside it.
```

- Keep **first failure, latest failure and last success separately** (`first_failure_ /
  last_failure_ / last_success_`; steamvr input: first setup / first failed setup / latest
  setup; post-AA `first_ / latest_`). The first failure is the root cause; later churn
  (cleanup errors, focus loss, device reset) must not erase it. Print "latest" only if it
  differs from "first". Console history keeps the first and last error through rollover and
  merges repeats (`repeats=N`); failure history keeps first + 8 recent + total count.
- **Diagnostics hold values, never resources**: no GPU leases, COM pointers or borrowed
  native pointers (`uintptr_t` identities + copied descriptors, `const char*` to static
  reason names). Never query D3D objects or native registries from a report. Reasoning:
  retained objects keep resources alive across device/session teardown.
- Hot-path diagnostics read fixed counters only; SHA-256, JSON, formatting and file I/O are
  control-thread work. Each per-eye substitution you add should count *rewrites, foreign
  bypasses* (calls from another thread/backend) and *contract mismatches* — the foreign
  counter proves you did not touch unrelated work.
- **A diagnostic failure must never veto production behaviour** (Overlord once returned the
  optional artifact-capture success as "view derivation succeeded", so a failed diagnostic
  rejected valid stereo views). Check disabled-ness and "already captured" *before* validating
  memory, and gate expensive observers by a startup-frozen selection.
- Label what a counter proves: "request only, not native acceptance", "CPU call returned,
  not GPU completion" (`gpu_retire_proof=none`), "submissions are not proof of visible
  stereo pixels", "a missing sample is not proof of success". Counters label evaluations,
  not hits.

### 1.3 Rejection-reason bitmasks ("why isn't X showing?")

Any feature gated by many identity/epoch/tracking prerequisites should record *every*
failed prerequisite, not just the first. Overlord `screen_display.hpp`:

```cpp
enum gate : uint64_t { owner=1<<0, epoch=1<<1, tracking=1<<3, reference=1<<4, recenter=1<<5,
                       world_scale=1<<6, hud_missing=1<<7, ... hud_output=1ull<<41 };  // 42 bits
inline void check(uint64_t& mask, bool valid, gate flag) { if (!valid) mask |= flag; }
std::string gates(uint64_t mask);   // "owner|epoch|hud_age"
```

One trace per consumer (fixed scope, Javelin, their HUDs, auxiliary view), with pair id,
publication id, device generation, owner:generation:revision, reference id and
expected-vs-actual ids, so a stale-handle bug is visible. `note_material()` keeps up to 8
matching material names (63 chars) plus an overflow counter. `post_aa.hpp` returns the first
failing named reason (`rtv_srv_resource_mismatch`, `device_mismatch`, ...) beside the expected
and actual descriptors, captured only on failure on the GPU-owner thread (no readback).

### 1.4 Wait-timing probes (find Present/XR pacing stalls)

`wait_timing.hpp`: 32 named observation sites (renderer maintenance, Present facade/dispatch,
xrWaitFrame, xrWaitSwapchainImage, WaitGetPoses, eye copy, composition, hands solver, reload
tick, ...).

| Rule | Detail |
|---|---|
| Gate | frozen startup option (`vr_debugWaitTiming`); disabled path reads no clock and no thread id |
| Wrap blocking calls as-is | scoped wrapper around the original `lock()`; no try-lock polling that would change behaviour |
| Two numbers | `wait_ns` (acquire) and `held_ns` (hold) recorded separately |
| Publish after unlock | the sample is published *after* the measured lock is released |
| Buckets | `alignas(64)` per site; counters waits >=100 us, waits >=1 ms, holds >=1 ms; recent ring of 8 only for slow samples |
| Context | thread-local `context{frame id, skeleton ts, stereo pair, eye}` tags samples; restored on unwind |
| Nested intervals | inclusive; report prints `nested_intervals=inclusive_do_not_sum` |
| Test seam | clock is a template parameter (disabled path, exact boundaries, exception unwind) |

### 1.5 Identity of the run (report header)

- Build: product version, configuration, git rev/dirty, compile date, module path.
  **Read PE identity from the loaded image, not the file on disk**: TimeDateStamp,
  SizeOfImage and the CodeView RSDS GUID+age (bounds-checked, <=64 debug entries,
  signature `0x53445352`). The GUID identifies the matching PDB. A *separate* `target
  identity` section identifies the original and cached **game** exe (size, SHA-256, PE
  timestamp, sentinel) — never conflate the two.
- Game context: map/level, in-game/paused/video state, frontend flag, and the **effective
  settings read live from the engine's own dvar table** (`source=native_current_values`),
  plus the engine settings that interact with VR (post-AA, display mode, vsync, max fps).
  Report what the engine has, not what the launcher profile says.
- Environment: OS via `RtlGetVersion` (not `GetVersionEx`), CPU, RAM, process uptime, **the
  actual D3D adapter name/vendor/LUID/dedicated MB/driver version** (this LUID is the one
  that must go to OpenXR; mismatch is `graphics_mismatch`), loaded module list (cap 512,
  `truncated=` flag; explains overlay-tool/hook overlap).
- Crash-safe native reads: keep `__try/__except` in tiny destructor-free leaf functions;
  string readers copy byte-by-byte with a bound and print `[truncated]`. Escape/truncate
  every string (`quoted_text` JSON-escapes, 512 chars).

### 1.6 Runtime status checklist (copy into any VR mod)

backend + selection reason; failure ring (first + recent: tick, session, frame, repeats, XrResult/
HRESULT, operation, stage); frame submission counters (completed / world / menu_only / empty,
last layer counts, age); loader_loaded / instance / session / session_running; last init stage;
session generation + READY/STOPPING/begin/end counts; implicit API-layer policy and runtime
override source/manifest (OpenXR runtime-selection debugging); recommended eye size/format and
the **submit-format probe**; native resolution state (requested vs scene vs desktop,
samples, rebuild count); frame protocol phase and `present_pending`; OpenVR WaitGetPoses
count/last/max/avg with histogram (>=2/11/50/100 ms), compositor frame timing; **thread
split** (renderer thread vs scene owner vs Present owner, `owner_contract` valid, violation
count — the Present-owner rule is runtime-verified, not assumed); scene mode requested vs
effective; head-pose bridge (recenter count, world scale, roll local/input-horizon/
output-horizon with min/max/peak — comfort acceptance: the horizon must stay level; base
camera policy, roomscale origin offset); per-hand pose quality (position valid vs tracked,
`interaction_ready`, analog value+source, press counters with generation). OpenVR input:
named `VRInputError_*` table (only when backend is OpenVR *and* the reason is an action-API
failure; compositor/tracking codes use other enums), event ids 100/101/102/108 and
1701-1708 (binding/manifest load results), header `event_application_attribution=unverified`.

### 1.7 Input diagnostics semantics

Keep four things distinct: **action availability, runtime focus, recorded loss conditions,
native gameplay acceptance.** "Active/valid means available, not pressed"; losses are
availability transitions, not proof of a physical disconnect; hand pose needs *both* grip and
aim valid; "not a rendering verdict". Per channel: `ever_valid`, `valid_samples`,
`activity_samples`, `last_valid_age_ms`, `losses`, `recoveries`, `api_failure_samples`, last
loss (reason, code, backend, sequence, reference, `runtime_focus_known`, gameplay_active),
last rejection, plus a bounded transition ring (8, oldest->newest, `discarded` counter;
identical unavailable frames do not fill it). Fresh = sampled age <= 150 ms; summaries
distinguish never-published / stale / never-available ("focus alone does not establish
working controllers") / lost-since. One automatic status save after the first input API
failure with 1 s settling (the player need not run anything). OpenVR additionally keeps
first/latest setup and binding probe (manifest path, bounded JSON inspection — 1 MiB,
nesting 32, <=16 binding files, at most 4 probes per init with the last reserved for an
API failure), the first manifest-load-failure event and the last 8 events; serial numbers
are not collected. Manifest reader rules (`input_manifest.hpp`): regular file only, size
cap before allocation, changed-during-read detection (`gcount != size || peek != EOF`),
nesting limit via throwing parse callback, relative-only binding paths (`..`, absolute,
`:`, NUL, URLs rejected; URLs descriptive, never fetched).

### 1.8 Per-feature status commands and opt-in probes

- Every feature that affects commands/state ships an on-demand `vr_<feature>_status` that
  prints and atomically writes a bounded report (`vr_fire_status`, `vr_input_status`,
  `vr_reload_status`, `vr_carry_status`, ...). `vr_status` and `vr_input_status` save the
  complete report to the shared latest-file path and say whether saving succeeded.
- Debug overlays and recorders are **opt-in, selected at startup and frozen for the process
  lifetime** (`debug_options::initialize` via `std::call_once`; console edits affect the
  next process). 13 probes in Overlord: view probes, auto snapshots, CPU perf capture, scene
  models, menu input, weapon events, vehicle, reload-well, HK-slap, bolt, cover-push overlays,
  wait timing, audio output. Debug overlays share the stereo line renderer (<= 192 segments,
  one instanced draw per batch) and allocate nothing when off. Reason: left-on probes cost
  real CPU (16-frame stack capture per transaction, hashing 0x501C8 bytes x256, `VirtualQuery`
  per diagnostic call). Do **not** disable the probe that production stereo publication
  depends on (`vr_engineProbe`) as "diagnostics".
- Throwaway RE samplers (`native_flare_probe.cpp`, Debug only): Release compiles the wrappers
  as inline pass-throughs so hook sites need no `#ifdef`; armed by command with a delay +
  window; per-channel slots with `atomic_bool ready`; run object held by `shared_ptr` in an
  atomic so a late callback is dropped, never waited on; destructor-free SEH leaf swallowing
  only ACCESS_VIOLATION/IN_PAGE_ERROR; `original()` called exactly once with no observer lock
  held; validate the layout before trusting it (allocator stride, count deltas).

### 1.9 Region capture: bounded recorder + loss-aware analyzer

`vr/region_capture.cpp` + `tools/analyze_region_capture.py` (`docs/vr-region-capture.md`).
Use this lightweight recorder *before* enabling GPU census/readback probes, which change the
measured workload.

- Fixed 16,384-row **MPSC queue; producers never wait** (full/contention = counted drop). Writer
  thread flushes ~200 ms. Defaults OFF; auto-stop 10 min / 128 MiB (`vr_perfStart [seconds]`,
  `vr_perfMark normal|slow|visual`, `vr_perfStop`; requires the startup debug option).
- Binary `H2VREG01`: 8-byte magic, LE uint64 QPC frequency/PID/unix ms, then 88-byte rows
  (11 x u64). Rows: stage begin/end (present, WaitGetPoses, submit, conversion, dynamic
  upload), owner per eye, eye origins, writer health (cumulative drop count), markers,
  nested phase begin/end/state.
- **Analyzer rule: never count an interval that may cross lost events.** Between consecutive
  health rows where the drop counter rose it builds a loss window and excludes every
  overlapping timing (`timing_intervals_excluded_for_loss`); a lost phase end is not paired
  with a later begin. Per-thread begin/end stacks; nested waits pushed on a stack (native
  waits pump jobs and legitimately nest). `clean_stop` only means a footer exists. Partial
  tails are reported (`trailing_bytes`). Outputs: `timeline.csv` (per-second n/mean/p95/max),
  `phases.csv`, `lists.csv`, `evidence.json` (slow spans >= 33.333 ms).
- **Metric traps:** successful right-eye Submit *interval* is application stereo cadence, not
  HMD display FPS/reprojection or GPU time; CPU wall != submission != GPU != display FPS;
  nested phases must not be summed; do not claim FPS gains without measurement; do not
  attribute individually changes that shipped together; probes must not change the measured
  workload. The recorder embeds its limits in the report ("CPU wall time only; DrawIndexed on
  owner thread only; list hashes are CPU descriptors, not bindings").
- Use `K32QueryWorkingSetEx` for small dynamic-arena spans with a `VirtualQuery` fallback; never
  cache page permissions across native calls.

### 1.10 Cross-eye lockstep census (only if you re-run the engine per eye)

When per-eye engine re-execution shows artifacts (reflection jumps, missing transparency,
black surfaces), Overlord found causes by instrumenting the D3D boundary and diffing the two
eyes. Build a small version early (steps 1-2), the rest only when an artifact demands it:

1. Record the ordered GPU call stream per eye (api, caller, args, VS/PS/CS, target id, RTV/DSV,
   depth/blend/raster/layout state, blend factor, topology, index buffer, viewport/scissor,
   hashes of CB/SRV/UAV binding sets); compare in ordinal lockstep; report mismatch categories,
   first mismatch ordinal and both signatures. A healthy stereo pass has the *same* call stream
   in both eyes; divergence points at stale per-eye state.
2. **Read-before-write candidates:** a resource eye 1 reads before writing but eye 0 wrote is a
   per-eye-isolation candidate (history textures).
3. Per-eye constant-buffer provenance: hash content, record upload source (map/update/copy),
   uploader caller/thread; classify slot use via shader reflection (used/unused/unknown) so
   "left bound / right null but unused" is not a false alarm; byte/word diff with float
   interpretations. Content never uploaded since tracking began is `content_unknown`, never
   "equal".
4. Dynamic-arena families: a *constant nonzero* StartIndex delta between eyes is the signature
   of a consumed-sequentially shared dynamic arena (running the scene twice advances the
   cursor; rebase/restore per eye).
5. Same-pair GPU readback is the only thing that exposed some bugs (draw counts, shaders and
   resource identity looked identical). Compare byte ranges, restrict to the exact indexed
   range, and keep the diagnostic copy itself out of the observer stream.
6. Bytecode reflection (`D3DReflect` + disassembly) + a write-provenance log at the D3D API
   boundary discovers which shader consumes which resource when you cannot read engine source.

### 1.11 Runtime code snapshot (when the exe is encrypted on disk)

If the game's `.text` is encrypted/obfuscated on disk (H2-only: it is), offline disassembly of
the exe is useless. `runtime_code_snapshot.cpp` dumps validated executable ranges of the
*running* process to `minidumps/runtime-code/<name>-<rva range>.bin` + `manifest.txt` (state
`in_progress` -> `complete|incomplete`, per-region SHA-256, scope string). Rules: take it at
the loader boundary after target validation and **before your own patches** (the manifest
records `post_existing_h2mod_hooks_pre_vr_observation_calls`); validate each range by walking
`VirtualQuery` regions (committed, executable, no GUARD/NOACCESS, overflow-checked, <= 1 MiB);
decode known prior patch sites (E8/E9 rel32 -> target) so analysts recognise detours; capture
the code around any recurring crash address once instead of re-running the game. Byte masks
for hook verification are compared against this decrypted runtime memory.

---

## 2. Support bundle (one-button player report)

Goal: the player presses one key, uploads one file, and you can answer "what build, what
runtime, what failed first" without a second round trip. Overlord: `support_bundle.cpp`,
`support_archive.cpp`, `docs/vr-diagnostics-workflow.md`.

| Aspect | Overlord behaviour |
|---|---|
| Trigger | `vr_diagnose` or **Ctrl+Shift+F8**: `GetAsyncKeyState` polled in a 10 ms scheduler loop, edge-detected, honoured only if the foreground window belongs to this process or its console |
| Worker | `utils::latest_task_worker` (replace-latest); manual request always beats an automatic checkpoint (`admission_mutex`); second manual request prints "already running"; `stop()` sets stopping, notifies the condvar, joins before VR teardown |
| Sequence | README -> core-1 / status-1 / trace-1 immediately -> wait **5 s** (cancellable condvar) -> core-2 / status-2 / trace-2 -> saved settings JSON (profile path, launcher-resolved values, risk settings, startup preflight) -> recent console history -> existing artifacts |
| Fresh before historical | two *fresh* full statuses ~5 s apart; historical files labelled `previous-status-N`, `existing-*`; manifest states association with this run "is not assumed" |
| Dumps included | at most the 2 newest `.txt` and 2 newest `.dmp` from `minidumps/` whose names start with `overlord-crash-/emergency-/scene-stall-/backend-stall-/interop-stall-/soft-freeze-`; directory scan capped at 2048 entries |
| Limits | 8 MiB per file, 24 MiB total; over-limit entries are **omitted and noted** in `manifest.json` (never silently truncated); core summaries first |
| Entry safety | id charset `[A-Za-z0-9_-]`; id = `<mod>-diagnostics-<UTC>-p<pid>-<tick>-<serial>`; refuses an existing path; write-check probe file; entry names rejected if they contain `/ \ :` or are `.`/`..`; includes read `size+1` bytes and compare `gcount` (changed-during-copy detection), `symlink_status` rejects non-regular files |
| Publication | files staged to disk first (a ZIP failure leaves usable text), ZIP written to `package.partial`, then `MoveFileExW(MOVEFILE_WRITE_THROUGH)` into the uploadable name — an incomplete ZIP is never presented as complete; staging removed after publish. minizip with `fill_win32_filefunc64W` (Unicode paths), check `zipClose` (full disk may surface only at the central directory) |
| Automatic checkpoints | poll 1000 ms; signature = `session_generation<<32 ^ retained_failures<<8 ^ state<<2 ^ native_ready ^ 1`; write a status snapshot only when the signature changes; failure retries after 30 s; never replaces a pending manual task |
| UX | console prints the hotkey at startup; opens Explorer with the ZIP selected (`SHOpenFolderAndSelectItems`), reveals the staging dir on failure; "Nothing was uploaded automatically"; the README asks the player to describe what they saw because the package cannot observe the HMD |
| Not included | game assets, whole profiles, executed console commands, heavyweight probes |

Also add: a mandatory honest-status notice in the launcher ("Beta; saves may break; run
`vr_diagnose` and attach a minidump when reporting"), and a build-type banner (Debug is not
for frame-rate measurement).

---

## 3. Watchdogs

Two independent detectors (verified in `vr_component.cpp`):

| Detector | Runs on | Trigger | Dump focus thread | Files |
|---|---|---|---|---|
| Soft freeze | scheduler async loop, 250 ms | Present in progress for >= **5 s** | last Present thread | `overlord-soft-freeze-*` |
| Scene-hook stall | dedicated watchdog thread, `Sleep(250)` | scene hook depth != 0 for >= **5 s** since entry | scene thread | `overlord-scene-stall-*` |
| GPU-interop stall (cross-device copy gate) | same thread | depth != 0 for >= **1 s** | interop thread | `overlord-interop-stall-*` |
| Backend CPU stall | same thread | depth != 0 and >= **5 s since `last_progress_tick`** (not since entry) | backend thread | `overlord-backend-stall-*` |

- **Double-read confirmation:** re-read the watchdog record and require the same
  `sequence/depth/entered_tick(/phase)` before reporting, and report **once per sequence**.
  This avoids false positives on a stall that cleared between samples.
- Hook entry/exit are depth-counted atomics (`depth`, `entered_tick`, `thread_id`, `sequence`);
  the backend record is a seqlock (retry 4 times; zeros if inconsistent); backend ids stay
  monotonic across probe resets so a late wrapper cannot clear a new owner. Watchdog state and
  tokens stay active even with trace storage off.
- Each stall writes `<stem>.txt` (Present state/thread/frame/result, device-removed reason,
  **device-creation evidence**: factory count before device, requested vs actual flags,
  single-threaded flag, multithread-protect state, creation thread, GPU-queue gate contention;
  then the full trace) and `<stem>.dmp` via `exception::write_process_minidump(path,
  focus_thread)`. Thread-targeted dumps matter: the writer thread is not the stuck thread.
- GPU queue gate: record a *waiter record* (thread id, client enum, start tick) so a watchdog can
  tell who is stuck on the gate; blocking OpenVR calls must never hold it.
- With the snapshots option on, every 2 s the watchdog persists status + live trace + renderer
  evidence; a "safe to close" console message is printed only when terminal evidence, live
  trace and status were all written from an identical stable terminal signature (before ==
  after). Once proven, periodic serialization stops (the terminal report is multi-megabyte).
- Hang triage: a missing `runtime_submit_result` does not excuse ignoring the preceding GPU
  gate; `Responding=True` does not rule out a render livelock; if the console cannot run
  commands use watchdog evidence or a live thread dump instead of re-entering status commands.
- Shutdown: component finalization can only complete in Present-post (listeners are
  unsubscribed only once `runtime.shutdown_complete()`); stop+join the watchdog first.

---

## 4. Crash evidence

Problem: the crash path is hostile (heap may be corrupt, a lock may be held by the faulting
thread, the stack may be exhausted), and a manually mapped game image is not in the loader
module list, so a plain minidump cannot be unwound or symbolicated.

### 4.1 Crash-path flow (Overlord `exception.cpp`)

1. Unhandled-exception filter is installed and `SetUnhandledExceptionFilter` is hijacked
   (game/middleware cannot replace it). Harmless codes (integer/float overflow, single-step)
   continue execution.
2. **Emergency dump thread**, created in the component constructor (before any fault): the
   filter copies `EXCEPTION_RECORD`+`CONTEXT` into static storage, `sprintf_s`-formats a fixed
   text report into a static buffer, signals the thread and waits <= 15 s; the thread writes
   `minidumps/overlord-emergency-<ts>-p<pid>-t<tid>.dmp/.txt` straight to the final file. Why:
   heap/stack corruption can fault again inside `std::string`/ZIP code.
3. Only then the "historical" path: `.dmp` + `.txt` raw sidecars first
   (`overlord-crash-<mode>-<ts>`), ZIP last. Never read, format or compress a whole dump
   inside exception handling.
4. Recovery: redirect `Rip` to a stub that either `Com_Error(ERR_DROP)`s (recoverable only if
   on the main game thread, >= 1 min since the last recovery and < 3 recoveries) or shows a
   dialog, suspends other threads and `TerminateProcess`es. `cg_legacyCrashHandling` forces the
   dialog path.

### 4.2 Bounded evidence planner (`dump_evidence.*`)

`exception::evidence::prepare()` builds a fixed-size `plan` in static storage
(`dump_plan`, off the possibly exhausted crash stack) **before** `MiniDumpWriteDump`:

- No allocator, no loader enumeration, no DbgHelp symbol calls, no graphics API calls, no waits
  on locks the faulting thread may hold.
- Limits: 256 memory ranges, 16 modules, 64 candidate PCs, **8 MiB byte budget**, function
  table <= 2 MiB, unwind chain depth 8. Limits hit are recorded as bit flags in
  `report.limits` (1 range cap, 2 byte budget, 4 module cap, 8 candidate cap, 16 unwind chain,
  32 no context, 64 stack read failure, 128 function table too large).
- Content: RIP; 14 GPRs x 64 bytes behind each pointer; 16 KiB of stack words from RSP. Every
  stack word pointing into executable memory *nominates* a candidate PC (`stack_candidates_only=1`
  — never claimed as a proven call chain). Per candidate: 128 bytes of code, the module's PE
  headers (0x1000), the `.pdata` RUNTIME_FUNCTION table (binary-searched for the function) and the
  chained UNWIND_INFO, so **offline unwinding works without the game binary**.
- Every span is validated with `VirtualQuery` (MEM_COMMIT, readable, not PAGE_GUARD/NOACCESS);
  overlapping/duplicate ranges are deduplicated.
- Fed back through `MINIDUMP_CALLBACK_INFORMATION` as MemoryCallbacks.
  **Pitfall: `IsProcessSnapshotCallback` must set `Status = S_OK`.** `S_FALSE` makes DbgCore
  treat the live process handle as a PSS snapshot handle. Also `ReadMemoryFailureCallback`
  returns `S_OK`, `MiniDumpIgnoreInaccessibleMemory` is set.
- **Custom user stream** `0x4F564C44` ("OVLD"): fixed little-endian u64 header (version, focus
  thread, exception code, pc, sp, sampled tick, range/module/candidate counts, skipped,
  requested bytes, budget, scanned stack bytes, context_available, stack_candidates_only,
  limits) + 16 module records (base, size, timestamp, function-table rva/size, scheduled) +
  256 ranges (address, size, kind: headers/function_table/unwind_info/code/registers).
  `static_assert(sizeof(stream)==128+16*48+256*24)`. `tools/read_crash_evidence.py` decodes it
  with only `struct`/`mmap` (no game binary, no pip), validates every offset, cross-checks which
  scheduled ranges actually landed in the memory list (`captured`), caps directory count 4096 /
  memory rows 65536, and treats older dumps as `absent`.
- Dump type is *not* full-memory: `IgnoreInaccessibleMemory | WithHandleData | ScanMemory |
  WithProcessThreadData | WithFullMemoryInfo | WithThreadInfo | WithUnloadedModules`.

### 4.3 Gate, writers, supplements

- **All dump writers share one try-only `std::atomic_flag` gate.** DbgHelp is process-global and
  single-threaded; a concurrent request returns false/empty immediately instead of waiting.
- Three writers: `create_minidump` (temp file DELETE_ON_CLOSE -> string; on failure MessageBox +
  `TerminateProcess(123)`), `write_process_minidump(path, focus_thread)` (watchdogs: null
  exception, `RtlCaptureContext` for the current thread or **briefly suspend only the target
  thread**, `GetThreadContext`, resume — a pre-dump sample, not an atomic snapshot; tests prove
  the thread is not left suspended), `write_exception_minidump(path, ptrs, exception_thread_id)`
  (direct to final file for the dedicated writer thread).
- **Supplement providers**: `register_dump_supplement` accepts up to 4 process-lifetime,
  allocation-free callbacks, each returning a fixed buffer <= 512 KiB, with a stream type above
  `LastReservedStream`, no duplicate types, never waiting on a producer lock. Overlord's one
  provider is the D3D11 `CreateTexture2D` flight recorder (`0x4F565458` "OVTX": 256 recent
  events + 32 active ops + one latched first failure, 29 x u64 per event; first
  device-removed/hung/reset error outranks an earlier ordinary failure; descriptors only —
  no pixel data, no COM retain; try-only publish guard; passes through unregistered devices
  sharing the vtable; preserves HRESULT, output pointer and `LastError`). Active (never-returned)
  calls survive ring wrap and a blocked call.
- Keep matching PDBs (private; they can embed local source paths). A dump is interpretable
  only with the PDB of the exact build; do not read an old dump with a rebuilt PDB.
- A D3D11 error dialog / device removal may produce **no** dump (dialog path does not go
  through SEH): record HRESULT, `GetDeviceRemovedReason`, last Present result/timestamp in the
  status report.

---

## 5. Evidence gating discipline

### 5.1 Reporting labels

- Tag every claim **confirmed** (observed), **decision**, or **pending** (needs runtime/HMD
  validation). Separate *source inspection*, *offline tests*, *build success*, *desktop run*,
  *headset acceptance*. Build+deploy success is engineering only.
- Test output says what it is not: "CPU/WARP only; no H2/HMD acceptance". Hardware probes pass
  messages still tell a human to verify (e.g. "pulsing magenta left eye and cyan right eye").
- Record dated capture provenance + SHA-256 in fixtures built from live observations; keep
  withdrawn fixes honest (a mathematically valid fix that the operator saw no improvement from
  was withdrawn, not kept).
- Reproduce the pre-fix failure first in the regression test, then fix.

### 5.2 Engine-evidence gating (strict, enum-valued, immutable)

Overlord gates production stereo behind an evidence apparatus. A new port should copy only the
principles first:

- **States are enums, not booleans:** `not_observed / observed / incomplete / truncated /
  invalid_shape / unreadable` plus terminal `complete / failed / opaque`. "Zero hits" is
  distinguishable from "census did not run". Bounded sample storage overflowing makes evidence
  `truncated` (fatal to completeness of evidence, not to the enclosing transaction).
  **Unknown is never equal** (unobserved content is `content_unknown`, not "same").
- **One-shot immutable captures:** `one_shot_artifact<Size>` with `atomic_uint32` state
  0=empty / 1=producer copying / 2=immutable-readable; the producer copies once with release;
  consumers read with acquire; after ready the bytes never change for the process. Hot-path
  callbacks never serialize or write files; only the diagnostics control thread does
  (`checkpoint()`, try/catch -> bool). Manifests list each artifact with state, size, SHA-256
  and `delta=<name> changed_bytes first last` between pairs (proves exactly which bytes a
  per-eye derivation changes). FNV-1a content signatures version manifests by exact bytes.
- **Terminal evidence is immutable:** after a complete report, later calls, a replacement
  device or `invalidate_device` must leave the report `memcmp`-equal. A report is a snapshot
  tagged with a device generation.
- **Strict acceptance** (`accepts_complete_execution_evidence`, 6 booleans: observation,
  backend-target frame, output-merger, draw-indexed, execution census, terminal device
  identity consistent): each gate requires `attempts==1 && completions==1 && failures==0`,
  non-zero positive counters, zero for every error counter (context/thread mismatches,
  overflows, query failures, hook failures, foreign context, admission collisions), report vs
  status field-by-field consistency, one device generation and one D3D context across probes,
  and frame-boundary arithmetic (`end_present_pre == start_present_post + 1`, successful
  Present, same thread). Manifest state machine: waiting_for_level -> execution_pending ->
  incomplete | **opaque** | failed | complete; *opaque* = deferred contexts / ExecuteCommandList
  seen, so the census cannot account for all work and the result is unacceptable.
- **Readiness comes from the current process:** files left by another process (or an earlier
  run) cannot satisfy readiness; a manifest embeds the hash of the immutable terminal report.
  Retired experiments get explicit `state=retired` manifests with the reason so stale artifacts
  are not mistaken for a runtime path.
- **Declare the side-effect budget** in every evidence file (`additional_gpu_commands=0
  target_replacements=0 command_replays=0 openvr_calls=0 registry_writes=0`) and prove
  command-stream immutability by hashing before/after.
- **Probe -> gate -> modifier -> retire lifecycle:** observation hooks first, strict gate, then
  production behaviour; failed approaches stay as `retired` with the reason. Overlord's two
  dead ends (H2-only): synthesizing a second scene record before `R_EndFrame` (engine's fatal
  "Tried to use (null)") and calling the stateful scene owner twice (corrupts transient
  surface-builder state); production derives per-eye view slots from the engine's *pure*
  finalizer called twice per family. Rule: never call a stateful engine "owner" twice per frame
  unless its transient state is proven idempotent.
- **Validate before trusting a layout** ("did the emission I think happened actually
  happen?"): stride/count assertions before and after.
- **Engine budget overflow warnings are acceptance failures:** grep native logs for
  "exceeded"/"not drawing" during stereo tests (Overlord: `MAX_SCENE_SURFS_SIZE ... exceeded -
  not drawing surface`).
- Read-only live observation (Frida / read-only memory sampling) to capture native contracts
  before patching; session tokens and entity numbers are evidence, not keys; historical
  status files may belong to another level — compare instance/binding and PDB.

### 5.3 Hook-site evidence

Pin the exe (size + SHA-256 + in-memory sentinel), record the identity in a mutex-protected
snapshot, and mask-verify every patch site; mismatch = skip the feature, log the first
mismatching offset. Gate the stereo bridge on the same identity as the probes (target mismatch
-> bridge never active even if views are published). Offline census tooling (Capstone over a
clean `.text` dump, assert exact per-field site counts) generates compile-time all-or-none patch
tables with before-bytes; runtime never scans-and-patches (`tools/generate_scene_surface_contract.py`).

---

## 6. Test pyramid (what each level proves)

Recipes with code are in `16-test-harness-recipes.md`.

| Level | Proves | Does **not** prove | Overlord examples |
|---|---|---|---|
| Pure CPU policy suites (header-only, injected `check`) | math, state machines, ammo conservation, input edges, IK, camera policies | engine/GPU/HMD behaviour | `vr-weapon-mechanics-tests`, `vr-hand-pose-tests`, `vr-controller-input-tests`, `vr-campaign-policy-tests`, `vr-aim-assist-tests` |
| Production-source slicing + boundary fakes | the *actual* adapter code with its engine/OS endpoints replaced | the real engine endpoint | `tests/background_update_tests.py`, `config_validation_tests.py`, `vr/native_*_adapter_tests.py`, `aim_assist_adapter_tests.py` |
| Synthetic native layouts | decoders/verifiers on byte-exact fake structs, PE images, bytecode | that the live engine layout matches | `game-loader-tests`, `vr-native-binding-tests`, bytecode inspectors |
| asmjit bridge tests | register/flag/SIMD/stack preservation of mid-function hooks incl. far relays | the hook site in the real binary | `camera_bob_tests`, `ads_sprint_tests`, `vr-native-menu-hook-tests` |
| WARP D3D11 | resource contracts, blit/blend/colour math, hook vtable behaviour, state restore | H2's native call chain, AMD/NVIDIA drivers, the HMD image | `vr-desktop-mirror-tests`, `vr-d3d11-eye-resource-isolation-probe`, `vr-engine-ssr-history-probe`, `vr-d3d11-draw-indexed-probe` |
| Mock runtime (fake `openxr_loader.dll`, strict SDK doubles) | lifecycle, call order, failure recovery, exact-pose submission, eye identity | installed runtime/layers, headset | `vr-runtime-mock-smoke`, `vr-runtime-no-loader-smoke`, `vr-openvr-lifecycle-tests` |
| Real-loader probe (opt-in) | loader/layer policy reaches a safe state | acceptance | `vr-runtime-real-loader-probe` |
| Hardware probe (opt-in) | runtime + devices deliver complete stereo pairs (`submitted>=90`, left==right released) | acceptance in the game | `vr-steamvr-hardware-probe` |
| Artifact audit (EXE + PDB) | production components were not compiled out; mutable storage writable | semantics | `tests/vr/client_feature_parity_tests.py`, `vr-aggregate-publication-tests` |
| Launcher in-process probe | embedded UI actually renders/bridges | game | `tools/verify_launcher.py` (`-launcher-smoke`) |
| In-game observe-only probes | census of hooks, threads, record identities (runtime evidence) | acceptance | `engine_view_probe`, `engine_backend_probe`, `gpu_census` (the `*_probe_smoke.cpp` files are CPU bookkeeping tests of these, *not* in-game runs) |
| **HMD acceptance** | a human in the headset with a checklist | — | section 10 |

Mock proves *our* code, the real-loader probe proves loader/layer policy, the HMD proves
acceptance. Three more rules:

- **Plan test seams up front.** Race tests need production hooks such as
  `test_arm_admission_pause(stage)`, `reset_for_tests`, `*_for_tests` counters, and
  `H2VR_*_TESTING` defines that swap hot-path hooks for stubs. Link-time stubs replace
  diagnostics/engine boundaries (`diagnostics_stub.cpp`, `native_menu_stubs.cpp`,
  `gpu_census_stubs.cpp`) so subsystems link without the game.
- **One standalone executable per concern.** premake `test_executable(name, output_dir)` in
  `premake/vr_tests.lua` (flag `--with-vr-tests`, ~60 targets): each compiles only the
  header/cpp under test; feature dependencies stay explicit next to each target; weapon catalog
  as a test library. The docs map change area -> target (`development.md` table) — keep that
  table current. Do not run every `*probe.exe` by filename; select by change area.
- **Platform caveat for porters:** all of these are Windows+MSVC (python adapters use
  `vswhere`+`vcvars64.bat`; some require VS 16.x `installationVersion` and WARP; killfeed needs
  the gsc tool + Lua). Only the pure-math/policy header suites are portable. Hosted CI runners
  are **elevated (high integrity)**, which changes runtime-override behaviour; tests branch on
  `process_integrity_level()`. Some tests need a larger stack (`/STACK:8388608` for the
  spatial-panel test; Debug keeps big pixel buffers in `main`'s frame).

### CI (Overlord `build.yml`)

Windows x64 (`windows-2022`), matrix `RelWithDebInfo / Debug / Release`: controller-binding
unittests -> premake generate -> build client -> no-loader smoke -> **aggregate-publication
qualification** -> launcher `npm test` (Debug) -> draw-indexed WARP probe -> spatial-panel and
controller-input tests (non-Release) -> OpenXR mock smoke -> package overlay (which re-runs the
artifact audit) -> upload artifact. It never publishes or deploys. GitHub Actions success is
not headset acceptance.

---

## 7. Build pitfalls

### 7.1 MSVC v142 WPO miscompiles lock-free publication buffers

*The incident (confirmed, `docs/release-validation.md`):* with `/O1`/`/O2` + `/GL` `/LTCG`,
MSVC v142 19.29.30154 placed **nonzero-initialized mutable aggregates into read-only `.rdata`
while keeping the writes** (level-load write access violations: 15,416-byte carry-model buffer,
204,840-byte output-merger report; 30,024-byte backend-target report found by scan), or dropped
the publication write. Pinning the objects to a writable section fixed placement but **not** the
lost-write behaviour. Symbol presence checks passed on the bad build.

Policy and evidence (copy for any mod that publishes snapshots through globals):

1. **Disable WPO for every project in Release**, including common code and dependencies:
   `buildoptions {"/GL-"}`, `linkoptions {"/LTCG:OFF"}`; keep ordinary `/O1` and full symbols.
   Remove per-buffer section workarounds. Re-enable only after qualifying the specific compiler
   *and* fresh Release runtime acceptance.
2. **Qualification test** (`tests/vr/aggregate_publication_tests.cpp`): a 4100-byte struct of
   1025 items with a nonzero default; `noinline` exported `publish/read_publication/
   reset_publication`; the destination address stays private (passing it to an opaque API can
   hide the defect); `publish`, read back, `reset`, read the defaults back. Built in Release in
   CI and gated by the packager (must be newer than `premake5.lua`, `premake/vr_tests.lua` and
   its source; run with a 30 s timeout).
3. **Artifact audit** (`client_feature_parity_tests.py`): open the finished EXE + same-name PDB
   with dbghelp (`SymSetOptions` UNDNAME|FAIL_CRITICAL_ERRORS|EXACT_SYMBOLS|NO_PROMPTS so a
   mismatched PDB fails), enumerate ~35 production entry points and require each to exist as an
   emitted function (`SymTagFunction`, size > 0) — this catches whole components compiled out by
   build guards — and require the named mutable publication buffers to lie in a section with
   the writable flag (`0x80000000`). Run it for *every* configuration you ship (180 s timeout).
4. A passing Debug build proves nothing about Release; Release uses `FatalCompileWarnings`.
   Skinned-visibility/feature hooks must be present in every build; build mode is not a feature
   switch.

### 7.2 Configurations and flags (`premake5.lua`)

| Config | Flags | Use |
|---|---|---|
| Debug | `-debug` target suffix, `/bigobj`, `DEBUG` | diagnosis; 35-44 ms per eye pair at ~86-88% of a core (JMC/RTC helpers) — never measure performance on it |
| **RelWithDebInfo** | Speed, Release CRT, `NoRuntimeChecks`, `/JMC-`, `H2VR_PROFILE_BUILD` | **normal gameplay and performance baseline**; full PDBs |
| Release | Size (`/O1`), `FatalCompileWarnings`, `/GL-`, `/LTCG:OFF` | packaged delivery candidate |

Link: `/DYNAMICBASE:NO /SAFESEH:NO /LARGEADDRESSAWARE /LAST:.main /STACK` (host must not be
randomized into the game's range; 16 MiB stack because game threads inherit the host's), static CRT,
`characterset ASCII` with explicit `/utf-8` source encoding, C++latest, `/bigobj`. A few TUs are
`NoPCH` (huge weapon-catalog TU; loader TUs that must compile standalone in tests). Surface
`build_config.hpp` (which config is running) to the launcher and report. Do not attribute a
performance change to a fix when build optimization changed in the same step.

- Project references: `dependson` utility projects (TLS helper DLL, pinned OpenXR loader,
  launcher UI generation) are turned into real `ProjectReference`s by an MSBuild override so a
  direct `.vcxproj` build still produces prerequisites (`premake/msbuild_dependencies.lua`).
- OpenXR loader: build the pinned Khronos SDK with CMake (`tools/build_openxr_loader.py`;
  dynamic loader, no layers/tests) and ship it app-local as `openxr_loader.dll`; one optimized
  loader serves both client configs.
- Pinned toolchain for the launcher: Node >= 22.12, WebView2 SDK fetched from NuGet (size-capped,
  only header + static lib extracted), `npm ci` only when the lockfile is newer, serialized by a
  lock file; embed `dist/` into RC resources (<= 512 files, 16 MiB); statically link the
  WebView2 loader (Evergreen runtime installed separately).
- A source-only snapshot without Git must still build (`GIT_*` macros fall back to `"unknown"`).

---

## 8. Debugging workflow

1. **Identify the run:** build commit + dirty flag, EXE/PDB hashes, process start time (matching
   files on disk do not prove the running build), map/checkpoint, headset, runtime, GPU/driver,
   settings.
2. **Save evidence before restarting** (status, traces, dumps, captures per incident together; note
   timing relationships; `vr_reinit` / `vr_recenter` change state — capture first).
3. Pick the smallest observation: status -> region capture (timing) -> focused trace -> dump.
4. Separate input, mechanical state, native result and visual presentation before tuning offsets.
5. Agree a capture window with the headset operator (section 10). Outside it assume the headset is off.

| Symptom | Inspect first | Limit of the conclusion |
|---|---|---|
| HMD waiting while desktop works | backend, init stage, tracking, native pairs, Submit, last error | tracking counters establish the pose path only |
| Game and SteamVR on different GPUs | adapter identity / `graphics_mismatch` | verify device match before changing scene code |
| One-eye artifacts, reflection jumps, missing transparency | pair/eye/generation, owner failures, isolation status, region capture | same pointer/descriptor hash does not prove same contents or timing |
| Slowdown near an area/effect | owner/Present/WaitGetPoses/Submit phases in region capture | CPU wall, submission interval, GPU time, HMD FPS differ |
| Console produces no output | latest status snapshot, live trace, thread stacks in watchdog dumps | command may be unscheduled or waiting on a lock; not proof of init failure |
| D3D11 error dialog / device removal | HRESULT, `GetDeviceRemovedReason`, last Present result | dialog path may bypass SEH, so no dump |
| Process crash | raw `.dmp`, `.txt`, exception code/context, matching PDB, `read_crash_evidence.py` | faulting module = location, not necessarily root cause; async GPU-driver crashes follow their cause |
| Wrong weapon/hand behaviour | feature status commands, interaction diagnostics | separate input / mechanical / native / visual |

Record conclusions with this structure (and say what was not covered):

```text
Status: implemented / offline tests passed / hardware accepted / pending validation
Build identity: commit, workspace changes, EXE/PDB hashes, running process identity
Reproduction: level, devices, configuration, actions, symptoms
Evidence: files, time range, completeness, missing data
Cause: established facts vs inferences
Fix constraints: ownership, generations, native behaviour, rejected approaches
Validation: commands and results; scope actually observed
Remaining limits: untested devices, levels, reconnection, long-term operation
```

---

## 9. Packaging, deployment, release, licensing

### 9.1 What a release artifact is

A source publication, a client overlay and a complete playable distribution are different
artifacts. Overlord ships a **client overlay** only and says so ("Client overlay only: base
game/mod data and clean-install acceptance are separate"). Do not advertise an overlay as a
first-install package until the required data is assembled with redistribution rights and
tested on a clean install.

Package contents (`tools/package_client.py`): `<mod>.exe`, app-local `openxr_loader.dll`,
`<mod>.vrmanifest`, SteamVR `cover*.png` (920x430 `image_path`, 600x900 `image_path_capsule`,
460x215 export), `vr_input/*.json`, loose project scripts, compiled mod fastfiles, LICENSE,
THIRD_PARTY_NOTICES, docs and **per-dependency licence files** (GSL, WinToast, asmjit, CascLib,
curl, minhook, OpenVR, OpenXR (MIT/Apache/CC-BY/jsoncpp), rapidjson, sol2, stb, udis86, zlib, ...).
**Never game assets**, user settings, logs, saves or raw extraction dumps. PDBs are excluded
from archives and retained privately (local source paths); local staging may include the PDB.

### 9.2 Packager rules

- Qualification gates run before collecting anything: artifact audit against the exact built
  EXE/PDB, and for Release the aggregate-publication probe (must be newer than its inputs).
  A failed/timed-out check aborts packaging.
- Validate the `.vrmanifest`: exactly one application whose `binary_path_windows` is the packaged
  exe; `image_path`, `image_path_capsule`, `action_manifest_path` relative, forward-slash, no `:`,
  no `..`, present in the file set; every `default_bindings[].binding_url` exists.
- Inputs: missing, empty, symlink or duplicate package inputs are rejected.
- Release name is a safe single path component (<= 64 chars, no reserved Windows names
  CON/PRN/AUX/NUL/COMn/LPTn, no trailing dot/space, no control or `<>:"/\|?*`).
- Archive mode requires a clean, committed tree (`git status --porcelain` empty), names the zip
  `<mod>-<config>-<git short rev>.zip`, writes via temp + rename. Local candidate staging
  (`--staging-directory`) writes `release-info.json` with `acceptance_status=candidate`,
  `runtime_acceptance=pending`, `delivery_ready=false`.
- Base mod data is included only from an explicit `--base-data` directory and only names listed
  in the zone-source CSVs; the packager admits official-language resource sets.
- **Version is a single source of truth** (`version.json`: display name <= 64 chars, four-part
  file version 0..65535, prerelease flag) generated into `version.h` by a premake action; Git
  metadata is diagnostic provenance only (fallback `unknown`); the displayed release name is
  never derived from commit counts or tags.

### 9.3 Deployment is transactional (`tools/deploy_client_pair.py`)

Validate the *whole set* before touching the install: both configs' binaries/PDBs, manifests,
covers, matching `vr_input` packages (byte-identical across configs, JSON-parsed, binding
paths path-contained), matching compiled sound/script resources. Then per file: back up the old
file (SHA-256 verified) under a unique `output/deployments/<utc>-<id>/`, copy to
`<name>.staging-<token>`, verify its digest, `os.replace` everything, verify each destination
digest. On **any** exception roll back in reverse order from the backup (unlink if there was no
previous file), always delete staging files, always write `deployment.json`
(`deployed:false` on failure; before/after hashes). Never touch user profiles. Confirm the game
process exited first; to confirm the *running* build check process start time and module identity.
Python `unittest` for this (mock `os.replace` failing on the Nth call, missing/mismatched
inputs, path escape) is in `16-test-harness-recipes.md`.

### 9.4 Rules for the shipped product

- **Pin the supported game build and refuse others with a clear message** (exact size, SHA-256,
  PE sanity, sentinel; relocated images refused). State the supported game build, runtime,
  tested headset/controllers and tested missions in release notes; list what was HMD-validated
  and on which headset and runtime. README honesty: list only validated hardware combinations;
  publish no unverified minimum specs or performance promises; list known issues (overlay tools
  that hook Present, e.g. MSI Afterburner, can prevent start).
- Runtime selection documented (OpenXR default, OpenVR manual backup); no automatic API switch
  that hides a failure; selection happens at init (`vr_reinit` after connecting). SteamVR
  registration: relative manifest paths, `AddApplicationManifest` after `VR_Init`, identify the
  process before input init, separate app keys for normal/debug builds; registration failure must
  not disable rendering or input.
- **Launcher preflight is part of release readiness** (blocking errors vs acknowledge-once
  warnings; fixes recompute the report; no "don't show again"): incompatible native graphics
  options (supersampling must be 1x, shader preloading, shadow caches, fill-remaining-VRAM),
  missing/corrupt game files, app-local OpenXR loader, config readability/size cap. See `09`.
- Updater/telemetry: ship none by default; manual updates from the releases page. Disable inherited
  upstream updater/MOTD code paths rather than leaving a dormant network feature.
- Tag the validated source used for binaries; make the matching source and dependency pins
  available (GPL). Upload only reviewed artifacts.

### 9.5 Licensing and provenance

- **Overlord is GPLv3, inherited from H2-Mod/IW6x/S1x.** A fork or reuse of its code stays GPLv3
  and keeps notices (`LICENSE`, `THIRD_PARTY_NOTICES.md`, `docs/source-provenance.md`). A new Git
  root does not change ownership or licence terms. Code derived from another GPL mod inherits the
  obligation; a permissive reference project can be credited as research, not copied.
- Dependencies keep their own licences (collect them per release); fonts under OFL stay OFL and
  need their notices; a licence identifier alone does not recover missing attribution for a merged
  font; a git submodule with no licence file is an explicit review item, not clearance.
- **Never ship extracted game assets.** Provenance-review inherited fonts, voice audio and
  artwork before public redistribution (Overlord lists these as open items). Fixtures use
  skeleton names/bind transforms + SHA-256 only, no meshes or animation curves.
- Custom assets: compile only your own sources into the engine's package format in a throwaway
  workspace (copy only exe/DLLs, junction the data directories so the real install is never
  written), run the linker/verify/dump, **read the compiled result back** and compare against
  inputs (alias names, recordings, bone/surface counts) so "compile success cannot hide a
  missing variant"; pin the extraction tool revision; ship only the compiled artifact; keep
  sources and local engineering notes out of VCS. (H2: FLAC inputs need constant 1024-sample
  blocks; the `0x20` sound-alias flag bit is "random start offset" not `is3d`, so one-shots need
  `flags & 0x21 == 0` — asserted at input and at readback.)
- Scope/ethics: single-player/offline games the user owns; no DRM/anti-cheat bypass (see SKILL).

---

## 10. Acceptance tiers and capture protocol

### 10.1 Tiers (never promote by transfer)

`implemented` -> `offline tests passed` -> `hardware accepted` -> `pending`. For assets/profiles:
`captured -> candidate -> desktop_verified -> hmd_pending -> hmd_verified` (or `fallback` when
required assets/transactions are unavailable); nothing goes from `candidate` straight to
`hmd_verified`.

- Record the scope: level, devices, actions, configuration. **Never transfer acceptance across
  build configurations or from an earlier binary to the staged Release candidate.** An unchanged
  version label does not identify an unchanged executable. Executable, loader or gameplay-resource
  changes need fresh acceptance for the affected runtime scope; docs-only changes do not.
- Record desktop and HMD acceptance separately with their actual tested scope. User confirmation
  establishes visual acceptance in the tested scope without inventing counters.
- A known unresolved crash, failed level load or missing game output **blocks delivery** in the
  affected scope. Candidates may be packaged and deployed locally for retest, but stay
  `runtime_acceptance=pending`, `delivery_ready=false` until evidence is recorded.

### 10.2 Required evidence for a release candidate (`release-validation.md`)

1. Audit the exact staged configuration's EXE + matching PDB (feature entry points, all mutable
   storage, packaged resources) and run the aggregate-publication test built for that policy.
   Writable storage does not prove updates survived optimization.
2. Run the staged executable in a **fresh process**, load a campaign level through the supported
   startup path and reach a playable scene; for a regression repeat the level/path that failed.
   When touching FX definitions, asset retirement or saved state also exercise death/checkpoint
   reload and Resume from the main menu.
3. Validate the affected gameplay/rendering path and the runtime/controller combination claimed.
4. Record artifact identity and results: configuration, product version, PE/PDB identity, tested
   level/runtime/controller path, observed outcome.

### 10.3 HMD acceptance checklist (a human, in the headset)

Binocular alignment and parallax (no swimming/orbiting on turns); transparency, particles,
reflections, terrain edges, lens flares, dynamic lights; weapon/hand alignment, scopes/optics;
all controls and their re-arm behaviour (held-at-entry, pause, recenter, focus loss); pause,
menus, video/loading theater; recenter; HMD standby/resume (no 180-degree body flip); focus
loss/dashboard; window resize and device recovery; checkpoint/death reload; horizon stays level
(comfort); exit/shutdown. Check *both eyes*. Compare desktop vs HMD. For rendering changes also
check focus transitions and device lifecycle.

### 10.4 Coordinated capture window (headset-dependent reproduction)

Agree the window explicitly with the operator; outside it assume the headset is **off** (running
game/SteamVR processes are not evidence of a valid live reproduction).

1. Finish script/profile preparation and offline tests first (the operator need not wear the HMD).
2. Request **fresh** readiness confirmation (never reuse a previous window's); explain the actions
   first; a delay taking the item must not terminate the capture.
3. Immediately **before** the command, announce start in plain language; run exactly **one**
   window with no parallel analysis or other commands.
4. Immediately **after** it returns, announce the end (usable / aborted / empty), including
   failures, and that the operator may pause or remove the headset. Tool output does not replace
   either message.
5. Review offline. A second window needs its own readiness, start and end messages. **Never
   silently extend or retry**; invalid spans do not extend the agreed deadline. Default 30 s,
   explicit maximum 60 s (`vr_perfStart 30` auto-stops).
6. Ineligible samples (paused, unfocused controller, stale snapshot, no held item) are marked,
   not aborting; an entirely invalid window ends on time with no data.

**External read-only witness** (`tools/capture_equipment_visibility.py`): opens the process with
`PROCESS_QUERY_INFORMATION|PROCESS_VM_READ` only (never write/inject/suspend); requires an
operator-supplied address profile containing the **exact EXE SHA-256** (hash the running image;
mismatch aborts before any offset read); addresses sanity-bounded, reads <= 1 MiB; re-checks
readiness after the scene read and marks the sample ineligible if state changed; staleness gate
150 ms (non-finite/negative invalid); JSONL rows flushed, begin/end records, `validity_runs`
run-length encoded reasons; `--operator-ready` required; exit code 2 for no usable window.
It is best-effort, "never proof of a coherent GPU frame or HMD visibility". The in-process module
publishes fixed-layout shared structs each starting with sequence + reference generation + QPC
timestamp so an outside reader can judge staleness (the snapshot contract extended across
processes). Profiles contain build-specific addresses and live in an ignored incident directory.
Engine cvars can be found by hashing the name and scanning the cvar pool (one address covers many
variables).

---

## 11. Quick checklist for a new port

- [ ] `vr_status` writing atomically to a known path, with run identity (PE GUID, game hash, adapter LUID, effective engine settings).
- [ ] First-failure retention everywhere; try-lock producers; `dropped/snapshot_busy` counters.
- [ ] One-key support ZIP (limits, manifest, fresh-before-historical, never auto-upload).
- [ ] Watchdog thread with confirmed thresholds and thread-targeted dumps.
- [ ] Crash path: pre-created writer thread, bounded evidence stream with `.pdata`, `IsProcessSnapshotCallback => S_OK`, offline reader script, matching PDBs.
- [ ] Startup-frozen opt-in debug probes; a region/perf recorder with loss-aware analyzer.
- [ ] Mock OpenXR loader + strict SDK doubles + WARP tests before the first headset session (see 16).
- [ ] WPO off in Release + aggregate publication test + EXE/PDB artifact audit; ship RelWithDebInfo for play/perf.
- [ ] Packager/deployer that validate everything first, roll back on failure, record `delivery_ready=false` until accepted.
- [ ] Release notes: supported game build, runtime, tested hardware, what was HMD-validated; licences and provenance reviewed.
