# 03 — Reverse-engineering playbook: what to find, how, and what not to retry

Phase 2. The goal is a short list of **proven native boundaries** (camera, view
slots, backend per-view owner, command builder, skeleton, fire, UI, script VM),
each with an address, a byte contract, a meaning and runtime evidence. Overlord's
method is *observe first, gate on evidence, only then modify, and retire failed
experiments explicitly*. Most of this file is how to do that cheaply.

Contents: 1 principles · 2 toolbox · 3 getting analysable code (encrypted
`.text`) · 4 offline census (Capstone) · 5 runtime observe-only census ·
6 target list · 7 engine-property audits · 8 failed experiments and pitfalls ·
9 debugging accelerators · 10 H2 reference addresses.

---

## 1. Principles

1. **Probe -> gate -> modifier -> retire.** Install read-only hooks that record
   counts, threads, argument identities and before/after hashes into bounded
   rings. A strict acceptance gate decides when evidence is complete. Only then
   turn the hook into a modifier. When an experiment fails, leave an explicit
   `state=retired` record with the reason so stale artifacts cannot be mistaken
   for a runtime path (`diagnostics/renderer_evidence.cpp`).
2. **Evidence labels.** Write every finding as **address/RVA, signature, meaning,
   evidence, build**, and tag it *confirmed* (observed at runtime), *decision*
   or *pending*. Source reading, offline tests, successful builds and headset
   acceptance are four different things (see 11). Trust call-chain evidence over
   reconstructed SDK struct labels and over inline offset comments from other
   builds.
3. **Never accept "the math passes" as proof.** The matrix-culling tightening
   (translate the frustum apex rearward so one allowance covers both eyes) was
   mathematically tight, passed its tests, gave no improvement for the operator
   and was withdrawn. Do not keep a candidate fix because its unit tests pass.
4. **Sample once, in a state you control.** Capture pristine code *before* your
   own patches (3.2), capture engine state at the boundary you will later modify
   (before/after a call), and keep the capture immutable.
5. **Every observer declares its side-effect budget.** Evidence files in Overlord
   print `additional_gpu_commands=0 target_replacements=0 command_replays=0
   registry_writes=0`. Observers call the original exactly once, with no lock held,
   never issue GPU work, never dereference engine pointers they cannot validate.
6. **Hot paths only touch fixed counters.** Formatting, SHA-256, JSON and file I/O
   belong to a control thread; producers drop and count rather than block.
7. **Agree capture windows with the headset operator** (explicit start/end
   announcements, <= 60 s, ineligible samples marked not aborted, no silent
   retry). External reads are best-effort and never proof of a coherent GPU frame
   or HMD visibility.

---

## 2. Toolbox

| Purpose | Tools / Overlord practice |
|---|---|
| Static analysis | IDA / Ghidra / Binary Ninja; community symbol maps for the lineage (CoD: H2-Mod/IW6x/S1x `game/symbols.hpp`); leaked PDBs, console builds, open-source sibling engines (Quake 3 -> id Tech -> IW). **Treat inherited tables as unverified** (02 section 8.2). |
| Dynamic | x64dbg; Cheat Engine / ReClass.NET for live structs; Process Monitor. |
| GPU | RenderDoc / PIX / Nsight to find the projection/view constant buffers, pass order, reverse-Z, depth-hack draws. A capture must show a projection you can reproduce bit-for-bit for the mono case. Overlord's own in-process equivalent is the D3D-boundary census (section 5). |
| Passive observation | Frida or read-only memory sampling to capture native contracts before patching: bounded sessions that change no arguments or returns; session tokens and entity numbers are evidence, not keys; historical status files may belong to another level (compare instance and PDB). |
| Offline disassembly | Python + **Capstone** (`tools/generate_scene_surface_contract.py`) over an owned `.text` + `.pdata` dump. |
| In-process dev tooling | ImGui overlay (entity/asset/material/shader browsers and live editors, world-space debug draw, script console), loaded lazily; asset viewers re-resolve names each frame and never cache asset pointers (zones unload); techset editor can swap/restore a technique slot at runtime to see which pass a draw uses. Shader disassembly reads bytecode from `SetPrivateData` (DEBUG only). |
| Asset capture | `tools/asset_dump.py` (bounded typed-record reader for ZoneTool dumps), CASC readers (`tools/casc_images.py`, `casc_slice.cpp`) to measure grip/magazine geometry. **Keep extracted assets out of the repo.** Pin the extraction tool revision and refuse manifests that are not `status: complete`. |
| Cheap probes | Throwaway CPU-only samplers (`native_flare_probe`: Release compiles the wrappers as inline pass-throughs, timed run with slots per channel so one eye is not aliased away, validates layout assumptions such as allocator stride 48 and vertex count +4 before trusting them, dumps raw vertices/matrices as floats with non-finite flagged). |
| Evidence formats | Binary recorder with magic/version/footer and an offline analyzer (`vr/region_capture.cpp`, `tools/analyze_region_capture.py`); bounded crash evidence inside minidumps (`tools/read_crash_evidence.py`); hash-bound external witness (`tools/capture_equipment_visibility.py`). See 11. |

---

## 3. Getting code you can analyse

### 3.1 Identify the packer/DRM situation first (00 section 1)
Entropy, section names, stub signatures, self-checksumming. Observe protection
(Overlord's `arxan.cpp` snapshots 14 ntdll `Dbg*` prologues once a second and logs),
do not defeat it.

### 3.2 Encrypted `.text`: dump the decrypted code from the running process
H2's on-disk `.text` is encrypted, so offline disassembly of the exe is useless
and every RVA/mask must be established against **decrypted runtime memory**
(`diagnostics/runtime_code_snapshot.cpp`):

- After the loader validated the target and **before** your own hooks patch
  anything, copy selected live executable ranges to
  `minidumps/runtime-code/<name>-h2+<startRVA>-h2+<endRVA>.bin` plus a
  `manifest.txt`.
- Each range is validated by walking `VirtualQuery` regions through
  `validate_executable_target` (EXECUTE_READ/READWRITE/WRITECOPY, no GUARD/NOACCESS,
  overflow-checked, <= 1 MiB per range); failure marks the capture
  `state=incomplete`. The manifest is written first as `in_progress` (capture id
  = pid-tick, expected region count), then rewritten `complete|incomplete` with
  per-region begin/end/size/file/SHA-256.
- The manifest records scope: `post_existing_h2mod_hooks_pre_vr_observation_calls`:
  the dump contains the *base mod's* hooks but none of yours. It decodes known prior
  patch sites (`E8/E9 rel32 -> target`) and skips an entry already detoured by
  another component.
- Region bounds come from runtime disassembly and the PE exception metadata; some
  are explicitly labelled *clusters*, not function bounds. About 60 regions cover
  the scene owner/adapters, view-slot allocator/initializer/finalizer/projection
  builder, `render_scene`, the extended-scene selective copy, scene-worker consumer,
  menu find/dispatch, worker enqueue/dispatch/wait, backend command dispatch and
  view copy, depth-hack projection, target select/prepare, OM set render targets,
  record classifier, GPU query pool/publish.
- **Dump the code around any recurring crash site once** (a crash AV address inside
  a worker motivated a region) so the failure can be resolved without another game
  run.
- A crash handler for a hand-mapped exe must add module/function-table/unwind
  ranges to the minidump (02 section 2.9, 11).

### 3.3 Hash-named cvars and in-memory discovery
Engines that store cvar names as hashes (H2: FNV-like, seed `0x319712C3`,
multiplier `0xB3CB2E29`, XOR then multiply per lower-cased byte) need a
name/description table dumped once (`dvar_list.json`, ~3,600 entries) because many
VR features are just cvar toggles and the binary ships no names. An external tool
can find a variable by hash by scanning the engine's cvar pool for the hash
(96-byte records, hash at +0, type byte at +8, value at +16, boolean type 5)
instead of hard-coding a value address: one pool address covers many variables.

---

## 4. Offline census with Capstone (how to patch an ABI safely)

Use when a VR need forces you to change engine *data layout* or limits (arenas,
packed IDs, pool sizes). Method from `tools/generate_scene_surface_contract.py`:

1. **Input:** a clean, owned dump of `.text` and `.pdata` (not the running image,
   which may already contain hooks). Decode every function in `.pdata` with
   Capstone (`skipdata`) so only real code is considered.
2. **Census every instruction that references the structure** by immediate *or*
   non-RIP memory displacement (`0x62b700 <= v < 0x66b700`), preserving each field
   addend. A comment records that a literal-base-only search missed 24 sites and
   "caused two crashes". Exclude coincidentally equal RVAs/strings outside the
   audited code span, and assert none of the member range appears outside it.
3. **Assert exact census counts** (`field_counts`, 89 base sites, 59 mask sites for
   H2) so a different native build or an incomplete audit fails generation.
4. **Emit a compile-time header** of `{address, before[15], after[15], size, field,
   dynamic}` with reasons, plus guard entries with `old == new` for "bank identity"
   instructions. The runtime verifies every site and applies all-or-none (02
   section 6.2).
5. **Review the generated table**; verification proves the listed sites matched,
   not that the list is complete, so the coverage census is itself a test.

Also generic: an `E8 rel32` operand check, a prologue check, and a unique-match
rule are the primitives; the census is how you find *which* sites exist.

---

## 5. Runtime observe-only census

### 5.1 Per-hook observation
- Record stages (`enter`, `before_original`, `after_original`, `after_mod`,
  `leave`) with caller `_ReturnAddress()`; diagnostics then show other writers of
  the same state (camera values and callers at each stage).
- **Diff struct memory** around the original call to find what it writes
  (`backend_target_prepare_probe_stub` captures record bytes before/after and
  reports changed ranges). Overlord's one-shot immutable captures
  (`one_shot_artifact<Size>`: `atomic_uint32` state 0 empty / 1 copying /
  2 immutable; written once with release, read with acquire; only the
  diagnostics thread serialises) include scene descriptor before/after, view slot
  before/after initializer, left/right eye slots, backend view-source before/after,
  record before/after target-prepare, target registry. The manifest lists SHA-256,
  metadata words and `delta=<name> changed_bytes first last` between pairs: a
  binary diff proving exactly which bytes your per-eye derivation changes.
- **Read-only witnesses:** compare an expected value (a bone matrix you solved)
  before and after native work to prove nothing else overwrote it. Hash the
  backend command stream before/after observation (`command_stream_immutable=yes`
  iff address, bytes, count and hash all equal).
- **Content signature** (FNV-1a 64) versions manifests by exact bytes so "written"
  means "these exact bytes", and readiness must come from the **current process**
  (files from another process cannot satisfy it).

### 5.2 Strict acceptance
`accepts_complete_execution_evidence` has six booleans (observation complete,
backend target frame complete, output merger complete, draw-indexed complete,
execution census complete, terminal device identity consistent). Each requires
attempts == 1, completions == 1, failures == 0, non-zero positive counters, zero for
every error counter (context/thread mismatch, overflow, query failure, hook failure,
foreign context, admission collision), report-vs-status field consistency, a single
device generation and D3D context across all probes, and frame-boundary arithmetic
(`end_present_pre == start_present_post + 1`, `SUCCEEDED(present_result)`, same
thread). Manifest states: `waiting_for_level -> execution_pending -> incomplete |
opaque | failed | complete`. **Opaque** means deferred contexts or
`ExecuteCommandList` were seen, so the census cannot account for all work: not
acceptable. Evidence states are enums (`not_observed/observed/incomplete/truncated/
invalid_shape/unreadable`), never booleans; "zero hits" must not be confusable with
"did not run".

### 5.3 Cross-eye lockstep census (build a small version early if you choose strategy A)
Record the ordered sequence of GPU calls for eye 0 and eye 1 (api, caller, args,
VS/PS/CS/GS/HS/DS, RTV/DSV, depth/blend/raster/layout, topology, index buffer,
viewport/scissor hashes, hashes of CB/SRV/UAV/output bindings), compare in
ordinal lockstep, and report mismatch categories with the first mismatch ordinal.
A well-behaved stereo pass has the same call stream in both eyes; divergence points
at engine paths that depend on stale per-eye state. Add:

- **Shared-resource detection:** per eye, the set of resources written;
  `right_first_read_before_write_candidate` flags something eye 1 reads before
  writing but eye 0 wrote (needs per-eye isolation).
- **Per-eye constant-buffer provenance:** hook Map/Unmap/UpdateSubresource/Copy/
  Set*ConstantBuffers; 128-bit content hash, upload generation and caller; classify
  slot use via shader reflection (used/unused/unknown) so "left bound / right null
  but unused" is not a false alarm; byte/word diff with float interpretation and
  `D3DReflect` variable attribution.
- **Dynamic-arena snapshots:** dynamic geometry arenas are consumed sequentially, so
  running the scene twice advances the cursor; detect constant `start_index_delta`
  between eyes and prove the cursor is restored between eyes.
- **Shader-consumer discovery** with bytecode reflection plus a write-provenance log
  at the D3D boundary (which pixel shader reads the scene mip chain at which SRV
  slot) when you cannot read engine source.
- **Owner-pass proof:** content hashes, non-zero byte counts and `eyes_distinct`
  from a same-pair readback, with HRESULTs surfaced.
- Meta-lesson: Overlord fixed artifacts cause by cause (SSR history, CB content,
  dynamic arena cursors, effect index rebasing) by diffing the two eyes at the D3D
  boundary. See 05 for the fixes.
- **A same-pair GPU readback may be the only thing that exposes a bug**: draw counts,
  shaders and resource identity looked identical in the transparent-shadow case.

### 5.4 Cost of probes
Left-on debug probes cost real CPU (16-frame stack capture per transaction, hashing
`0x501C8` bytes x 256 transactions, `VirtualQuery` per call, a watchdog every 2 s).
Freeze probe selection at startup (`debug_options` in `post_unpack`); never "disable
diagnostics" that production gating depends on (`vr_engineProbe` controls stereo
publication).

---

## 6. Target list (in the order you need them)

For each: how to find it, what to record, **exit evidence**. H2 examples are in
section 10 (never reuse addresses).

### 6.1 Graphics device and swapchain
Import `D3D11CreateDevice(AndSwapChain)` / `CreateDXGIFactory*`; substitute through
the import resolver (02 section 11). Present/ResizeBuffers via dummy swapchain on the
game's own device (vtable slots 8 and 13); also `Present1` if used. Shader creation
wrappers can be detoured to tag bytecode. *Exit:* a promoted device generation and an
active swapchain epoch logged on the first real Present.

### 6.2 Final camera / view definition
Engine names: `refdef`, `viewParms`, `CViewSetup`, `FSceneView`, `cg.refdef`. Search
for the FOV cvar (`cg_fov`), follow readers to the struct holding `fov_x, fov_y,
origin[3], axis[3][3]`; or find the view-projection constant buffer in a capture and
search memory for the camera origin floats. Then find **the last function before
rendering that writes it**: that is the camera composition boundary (06).

- Arbitration of scripted/sniper/mounted/remote/ladder/optic cameras happens at this
  single point through a request/policy object, not scattered patches.
- Find the engine's **separate view-origin for LOD/cull/streaming** and update it
  with the composed camera (H2 global, overwritten when `cg_paused==2` or the head
  pose was applied).
- Find the **command angle format**: H2 packs 360 degrees into 2^24
  (`packed * 360/16777216 + ps->delta_angles`), not Quake's 16 bits; writing a head
  yaw into a command must account for `delta_angles`. For a remote camera use the
  latest usercmd's `serverTime`, the time the engine will use.
- Free-camera mode (`cg_paused == 2`) is a stereo test tool: check eye separation and
  culling without player constraints.
- *Exit:* observe-only stages show who writes the camera and in what order; composing
  a known offset moves the image with no other writer undoing it.

### 6.3 Scene render entry and view slots (frontend)
The function that turns a camera into a renderable view (`R_RenderScene`), its
view-slot allocator/initializer/finalizer, and the draw-surface generator that culls
and builds lists from the slot. **Determine the layout of the view slot** by diffing
slots across frames with known camera motion: origin, axis, view-projection (+inverse),
tan-half-FOV scalars, near/reverse-Z scale. Find the **finalizer**, the *pure*
function that derives matrices from origin/axis/projection, and call it instead of
writing derived matrices yourself, so jitter and other conventions stay intact (it is
also what makes per-eye derivation possible without replaying the stateful owner,
section 8).

### 6.4 Render backend (if threaded)
Per-record execution on the render thread (record function, target prepare / world
owner, view bind, command dispatch, PostFX destination), per-view constant uploads
(projection cache, eye position, previous view-projection for TAA/SSR, depth-hack
projection builder), the resource registry (targets by index, ping-pong buffers,
history textures). The **per-view owner transaction** is the only per-eye boundary
Overlord trusts. Record: the claim conditions (record type == world), what the owner
writes (struct diff), thread identity.

### 6.5 Input command pipeline
Quake lineage: `CL_CreateCmd` -> `CL_KeyMove/CL_MouseMove/CL_NormalInput` ->
`CL_FinishMove` -> store in the command ring (`cmds[cmdNumber & MASK]`). Find the
command struct by watching writes when pressing W/mouse buttons; the angle clamp
(pitch limit) and the player-state update that copies command angles
(`PM_UpdateViewAngles`); the key-binding executor and the **notify-on-command route**
scripts listen to (usercmd bits alone do not reach `notifyoncommand` waiters). Note
signed-char move axes (`[-127,127]`), button bit values, packed-angle scale.
*Exit:* a hook at the last packing boundary (after all other input sources, before
the command enters history) can set a button and see the player act.

### 6.6 Player state and entities
`playerState_s` (origin, velocity, viewHeight, viewangles, delta_angles, pm_flags,
weapon fields), `gentity_s`, `centity_s`. Two copies exist: the **server-side** ps
(`g_entities[0].client`) and the **client-predicted** ps
(`CG_GetPredictedPlayerState(0)`); handle both. Find the flag word that marks "not in
normal control" (mounted, scripted, vehicle: H2 `e_flags & 0x103000`, separate from
`pm_flags`) and gate every VR mutation on it. Record entity-table size and sentinels
(H2 4000 slots, world/none 3998/3999).

### 6.7 Viewmodel and skeletons
The animated object (DObj/skeletal mesh) for first-person arms and gun; the
**skeleton calculation function** that writes bone matrices after animation blend
(hook *after* the original to overwrite bones, 08); client and server entity->DObj
lookups; skinning/submit boundaries; bone/tag names (`tag_weapon`, `tag_flash`,
`j_wrist_*`, elbow/shoulder, finger joints); the depth-hack/viewmodel projection.
Matrices are renderer-origin-relative; that origin can be far from the head after
crouching, so store world space and subtract once at apply time. A single-model DObj
without an animation tree enters a static shortcut that ignores injected bones
unless a skeletal-path flag is requested. A DObj with no animation tree has collapsed
joint translations (use the model's bind pose as source).

### 6.8 Weapon fire path
Fire function, the shot-parameter builder (start/direction), bullet trace, spread
computation, ammo fields (clip/stock per weapon), reload start/finish, weapon switch.
Prefer a lower-level explicit-parameter bullet routine under the real attacker for
independent fire; ask the engine for **live** capacity (`clip_capacity(ps,weapon,alt)`)
rather than static `WeaponDef` fields; fuse/timing fields: trust the call chain.

### 6.9 UI
2D draw command stream (quads/text), HUD dispatcher, menu system (LUI/Scaleform/
Coherent), cursor/mouse input to menus, video playback. If the UI is script-driven,
find the UI VM so pointer events can be injected as UI events (`processEvent` under
the UI critical section) instead of faking mouse input.

### 6.10 Scripts and scenes
Script VM entry, notify system, builtin table, `disableweapons/enableweapons`, player
link-to-entity (rides, cutscenes), vehicle attach, camera tags. **First restore script
diagnostics** (shipped builds strip error text), then locate functions by token and
extent. Evidence for scene identification is native state (linked parent entity, its
`animname`/`classname`, level flags, object variables), never time or health. See
02 section 13 and 10.

### 6.11 Assets, VFS and memory
Asset pools and their fixed sizes, zone loader (custom zones), the VFS interface
(loose-file overlay), asset header lookup (override), image/streaming files, the
pre-unload boundary, memory pool init. Record which tables are indexed by pool index
(model-indexed float tables) before you enlarge a pool.

### 6.12 Threads, frame boundaries and shutdown
Thread contexts (main, backend/render, workers, server, cinematic, window, input,
database), where the frame-state transition and frontend handoff happen, the engine's
unload order (driver libraries such as NVAPI), the quit paths. Record which thread a
hook runs on at each phase, then build the scheduler pipelines (02 section 9).

### 6.13 Save games
What the engine serialises (handles that embed layout-derived offsets, such as
lighting handles); checkpoint reload semantics (generation counters must advance even
when pointers/time repeat).

---

## 7. Engine-property audits (cheap, high value)

- **Enumerate what is view- or history-dependent.** Dump the shader code-constant
  source enum (`MaterialConstSource`, ~680 entries in H2 `game/assets.hpp`) and mark
  the view-dependent subset: view/inverse/projection/view-projection matrices (and
  transposes), world-view-projection, near-plane origin/dx/dy, eye offset, head-rotation
  rows, viewport dimensions, scene UV multipliers, inverse scene projection, depth-from-
  clip, clip-space lookup scale/offset, and all previous-frame/temporal constants
  (prev-frame VP, SSR/motion-blur UV-to-previous matrices, motion matrices, previous eye
  position), plus SSAO/MDAO/DOF/motion-blur/SSR parameter blocks. That list is what must
  be per-eye or per-eye-history.
- **Technique set structure** shows the pass list a per-eye world render must survive
  (H2: 252 techniques = 4 groups of 63 variants: base / instanced / subdiv patch / no
  displacement; z-prepass, velocity, shadow, unlit, emissive, lit variants, deferred
  light volumes, thermal, debug).
- **Capacity warnings:** grep engine logs for `exceeded`, `not drawing`, `too many`,
  `too much` during a stereo run; treat as acceptance failures (02 section 12).
- **Arrays the engine does not clone per eye:** anything stored outside the scene
  record (dynamic-light descriptors keep the center-eye origin; instanced model passes
  take origin from the light descriptor while ordinary passes take the backend eye ->
  opposite half-IPD errors per eye, seen as a detached translucent lighting layer that
  "moves outward in opposite directions in the two eyes"). Audit **all** per-view data,
  not just the camera slot.
- **Flags that leak across views:** "draw with viewmodel" FX flag maps to the scene
  depth-hack bit, so world-space shell casings draw in front of the independently
  rendered weapon.
- **Units and conventions:** read gravity/jump/step cvar descriptions (H2 gravity is in
  "inches per second squared": 1 unit = 1 inch = 0.0254 m), projection conventions
  (reverse-Z, infinite far), near-plane scripts (a script setting near 0.01 on scope entry
  can reject stereo publication).
- **Native self-reporting:** list console commands/cvars early; many comfort toggles
  already exist (FOV, bob, shake, DOF, motion blur, blur, culling, LOD, render-thread
  switches).

---

## 8. Failed experiments and pitfalls (do not retry blindly)

| Experiment / pitfall | What happened | Do instead |
|---|---|---|
| **Dual scene owner, variant 1:** synthesise a second scene record before the end-of-frame boundary ("dual-record") | Fatal engine drop ("Tried to use (null)") in the one frame it ran; the next level entry in the same process succeeded once the one-shot stopped mutating. The frame finaliser had already invalidated transient state. | Preserve the exact natural scene transaction. |
| **Dual scene owner, variant 2:** call the scene owner a second time | Corrupts the surface builder's transient state (same-build A/B proved it). | **Record-local stereo view-slot derivation:** derive each eye's slot from the center slot with the engine's *pure* finalizer (called twice per family), publish the pair before the draw-surface generator, then run eye 0 on the engine's own record and eye 1 on a private clone inside one per-view owner transaction. Never call a stateful owner twice per frame unless you proved its transient state idempotent; run an A/B in the same process before trusting a replay strategy. |
| Poll D3D queries from Present | NVIDIA driver fault in Present (race with `CopyResource/End` on the single-threaded device) | Poll on the renderer thread that produced the copy (02 section 9). |
| Force `ID3D11Multithread` protection | Changed every game call, same kernel failure | Only query it; keep GPU work on the owner. |
| Literal-base-only operand search when relocating an arena | Missed 24 `base+field` sites; two crashes | Whole-image Capstone census (section 4). |
| Relocating only direct references | Compensated (`stack_base - arena`) LEAs shifted sort/end pointers | Search for inverse operands. |
| Raising only a packed-ID capacity check | IDs wrap into earlier surfaces | Change the unit coherently. |
| Generic far relay at a live-RAX site | Destroyed the dvar pointer before the bridge could save it | Preserving near relay. |
| Plain C++ detour on a leaf whose caller expects RCX preserved | Crash at load | Bridge that preserves volatile registers. |
| Optimiser (MSVC v142 WPO) | Publication buffers placed in `.rdata` / writes dropped | WPO off, readback test, PE/PDB audit (02 section 16). |
| Cull-frustum tightening via virtual apex | No improvement; withdrawn | Keep only changes that move an observed symptom. |
| Hooking a function and returning a cached "latest" value for a queued record | Eviction must be "not found", never "latest"; record addresses are reused | Claim by camera identity before and after; key queued data by pair/publication. |
| Bytecode redirect that skips `OP_checkclearparams` | `OP_clearparams` released the caller's references; unrelated crashes later | Execute the original prologue, redirect the body (02 section 13). |
| A theory-driven culling fix, a time-based or health-based phase guess | Looked right, wrong in play | Identify scripted phases by native flags/links. |
| Debug-only install gate | Optimised builds fell back to native first-person rendering | Build mode is not a feature switch. |
| Hiding by mutating native assets | Shared-surface hide mask hid the whole frame surface | Renderer-owned immutable surface-metadata variants; never mutate assets. |
| Replacing engine pointers by clamping to slot zero | Wrong resources | Registry maps mod descriptors to a real source slot; unregistered pointers pass through. |
| Trusting "readable" memory | Not lifetime, ownership or coherence | Identity + generation checks (02 section 5.3). |
| Leaving `FILE_SHARE` open between hash check and map | Check-to-use race | Lease (02 section 2.2). |

---

## 9. Debugging accelerators (build these in Phase 1-2)

- **Console + status commands**: `vr_status`, `vr_diagnose`, `vr_reinit`, `vr_recenter`
  (see 11); every subsystem has a `*_status` command returning counters and last
  decision strings.
- **`startmap <map>`** to skip menus; god/noclip/notarget; free camera; **asset and
  entity browsers**; script console that queues onto the server pipeline.
- **Observation hooks stay in the code** behind startup-frozen options (zero cost when
  off); a probe that mutates is a different component.
- **World-space debug draw math** (works with any refdef with tan-half-FOV scalars):
  `t = (dot(p-org, axis[1]), dot(p-org, axis[2]), dot(p-org, axis[0]))`; reject
  `t.z < 0.01`; `x = W/2*(1 - t.x/fovX/t.z)`, `y = H/2*(1 - t.y/fovY/t.z)`.
- **Dev overlay input ownership:** borrow the engine's UI key-catcher bit, observe the
  native setters, expose `captures_input()` so VR input stands down.
- **Live hook-ownership observers**: composition observers use exclusive registration
  handles; a diagnostic owns its layer and cannot replace another.

---

## 10. H2 reference addresses (examples only; image base `0x140000000`, MW2CR SP)

Pin your own; these illustrate *what kind* of thing to record.

| Item | H2 example |
|---|---|
| Camera composition: call site -> function | `0x1403ACB2D` -> `0x1403B07C0` (call-site rewrite); LOD origin helper hook `0x140781090`, LOD origin global `0x14EEE0890`; `cg_paused` ptr `0x141E39FC0` |
| `R_RenderScene` | fn `0x14077B820` (call `0x1403CA23E`), sig `(int localClient, void* sceneDescriptor, u32 sceneRecordIndex, float lodScale, int drawType)` |
| View-slot allocator / initializer / finalizer / draw-surface generator | `0x14076D5D0` / `0x14077E340` / `0x14077F550` / `0x140778E60` |
| Backend | record `0x1407A6BE0`, target prepare/world owner `0x1407A7DA0`, view bind `0x14079F6F0`, command dispatch `0x14079EAE0`, PostFX destination `0x1407B09A0`, depth-hack projection `0x140786D30`, view copy `0x14078B5D0`, target select `0x1407892C0` |
| Scene layout examples | view slot origin `+0x100`, view-projection `+0x80`, inverse `+0xC0`, reverse-Z/near scale `+0x148`; backend view copy 0x170 bytes; scene descriptor 0x501C8 bytes |
| Input command | create `0x1403D20F0`, normal input `0x1403D13C0` (call `0x1403D2223`), FinishMove `0x1403D0300` (call `0x1403D22A2`), client view angles `0x141E8A65C`, yaw `0x141E8A660`, key-binding executor `0x1403CF1E0`, notify-on-command route `0x1403D3A90` (whitelist 37 `+gostand`, 83 `+sprint`), `PM_UpdateViewAngles` `0x14068D7B0` |
| Predicted player state | `CG_GetPredictedPlayerState` `0x14037F720`; ps layout: `commandTime 0x4c`, `pm_flags 0x54`, `e_flags 0x58`, `delta_angles 0x98`, `viewangles 0x108`, `viewHeightCurrent 280` |
| Skeleton / fire | skeleton calc `0x140657D80`; skinning/submit `0x14071F190`, `0x140774F10`; fire `0x140518A40`, shot params `0x140518880` (call `0x140518B91`), bullet trace `0x1404CBFE0`, spread `0x140518C2E` |
| Shader create wrappers | PS `0x140758A80`, VS `0x140758B30`, DS `0x140758960`, HS `0x1407589F0`, CS `0x140758830` |
| Script VM | opcode dispatch hook `0x1405C90A5`; builtin table patches `0x1405BC7BC`, `0x1405BCB6C`; damage `0x1404BD2E0`; `Scr_RunCurrentThreads` `0x1405C8370` |
| Scheduler hook sites | R_EndFrame `0x14076D7B0`, main `0x140417FA0`, G_RunFrame `0x1404CB030`, Hks `0x140327880` |
| Surface arena | native arena offset `0x62b700`, 256 KiB -> 512 KiB, frontend stride `0x1039300`, warning string `0x1409afa50` |
| Typed entry points | `vr/h2/entrypoints.hpp` (`server_entity_dobj 0x1405A6ED0`, `client_entity_dobj 0x1405A6DD0`, `angles_to_quaternion 0x140613590`, `clip_capacity 0x1406A3A60`, `weapon_selection_request` global `0x141E8A628`) |
| Version sentinel | `*(DWORD*)0x140123456 == 0xE465E151` |
| Constants | usercmd angles `360/2^24`, entity world/none `3998/3999`, 4000 entity slots (stride 760), thread contexts `MAIN 0, BACKEND 1, WORKER0-7 2-9, SERVER 0xA, CINEMATIC 0xB, WINDOW 0xC, INPUT 0xD, DATABASE 0xE, STREAM 0xF` |
