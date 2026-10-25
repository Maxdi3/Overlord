# 13 — Porting checklist (milestones with exit criteria)

Copy this into the new project as `docs/vr-port-plan.md` and tick items off. Each milestone's exit
criterion must be *observed* (log, capture or headset), not assumed. Tag every claim
**confirmed / decision / pending** (see `SKILL.md` rule 8). CPU, mock-runtime and WARP results never
establish headset acceptance; record them separately.

Rules for the whole plan:
- Do not start M7 (hands/weapons) before M4 (stereo) and M5 (head tracking) are solid.
- A mathematically valid fix that the operator reports as no improvement is **withdrawn** (Overlord withdrew a tight per-eye culling candidate). Do not keep a fix because its tests pass.
- A fix is accepted only after the *pre-fix failure was reproduced first* by a test or capture.
- Build mode is not a feature switch: every production hook installs in every configuration (M9 audits this).

## M0 — Feasibility (see 00)
- [ ] Executable pinned (hash, size, sentinel bytes); single-player/offline; no kernel anti-cheat; legal check
- [ ] Graphics API and engine identified; stereo strategy chosen (A/B/C)
- [ ] Projection convention known (reverse-Z? row/column? handedness?); world scale measured (units per metre, e.g. 1 unit = 1 inch = 0.0254 m)
- [ ] **Embedded VMs inventoried**: mission script VM, UI VM, mod-script option (see 17); thread that runs each; whether error text is stripped
- [ ] **Engine budgets inventoried**: grep the engine's strings/log for "exceeded", "not drawing", "Too many", "Too much" (per-frame arenas, lighting caches, transparent lists); list which ones stereo will double
- [ ] **Save-game map**: what the engine serialises by pointer/name/handle (FX definitions, lighting handles, script fields) so that mod-created objects and enlarged caches do not corrupt saves
- [ ] **Risky graphics options identified** (supersampling, shader preloading, shadow caches, "fill remaining VRAM"): which are blockers, which are warnings (see M9 preflight)
- [ ] **Toolchain qualification** planned: compiler/linker version, WPO/LTCG policy (see M9)
- [ ] Per-eye data list started: every view-/history-dependent constant or array (camera slot, dynamic lights, dynamic index arenas, SSR/AA/motion-vector history, flares, fullscreen blurs, tessellation constant buffers)
- [ ] Upstream mod/network services identified if forking (updater, MOTD, telemetry) so they can be neutralised
- [ ] Third-party/licence and asset-provenance review started (fonts, audio, models; never ship game assets)
**Exit:** written target sheet with the above.

## M1 — Host and hooks (see 02)
- [ ] Mod loads (host loader or proxy DLL); console/log output works; image base verified *before* imports/TLS patching
- [ ] Component system with lifecycle (`post_start/post_load/post_unpack/pre_destroy`); decide which failures are fatal (throw out of `post_unpack` kills the process) and which disable only the feature (catch, `reason` string, leave native)
- [ ] Settings persisted in the game config; if the engine saves by name hash, a constexpr hash header shared with the launcher; init-only settings frozen before the binary loads
- [ ] Detour, call-site hook and byte-verification utilities; **verify ALL sites then write all, rollback on failure, never partial**; relays allocated near the image *before* patching; decide per site whether any register is live across the patch (near vs far relay); `E8 rel32` target decoding for call sites
- [ ] Scanning is an offline/RE aid only; runtime installs use pinned addresses + masked-byte verification (or a compile-time contract table generated offline by a whole-image disassembly census)
- [ ] Typed symbol table; every entry a feature depends on has a prologue/mask check (inherited tables contain duplicates and stale entries); `static_assert(offsetof/sizeof)` on every layout used (trust asserts over comments)
- [ ] Scheduler pipelines bound to native frame functions with `is_executing()` assertions; thread contexts named
- [ ] Crash handler writes bounded minidumps (user stream with ranges and code around PCs); `IsProcessSnapshotCallback` returns S_OK
- [ ] Shutdown ordering decided: drain the renderer, retire your COM/XR owners, *then* let the engine unload driver libraries (hook the quit wrappers, bytes verified)
- [ ] `vr_status`-style status command skeleton; bounded trace ring; no per-frame allocation/I/O on hot paths
**Exit:** game runs unchanged with the mod loaded; a test hook logs once per frame; clean quit and crash-dump both verified.

## M2 — Graphics and runtime (see 04)
- [ ] Device and swapchain captured via the game's own factory; filter by HWND/BufferUsage/device COM identity; epoch tag; candidate -> active only after a successful non-TEST Present; `ResizeBuffers` notifies **before** the original so backbuffer references are dropped; deferred install/shutdown when requested from inside a callback
- [ ] OpenXR instance/session on the game adapter (LUID check); incompatible implicit API layers disabled; runtime selection policy (explicit override wins; no silent backend switching); swapchains; actions attached
- [ ] Frame loop inside Present: wait/begin/locate after post; transfer/end before pre; `XR_SESSION_LOSS_PENDING` from a swapchain wait is success; `XR_TIMEOUT_EXPIRED` leaves the image legal to release later; first-failure `XrResult` preserved
- [ ] D3D immediate-context queries (copy completion) polled on the renderer thread that issued the copy, not from Present
- [ ] Solid-color (red left, blue right) frames in the headset
- [ ] Session state handling (focus, stopping, loss), resize/device recreation, retryable teardown
**Exit:** stable coloured frames for 10 minutes; headset removal/return works; `vr_status` reports correctly.

## M3 — Mono-to-headset sanity (temporary, see 04)
- [ ] Copy the game's backbuffer (or pre-UI scene target) into both eyes as a flat screen-quad layer (not a projection layer) to check colour and format
**Exit:** correct colours and gamma. Remove before release.

## M4 — True stereo (see 05)
- [ ] Final camera and render-scene boundary found; observe-only census hooks first
- [ ] Eye view derivation (offset along the engine's left axis by IPD/2 x scale, asymmetric projection from runtime FOV tangents, engine finalizer; near plane lower bound half-IPD/8)
- [ ] Per-eye execution (strategy A or proven B); claim records by camera identity before and after; no implicit "next eye"; publish views early, clone payload late; engine job waits joined before the snapshot
- [ ] Native scene resolution forced to the runtime eye extent through the engine's own resize path (supersample must be 1; refuse mid-session changes)
- [ ] **Per-view data audit** (not just the camera slot): dynamic light descriptors, dynamic index origins, model eye offset, XModel placement cache, FX camera planes, depth-hack flags, tessellation constant buffer, LOD origin
- [ ] Culling union, FX culling, tessellation; per-eye temporal histories isolated *before* the eye's first read (TAA/SSR/motion vectors)
- [ ] Screen-space effects audited: lens flares converted to world-space quads per eye (bump dependent matrix-cache version stamps on entry and exit, restore inputs only), fullscreen blurs re-run per eye after the display transform, menu blur zeroed at source, damage overlay composed over the union of both eyes' FOV
- [ ] **Capacity audit:** run the heaviest levels with a stereo capture window; **zero** engine "exceeded / not drawing / too many" warnings; any arena enlarged as an ABI-coherent patch (scaled-ID unit change, base+field operands, folded/inverse operands, bank spacing = native frontend stride, verify-all-then-write with rollback, census coverage test, savegame-embedded values guarded)
- [ ] Never-wrong-image rules: no half pairs, no desktop fill, no other-eye fill
- [ ] Exit gate for production is the **execution census** (admission), not the report-only GPU census; probes stay fingerprint-gated and off by default
**Exit:** in headset, correct scale and parallax, no swimming when turning the head, transparency/particles/reflections/shadows identical in both eyes, no edge gaps; same-pair GPU readback shows no unexplained left/right differences; capacity audit clean.

## M5 — Head tracking (see 06)
- [ ] Pose publication, recenter, world scale. Overlord's recenter sets the **full position including height**, heading from the horizontal forward direction (not fused yaw), only on upright non-singular poses, only while FOCUSED and views/head TRACKED
- [ ] Camera composition at the final view boundary; engine refdef order angles -> axis -> origin
- [ ] Command angle packing/units understood (H2: 24-bit turn, `360/2^24`, plus `delta_angles`); HMD yaw/pitch into command view angles at FinishMove; the engine's pitch clamp undone only for the tracked player state and matching command time (restore `delta_angles` pitch)
- [ ] **Heading math:** body/shoulder/scripted-camera heading from the rotation matrix (`atan2`), Euler only for command packing; test headset standby/resume (a 180-degree flip once); command history stores physical heading
- [ ] Command-time history (no double rotation in prediction replays); room-scale turn-about-head compensation
- [ ] Camera shake/bob handled (bob is a toggle, default ON, scoped to the player's camera call sites via register-preserving bridges); engine FOV rewrites neutralised
**Exit:** you can walk, look, recenter, and gameplay "look direction" matches the head; a test that replays the same command history repeatedly (Overlord replays it 1000x) gives identical results.

## M6 — Input (see 07)
- [ ] Action set and bindings for Touch/Index/Vive; profile detection; **per-profile differences documented** (some profiles lack buttons: re-home the function or accept loss); OpenVR manifest with boolean actions + binding-side thresholds, haptic amplitude 0, localization block, manifest + binary deployed as a pair
- [ ] Canonical frame with sequence/press/release counters and generations; focus-loss neutralization; analog hysteresis (value press .55/release .45; Index trigger .01/.005, squeeze .15/.05)
- [ ] Movement, snap/smooth turn (90 deg/s, 0.2 deadzone, dt cap 50 ms), jump, sprint, crouch/prone via native bindings (command-notification queue as well as command bits for script listeners)
- [ ] Script-visible input: identify the receiver (binding notify vs polled state vs weapon state); polled scripts get a held bit or >= 100 ms visibility
- [ ] Menu toggle, recenter long-press; neutral rearm after any focus/context/binding change
- [ ] Producer continuity for native slow motion (breach/bullet-time)
**Exit:** full traversal of a level without keyboard or mouse; a held trigger across a pause/focus change never fires on return.

## M7 — Weapons and hands (see 08)
- [ ] Skeleton hook: request ALL bones, **verify the calculated bits afterwards**, solve at most once per `(object, matrices, epoch)`, latch rejections, binding cache with asset epoch; structural rig admission (unique bone names, topology, muzzle axis within 5 degrees) with exact name + capacity + attachment topology for physical mechanics
- [ ] Gun at the controller (authored wrist-to-weapon offset; grip-local wrist point, rotated with the controller); arm IK with oriented bend plane, forearm twist finalisation, empty-hands object
- [ ] Muzzle-origin firing with eye-to-muzzle obstruction check; hip-fire spread policy; recoil on the weapon (never the HMD); fire delivery classes (bullets/projectile/thrown/scripted device); weapon-fired notification in engine order
- [ ] Render-record binding of solved poses (object + matrices + epoch + stereo pair identity; byte-compare bones before/after native skin)
- [ ] Viewmodel part hiding without editing assets (surface mask / immutable metadata clones / skin partition); the hook installs in every build
- [ ] Weapon switching via holsters (native inventory authoritative); identity = token + generation; pickup guards
- [ ] **Physical reload** for one family: pure planner with conservation check, compare-and-commit with escrow, block the earliest native reload-begin and fire/consume paths, reconcile external mutation, randomized event-sequence tests (ammo conservation checked after every commit) **before** any presentation
- [ ] Hand arbiter with witness model; fresh-press acquisition; release resolution order
- [ ] Authored data budget estimated: per-receiver pose/profile folders (Overlord has about 47 receiver families, a 46-entry trigger-finger table, mirrored left poses, captured bone fixtures with source SHA-256)
- [ ] Grenades, melee; auto-ADS/optics; aim assist
**Exit:** complete a combat encounter using only physical interactions; ammo-conservation fuzz passes; two copies of one gun do not interfere.

## M8 — UI, scenes, scripts and comfort (see 09, 10, 17)
- [ ] Menus captured onto a panel with ray pointer; pause dim owned by the compositor; tracking/pointer keep running during pause; frontend UI-only frames
- [ ] Video theater mode; loading screens
- [ ] HUD strategy (gun ammo via the engine's own widget, waypoints, subtitles by native flags); crosshair off; hide-HUD gate without dvar writes
- [ ] **Script VM integration** (17): call bridge with stack isolation and guarded calls, restored error diagnostics, builtin table extension, interceptors restricted to call sites, bytecode contracts (signature + mask + operand cross-check + uniqueness; whole-statement skips; OP_clearparams rule), embedded `scripts/vr_*` sources, plugin state in engine save state
- [ ] **UI VM integration** (17): wrap definitions without copying game code, key-based prompt replacement with exact-locale fallthrough, animation-pool and GC pitfalls handled, LUI critical section for event injection
- [ ] Every scripted scene classified (pure `classify(evidence)`, identity not names, persistent fields not transient flags) with camera/input/hands policy; fail-safe `release` on loss of observation; load recenter once
- [ ] Vehicles, turrets, ladders, carrier locomotion; scripted waiters driven through the flat game's own triggers
- [ ] Comfort options and defaults (snap/smooth turn, vignette off, blur/DOF/motion blur off, horizon level, bob toggle)
**Exit:** full campaign playthrough log with per-mission notes; checkpoint reload and Resume tested for every adapter that touches saved state.

## M9 — Hardening and release (see 11)
- [ ] `vr_status`, per-feature status, `vr_diagnose` support bundle (bounded, never auto-upload), stall watchdog, crash evidence reader
- [ ] **WPO qualification:** release configuration with whole-program optimisation off (`/GL-`, `/LTCG:OFF`) unless the toolchain is re-qualified; publication-buffer readback test (`aggregate_publication_tests`) and EXE/PDB section audit gate packaging; feature-parity test reads emitted symbols from the PDB so no hook is compiled out
- [ ] **Risk-settings preflight** in the launcher: supersampling != 1 is a blocking error; shader preloading, shadow caches and "fill remaining VRAM" are warnings needing per-launch acknowledgement; one-click atomic fix; also a support tool for crash/black-screen triage
- [ ] CPU, mock-runtime (fake `openxr_loader.dll` with call-order enforcement and eye-pixel readback) and WARP test targets in CI; strict OpenVR doubles; seeded fuzzing of ammo conservation; cadence sweeps 45–144 Hz
- [ ] HMD acceptance checklist per headset/runtime recorded **separately** (implemented / offline tests passed / hardware accepted / pending); never transfer acceptance across build configurations
- [ ] Launcher with settings, first-run guide, preflight, honesty notice; settings schema from one native catalog (no second option list in JS)
- [ ] Packaging without game assets, saves, logs or PDBs; licence notices (OFL fonts etc.); version-pinning message; deployment pair (binary + input manifest + bindings) with hashes, backups and rollback; compiled `.ff` zones tracked, raw sources not
- [ ] Upstream update/MOTD feeds neutralised in a fork
- [ ] Release gate: exact staged Release binary starts in a fresh process, loads a level, death/checkpoint reload and Resume pass for anything touching FX/asset retirement/saved state; known unresolved crash blocks delivery
**Exit:** a tester on a different machine completes the first mission from the release package; candidate flags (`acceptance_status`, `runtime_acceptance`, `delivery_ready`) honestly reflect what was observed.

## Cross-cutting pitfalls to re-test at every milestone
| Pitfall | Test |
|---|---|
| second-eye / replay re-simulation | duplicate-sequence and both-eyes tests on every consumer |
| stale data after recenter/focus/load | generation-bump tests; neutral-rearm tests |
| address reuse | identity + epoch tests; asset-unload tests |
| partial patch install | single-site-mismatch test leaves everything untouched |
| presentation gate reuse | acquire/release x skin-before/after orderings |
| native warnings hidden by a "working" build | grep logs for "exceeded/not drawing" on every capture |
| plugin state lost on load | save/load round trip per adapter |
| Debug-only behaviour | run the parity audit on Release |
