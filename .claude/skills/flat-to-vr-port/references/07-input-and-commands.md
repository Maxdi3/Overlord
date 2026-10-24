# 07 — Controller input and command injection

Overlord files: `vr/openxr_input.cpp`, `steamvr_input.cpp`, `controller_input.cpp/.hpp`,
`analog_input.hpp`, `digital_button_gate.hpp`, `controller_calibration.cpp`,
`controller_pose_pipeline.hpp`, `controller_haptics.cpp`,
`gameplay/controller_component.cpp`, `controller_locomotion.hpp`,
`controller_buttons.hpp`, `controller_stance.hpp`, `controller_firing.hpp`,
`data/vr_input/*.json` (OpenVR manifests). Docs: `vr-controller-interaction.md`,
`vr-stance-controls.md`, `vr-trigger-discipline.md`, `vr-hand-alignment.md`.

## 1. Action set (OpenXR)

Global actions: `move` (vec2), `turn` (vec2), `sprint`, `jump`, `menu_toggle`,
`menu_recenter` (tap = pause/back, hold = recenter).
Per-hand actions (subaction paths `/user/hand/left|right`): `trigger` (float),
`squeeze` (float), `trigger_touch` (bool), `primary` (A/X), `secondary` (B/Y),
`grip` (pose), `aim` (pose), `haptic` (vibration).

Binding recipes per interaction profile:
| Profile | axis | squeeze | primary L/R | secondary L/R |
|---|---|---|---|---|
| `/interaction_profiles/oculus/touch_controller` | thumbstick | `squeeze/value` | x/a click | y/b click |
| `/interaction_profiles/valve/index_controller` | thumbstick | `squeeze/force` | a/a | b/b |
| `/interaction_profiles/htc/vive_controller` | trackpad | `squeeze/click` | — | — |

Each profile also has an **analog policy**: thresholds with hysteresis for value,
force (Index) and click (Vive), so "grip held" means the same thing everywhere. After
`xrSyncActions`, read `xrGetCurrentInteractionProfile` per hand. Profile changes
reset latches and bump `pose_reference_generation`.

## 2. The canonical frame (published once per prediction)

```cpp
struct frame {
  uint64_t sequence, reference_generation /*recenter*/, pose_reference_generation /*binding*/, continuity_generation;
  clock::time_point sampled_at; int64_t target_display_time; bool focused;
  digital_action sprint, jump, menu_toggle, menu_recenter;
  std::array<digital_action,2> trigger, trigger_touch, squeeze, primary, secondary;   // [0]=left [1]=right
  std::array<analog_value,2> trigger_analog, squeeze_analog;
  std::array<float,2> move, turn;
  std::array<hand_pose,2> grip, aim;               // calibrated + optional stabilization
  std::array<hand_pose,2> runtime_grip, runtime_aim, sdk_grip, sdk_aim; // diagnostics
};
struct digital_action { bool active, down; uint64_t presses, releases, generation; };
```
- **Edges are counters, not booleans.** A consumer remembers the last `presses`
  value it saw, so a short tap between two game ticks is never lost.
  `generation` bumps when input became inactive or stale for >150 ms, which
  prevents replay after focus loss.
- Poses carry validity *and* tracked flags. Precision interactions require
  position+orientation **valid and tracked** (`interaction_ready`).
- Focus loss neutralises all buttons and poses. Rearm requires neutral input.
- Hand side (left/right) is never collapsed into a global "fire" before the
  gameplay topology decides which hand holds what.

## 3. Grip calibration

`wrist = grip_position − scene_view_offset + grip_rotation · wrist_offset_native_units`,
`control_rotation = aim_rotation`. Inward is mirrored between hands. Tracking
convention: Y up, Z back. Two banks (legacy/standard) with presets; zero means
"use the runtime grip origin". Calibration changes set `orientation_settling`,
which fences velocity history so a slider change is not detected as a throw.

## 4. Injecting into the game's command

Hook the command builder at three points:
```
create_command_stub(out, client)          // wraps CL_CreateCmd
  reset stale policies if >150 ms since last command
  original → builds cmd (calls normal_input, finish_move internally)
  post: story/scripted buttons, sentry/vehicle fallbacks, auto-ADS → OR into result->buttons
normal_input_stub(cmd, frametime)         // after native keyboard/mouse movement
  gameplay = enabled && head pose fresh && no recenter pending && same reference generation
             && cgame initialized && no key catchers (menus) && not paused
  buttons: cmd->buttons |= mapped held bits (never clear native keyboard bits)
  jump/sprint presses: also call the native key-binding notify (+gostand, +sprint) so
                       script waiters (notifyoncommand) fire
  stance: run the native stance binding command; don't write pm_flags
  fire: trigger policy (trigger discipline, weapon readiness, mechanics) → attack bit;
        clear the attack bit when VR mechanics say "not ready"
  locomotion: forwardmove/rightmove += round(axis*127) clamp ±127, relative to the HEAD yaw
  turning: client yaw += delta (snap or smooth); discontinuous → reset temporal history
finish_move_stub(client, cmd)             // last step before packing angles
  write HMD view angles into client angles (06), then original
update_view_angles_stub(ps, cmd)          // in Pmove: restore the HMD pitch after the native clamp
```
**Never edit `CL_GetUserCmd`.** Prediction re-reads old commands, so edit
only before insertion into the ring.

Movement direction: locomotion is relative to the head yaw (or optionally the
off-hand controller). Since the view yaw equals the head yaw (06), native
`forwardmove` already works head-relative.

## 5. Menus

When a menu or key catcher is active, gameplay injection stops. The menu pointer
uses the `aim` pose ray against the menu layer, mapped to the game's cursor
plus click (09). Stick navigation is the secondary method.

## 6. Haptics

Semantic events (shot, magazine seated, slot hover) are queued in a bounded
queue, coalesced, then sent with `xrApplyHapticFeedback` on the Present-owner
thread. Optional: haptic failure never disables rendering.
