# 01 — Architecture (Overlord's layering, generalised)

This file is the contract between the layers: who may call whom, who owns which state, which thread
does what, and how long a sample stays valid. Every number below was read from Overlord's code
unless marked *(doc)* (design documents) or *(unverified)*.

## 1. Layers and dependency direction

```
launcher (separate process: settings UI, preflight, launch; never loads game components)
        │  spawns  "overlord.exe -singleplayer"
        ▼
host loader ── LoadLibrary's the ASLR-stripped game image at its preferred base, resolves
        │      imports, runs the component lifecycle (post_start/post_load/post_unpack/pre_destroy)
        ▼
infrastructure components (no gameplay rules)
  ├─ d3d11            device/swapchain wrapping, Present pre/post, resize, device generations
  ├─ vr runtime       OpenXR / OpenVR backends behind one facade; frame loop inside Present
  ├─ stereo renderer  hooks engine render boundaries; per-eye views/records/resources
  ├─ head_pose_bridge tracking → engine coordinates, recenter, camera composition
  ├─ controller_input immutable per-frame input snapshots (poses, actions, generations)
  ├─ menus/HUD        native UI capture → compositor layers, pointer mapping
  ├─ scene injection  mod-built geometry submitted through the engine's own scene path
  ├─ script/UI VMs    GSC bridge + LUI/Lua adapters (see 17)
  └─ diagnostics      status snapshots, bounded traces, crash/stall evidence
        ▲
        │ consumes snapshots only (never calls the runtime)
gameplay layer (vr/gameplay)
  ├─ command adapter  usercmd injection (move, turn, buttons, view angles)
  ├─ interaction coordinator + hand arbiter (who owns each hand this frame)
  ├─ domain runtimes  weapons/reload mechanics, carry/holsters, equipment, world use
  ├─ presenters       hand IK, weapon pose, attachments, falling items, haptics
  ├─ campaign/scene adapters  scripted cameras, vehicles, turrets, ladders, climbing
  └─ native adapters  verified calls into game functions (fire, give/take, select…)
```

Rules (from `src/client/component/vr/gameplay/README.md` and `vr/README.md`):
- Gameplay → snapshots/adapters, never the reverse. The runtime must not include gameplay headers or call gameplay rules.
- Portable policy/math headers contain **no native addresses or engine types**. Native hooks, offsets and thread checks live in thin adapter files (`native_*.cpp`, `vr/h2/`).
- Pure decision code (`*_policy.hpp`, `*_feed.hpp`, `*_gesture.hpp`) works on *copied* inputs, so it runs in CPU unit tests. Scene adapters: pure `classify(evidence)` plus a thin `observe()` that is the only code touching the script VM.
- **A readable pointer or a cached pose is not proof of current ownership** (`vr/README.md`). Identify by identity + generation, not by address.
- One runtime frame is published for all consumers; no gameplay code polls the XR runtime for a fresher pose.

## 2. Host, lifecycle and what "fail closed" means

- Components register statically (`REGISTER_COMPONENT`). Hooks are installed in `post_unpack`, which Overlord triggers on the game's first `GetProcAddress("InitializeCriticalSectionEx")`, i.e. after the game unpacked itself and before its main loop. Saved settings are applied with `scheduler::once(..., pipeline::main)`.
- **A throw out of `post_unpack` is fatal** (message box + exit). Feature-level fail-closed therefore has to be implemented inside each component: verify bytes, catch, record a `reason` string, leave native behaviour (physical climb, vehicles, flare, airport adapters all do this). Reserve a throw for contracts without which the whole mod is unsafe (e.g. the process-shutdown entry points).
- `INJECT_HOST_AS_LIB` is defined: the host exe (base `0x160000000`) maps the cached game image at its preferred base with `LoadLibrary`; module-handle queries are spoofed so the game sees the host as itself. The `.payload` loader path is compiled out (see `12-overlord-file-map.md`).
- Loader details worth copying: the cache is compared to the ASLR-patched game bytes on every start and replaced atomically; a lease keeps the verified file unchanged until mapped; image base is verified *before* imports/TLS are patched; IFEO `MaxLoaderThreads=1` limits parallel DLL loading; TLS donor DLL patches the TLS index and the current TEB block.
- VR init is attempted **once per D3D device generation**. Retries need an explicit `vr_reinit` or a device change. There is no automatic backend switching (it would hide failures). OpenVR metadata queries run strictly before loading OpenXR; recenter only when the session is FOCUSED and views/head are TRACKED.

## 3. Execution contexts and threads

Overlord does not key assertions on OS thread ids. It binds **named pipelines to hooked native frame functions**; the scope is a *dispatcher scope*, not a permanent thread assignment (the engine runs startup and in-level server frames on different OS threads).

| Pipeline | Bound to (H2 example) | Work |
|---|---|---|
| `main` | after the main frame function | asset ownership/preparation (rigid parts, skeletal props, runtime model registration; slow prep every 250 ms), settings application, `on_game_initialized` polling |
| `server` | after `G_RunFrame` | **all script-VM access and native gameplay writes**: weapon fire adapters, ammo compare-and-commit, interaction arbitration, scene adapters, Lua `run_frame` |
| `renderer` | inside the `R_EndFrame` hook | VR frame maintenance (`poll_capture`, `prepare_frame`); D3D immediate-context queries; native 2D overlay draws |
| `lui` | before the Hks frame | LUI event injection and notifications from GSC |
| `async` | private 10 ms thread | file writes (status reports), never touches the game |

`scheduler::is_executing(pipeline)` is used as a precondition (`scene_models::register_runtime_models`, `rigid_part::create*`, `skeletal_model::create` return false off `main`; ammo commit APIs reject callers outside the server callback scope even on the same OS thread).

Other threads:
| Thread | Work |
|---|---|
| Present owner (game render/present thread) | XR `xrWaitFrame/BeginFrame/LocateViews/SyncActions` after Present-post; eye copy + `xrEndFrame` at Present-pre; `xrEndSession` only in STOPPING |
| Game client/command path | command building (input injection, view angles), camera setup (`refdef`) |
| Render backend thread | per-eye re-execution of world records, resource isolation; owner pass |
| Render worker threads | native skeleton/scene preparation jobs (nested; `preparing` is `thread_local`, saved/restored) |
| Watchdog thread | stall detection, status/evidence snapshots (reads published copies; suspends only the target thread briefly to copy registers) |
| Async writer | status/dump files; checks a revision before writing |

Concrete thread rules learned from failures:
- **D3D11 immediate-context queries stay on the renderer thread that produced the copy.** Polling copy-completion queries from Present raced `CopyResource/End` on the game's single-threaded device and surfaced as an NVIDIA driver fault inside Present. Polling therefore runs at the `R_EndFrame` hook. Also: that hook runs *after* the frame's scene records are built, so it cannot define the pose for them; a per-eye pose is sampled and frozen at the per-view owner.
- **No new locks across native/GPU work, with one exception:** the owner pass joins H2's own recursive GPU-context mutex (handle read from a global, 100 ms wait) instead of adding a lock of its own. Mod locks are short mutexes around copying only; never across native or GPU calls; status getters must not take the lock held across XR pacing; try-only locks report `snapshot_busy` rather than waiting.
- **Script VM calls only on the server pipeline.** The bridge has no locking; stack isolation protects stack state, not concurrency. Command construction and render threads read published `{epoch, timestamp}` snapshots.
- Render workers never wait on asset discovery: preparation runs on `main`, publishes complete immutable sets atomically, retains them until the native asset drain.
- The engine's thread contexts (H2: `MAIN, BACKEND, WORKER0-7, SERVER, CINEMATIC, WINDOW, INPUT, DATABASE, STREAM`) are named through its thread-id table so profilers can correlate VR stalls (`thread_names.cpp`).

Cross-thread communication: copied snapshots + monotonically increasing generations/epochs; "published / submitted / now" triplets (render thread compares the frozen submission with the live publication and **omits on mismatch**, no locks around gameplay).

## 4. Authority model

| State | Owner |
|---|---|
| Raw poses and actions | XR backend (sampled once, normalised, published) |
| Semantic edges (pressed/released) | Input frame service (derived exactly once per sequence) |
| Body frame, hand occupancy | VR gameplay layer (arbiter) |
| Player capsule, stance, inventory, ammo, damage, scripts, saves | **Game engine**, changed only through verified adapters/native commands |
| Mission/story state | the game's scripts; plugin state is stored *inside* script objects so saves carry it |
| Pending interaction intent | Domain system (speculative, cancellable) |
| Weapon mechanical partition (chamber vs magazine vs held) | Weapon mechanics, committed through an accepted engine transaction (escrow + compare-and-swap, see 5) |
| Hands, parts, sounds, haptics | Presenters, derived from committed state; never create gameplay state |

### Rule-4 nuance: "VR keeps no second writable copy of native state"
Overlord does keep **mechanical state the engine has no concept of** (chambered round vs magazine, rounds held in a hand or falling, tube shells, which physical gun an ammo cell belongs to). The correct statement is:
- The engine's cells are the only truth for *totals*; VR proposes, the engine confirms, then VR commits ("propose -> engine confirms -> commit", never "VR decides, engine follows").
- VR-held rounds are **escrow**: moved out of reserve at draw, refunded exactly once on release/cancel/handover, returned to the pool that shares the source's native ammo identity (`reserve_identity`), never guessed by caliber.
- Every native write is a **compare-and-swap**: `compare_commit_owned(ps, id, before, after)` re-reads the native cells, requires them to equal `before`, then writes `after`. If the compare fails the mechanical state *and escrow are retained*, nothing partial is written, no success feedback is emitted, and the input edge is consumed (a rejected action never becomes a queued retry or a different action). Two-cell operations use one atomic pair write.
- Commit APIs accept callers only inside the server dispatcher scope; prediction/render/console workers cannot write.
- **Conservation invariant** enforced by pure planners: `total_rounds(before) == total_rounds(after) + rounds_spent`, with `usable = reserve + live weapon contents + unconsumed held payloads + independent falling/held/unsettled payloads`; refunds/transfers preserve the total, spent cases never count.
- A shot advances mechanical state only after the engine actually consumed ammo (observe before/after around the native consume call; fault if the projection differs). "Never refund after entering native damage/scripts".
- **Reconciliation on external mutation:** native loaded went up -> move it into reserve (do not fake a physical load); went down -> accept without refund and mark the chamber unconfirmed; unexplained mismatch -> `fault`, release escrow safely. Death, checkpoint restore, script give/take, forced switch, cutscene and map transition invalidate expected revisions and rebuild from a fresh native read; on a level/checkpoint timeline change obsolete escrow is discarded, **not** refunded into restored counts.
- One engine ammo projection per weapon definition, but VR can hold two copies of one gun: a server-owned `clip_ledger` of physical instances selects which instance is projected into the native cell; a non-projected same-model instance never inherits another gun's debit.
- Save/load: `settle()` every escrow first, read shared pools once, persist only stable logical fields (not pointers/timestamps/leases/generations), validate read-only against current assets before any take/give, stage restore atomically; hand/trigger leases require fresh input after restore.

### Presentation gates are looser than authority gates
Never reuse an authority/firing predicate for pure presentation. Examples: `pose_ready` (holding hand, grip identity, tracking/camera freshness) vs `ready = pose_ready && owner.can_fire()` (a foregrip-only carry must still show the held magazine); render-owner comparison uses only the fixed rear carrier identity while firing/interaction authorization uses the complete owner (avoids a magazine blink across 92 assemblies). Test both orderings (acquire/release x skin-before/after).

## 5. Identity, revisions and idempotence

**Identity = definition + instance generation (+ revision), never a name, a token alone or an address.**

| Identity | Parts | Why |
|---|---|---|
| Weapon instance | engine token (definition) + generation (physical gun across grip/holster/world transfers; 0 = legacy native view) | two copies of one definition must not fight for the same mailbox |
| Hold | `revision` (any topology change), `rear_revision` (changes only when the control hand changes: "support changes must not rearm the firing hand"), pose_rear | feedback, HUD lease, haptics, recoil entries and render bind are keyed on `id + rear_revision` |
| Hand-interaction target | provider-local `object_identity{value, generation}`; world entities may have generation 0, head gestures value 0 + tracking reference | validity is provider-local |
| Skeleton/render record | `(object, matrices pointer, epoch)` + byte-identical bone witness; stereo pair id + publication + eye + natural camera | "a record address alone is not a frame ID"; addresses are reused |
| Entity | entity number + generation ("entity key"); script identity compares linked parent *object ids*, not shared animation names | recycled entity numbers |
| Asset | pointer + **asset epoch** bumped in `fastfiles::on_pre_unload` | addresses are reused after unload |
| Lease | `(revision + next) * 2 + side` style numbers so two tools with one native token have distinct identities | stale sessions can't match a later holding |
| Queued render data | keyed by pair/publication; eviction -> not-found, never "latest" | no latest-pose fallback |

**Idempotence rule:** every consumer must be idempotent for the 2nd eye and for prediction replay. Concrete forms: `if (input.sequence == sequence_) return;` before any operation; solve a skeleton once per `(object, matrices, timestamp)` and **latch a rejection as well as a success**; replay-safe press edges cached per `serverTime` (older times use `cmd & ~oldcmd`, same time returns the cached edge); `replaying_trigger = last_trigger_time == cmd.serverTime` avoids consuming charges twice; command history stores the physical heading so replays do not double-rotate; a duplicate frame is a no-op; the dynamic-upload advance runs exactly once per pair after both eyes; one clock sample per native packing job shared by both eyes ("individual parts must not sample their own clock"). Simulation never runs from a render callback or prediction replay.

**Claims and handles:** record addresses are recycled, so a renderer claim is acquired by camera identity before **and** after; there is no implicit "next eye". Retain every descriptor until queued rendering has finished (no slot recycling); retirement only at the native drain barrier, never on weapon switch or checkpoint. Fixed-size registries need an (N+1)th-registration test (a ninth zone-unload observer once broke boot; the registry is now growable, startup-only).

## 6. Generations and what bumps them

| Counter | Bumped by | Consumers reject |
|---|---|---|
| `reference_generation` | recenter, origin change, map load, world-scale change | pending spatial interactions, hover, motion history |
| `continuity_generation` | producer continuity break (slow-motion gap, tracking resume) | held-input gates, throw/melee history |
| `pose_reference_generation` | binding/model change (not physical motion) | velocity computation |
| input `generation`, `presses`, `releases` | focus/action loss, reconnect, digital latch reset | edge gates (armed-after-neutral) |
| device generation | D3D device recreation | every VR resource |
| swapchain epoch (`SetPrivateData`) | swapchain promotion | stale swapchain Present |
| `pair_id`, `publication`, `output_eye` | per stereo pair | stereo identity lookups |
| `recenter_count` | each recenter | shot admission (`recenter_count == reference_generation`) |
| asset epoch | zone unload | rig/binding caches, rigid-part sets, fonts, FX |
| scene `epoch` / `position_epoch` | scene change, player-state pointer change, command-time rollback / camera-translation ownership change | camera anchors |
| instance generation | physical gun lifetime | leases, ledgers |
| level timeline | load/restart | escrow, scene adapters |

## 7. Freshness (staleness) values per subsystem

There is no single global timeout. Every consumer rechecks `now >= sampled_at && now - sampled_at <= limit`, plus `input.focused`, `!orientation_settling`, `spatial.generation == input.reference_generation`, and `cl_paused == 0 && keyCatchers == 0`.

| Subsystem | Value | Source |
|---|---|---|
| Input/pose consumers (fire, muzzle, grips, carry, tube, underbarrel, world use, scripted arms, locomotion, quick reload, render caches, attachment solves, submission-pool reuse guard) | **150 ms** | many |
| Weapon feedback events and haptic mailbox | **100 ms** | `weapon_feedback.hpp`, `controller_haptics.hpp` |
| Controller calibration settle | 200 ms | `controller_calibration.hpp` |
| Smooth-turn / locomotion step cap | dt capped at **50 ms** (`clamp(dt, 0, 0.05)`) | `controller_locomotion.hpp` |
| Damage overlay and narrative captures | **250 ms** (`damage_screen::current`, `narrative_ui`); an earlier note said 350, the source says 250 | |
| Snowmobile boarding dwell | 250 ms of uninterrupted evidence | |
| Magazine insertion guard after a latch knock; belt cover push cooldown | 300 ms | `physical_reload_gesture.hpp`, `belt_cover_push.hpp` |
| Script polling visibility for a tap (50 ms script poll) | held >= 100 ms | oilrig/sentry adapters |
| Arm takeover envelope | 200 ms; reset when command time jumps > 250 ms | `scripted_arms_policy.hpp` |
| HUD smoothing reset | dt > 0.25 s or distance jump | `weapon_hud_smoothing.hpp` |
| Controller profile retry | 4 tries x 250 ms | `controller_profile_retry.hpp` |
| Haptic pulse | one pending per hand, <= 0.1 s, expiry 100 ms | |
| Recoil decay | `exp(-t/0.22 s)`, zero after 2 s | `weapon_recoil.hpp` |
Under native slow motion server consumers tick less often while tracking continues; input frames therefore carry **producer continuity** so "held but stale" is distinguishable from "held through a slow tick"; input freshness and geometry freshness have independent limits. Do not widen the windows without bounded render-rate motion history plus server-side settlement.

Held-input rules shared by all gates: a button held at entry/resume needs a neutral sample plus a fresh press (`armed` only after seeing released); a button must be seen released after focus/context restore; tracking loss is **not** a release (deferred release; "a proven physical release ends its session even when pose is lost"); per-hand pose admission precedes edge consumption (a missing opposite hand must neither swallow nor replay this hand's release).

## 8. Transaction vocabulary

The design docs *(doc: `docs/vr-gameplay-interaction-architecture.md`)* describe a transaction envelope; the code realises it per domain rather than as one struct:
```cpp
// design vocabulary (not a single type in the code)
struct GameplayTransaction {
  TransactionId id; Domain domain; Intent payload;
  uint64_t inputFrameId, contextGeneration, referenceSpaceGeneration;
  uint64_t expectedNativeRevision, expectedDomainRevision;
};
enum class TransactionResult { Accepted, Rejected, Duplicate, ContextChanged, ExternalMutation, Unsupported };
// code shape: pure planner  plan(rules, state, request) -> { accepted, next, before, after, feedback, rounds_spent }
//             rejects if request.revision != state.revision or instance generation differs
//             then   native compare-and-commit(before, after)  ->  publish next state  ->  feedback
```
- Execute only on the verified engine thread/boundary; replaying the same id/sequence never applies twice.
- After the result, publish a new native observation; presentation converges on it.
- **Arbiter witness invariant:** providers `offer` candidates, the arbiter `resolve`s once per batch (deterministic sort; candidate-buffer overflow rejects *all*, truncation never picks a winner), providers act only on `granted(...)`, then report `observed(...)` the relation that actually exists; any accepted decision not witnessed becomes `commit_failed`; sessions not witnessed this frame are dropped. A grant is real only if the provider reports it afterwards. Bounded: 8 sessions, 64 candidates; overflow rejects new requests, never evicts existing sessions.
- One release edge = one request: stow into an empty slot, atomic exchange with an occupied slot, reject-and-retain when blocked (never falls through to a world drop), native drop only after native inventory removal and a real world entity are confirmed; tracking/focus loss and pause are never deliberate drops.

## 9. Lifecycle and failure

- Tracking loss neutralises buttons and freezes interactions (retaining state); recenter bumps the reference generation and rebuilds body/slot frames and never computes velocities across the discontinuity; external native mutation invalidates pending transactions; load/rollback (command time going backwards, entity key change) resets and re-restores from VM state.
- Fail-safe defaults: an exception while reading script state produces `phase::release` (movement/turn/world-use denied, weapons suspended, arms authored); "loss of a script observation cannot grant weapons or a QTE". Unknown/ambiguous state fails closed: wrong prologue -> feature off with a console error, duplicate script relations -> all rejected, a faulted mechanical instance releases escrow instead of guessing.
- **Shutdown ordering:** `Com_Quit_f` wrapper and `Sys_Quit` are hooked (bytes verified, hard fail otherwise); both call `R_SyncRenderThread()` first (drain the renderer), then `component_loader::pre_destroy()`, because native `Com_Shutdown` later unloads NVAPI whose private interfaces may still be attached to a device the mod retains. Neither hook runs on cancellation or renderer restart. `pre_destroy` is idempotent and thread-safe (state machine with a teardown thread id and a condition variable; re-entrant calls on the teardown thread return; others wait). `ExitProcess` is redirected to the same path as a fallback. Release XR/COM owners **before** the engine pulls driver libraries.
- Teardown retains the object graph so it can be retried; the first failing XrResult is preserved over cleanup results.
- Asset-lifetime hooks: `fastfiles::on_pre_unload` drops every cache/lease that holds zone pointers (models, fonts, FX variants, identity registries, partition caches); `hks_shutdown_stub` resets VM-held objects.

## 10. Settings and configuration

- Engine-native config variables (dvars/cvars) registered by components with a `SAVED` flag so they persist in the game's own config file; one constexpr table (`vr/settings.hpp`) lists every setting (names, defaults, ranges).
- If the engine stores config by hashed name (H2 dvars: registration takes a hash and an empty name), put the **hash function in a constexpr header** (`game/dvar_hash.hpp`) the launcher includes, so the launcher can read/write the saved config offline (it accepts either the plain name or `0x<hash>`; on save it removes both spellings and writes one).
- The launcher edits the same saved values. **Initialization-only settings are frozen before the game binary loads** (pose pipeline, backend, debug probes via `call_once`); console edits never touch a live hook graph.
- Risk-settings preflight (launcher): SSAA != 1x is a **blocking error**; shader preloading, shadow caches and "fill remaining video memory" are warnings that must be acknowledged for that launch; a one-click atomic "Fix" writes `r_ssaaSamples=1`, `r_preloadShadersFrontendAllow=0`, `r_preloadShaders=0`, `sm_cacheSunShadow=Disabled`, `sm_cacheSpotShadows=Disabled`, `r_fill_texture_memory=0`. The scene-resolution forcing path refuses `supersample_count != 1` and mid-session changes.
- Console: `vr_status`, `vr_reinit`, `vr_recenter`, `vr_diagnose`, and per-feature `*_status` commands (counters + last decision strings written to `minidumps/overlord-*.txt`).

## 11. Build and process policy that affects architecture

- **Build mode is not a feature switch.** A Debug-only install gate made optimized clients fall back to native first-person rendering (skinned-visibility hook). Every production hook must install in every configuration; a feature-parity test reads the emitted function symbols from the PDB (`client_feature_parity_tests.py`).
- **MSVC v142 WPO/LTCG miscompiled publication buffers** (nonzero-initialized mutable aggregates placed in `.rdata` or their writes dropped; a writable-section workaround did not fix lost writes). Policy: `/GL-` and `/LTCG:OFF` for every project, `/O1` retained, normal C++ storage, `aggregate_publication_tests` + PE/PDB audit gating packaging. A passing Debug build proves nothing about Release.
- The launcher process never starts game components; the game process never loads WebView2.
- A VR fork of an upstream mod must neutralise the upstream network services (updater `server_urls` empty, MOTD disabled, a no-auto-update marker) because the feed could replace VR binaries.
