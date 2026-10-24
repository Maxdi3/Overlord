# 01 — Architecture (Overlord's layering, generalised)

## Layers and dependency direction

```
launcher (separate process: settings UI, preflight, launch)
        │  spawns  "game.exe-host -singleplayer"
        ▼
host loader ── loads original game binary at fixed base, resolves imports,
        │      runs component lifecycle (post_start/post_load/post_unpack/pre_destroy)
        ▼
infrastructure components (no gameplay rules)
  ├─ d3d11            device/swapchain wrapping, Present pre/post, resize, device generations
  ├─ vr runtime       OpenXR / OpenVR backends behind one facade; frame loop inside Present
  ├─ stereo renderer  hooks engine render boundaries; per-eye views/records/resources
  ├─ head_pose_bridge tracking → engine coordinates, recenter, camera composition
  ├─ controller_input immutable per-frame input snapshots (poses, actions, generations)
  ├─ menus/HUD        native UI capture → compositor layers, pointer mapping
  └─ diagnostics      status snapshots, bounded traces, crash/stall evidence
        ▲
        │ consumes snapshots only (never calls runtime)
gameplay layer (vr/gameplay)
  ├─ command adapter  usercmd injection (move, turn, buttons, view angles)
  ├─ interaction coordinator + hand arbiter (who owns each hand this frame)
  ├─ domain runtimes  weapons/reload mechanics, carry/holsters, equipment, world use
  ├─ presenters       hand IK, weapon pose, attachments, falling items, haptics
  ├─ campaign/scene adapters  scripted cameras, vehicles, turrets, ladders, climbing
  └─ native adapters  verified calls into game functions (fire, give/take, select…)
```

Rules (from `src/client/component/vr/gameplay/README.md`):
- Gameplay → snapshots/adapters, never the reverse. The runtime must not include
  gameplay headers.
- Portable policy/math headers contain **no native addresses or engine types**.
  Native hooks, offsets and thread checks live in thin adapter files.
- Pure decision code works on *copied* inputs so it can run in CPU unit tests.

## Authority model

| State | Owner |
|---|---|
| Raw poses and actions | XR backend (sampled once, normalised, published) |
| Semantic edges (pressed/released) | Input frame service (derived exactly once) |
| Body frame, hand occupancy | VR gameplay layer |
| Player capsule, stance, inventory, ammo, damage, scripts, saves | **Game engine**, changed only through verified adapters/native commands |
| Pending interaction intent | Domain system (speculative, cancellable) |
| Weapon mechanical partition (chamber vs magazine) | Weapon mechanics + accepted engine transaction (must sum to the native total) |
| Hands, parts, sounds, haptics | Presenters, derived from committed state; never create gameplay state |

## Transaction protocol for any gameplay mutation

```cpp
struct GameplayTransaction {
  TransactionId id; Domain domain; Intent payload;
  uint64_t inputFrameId, contextGeneration, referenceSpaceGeneration;
  uint64_t expectedNativeRevision, expectedDomainRevision;
};
enum class TransactionResult { Accepted, Rejected, Duplicate, ContextChanged, ExternalMutation, Unsupported };
```
- Execute only on the verified engine thread/boundary.
- Replaying the same id never applies twice.
- After the result, publish a new native observation; presentation converges on it.

## Thread model (Overlord)

| Thread | Work |
|---|---|
| Present owner (game render/present thread) | XR `xrWaitFrame/BeginFrame/LocateViews/SyncActions` after Present-post; eye copy + `xrEndFrame` at Present-pre |
| Game client thread | command building (input injection, view angles), camera setup (`refdef`) |
| Game server thread | weapon fire, inventory/ammo transactions, interaction settlement |
| Render backend thread | per-eye re-execution of world records, resource isolation |
| Watchdog thread | stall detection, status/evidence snapshots (reads published copies) |

Cross-thread communication: copied snapshots + monotonically increasing
generations (`reference_generation`, `device generation`, `pair_id`,
`publication`, `recenter_count`). Short mutexes only around copying, never across
native or GPU calls. Status getters must not take the lock held across XR pacing.

## Lifecycle and failure

- Components are registered statically (`REGISTER_COMPONENT`). Hooks are
  installed in `post_unpack` (after the game unpacked itself and before its main
  loop). Saved settings are applied with `scheduler::once(..., pipeline::main)`.
- VR init is attempted once per D3D device generation. Retries need an explicit
  `vr_reinit` or a device change. There is no automatic backend switching.
- Failure handling: tracking loss neutralises buttons and freezes interactions;
  recenter bumps the reference generation and rebuilds body/slot frames;
  external native mutation (death, checkpoint, script give/take) invalidates
  pending transactions and forces reconciliation from a fresh native read.
- Shutdown: `pre_destroy` is idempotent and thread-safe (`component_loader.cpp`).
  The game's quit path calls it before driver teardown; ExitProcess is
  redirected to it as a fallback.

## Settings

- Engine-native config variables (dvars/cvars) registered by components with a
  `SAVED` flag, so they persist in the game's own config file.
- The launcher edits the same saved values. Initialization-only settings (backend,
  pose pipeline) are frozen before the game binary loads.
- The console exposes `vr_status`, `vr_reinit`, `vr_recenter`, `vr_diagnose`, and
  per-feature `*_status` commands.
