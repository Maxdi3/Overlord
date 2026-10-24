# 13 — Porting checklist (milestones with exit criteria)

Copy this into the new project as `docs/vr-port-plan.md` and tick items off.
Each milestone's exit criterion must be *observed* (log, capture or headset), not assumed.

## M0 — Feasibility (see 00)
- [ ] Executable pinned (hash, size, sentinel); single-player, no kernel anti-cheat
- [ ] Graphics API and engine identified; stereo strategy chosen (A/B/C)
- [ ] Projection convention known (reverse-Z? row/column? handedness?)
- [ ] World scale measured (units per meter)
**Exit:** written target sheet.

## M1 — Host and hooks (see 02)
- [ ] Mod loads (host loader or proxy DLL); console/log output works
- [ ] Component system with lifecycle; settings persisted in the game config
- [ ] Detour, call-site hook and byte verification utilities; fail-closed install pattern
- [ ] Crash handler writes minidumps
**Exit:** game runs unchanged with the mod loaded; a test hook logs once per frame.

## M2 — Graphics and runtime (see 04)
- [ ] Device and swapchain captured; Present pre/post and Resize events; device generation
- [ ] OpenXR instance/session on the game adapter (LUID check); swapchains; actions attached
- [ ] Frame loop inside Present: wait/begin/locate after post; transfer/end before pre
- [ ] Solid-color (red left, blue right) frames in the headset
- [ ] Session state handling (focus, stopping, loss), resize/device recreation
**Exit:** stable colored frames for 10 minutes; headset removal/return works; `vr_status` reports correctly.

## M3 — Mono-to-headset sanity (temporary)
- [ ] Copy the game's backbuffer (or its pre-UI scene target) into both eyes as a flat
      screen quad layer (not a projection layer!) to check color and format
**Exit:** correct colors and gamma. Remove this before release.

## M4 — True stereo (see 05)
- [ ] Final camera and render-scene boundary found; observe-only census hooks
- [ ] Eye view derivation (offset along the engine's left axis by IPD/2 × scale,
      asymmetric projection from runtime FOV tangents, engine finalizer)
- [ ] Per-eye execution (strategy A or proven B), per-eye targets → ring → swapchain
- [ ] Culling union, FX culling, tessellation, LOD origin
- [ ] Per-eye temporal histories (TAA/SSR/motion vectors), viewmodel depth-hack fix
- [ ] Screen-space effects audited (flares, vignettes, blur, damage)
- [ ] Never-wrong-image rules: no half pairs, no desktop fill
**Exit:** in headset, correct scale and parallax, no swimming when turning the head,
transparency/particles/reflections identical in both eyes, no edge gaps.

## M5 — Head tracking (see 06)
- [ ] Pose publication, recenter (yaw-only, tracked-only seeding), world scale
- [ ] Camera composition at the final view boundary
- [ ] HMD yaw/pitch into command view angles at FinishMove; pitch clamp fix
- [ ] Command-time history (no double rotation in prediction replays)
- [ ] Room-scale translation and turn-about-head compensation
- [ ] Camera shake/bob disabled
**Exit:** you can walk, look, recenter, and gameplay "look direction" matches the head.

## M6 — Input (see 07)
- [ ] Action set and bindings for Touch/Index/Vive; profile detection
- [ ] Canonical frame with counters and generations; focus-loss neutralization
- [ ] Movement, snap/smooth turn, jump, sprint, crouch/prone via native bindings
- [ ] Menu toggle, recenter long-press
**Exit:** full traversal of a level without keyboard or mouse.

## M7 — Weapons and hands (see 08)
- [ ] Skeleton hook: gun at the controller (authored wrist-to-weapon offset)
- [ ] Muzzle-origin firing with eye-to-muzzle obstruction check
- [ ] Arm IK, finger poses, support-hand two-handed aim
- [ ] Hip-fire spread policy, recoil on the weapon, optional aim assist
- [ ] Auto-ADS / optics
- [ ] Weapon switching via holsters (native inventory authoritative)
- [ ] Physical reload for one family, with ammo-invariant unit tests first
- [ ] Grenades, melee
**Exit:** complete a combat encounter using only physical interactions.

## M8 — UI, scenes and comfort (see 09, 10)
- [ ] Menus captured onto a panel with ray pointer; pause dim; frontend UI-only frames
- [ ] Video theater mode; loading screens
- [ ] HUD strategy (gun ammo, waypoints, subtitles); crosshair off
- [ ] Every scripted scene classified with camera/input/hands policy
- [ ] Vehicles, turrets, ladders
- [ ] Comfort options and defaults
**Exit:** full campaign playthrough log with per-mission notes.

## M9 — Hardening and release (see 11)
- [ ] `vr_status`, per-feature status, support bundle, stall watchdog
- [ ] CPU, mock-runtime and WARP test targets in CI
- [ ] HMD acceptance checklist per headset/runtime recorded separately
- [ ] Launcher with settings, first-run guide, preflight
- [ ] Packaging without game assets; licence notices; version pinning message
**Exit:** a tester on a different machine completes the first mission from the release package.
