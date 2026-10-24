# 06 — Head tracking, camera composition and aim coupling

Overlord files: `vr/head_pose_bridge.cpp/.hpp`, `camera.cpp`,
`vr/camera_rig.hpp`, `scripted_camera.hpp`, `scripted_view.hpp`,
`scripted_position.hpp`, `roomscale_origin.hpp`, `game_view.hpp`,
`gameplay/controller_component.cpp` (FinishMove/UpdateViewAngles),
`gameplay/campaign/sequences/camera_policies.hpp`, `gameplay/scripted_camera_reference.cpp`.
Docs: `docs/vr-camera-architecture.md`, `vr-hmd-resume-heading.md`,
`vr-camera-bob.md`, `vr-stabilization.md`.

## 1. Publishing the tracked pose

The runtime calls `publish_tracking_pose({position_meters, orientation3x3}, sample_id, sampled_at, reference_policy)`
once per prediction (04). The bridge:
- converts tracking space → engine space (14),
- applies **recenter**: on request, the next *fully tracked* (position + orientation
  valid and tracked) sample becomes the origin. Recenter resets **yaw only**
  and preserves physical pitch/roll. An inferred pose can continue an established
  origin but cannot seed a new one,
- stores `local_orientation`, `local_position_units`, and `local_angles` (yaw/pitch/roll),
- bumps `recenter_count` (the reference generation everyone checks),
- marks the pose stale after **150 ms**. Every consumer checks freshness.

## 2. Two separate couplings: camera and aim

Keep these separate; mixing them causes double rotation or drift.

### a) Camera (what the eyes see)
At the **final view-setup boundary** (H2: hook the call at `0x1403ACB2D` to
`set_viewpos_now_stub`), after the engine computed `refdef.origin/axis` from
the player state:
```cpp
apply_camera(refdef.org, refdef.axis, cmd_time, native_angles, camera_request, rotation_ref, body_mode)
  base_yaw   = native yaw from command/angles (or extract yaw-only basis from axis)
  head_yaw_in_cmd = command-history lookup: yaw that this command already got from HMD
  composed   = scripted_camera.compose(original_axis, head_axis, head_meters,
                                       base_yaw, head_yaw_in_cmd, policy)
  origin    += roomscale turn compensation + base_yaw_axis · (head_offset_m * world_scale)
  axis       = composed.axis    // full HMD pitch/roll/yaw relative to body yaw
```
The eye offsets are applied later per eye (05), so this is the **center-eye** camera.

### b) Aim / gameplay view angles (where the player "looks" in game logic)
At **FinishMove** (the last step before the command is packed into history),
write HMD yaw/pitch into the client view angles (`apply_game_view`). Native
movement direction, sprint, interaction traces, AI awareness and scripted
"look at" checks then all agree with the head.
- Record per command time how much HMD yaw/pitch was included
  (`record_game_command`), so the camera never adds it twice and
  prediction replays resolve the same values (`resolve_game_pitch`).
- The engine clamps pitch (±85°). In `PM_UpdateViewAngles`, after the original
  runs, restore the exact HMD pitch for tracked commands only
  (`update_view_angles_stub`).
- Snap/smooth turning modifies the **native client yaw**, not the HMD reference.

**Weapon aim is separate again**: weapons fire from the controller muzzle (08).
The view angles are just the head.

## 3. Room-scale translation

Walking physically moves the camera by head offset × world scale. Native
collision/capsule stays where the player entity is. Overlord turns about the
current tracked head (not the recenter point) by compensating a shared
world-origin offset when native yaw changes (`roomscale_origin.hpp`). Hands
and body equipment use that same origin. A bounded command-time history stops
prediction replays from integrating a turn twice.

(Optional, not in Overlord: push the player capsule toward the head when the head
leaves the capsule, so walls stop you physically leaning through.)

## 4. Camera policy system for scripted scenes

A scene selects a policy. Composition is generic:

| Dimension | Values |
|---|---|
| head rotation | `fixed`, `pitch_roll`, `limited`, `free` |
| head translation | `fixed`, `attenuated` (scaled, spherical limit), `tracked` |
| script rotation | `override_view`, `additive` (relative native change), `ignore` |
| axes | `yaw`, `yaw_roll`, `full` |
| source | final native view or a named tag (`tag_player`, `tag_origin`) |
| entry | preserve heading or align once on entry |

Priority (Overlord): fixed thermal optic > mounted weapon > scripted scene >
ordinary gameplay. The same `camera_request_for` is used by both the command
path and the render path. **Only the policy that owns rotation may write HMD yaw
into native commands.** Detect scenes through explicit native identity and phase
(linked entity, animation name, script flags), not just the map name.

Rules:
- Compose rotations with matrices, never by adding Euler angles. Track branch continuity
  so a native pitch crossing vertical doesn't flip yaw by 180°.
- Invalid sources hold the current heading. Recovery must not jump.
- Checkpoint load: reset camera/command baselines and request recenter once.

## 5. Things that must be disabled or adapted in VR

- Camera bob, view kick, weapon sway, view shake → off or reduced
  (`camera_bob.cpp` hooks four sites). Keep native recoil only on the *weapon*.
- Scripted FOV changes (zoom) must not change the HMD projection. Optics are rendered
  separately (`docs/vr-native-ads.md`).
- Death/third-person cameras: fade, or present with a policy.

## 6. Comfort controls

- Snap turn (default 30–45°) and smooth turn (deg/s), each with a deadzone.
  A discontinuous turn resets temporal history (05).
- Optional head/hand/desktop stabilization: bounded one-pole filters whose
  half-life shrinks with speed (`pose_filter.hpp`, `stabilization.cpp`).
- Resume heading after headset removal (`vr-hmd-resume-heading.md`).
