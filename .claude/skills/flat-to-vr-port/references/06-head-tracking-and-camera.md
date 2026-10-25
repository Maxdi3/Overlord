# 06 — Head tracking, camera composition and aim coupling

The head pose has to reach the game through **two separate doors** and one
shared frame:

```
runtime sample (once per prediction, 04)
        │ publish_tracking_pose()
        ▼
 head-pose bridge  ── recenter reference, coordinate conversion, freshness,
        │              optional head filter, command history, policy rig
        ├─► (A) COMMAND door  FinishMove  → HMD yaw/pitch into command angles  (aim/movement/AI see the head)
        ├─► (B) CAMERA door   view setup  → full HMD rotation + position into the refdef (what the eyes see)
        └─► (C) spatial frame (tracking→world, body anchor) → hands, holsters, IK (08)
```

Keep A and B separate. A writes only the *yaw/pitch contribution* into the
game's own input; B composes the *full* tracked rotation and translation on top
of the game's resulting view and removes whatever A already contributed. Mixing
them causes double rotation, drift, or a camera that snaps on every script rotate.

Overlord files (all under `src/client/component/` unless noted):
`vr/head_pose_bridge.cpp/.hpp`, `camera.cpp`, `vr/game_view.hpp`,
`vr/continuous_view_angles.hpp`, `vr/camera_rig.hpp`, `vr/scripted_camera.hpp`,
`vr/scripted_view.hpp`, `vr/scripted_position.hpp`, `vr/roomscale_origin.hpp`,
`vr/remote_view.hpp`, `vr/remote_look.hpp`, `vr/body_pose.hpp`, `vr/pose_filter.hpp`,
`vr/stabilization.cpp`, `vr/camera_bob.cpp`, `vr/camera_bob_bridge.hpp`,
`vr/gameplay/controller_component.cpp` (FinishMove / UpdateViewAngles),
`vr/gameplay/campaign/sequences/camera_policies.hpp`,
`vr/gameplay/campaign/scripted_sequences.hpp` (`camera_request_for`),
`vr/gameplay/scripted_camera_reference.cpp`. Docs: `docs/vr-camera-architecture.md`,
`vr-hmd-resume-heading.md`, `vr-shared-tessellation-and-head-pitch.md`,
`vr-camera-bob.md`, `vr-stabilization.md`. Tests: `tests/vr/game_view_tests.hpp`,
`horizontal_heading_tests.hpp`, `camera_policy_tests.hpp`, `roomscale_origin_tests.hpp`,
`body_pose_tests.hpp`, `stabilization_tests.cpp`, `camera_bob_tests.cpp`.

---

## 1. Publishing the tracked pose and the recenter contract

The runtime adapter calls once per prediction (Present-post, see 04):

```cpp
publish_tracking_pose({position_meters, orientation3x3}, sample_id, sampled_at, reference_policy);
```

The bridge (`head_pose_bridge.cpp`):

1. **Validates**: finite position; rotation rows unit within 0.05, pairwise dot
   < 0.05, determinant within 0.05 of 1. Invalid -> count it, invalidate the pose
   (and the spatial frame, filters, body estimator). Never "repair" a pose.
2. **Converts** tracking space to engine space (axis permutation in 14) and builds
   the absolute engine orientation (rows forward/left/up).
3. **Recenters** if a recenter is pending (below).
4. Computes `local_position_units = tracking_to_engine(R_ref^T · (p − p_ref)) · world_scale`,
   `local_orientation = engine_basis(R_ref^T · R_head)`, `local_angles` (continuous Euler,
   command input only), `local_heading` (geometric, section 3), and horizon roll for diagnostics.
5. Stamps `pose_sampled_at`. Every consumer (`apply_game_view`, `apply_remote_control`,
   `get_tracking_reference`, `get_spatial_frame`) rejects a pose older than **150 ms** or
   with a sample time in the future. `get_spatial_frame` additionally requires
   `camera_frame.generation == recenter_count`.

OpenXR specifics feeding it (`openxr_runtime.cpp`): in native-stereo mode the *render
head* is the **midpoint of the two eye positions** with `slerp(qL, qR, 0.5)`; eye
offsets are `R_head^T (p_eye − p_head)`. If `|dot(qL,qR)| < 0.99999` (about 0.51 deg
between eyes) the family is rejected as canted (a shared-camera engine cannot show it).
Quaternion validity: `|q|² − 1` within 0.01.

### Recenter

| Question | Overlord's answer |
|---|---|
| What starts pending? | `recenter_pending = true` at startup, on `set_enabled`, on every new OpenXR session/OpenVR connection ("a new LOCAL space cannot reuse a reference from an older session"), on `XrEventDataReferenceSpaceChangePending(LOCAL)` once `predictedDisplayTime >= changeTime`, on `vr_recenter` (menu button hold ~1 s), and on checkpoint load. |
| Which sample may seed it? | Only a sample whose adapter passes `reference_policy::allow_recenter`: session **FOCUSED** and both the view state and the head location carry the **TRACKED** bits (not merely VALID). An *inferred* pose can continue an established origin but never seed a new one; while a recenter is pending a `retain_reference` sample just invalidates (`head_reference_policy`, `openxr_runtime.cpp:109`). |
| What is stored? | `reference_pose = { full tracked position (x, y, z), yaw-only rotation about tracking-up }`. So **recenter zeros all three translation axes including height**: the camera lands on the engine's own eye position. Physical pitch and roll are NOT absorbed (`local_angles = {abs_pitch, 0, abs_roll}`). Recentering while seated gives seated play at native eye height; crouching later moves the camera down 1:1. (A port could keep floor-relative height instead; Overlord does not.) |
| Which heading? | `atan2` of the **horizontal projection of forward** when the head is upright (`up.z > 0` and horizontal length > 0.01), so a recovered pose never inherits an alternate Euler yaw. Over-pitched (past the pole) recenters keep the continuous Euler branch. Fused yaw is deliberately *not* used here (it couples simultaneous pitch+roll into the translation frame). |
| Generation | `recenter_count++` is the reference generation. Hands/camera/consumers compare it (`head.recenter_count == input.reference_generation`) and drop stale data; a new native camera application is required before the spatial frame is available again. |
| Side effects of the `vr_recenter` command | temporal history reset in the stereo owner pass, menu anchor re-captured, head filter + stabilization epoch invalidated. `request_recenter(true)` (checkpoint load) also clears command history, remote-view and scripted rig state. |
| Native facing is preserved | `game_view::state::rebase` adds `old.heading − new.heading` to `reference_offset_`, so after a recenter the player does not snap; old command-history entries are resolved *in the new reference* (section 4). |

Checkpoint/spawn: `reset_game_view()` discards derived camera/input history without
restarting tracking ("spawn/save-load can reuse both player pointers and native
command times"). `game_view::load_recenter` consumes the **first valid post-load
observation once**: if tracking/VM data is not ready the load stays pending;
ordinary gameplay consumes it without recentering so a later scripted scene cannot
inherit the reset (`scripted_sequences.cpp:192`).

Staleness is per consumer, not one global 150 ms: head pose 150 ms; body estimator
gap reset 150 ms; filters reseed at gap > 150 ms; haptics 100 ms; calibration settle
200 ms; smooth-turn dt cap 50 ms; remote look dt cap 50 ms; HUD/menu captures 250 ms;
damage overlay 250 ms (`damage_screen.hpp:54`).

---

## 2. Heading math: geometric heading, not Euler (the standby/resume 180° bug)

Three different "yaws" exist. Mixing them was the root of two real, captured bugs.

| Quantity | Computed from | Used for | Pitfall it avoids |
|---|---|---|---|
| **Command Euler angles** (`continuous_angles`) | nearest-branch Euler of the tracked rotation | the pitch/yaw numbers packed into the game's input command, and nothing else | pitch past ±90° would flip yaw 180° in the *command* |
| **Physical heading** (`horizontal_heading`, fused/tilt-free yaw) | the rotation matrix directly | body/shoulder/holster heading, body estimator, command-history bookkeeping, all camera-policy reference headings, world-frame hand/body axes | Euler branch ambiguity |
| **Recenter heading** | `atan2(forward.y, forward.x)` of horizontal forward (upright only) | the recenter reference | pitch/roll coupling into the translation frame |

```cpp
// rows = forward / left / up (engine basis). Fused (tilt-free) yaw, independent of Euler branch:
float sine = f.y - l.x, cosine = f.x + l.y;               // = atan2(2wz, w²−z²) in matrix form
if (hypot(sine, cosine) >= 0.01f) yaw = atan2(sine, cosine) * 57.29578f;   // else keep previous value
// sanity: pure yaw ψ: f=(cψ,sψ,0), l=(−sψ,cψ,0) → atan2(2sψ, 2cψ) = ψ
```

Singular only when the **up axis is fully inverted** (about 8° from inverted): keep the
last heading; upright recovery re-acquires the physical heading immediately. Pure pitch,
even leaning back past vertical, keeps heading.

Camera-rig helper (`camera_rig.hpp::yaw`): `atan2(f.y, f.x)`, falling back to the **left
axis** `atan2(−l.x, l.y)` when forward is vertical (`f.x²+f.y² < 1e-6`). The bridge's
`extract_h2_yaw_only` does the same (squared horizontal length < 1e-4) and counts the fallback in
`yaw_from_left_count`.

**The bug.** After headset standby/resume the continuous Euler state sat on the
alternate branch (captured: pitch ≈ 162, yaw ≈ 72–74, roll ≈ −177, physically the
same orientation as pitch 18, yaw −106, roll 3). Rendering used the full matrix so the
picture was right, but shoulders/body used the Euler yaw and were exactly 180° off (the
left shoulder 18 cm to the right of the camera). A later cutscene bug (entry/handback
of a scripted camera) consumed the same Euler yaw as a physical heading.

**The fix, as rules for any port:**
1. Keep nearest-branch Euler **only** as the native command representation (the engine's
   pitch/yaw pair needs continuity across the poles).
2. Derive body, shoulders, equipment anchors, scripted-camera entry/handback and
   command-history reference headings from the **matrix** (`atan2` form above).
3. Native command history stores the **physical heading and the cumulative reference
   rotation** next to the Euler contribution. Prediction that reads an older command then
   resolves it in the *current* reference instead of falling back to 0 (a half turn).
4. Every camera policy must give the same matrix on either Euler branch (the test runs
   `gameplay, free, locked, yaw, aligned, vehicle, authored` with a normal and an
   alternate-branch history through normal / invalidate-pose / recenter stages and
   requires the same camera within 0.003).
5. Replay the captured orientation 1000x in a test; also sweep pitch 0..270 both ways
   keeping yaw/roll, check pole recovery and invalid input (`horizontal_heading_tests.hpp`).

Hardware acceptance (actual standby/wake) remains separate from these offline replays.

Related pitch-pole facts: the engine camera contributes **world yaw only** to the VR
camera (game pitch and any game roll are stripped; hands and room-scale translation are
identical for pitch +60 vs −45); HMD pitch/roll are preserved identically for every game
pitch; over-pitch ("backlean to 120°") yields no yaw reversal and no folded pitch.

---

## 3. Aim coupling: command angles (door A)

### What gets written and where

Write the HMD yaw/pitch into the game's own per-tick command **at the last packing step**
(Overlord: the call site of `FinishMove`, after CreateCmd's clamp and every other input
source), so movement direction, interaction traces, AI awareness, sprint direction and
"look at" scripts all agree with the head. Hook call *sites* of the sub-steps
(NormalInput / FinishMove / UpdateViewAngles), not function entries, and only the command
path is intercepted.

```cpp
// head_pose_bridge::apply_game_view(native_angles, delta_angles, scripted_epoch, remote)  [game_view::state::apply]
fail (leave angles bit-identical) unless: target matched && enabled && pose_available &&
      !recenter_pending && pose fresh (<=150 ms, not future);
if (scripted_epoch) return false;          // script owns rotation: never feed HMD yaw back into its anchor
yaw   = remainder( native_yaw + remainder(head_yaw - previous_head_yaw, 360), 360 );   // DELTA, not absolute
pitch = remainder( -head_pitch - remainder(delta_pitch, 360), 360 );  // H2: positive command pitch looks DOWN;
                                                                      // delta_angles belongs to the engine
```

Key properties:
- The tracker stores only the **HMD contribution** (`contribution{generation, yaw, heading,
  reference_offset}`), never a second copy of native yaw. Native mouse/stick/snap turning and
  physical turning each contribute exactly once; 1000 repeated samples do not accumulate;
  yaw wrap uses the short signed delta; a NaN head sample cannot poison history.
- Snap/smooth turning modifies the **native client yaw**, not the HMD reference.
- Gate also on "normal gameplay": cgame initialised, no key catcher, not paused
  (`normal_gameplay()`), and `now − last_command > 150 ms` resets all consumers.
- Set the game view *independently of controller availability and neutral arming* (a head
  turn must always turn the player); only button/stick consumers arm on a neutral sample.

### Angle packing and `delta_angles` (check your engine; this is a silent-bug generator)

- H2 stores usercmd angles as a signed **24-bit turn in a 32-bit field**: 360° = 2^24, i.e.
  `degrees = packed * (360/16777216) + delta_angle` (0.000021457672 per count). Quake-lineage
  engines usually use a 16-bit short (360/65536); **check the target's scale before injecting**.
- The final engine view angle = decoded command angle + `ps->delta_angles` (the server's
  accumulated correction from teleports/script rotation). Injecting an absolute head angle
  without accounting for `delta_angles` makes the view jump after any script rotate. Overlord
  subtracts the delta from the pitch it injects and adds it back when resolving (below).
- H2 axes: yaw increases to the **left** (a right turn is negative yaw); positive command
  pitch looks **down**. `continuous_angles` itself is in "positive pitch looks up" convention,
  hence the minus sign.
- Offsets proved by disassembly, kept next to the evidence in `structs.hpp` (H2-only:
  `playerState.delta_angles = +0x98`, `viewangles = +0x108` — an older struct had 0x128, wrong).

### Pitch clamp undo (`PM_UpdateViewAngles` hook)

The engine decodes the packed command then clamps pitch (±85° in H2) and rewrites
`delta_angles`; a head leaning past 85° must not be clamped. Wrapper (`update_view_angles_stub`):

```cpp
delta = ps->delta_angles[0];
tracked = alive && (ps == predicted_ps || ps == g_entities[0].client)      // BOTH copies
          && resolve_game_pitch(cmd->serverTime, cmd->angles[0], delta, &pitch);
original(ps, cmd);                      // runs exactly once: yaw, roll, shield effects, parent Pmove constraints stay native
if (tracked) { ps->delta_angles[0] = delta; ps->viewangles[0] = pitch; }
```

`resolve_pitch` succeeds only when the history entry for **that exact command time** was
recorded as headset input AND its packed pitch matches AND the reset generation matches; it
then reconstructs `pitch = packed*360/2^24 + delta_pitch`, wrapped to ±180. Patch both the
**client-predicted player state and the server-side `g_entities[0].client`** because both
run the same Pmove; otherwise prediction and server disagree and the view rubber-bands.
Mounted/prone/script constraints stay native because only the pitch term is restored.

### 64-entry command-time history and rollback rules

`game_view::state` (and `roomscale_origin`, `remote_view`) keep a ring of **64** entries — the
same bound as the engine's usercmd ring — keyed by **command time** (`serverTime`), written by
`record_game_command(serverTime, angles[0], tracked)` right after CreateCmd returns (before the
command is consumed by prediction).

| Rule | Why |
|---|---|
| Lookup is by **exact** command time; a missing command never borrows a neighbour or "latest" | prediction replays old commands; borrowing double-applies yaw |
| Unmatched command at the camera door -> the camera transaction **fails** (`game_view_history_misses++`) unless a script owns the camera | never guess the input epoch |
| `command_time < latest_time` (checkpoint/map load) clears the ring | a new timeline cannot resolve against the old map |
| Same-time checkpoint reset uses the native spawn heading, not the old HMD contribution | `reset_game_view()` |
| A *paused* render keeps its exact already-resolved command beyond ring eviction | pause renders the same command forever |
| Recenter (generation change) never subtracts yaw from a different reference: entries resolve via `head.yaw − (reference_offset_ − entry.reference_offset)` | native facing preserved across recenter |
| Prediction of an older command removes only that command's own head contribution; late tracking with older prediction applies the new pose once | older frames stay consistent |
| Evicted older prediction cannot poison the live origin | bounded history |

Whenever you remove "HMD rotation that was already put into a command", look it up using the
**exact timestamp the engine will use** (see remote camera, section 4).

Edit commands **before** they enter history (Overlord injects in CreateCmd/FinishMove, never on
read). Weapon ADS or other bits are OR-ed after FinishMove has packed the weapon (07).

---

## 4. Camera coupling: the final view boundary (door B)

### Where and in what order

Hook the call at the engine's **final view-setup boundary** (the place where it has just
computed `refdef.origin/axis` from player state and is about to publish the view). Overlord:
`utils::hook::call(0x1403ACB2D, set_viewpos_now_stub)` (H2-only); the stub:

1. observe(enter) -> `paused_free_move()` (test camera, below);
2. read the **predicted** player state; resolve scripted `sequence`, fixed sniper, mounted
   turret camera, remote (notebook) camera epoch;
3. build one `camera_request` (policy + epochs) — **all special camera owners are arbitrated at this
   single point** through a policy object (`policy.owns_rotation()`), not scattered patches;
4. ladder origin adjust (only if neither mounted nor sniper);
5. `head_pose_bridge::apply_camera(refdef.org, refdef.axis, command_time, native_angles, request,
   rotation_reference, body_mode)`; if applied, cache the camera origin;
6. javelin/screen camera/FOV override (must not move the tracked-hand reference frame:
   "keep tracking/hand space HMD-owned; only the scene/HUD camera changes");
7. observe(after_mod); **call the original** (`0x1403B07C0`) — i.e. mutate `refdef` **before**
   the original runs so everything the engine derives from it sees the head; then
   observe(leave).

Also overwrite the engine's separate **LOD/cull/streaming origin** after its camera helper
(hook of `sub_140781090`; H2 global `0x14EEE0890`) when `cg_paused==2 || camera_was_applied()`, so
LOD/streaming picks match the head position. Every engine has such a "view origin for
LOD/cull/streaming" distinct from the render camera.

Per-stage observation (`enter, before_original, after_original, after_mod, leave` with the
caller return address) is how Overlord detects *other writers* of the camera.

### What `apply_camera` computes

```cpp
base_yaw     = native_angles[1]                    // cgs->refdefViewAngles yaw (or mounted angles); axis-extraction fallback
head_axis    = local_orientation                   // full tracked rotation relative to recenter heading
cmd_head_yaw = history.resolve(command_time)       // how much HMD yaw this exact command already carried
base         = remainder(native_heading − cmd_head_heading, 360)     // remove what door A already added
composed     = camera_rig.compose({native_axis, head_axis, head_meters, native_heading,
                                   head_heading, cmd_head_heading, time, reference}, request, tag)
origin      += roomscale_turn_offset * world_scale
origin[w]   += Σ_i head_offset[i] * world_scale * base_yaw_axis[i][w]     // yaw-only base: pitch/roll come only from the HMD
axis         = head_axis × base_yaw_axis (and policy handling)
```

The camera is the **center eye**; per-eye offset and asymmetric projection come later (05).
Any scripted FOV change must not alter the HMD projection (optics are rendered separately,
`docs/vr-native-ads.md`).

Also publish a **spatial frame** for hands/body (`tracking_origin`, yaw-only `world_yaw_axis`,
`units_per_meter`, `generation`, `captured_at`, head position/forward/up, body anchor). The
tracking origin is translated by `(head_offset − local_position)·base_yaw_axis` so the **whole
tracking frame moves with the camera**: hands keep 1:1 motion relative to the head even when a
cinematic camera attenuates head motion. `tracking_to_world(frame, pose)`:
`pos = world_origin + Σ local[i]·units_per_meter·yaw_axis[i]` (14). If the head filter is on, hand
poses are composed with the same `tracking_correction`, so head and hands share stabilization.

### Remote camera: use the right command time

For a remote-camera mode (missile/UAV) the native camera evaluation reads the **latest usercmd's
`serverTime`** (when its remote-control button is set), not the predicted `ps->commandTime`.
Overlord passes `latest.serverTime` as the camera command time in that case (`camera.cpp:196-205`;
H2: evaluation at `0x1403B0485`, button `0x100000`). A mismatch looks up the wrong history entry
and double-applies yaw.

### Free camera as a test tool

H2's `cg_paused == 2` free-move camera (`paused_free_move`) builds angles from
`usercmd.angles*360/2^24 + delta_angles`, calls `AnglesToAxis`, then overrides `refdef.org` — the
minimal recipe for "write my own camera into the engine view state" (angles -> axis -> origin
order matters), and a way to test eye separation/culling without gameplay constraints. Lua/console
overrides exist (`camera_origin_override`).

---

## 5. Room-scale translation and turn compensation

Walking physically moves the camera by `head_offset × world_scale`; the native capsule/collision
stays where the player entity is (head/hands/muzzle can leave the capsule — see 08 for collision
checks of muzzle through walls). (Optional, not in Overlord: push the capsule toward the head so
you cannot lean through walls.)

**Turn about the head, not the recenter point** (`roomscale_origin.hpp`). When the *rendered native yaw* changes
(snap/smooth turn, script), the world basis rotates about the player origin, which would swing a
head that is physically 40 cm away from it. Compensate by moving a shared world-origin offset:

```
offset' = offset + world(latest.basis, head) − world(new.basis, head)     // metres; world(b,v)=Σ v[i]·b[i]
camera/hands/body origin += offset * world_scale
```

- Use the **final rendered** yaw (after the game's movement constraints) minus the HMD contribution
  already in the command ("rendered native yaw has already passed the game's movement constraints"),
  never the requested yaw.
- 64-entry command-time history so a replayed older prediction reuses its own captured turn origin
  and cannot integrate a turn twice; evicted entries return false; unknown older commands are rebuilt
  only when the command is known (`command_known`); time going backwards or a reference-generation
  change resets it. Only the *ordinary gameplay* profile (no epochs) uses it — scripted, vehicle and
  fixed-optic cameras reset it and own their origin. Test numbers: head position invariant across
  yaw changes 35/65/125/−175/125° within 0.002 m; both hands rotate rigidly about the head.
- Hands and body equipment use that same origin (it is baked into the spatial frame).

---

## 6. Camera policy system (scripted scenes, vehicles, optics, remote control)

Composition is generic and engine-neutral (`scripted_camera.hpp`, `camera_rig.hpp`: no map names,
addresses, VM or runtime calls). A scene adapter only *selects* a policy and supplies identity/phase.

| Dimension | Values |
|---|---|
| owner | `player`, `script` (`owns_rotation()` = script) |
| head rotation | `fixed` (identity), `pitch_roll` (`head × yawAxis(head_heading)^T`: physical tilt, no yaw), `limited` (per-axis clamp of the delta from an entry baseline), `free`, **`command_delta`** (head treated as *relative input*, remote vehicles) |
| head translation | `fixed` (zero, clears baseline), `attenuated` (`(head − baseline)·gain`, vector length clamped to `limit`), `tracked` (1:1) |
| script rotation | `override_view` (matrix from the source), `additive` (relative change since last frame: `full_base = full_base · prev_source^T · source`), `ignore` |
| axes | `yaw`, `yaw_roll` (pitch removed, continuous branch), `all` |
| source | final native view, or named tag matrix (`tag_player`, `tag_aim`, `tag_origin`, `none`) |
| entry | `preserve`, or `align` once (a one-shot "cut") |
| limits | degrees relative to the entry pose, default `[-90,-180,-180]..[90,180,180]` (pitch, yaw, roll); invalid limits -> 0 |
| translation | `gain = 0.10`, `limit = 0.05 m` (defaults) |

**Scripted head translation:** when a script owns the camera, added physical motion is scaled to
**10%** with a **5 cm** spherical cap, anchored at script entry (`vr_scriptedHeadScale`, default
0.10, range 0..0.25; 20 cm of physical motion -> 2 cm; a big step is capped at 5 cm; recenter
establishes a new baseline; stereo separation is unchanged). Ordinary gameplay stays 1:1. Translation
has its **own epoch** (`position_epoch`) separate from rotation, so entering combat does not re-anchor
the player and changing rotation policy does not move the head. A player-owned fully-tracked camera
keeps `position_epoch = 0` so entering a native link does not shift the entry.

### Named profiles (`camera_profiles`, values verified in `scripted_camera.hpp`)

| Profile | owner | head | translation | script rot | axes | source | entry |
|---|---|---|---|---|---|---|---|
| `gameplay` | player | free | tracked | ignore | yaw | view | preserve |
| `free` | script | free | attenuated | ignore | yaw | view | preserve |
| `aligned` | script | free | attenuated | ignore | yaw | view | align |
| `locked` = `fixed_scope` | script | fixed | fixed | override_view | all | view | preserve |
| `yaw` (breach) | script | free | attenuated | override_view | yaw | view | preserve |
| `vehicle` | script | free | attenuated | additive | yaw | view | align |
| `authored` | script | free | attenuated | additive | yaw | **tag** | preserve |
| `authored_full` | script | free | attenuated | additive | all | tag | align |
| `authored_yaw_roll` | script | free | attenuated | additive | yaw_roll | tag | align |
| `bounded` | script | limited | attenuated | override_view | yaw | view | preserve |
| `remote_control` | script | **command_delta** | fixed | override_view | all | view | preserve |
| `align_on_entry(p)` | `p` + entry=align, source=tag (default) | | | | | | |

Guarantees worth copying: `locked`/`fixed_scope` give a camera **bit-identical** to the native one
(memcmp) while the hands still get a read-only tracking reference; HMD commands are suspended there.
`head_translation::fixed` keeps exact native translation (rope/prop shots).

### Priority and ownership

`camera_request_for` (shared by the command path and the render path): **remote_control >
fixed thermal optic (`fixed_scope`) > mounted weapon > scripted scene > ordinary gameplay**. Epoch
bits tag the source (remote `1<<62`, scope `1<<63`). Only the policy that owns rotation may write HMD
yaw into native commands (`scripted_epoch` != 0 makes `apply_game_view` return false). Translation never
grants or revokes weapon permission. Detect scenes through explicit native identity **and** phase (linked
entity, animation name, persistent flags, DObj/tag lifetime), never the map name or a transient flag; a
churning temporary VM wrapper id for the same camera must not look like a new shot (139 false "new shots"
in 7.9 s were observed), while a new generation/entity/time rewind/level change is a new lifetime.

### Lifecycle rules (each one came from a bug)

- **Rotation is composed with matrices, never by adding Euler angles.** Repeated samples cannot apply a
  turn twice.
- **Entry alignment** is consumed once per native entry lifetime (`aligned_entry_`): armed when requested,
  waits for a valid new anchor, re-armed when policy returns to preserve mode or when time rewinds
  (checkpoint). Yaw / yaw_roll cuts keep physical head tilt. Recenter, repeated sampling and head motion
  never re-consume it.
- **Tag may arrive after the server link:** the camera waits (`policy.needs_tag()`) without consuming
  alignment. A helper tag with a wrong roll (a turret aim helper at 119° vs the real 8.56°) must not seed
  or consume alignment; losing the tag mid-shot keeps the last basis; invalid sources hold the current
  heading; recovery never jumps.
- **Pole-crossing:** with pitch discarded (`yaw_roll`, `yaw` axes), crossing the vertical with the
  discarded pitch must not create yaw/roll (sweep 5..120° both signs). For an *authored camera bone*
  that can point through the pole, follow the heading of the **same axis** between consecutive poses
  (choose the axis whose horizontal projection is largest in both poses, `score=min(|a_xy|²,|b_xy|²)`,
  add only that axis' atan2 delta) so pure pitch cannot reverse heading (`scripted_view::follow_authored`).
- **`limited`** clamps in the entry frame using **unwrapped** angles (clamps never wrap at ±180), survives
  recenter (limit not released), and absurd limits still give a valid rotation.
- **Handback:** when control returns to the player, `follow_native` uses the **actual rendered
  heading** (yaw of `head × base`, including partial constraints); `restore_command` writes the continuous
  Euler yaw back into the command **once** so the player's command angle matches the rendered one; older
  prediction holds the corrected view until its rebased command is consumed.
- Time going backwards resets the rig; reference (recenter) change recomputes baselines instead of jumping.
- Checkpoint load: reset camera/command baselines and request recenter **once** (`load_recenter`).
- Mounted orbit follows every rendered turn even while native command time is unchanged.
- A fixed scope/screen display plane (`screen_scope_layout.hpp`): 2.0 m away, half-width `2·tan30°`
  (60° wide), anchored to the head orientation at epoch start, translation by the attenuated head offset
  with gain clamped to [0, 0.25]; projects its corners per eye using homogeneous w (no near rejection); never
  modifies weapon/view angles.

### `remote_control` / `command_delta` (UAV, guided missile)

Treat head motion as a **virtual mouse/stick**, not an absolute orientation:
- Entry keeps native aim; per frame the head delta (wrapped to ±180°) is added to native pitch/yaw
  (`remote_view::apply`), and the head contribution already delivered through the command is removed from
  the late camera correction exactly once (`correction = head · consumed^T`: render only layers the
  **unconsumed** head delta over native aim).
- Where the engine takes **int8 stick wire values**, encode the delta into them with an accumulator so
  quantisation error carries over: `pack(x)=floor(clamp(x,−1,1)·127+0.5)`,
  `command = pack(base + sign·remaining/step)`, `step = rate·elapsed_ms/1000`, rate in (0, 720] deg/s.
  Anything beyond one signed-byte step stays pending (not lost to ±127 saturation). Native
  `CL_RemoteControlMove` negates both non-inverted gamepad look axes (H2) — verify polarity per axis.
- Stick look (`remote_look`): fresh (≤150 ms), epoch set, radial deadzone in [0.05, 0.5] rescaled,
  `|stick| ≤ 1.01`, must be **armed** (stick first returns inside the deadzone after a scene starts),
  dt clamped to 50 ms.
- Recenter discards pending turns; exit restores native facing, not the missile's heading.

---

## 7. Body / equipment anchor estimator

Equipment anchors (chest, holsters, shoulders for arm IK) must not wobble with every head nod. Estimate
an **inferred body** from the head only (`body_pose.hpp`, also a `head_locked` mode = center-eye position +
horizontal heading, never pitch/roll; setting `vr_bodyPoseMode`). State lives in **recenter-relative
metres**, so locomotion, snap turns and native crouching move the world base outside the filter and add no lag.

| Constant | Value |
|---|---|
| neck = head − eye + eye_reference, eye offset along forward | 0.08 m |
| horizontal dead zone / follow | 0.07 m; `blend = 1 − exp(−dt/0.12)`; hard cap lag 0.15 m (`movement = max((d−0.07)·blend, d−0.15)`) |
| vertical (physical crouch) | follows with tau 0.08 s |
| body yaw | follows only beyond **±35°** head-vs-body, tau 0.18 s, max 180°/s |
| hard reset | non-finite or > 10000 input; non-orthonormal basis (dot error > 0.002) or determinant < 0.998; reference change; gap > 150 ms; time reversal; jump > 0.75 m |
| determinism | updated once per distinct tracking timestamp (duplicate eye/camera queries cannot advance it) |

Heading input is the *physical* heading (section 2), never Euler yaw. Test expectations: a 0.4 m lean -> 0.33 m
body offset, 5 cm ignored; a 30° glance around an 8 cm neck pivot does not drag equipment; yaw wrap
170 -> −170 gives no spurious turn. Equipment stays upright regardless of head pitch/roll.

Shoulders for arm IK (`gameplay/shoulder_anchors.hpp`): center = `head − (forward·back + up·down)`,
sides `± left·half_width`, using the yaw-only **physical** heading axes (gravity stays world-up). Defaults
half width **0.18 m**, down **0.20 m**, back **0.08 m** (dvars `vr_shoulderHalfWidth/Down/Back`, ranges
0.10–0.35 / 0.05–0.45 / −0.10–0.30). Hit volumes and rendering share one body frame (08).

---

## 8. Stabilization filters (optional comfort)

All three default **off** (`vr_headStabilization`, `vr_handStabilization`, `vr_desktopStabilization`);
strength 0–100 (defaults head 30, hand 40, desktop 50), 0 bypasses (exact bypass). One-pole filter with a
**speed-adaptive half-life** (`pose_filter.hpp`):

```
half  = half_life * (strength/100) / (1 + (v/v_ref)² + (ω/ω_ref)²)
alpha = 1 − 2^(−dt/half)               // position lerp and quaternion slerp use alpha
then clamp residual lag: |out−raw| ≤ position_limit, angle(out,raw) ≤ angle_limit
```

| Profile | half_life | pos lag cap | angle lag cap | v_ref | ω_ref |
|---|---|---|---|---|---|
| head | 15 ms | 2 mm | 0.25° | 0.05 m/s | 30°/s |
| hand | 45 ms | 15 mm | 2° | 0.25 m/s | 120°/s |
| desktop | 120 ms | 0 | 30° | 1 m/s | 180°/s |

Reseed (output = raw) on: invalid pose, first sample, epoch change, strength change, sequence going
backwards, time going backwards, gap > 150 ms. Same sequence id returns the cached result (idempotent for the
second eye). `stabilization::epoch` increments on gameplay-state change/invalidate so loads and teleports never
smear. Do not filter the HMD pose used by the compositor submission (Overlord keeps the runtime's original pose
for submission); the **head filter is an application-space root correction** `filtered · inverse(raw)` captured
into the spatial frame (`tracking_correction`) and applied identically to camera, `tracking_to_world` and
hands. Hands are filtered once at publication; the grip filter defines one rigid correction also applied to
aim (keeps their relative transform); derive wrist-offset basis from the raw grip/aim pair. Mechanical controls
(steering, climbing, hinge gestures) should read the raw pose; visuals the stabilized one. Head filtering can add
lag/discomfort: keep optional and bounded. These are tuning values, not comfort guarantees.

---

## 9. Things to disable or adapt in VR

- **Camera bob is a user toggle, default ON, scoped to specific call sites** (`vr_cameraBob`, saved,
  default true: "Enable native movement camera bob in VR, including empty hands and body knife").
  Make it optional rather than patching dvars (`camera_bob.cpp`, `camera_bob_bridge.hpp`, `docs/vr-camera-bob.md`):
  - **Seven** verified `E8 rel32` call sites of two native "bob factor" routines (horizontal/vertical) are
    redirected: camera vertical+horizontal, ADS angular vertical+horizontal, overlay-ADS vertical, horizontal,
    roll. Each relay tags a thread-local `camera_scope` and returns 0 when VR is local and the option is off
    (`sample_camera`). Skipping any one site made the feature half-work (the original two-site policy missed
    the ADS angular kernels).
  - **Two** 5-byte `MULSS xmm6/xmm7,[rax+0x10]` sites (weapon-def bob factor read) go through an
    asmjit-generated bridge that saves flags, rax/rcx/rdx/r8–r11, xmm0–5 and the upper lanes of xmm6/7. Native
    weaponless factors default to 0, so the policy returns 1.0 **only when the native scale is exactly 0 and
    the call is in camera scope**; explicit nonzero native tuning is preserved.
  - Use `create_preserving_near_jump` for the MULSS sites (the native dvar pointer is **live in RAX**; a
    `mov rax,imm; jmp rax` relay destroys it) and `create_far_jump<image_base>` for the plain call sites.
  - Local-player gate: cgame initialised && `local_role(ps)==1` && head bridge enabled. Other consumers of the
    same routines (other entities) keep native results; native dvars are never temporarily overwritten.
  - Only positional bob and the ADS angular terms are removed when off; idle sway, recoil and viewmodel bob stay
    native. Keep native recoil only on the weapon (08).
  - For a new port the safe comfort default is "off"; Overlord ships it as a user option defaulting on.
- View kick/shake/DOF/motion blur/vignette: remove or reduce (native view kick must not rotate the tracked head).
- Scripted FOV changes (zoom) must not change the HMD projection; optics render separately.
- Death/third-person cameras: fade or present through a policy (`aligned`).
- Paused/menu: camera continues at tracked pose; menus are quads (09).

---

## 10. Comfort controls

Snap turn (default 30°, 5–90) and smooth turn (default 90°/s, 15–360, 0.2 deadzone, dt cap 50 ms) operate on
the **native client yaw** (07); a discontinuous turn resets temporal history (05) and roomscale compensation
(section 5). Optional head/hand/desktop stabilization (section 8). Resume heading after headset removal (section 2).

---

## 11. Port recipe and acceptance tests

1. Find the final view boundary and the per-command angle packing; confirm the angle scale (16-bit vs 24-bit)
   and whether the engine adds a `delta_angles`-style correction.
2. Implement the bridge: validation, coordinate conversion, recenter (tracked-only, full-position or your
   own height policy), generation counters, 150 ms freshness.
3. Door A: yaw as **delta**, pitch absolute minus engine delta, at the last packing step; record per command
   time in a ring equal to the engine's command ring; undo the engine pitch clamp for exact tracked commands.
4. Door B: `native_yaw − command_head_yaw` as the base, compose matrix rotation + yaw-axis-rotated translation;
   fail the frame on a history miss.
5. Heading from the matrix everywhere except the command Euler pair; add the standby/resume replay test first.
6. Add room-scale turn compensation; then policies for scripted scenes; then the body estimator.

Offline checks to write (each encodes a real bug): recenter keeps physical pitch/roll on the *first* pose;
engine pitch/roll never leak into the camera; tracking->engine conversion sign test (recenter at (0, 1.6, 0) m, then
pose (0.1, 1.7, −0.2) m at scale 10 -> forward 2, left −1, up 1 units); 1000 repeated samples don't accumulate yaw; exact-time history rules;
old-prediction replay; pole sweeps; alternate Euler branch through every policy; entry alignment once per epoch;
checkpoint rewind; NaN rejection; `memcmp` bit-identical camera for `locked`. CPU/mock tests do not establish headset
acceptance; record that separately.
