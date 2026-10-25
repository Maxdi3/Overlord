# 07 — Controller input and command injection

Goal: turn VR controller actions into the flat game's own per-tick command (move
axes, button bits, view angles) and into script-visible events, without ever
becoming a second authority over game state. Two halves:

1. **Sampling** (sections 1-5): runtime actions are sampled once per frame into an
   immutable *frame* with edge counters and generation counters.
2. **Injection** (sections 6-10): consumers read that frame at the game's command
   boundary, gate it, and write only additive/request-style changes.

Overlord files (all under `src/client/component/vr/` unless noted):
`openxr_input.cpp` (OpenXR actions), `steamvr_input.cpp` + `data/vr_input/*.json`
(OpenVR), `controller_input.cpp/.hpp` (frame), `analog_input.hpp`,
`digital_button_gate.hpp`, `controller_calibration.*`, `controller_pose_pipeline.hpp`,
`controller_haptics.*`, `controller_profile_retry.hpp`, `input_observation.hpp`,
`gameplay/controller_component.cpp` (command hooks), `gameplay/controller_locomotion.hpp`,
`controller_buttons.hpp`, `controller_stance.hpp`, `controller_firing.hpp`,
`controller_ads.hpp`, `menu_surface.hpp`, `native_menu.cpp`,
`src/client/component/input.cpp` + `native_ui_pointer.hpp` (desktop mouse/keyboard),
`src/client/component/movement.cpp` (script-declared PMove profile).
Docs: `docs/vr-controller-interaction.md`, `vr-stance-controls.md`,
`vr-trigger-discipline.md`, `vr-hand-alignment.md`, `vr-breach-control.md`,
`pmove-scripting.md`, `vr-runtime-rendering.md` (pose pipelines).

## 0. The rules, in one screen

1. **Sample once, publish an immutable frame.** Gameplay never polls the XR runtime.
2. **Edges are counters** (`presses`, `releases`), never booleans. A tap between two
   game ticks must not be lost; a held button at resume must not fire.
3. **Two kinds of generation**: per-button `generation` (input became active/stale)
   and frame-level `reference_generation` (recenter/level) + `continuity_generation`
   (tracking/producer discontinuity). Consumers re-arm on any change.
4. **Held-at-entry/resume needs a neutral sample plus a fresh press.** Applies to
   attach, menu return, focus loss, recenter, stale gap, future timestamp, sequence
   rollback, profile change, weapon switch, checkpoint, mode switch.
5. **Duplicate frame (same `sequence`) is a no-op**: the engine renders twice (stereo),
   predicts, and replays; every time-integrating consumer must be a pure function of
   (input sequence, generation).
6. **Tracking loss is not a release.** A real release during pose loss is deferred
   and delivered once when geometry returns; recovery never replays acquisition.
7. **OR VR bits into native buttons, never clear native bits** (only exception: veto
   your own fire request). Add stick axes to native axes, clamped.
8. **Edit the command before it enters history.** Never mutate on read.
9. **Request, don't force**: ADS/stance/sprint/jump are requests; the native game
   (PM code, collision, scripts) keeps final authority.
10. **A failed or refused native write consumes the edge** (no retry from the same press).

## 1. OpenXR action layer

### 1.1 Action set

One action set `gameplay`, attached once. **No subaction paths**: every per-hand
action is a separate action (`left_trigger`, `right_trigger`, ...) bound with an
explicit user path, and every query uses `XR_NULL_PATH`. (OpenVR works the same
way, see section 2.) That keeps left/right physical roles explicit; weapon ownership
and dominant hand are decided in gameplay, never in the binding.

| Action | Type | Notes |
|---|---|---|
| `move`, `turn` | vector2 | left axis = move, right axis = turn |
| `sprint`, `jump` | boolean | **stick click**: left click = sprint, right click = jump |
| `menu_toggle` | boolean | explicit app-menu button; never the system/dashboard button |
| `menu_recenter` | boolean | tap = pause/back, hold 1 s = recenter |
| `{left,right}_trigger`, `_squeeze` | float | bound to `trigger/value` and a per-profile squeeze path |
| `{left,right}_trigger_touch` | boolean | capacitive contact, independent of trigger click |
| `{left,right}_primary`, `_secondary` | boolean | A/X and B/Y class face buttons |
| `{left,right}_grip`, `_aim` | pose | `grip/pose` and `aim/pose`; identity pose-in-action-space |
| `{left,right}_haptic` | vibration | `output/haptic` |

Sequence per frame (Present-owner thread, at the existing post-Present tracking
boundary): `xrSyncActions` -> profile refresh (retry-limited) -> read channels ->
`controller_input::publish(frame)` -> consume haptic mailbox -> `xrApplyHapticFeedback`.

### 1.2 Per-profile binding table (corrected)

Left face-primary is **never** a gameplay primary: it is the dedicated tap/hold menu
control. The left-hand `primary` frame field therefore stays inactive on every
profile and backend (features keyed to "the rear hand's primary" are right-hand-only;
check this before adding left-handed support).

| Function | Touch (`oculus/touch_controller`) | Index/Knuckles (`valve/index_controller`) | Vive (`htc/vive_controller`) |
|---|---|---|---|
| move / turn axis | thumbstick | thumbstick | trackpad |
| sprint / jump | axis click L / R | axis click L / R | axis click L / R |
| trigger (OpenXR) | `trigger/value` | `trigger/value` | `trigger/value` |
| squeeze (OpenXR) | `squeeze/value` | `squeeze/force` | `squeeze/click` |
| trigger_touch | `trigger/touch` | `trigger/touch` | **none** |
| primary L / R | **L: unbound** (X -> menu_recenter) / A | **L: unbound** (A -> menu_recenter) / A | **none** |
| secondary L / R | Y / B | B / B | **none** |
| menu_recenter | **left X** | **left A** | **none** |
| menu_toggle | left menu button | **none** | left menu button |

Notes: Knuckles has no `menu_toggle`, so its left A is both "tap = pause" and "hold =
recenter". Vive has no recenter, no face buttons and no trigger touch: re-home the
function or accept feature loss, and write the loss into the binding description.
`trigger_touch` and `secondary` are only bound where the recipe has face buttons.
Two `menu_*` actions exist so the system button is never consumed and a simultaneous
HUD action is never emitted.

Overlord: `recipes[]` in `openxr_input.cpp`; the menu tap/hold gesture is
`menu_surface.hpp::menu_button` (section 8).

### 1.3 Analog actions -> digital (hysteresis policy)

Float actions are turned into pressed/not-pressed with a latch whose thresholds
depend on the **suggested binding source**, not the hardware identity ("the source
describes the suggested binding"):

| Policy | Press | Release | Used for |
|---|---|---|---|
| `value_button` | 0.55 | 0.45 | Touch trigger/squeeze, Vive trigger; also the default when the profile is unknown |
| `index_trigger` | 0.01 | 0.005 | Index trigger (near-zero "any pull" gate; real threshold lives in the game-side trigger discipline) |
| `index_squeeze` (force) | 0.15 | 0.05 | Index grip |
| `click_button` | 0.5 | 0.5 | Vive squeeze click (stateless: `value >= press`) |

Latch rules (`analog_input.hpp`): state drops to "up" when the input is inactive, the
policy changed, the clock went backwards, or more than **150 ms** passed between
samples. Valid policy requires finite values and `0 <= release <= press <= 1`. The raw
float is also published (`trigger_analog[h]`, `squeeze_analog[h]`) with its source tag.

Profile identification: after `xrSyncActions`, `xrGetCurrentInteractionProfile` is
queried **per hand** through a bounded retry (`controller_profile_retry.hpp`: 4
attempts, 250 ms apart, stops on first non-null profile; re-armed by a profile-changed
event or session init). A *newly identified* policy resets that hand's latch and
samples an inactive state, so the next active sample bumps `generation`: "changing
thresholds is not a physical press". A profile-changed event additionally invalidates
the whole frame, clears haptics and bumps `pose_reference_generation`.

### 1.4 Pose reading

Grip and aim are action spaces located with `xrLocateSpace(space, base, predicted
display time)`; the same predicted time as the rendered frame. Required for `valid`:
position-valid **and** orientation-valid bits, quaternion norm within 0.01 of 1 and
finite pose. The *tracked* bits are stored separately in `pose_quality`
`{position_valid, orientation_valid, position_tracked, orientation_tracked, known}`.
Per-channel `input_condition{reason, code, handle}` (29-value `input_reason` enum:
`action_inactive`, `pose_invalid`, `profile_mismatch`, `calibration_invalid`,
`pose_filter_rejected`, ...) records *why* a channel is missing; it is diagnostic only
and never grants gameplay admission.

Unfocused (`XR_SESSION_NOT_FOCUSED` or failed sync): every sampler is fed an
inactive sample, no runtime queries are made, the frame is still published (so
consumers see `focused=false` and neutral buttons).

## 2. OpenVR / SteamVR Input manifest (backup backend)

SteamVR input is data-driven; the code only asks for handles by name.

### 2.1 Layout

`vr_input/actions.json` + one binding file per `controller_type` listed in
`default_bindings` (`oculus_touch`, `knuckles`, `vive_controller`), referenced by a
relative `binding_url`. Ship the directory next to the executable and deploy it
**together with the binaries** (a binary-only deploy once left a stale manifest
without the trigger-touch actions; the pair deployer now stages, backs up and rolls
back the input package atomically). Packaging validates that every `binding_url`
exists and is neither absolute nor contains `..`.

Loader (`steamvr_input.cpp`): manifest path = `<exe dir>/vr_input/actions.json`
(**absolute path required**) -> `SetActionManifestPath` -> `GetActionSetHandle
("/actions/gameplay")` -> `GetActionHandle` per action. `UpdateActionState` only when
focused.

```jsonc
// actions.json (shape)
{ "action_sets":[{"name":"/actions/gameplay","usage":"leftright"}],
  "actions":[{"name":"/actions/gameplay/in/left_trigger","type":"boolean","requirement":"suggested"}, ...],
  "default_bindings":[{"controller_type":"knuckles","binding_url":"knuckles.json"}, ...],
  "localization":[{"language_tag":"en_US","/actions/gameplay":"<set label>","/actions/gameplay/in/jump":"Jump (right stick click)", ...}] }
```

- Names are fully qualified `/actions/<set>/in|out/<name>`; vibration is `out`.
- Types used: `boolean` (**all** buttons incl. trigger and grip/squeeze), `vector2`
  (move, turn), `pose` (grip, aim), `vibration`. **No float actions**: analog->digital
  is done by the binding, so the OpenVR path never sees raw trigger/grip values and
  `trigger_analog/squeeze_analog` stay unavailable (no second latch is applied).
- No subaction paths: every per-hand action is duplicated (`left_*`/`right_*`).
- `requirement` (`suggested`/`optional`) only affects SteamVR's binding UI, **not**
  `GetActionHandle`. The code has its own split: core handles (move, turn, sprint,
  jump, triggers, squeezes, primary/secondary, grip, aim) are **required** (any failure
  aborts input init); `menu_*`, `trigger_touch`, haptics are **optional** (failure
  zeroes the handle; haptics only bound if handle != 0).
- The `localization` block is mandatory for the binding UI: one string per action
  **and** one for the action-set path. Put semantics in the descriptions
  ("tap = pause, hold = recenter", "fire when holding rear grip").
- Pose actions: grip bound to `/user/hand/<side>/pose/raw`, aim to `/pose/tip`
  (identical in all three profiles). Poses are read with
  `GetPoseActionDataForNextFrame(TrackingUniverseStanding)` (predicted); quality maps
  from `eTrackingResult`: `Running_OK` = fully tracked, `Fallback_RotationOnly`
  = orientation-only.

### 2.2 Binding file vocabulary (what is in the shipped files)

| Concept | Syntax |
|---|---|
| Source modes | `button`, `trigger`, `joystick` (Touch path `input/joystick`; Knuckles path `input/thumbstick`), `trackpad` (Vive) |
| Inputs per mode | `click`, `touch`, `position` |
| Two logical inputs from one physical path | two `sources` entries on the same path in different modes, e.g. trigger as `mode:"trigger"` -> `click` (fire) plus a second `mode:"button"` -> `touch` (contact action) |
| Analog->digital thresholds (Knuckles only) | parameters on a `button` source: grip `force_input:"force"`, `click_activate_threshold:"0.08"`, `click_deactivate_threshold:"0.02"`; trigger `force_input:"value"`, thresholds `"0.01"/"0.005"` |
| Suppress SteamVR's own click haptics | `"haptic_amplitude":"0"` on button sources (Knuckles grip/trigger/trigger-touch; Touch trigger-touch) so only mod haptics play |
| Haptics | `"haptics":[{"output":"/actions/gameplay/out/left_haptic","path":"/user/hand/left/output/haptic"}, ...]` |

Touch and Vive use default click thresholds. **Do not** map trigger contact with
`force_input:"touch"` + logical `click`: a live comparison showed it stays false at
zero travel, while a separate `button`-mode source routing the real `touch` output is
true. A binding contract test enforces this (`test_trigger_touch_binding.py`,
`test_knuckles_binding.py`: thresholds `0 < deactivate < activate <= 0.01` for trigger,
`0 < deactivate < activate < 1` for grip, one source per action, output types match
declared action types).

### 2.3 OpenVR vs OpenXR differences that matter

| | OpenXR | OpenVR |
|---|---|---|
| trigger/squeeze | float + code latch (1.3) | boolean from binding thresholds; Knuckles grip **.08/.02**, trigger **.01/.005** |
| Index grip threshold | .15/.05 (force) | .08/.02 (force) |
| Profile detection | `xrGetCurrentInteractionProfile` per hand | `Prop_ControllerType_String == "knuckles"` on both role-assigned devices, retried with the same 4x250 ms helper; used only for prompt glyph choice |
| Bindings | suggested in code (`recipes[]`) | user-rebindable JSON; SteamVR keeps custom bindings |
| Pose basis | SDK grip/aim | binding-dependent (`raw`/`tip`); the *standard* pipeline normalizes it (section 5) |

The product-facing text of the binding differs by profile (e.g. "left X" vs "left A"
for recenter), so the prompt layer picks glyph text by `knuckles` identity (`hud_controller`).

## 3. The canonical frame

Published once per sampling under a mutex at the post-Present tracking boundary
(`controller_input::publish`); consumers read copies (`latest()` for visuals,
`latest_interaction()` for precision admission).

```cpp
struct digital_action { bool active, down; uint64_t presses, generation, releases; };
struct frame {
  pose_pipeline;                                   // legacy|standard, frozen per process
  uint64_t sequence;                               // ++ per sample; 0 = never sampled/invalidated
  uint64_t reference_generation;                   // recenter/level (= head recenter_count)
  uint64_t pose_reference_generation;              // binding/model change, NOT physical motion
  uint64_t continuity_generation;                  // producer discontinuity (see below)
  time_point sampled_at;  int64 target_display_time; // XrTime; 0 when the backend cannot say
  bool focused, move_active, turn_active;
  digital_action sprint, jump, menu_toggle, menu_recenter;
  array<digital_action,2> trigger, trigger_touch, squeeze, primary, secondary;   // [0]=left [1]=right
  array<analog_value,2>   trigger_analog, squeeze_analog;                        // OpenXR only
  array<float,2> move, turn;                       // clamped to [-1,1], finite
  array<hand_pose,2> grip, aim;                    // calibrated (+ optional stabilization)
  array<hand_pose,2> runtime_grip, runtime_aim;    // adapter basis, unfiltered
  array<hand_pose,2> sdk_grip, sdk_aim;            // before binding/reference adaptation (diagnostics)
  array<float,3> orientation_degrees, position_offsets_meters;   // active calibration
  bool orientation_settling;                       // calibration just changed: fence motion history
  input_observation source;                        // per-channel reasons; never grants admission
};
struct hand_pose { bool valid; tracking_pose tracking; pose_quality quality; };
```

- **Hand identity is physical.** The frame never collapses left/right into a global
  "fire"/"grab"; the gameplay topology decides which hand holds what.
- `interaction_ready(pose)` = valid && quality known && position+orientation valid &&
  position+orientation **tracked**. `latest_interaction()` sets `valid=interaction_ready`
  on grip/aim/runtime_* so precision interactions cannot consume *inferred* poses while
  the camera/menus still can. Buttons, counters and SDK witnesses are untouched.
- Pose validity and tracking quality are separate; `unknown` quality never authorizes
  precision interaction. Keep pre-adaptation (`sdk_*`) and adapted (`runtime_*`) poses.

### 3.1 Edge counters (`digital_sampler`)

```
sample(active, down, now):
  if active && (!was_active || now < last || now-last > 150ms): ++generation; down=false
  if active && down && !was_down: ++presses
  if active && !down && was_down: ++releases
```
- A consumer remembers the last `presses` it saw; a tap that starts and ends between two
  command reads still produces exactly one press. Unread taps coalesce (no catch-up queue).
- **A button held at resume increments `presses` with a new `generation`.** Only the
  generation change plus neutral-arming (section 4) stops that from firing. Never treat
  `presses` alone as "the user pressed".
- Inactive/unfocused/failed samples feed `(false,false)` so the *next* active sample bumps
  generation.

### 3.2 Producer continuity

`producer_discontinuity(prev, next)` is true on: pose pipeline change, `pose_reference_
generation` change, sequence regress, `reference_generation` change, focus change,
`orientation_settling` change, time regress, gap > 150 ms, or any pose validity/quality
change (including VALID -> merely inferred). On discontinuity `publish` bumps
`continuity_generation` and invalidates stabilization. Filter amount/epoch changes also
bump continuity (consumers drop velocity/contact history **without** touching buttons or
leases; a filter toggle is not a hand impulse).

`consumer_continuity::update(frame, now)`: changed if time went backwards, or (when the
frame carries a generation) the generation differs, else (legacy frame) a > 150 ms gap.
This is what lets a **slow simulation tick** (bullet-time consumers running less often
than tracking) keep a valid held input: a long consumer gap alone is not a reset when the
producer generation is unchanged. Input freshness and geometry freshness keep *separate*
limits; unannotated gaps stay conservative.

### 3.3 Calibration/stabilization inside `publish`

Apply calibration (section 5) -> detect discontinuity -> optional pose filter. Hand filter
amount is clamped 0..100 and forced to **0 outside gameplay or while settling** (exact
bypass at 0). Filtering is applied to the grip as one rigid transform and the aim is
recomposed through the same correction. Non-finite/non-orthonormal pose with filtering on
marks both of that hand's poses invalid (`pose_filter_rejected`).

### 3.4 Per-subsystem time constants (do not use one global number)

| Constant | Value | Where |
|---|---|---|
| Input sample age accepted by command consumers, menu, stance, ADS, fire, locomotion | **150 ms** (151 rejected) | everywhere `now - sampled_at` |
| Command gap that resets all command-layer policies | 150 ms | `create_command_stub` |
| Digital sampler / analog latch gap | 150 ms | sampler, latch |
| Haptic pending pulse expiry | **100 ms** | `controller_haptics.hpp` |
| Calibration settle window | **200 ms** | `controller_calibration.hpp` |
| Turn integration dt cap | **50 ms** | locomotion, remote look |
| Stance continuation / abandon | 450 ms / 1500 ms | `controller_stance.hpp` |
| Menu hold-to-recenter | 1 s | `menu_surface.hpp` |
| Menu stick repeat | 350 ms first, 100 ms after | `menu_surface.hpp` |
| ADS dwell in/out | 100 ms (40 ms thermal) | `controller_ads.hpp` |
| Script-polled tap visibility | >= 100 ms (script polls at 50 ms) | section 7 |
| Press-then-reach grab intent | 200 ms | `body_equipment.hpp` |
| Gesture detectors | slap 150, magazine-latch sweep 150, cover push 100, melee 120 ms | docs/vr-breach-control.md |

## 4. Arming and consumer gates

Gate classes (`digital_button_gate.hpp`), all **disarmed on generation change, inactivity
or counter regression, and re-armed only by an active-and-neutral sample**:

```cpp
struct digital_press_gate  { // one deliberate edge
  bool consume(a){ if(!a.active||a.generation!=gen||a.presses<presses) armed=false;
    pressed=a.presses!=presses; gen=a.generation; presses=a.presses;
    if(!armed){ armed=a.active&&!a.down; return false; } return pressed; } };
struct digital_button_gate { /* same, but returns a.down || pressed (level OR short tap) */ };
struct paired_button_gate  { /* either hand: {held, pressed} */ };
```
`digital_button_gate` returns `down || pressed` so a tap completed between samples still
yields one held pulse. Use the press gate for one-shot actions, the button gate for
auto-fire/sprint/jump, the paired gate when either hand may act.

### 4.1 Universal arming/invalidation list (every digital consumer)

Held-at-entry is **not** a fresh edge for: first sample after attach, menu -> gameplay
return, focus loss, tracking/controller loss then reconnect, recenter
(`reference_generation++`), stale (> 150 ms) recovery, future timestamp, sequence
rollback, binding/profile change, weapon switch/owner transfer, checkpoint replacement,
tracking continuity change, smooth<->snap turn-mode switch, calibration settling, a
pause > 1 s (needs another release). In every case the consumer must see a release (or a
neutral frame) and then a fresh press.

### 4.2 Consumer checklist for any new gesture

- [ ] Reject stale (`>150 ms`), future, unfocused, `sequence==0`, non-finite input.
- [ ] Reset on `reference_generation`, `continuity_generation`, button `generation`,
      owner/instance/epoch change, `orientation_settling`, sequence rollback.
- [ ] Same `sequence` twice: return the cached result, integrate nothing.
- [ ] Never trust `presses` without a generation check; arm on neutral.
- [ ] A missing sensor on one hand never cancels or poisons the other.
- [ ] Wide gaps (>= 1 s) never create catch-up travel; run at 45/72/90/120/144 Hz and at
      slow consumer cadences (200/250 ms).
- [ ] A rejected native write consumes the edge; require a fresh gesture.
- [ ] Physical gestures are evaluated in a **body-relative** frame so locomotion and snap
      turn neither create nor invalidate swings.

### 4.3 Tracking loss is not a release (deferred release)

Overlord's `grip_edge_gate` (`gameplay/grip_edges.hpp`): button counters survive a
missing pose, so a proven release is held as `pending` until the hand has geometry, then
delivered once (`released`); while waiting it reports `deferred`. A new press cancels an
older deferred release; recenter/focus loss cancel it; the drop receives zero velocity
impulse. "Do not acquire on resume": a held button can still authorize its *next* real
release. Throw velocity requires continuous samples; nine conditions zero it (reference
change, pose-reference change, continuity change, pipeline-mode change, focus loss,
calibration settling, 200 ms gap, position-quality loss, invalid pose). `neutral_grips()`
(active && !down) lets "return to body" releases settle even if an edge was lost.

## 5. Pose pipelines and grip calibration

**Wrist model** (`docs/vr-hand-alignment.md`): the controller's grip origin is not the
wrist.
```
wrist = grip_position - scene_view_offset + grip_rotation * (offset_m * units_per_meter)
control_rotation = aim_rotation
```
Position offsets (`inward/back/up`) are a grip-local wrist point (inward mirrored per
hand; tracking axes: Y up, Z back); angle calibration (`pitch/yaw/roll`) is applied to the
**aim** pose only, in controller-local axes. A controller rotated about a stationary wrist
must leave the solved wrist fixed; aim never translates the wrist.

`controller_calibration::calibration::apply`:
- Rotation order **yaw * pitch * roll**, roll sign negated, right-multiplied onto each
  aim orientation (`orientation * local`). Tracking axes +X right, +Y up, -Z forward.
- Angles within +-180 deg, position offsets within **+-0.5 m** (`max_hand_offset`), all
  finite, else the aim poses are invalidated. Zero angles return the exact input (no
  matrix round-off). Cannot invent tracking for a missing hand.
- Any change of angles or position starts a **200 ms** `orientation_settling` window
  (199 ms still settling, 200 ms resolved): a slider move must not read as a swing or
  throw. It hides nothing and releases no lease; consumers fence velocity history only.
- `runtime_grip/runtime_aim` keep the pre-calibration poses; `grip` keeps the
  calibrated/filtered one.

**Two banks and a frozen pipeline mode.** `legacy` (**default**,
`vr_controllerPoseMode` default index = legacy) and `standard` have separate saved
calibration banks (`vr_handOffset*/vr_handAngle*` vs `vr_standardHandOffset*/
vr_standardHandAngle*`); both default to zero ("None"), and the same numbers are not
interchangeable. The mode is read at launcher startup and frozen with `std::call_once`
before the game loads: "saving affects the next launch; `vr_reinit` and console edits
cannot move held objects". A provider function pointer lets standalone tests use the
shared defaults without game dvars.

- *Standard*: OpenXR publishes SDK grip/aim directly; OpenVR normalizes the bound
  `raw`/`tip` pose into the render model's static `openxr_grip`/`openxr_aim` component
  frames (binding queried once, exactly one binding, correct hand, rigid static component;
  bounded cache, retry <= 1/s; failure = `pose_reference_unavailable`, never a guess).
  A binding event bumps `pose_reference_generation` so history is fenced.
- *Legacy*: keeps older Touch-tuned wrist numbers valid by converting each runtime into a
  canonical Touch grip frame (e.g. VirtualDesktopXR under the Touch profile: 20.6 deg pitch
  about X, translation (+-0.007, -0.00182941, 0.1019482) m; a profile mismatch invalidates
  that hand with `profile_mismatch`). General lesson: define one canonical grip frame and
  convert each runtime into it so user offsets are comparable.
- Do not "auto-calibrate" an anatomical wrist: a palm estimate from simulated controller
  joints was rejected in headset; fitted pivots are not ground truth. Default to zero
  offsets and let users tune "rotate about the physical wrist, the virtual wrist stays".

## 6. Injecting into the game's command

### 6.1 Hook shape

One detour on the engine's per-tick command constructor plus **call-site rewrites** of
three of its sub-steps (call sites, not function entries, so only the command path is
intercepted):

```
create_command_stub(out, client)            // detour on CreateCmd
  set gameplay-active flag (publishes context change to the input layer)
  if now - last_command > 150 ms: reset ALL command-layer policies
  thread-local flags: building_client, normal_input_seen, remote_input_seen, tracked_view_applied
  original -> builds cmd (calls normal_input, finish_move)
  post (still before the command is stored in the ring):
     vehicle/scripted branch fallback (they skip NormalInput): fill the same axes once
     story-scene buttons, sentry-placement attack, auto-ADS request bit,
     special-equipment command bits, record_game_command(serverTime, packedPitch, applied)
  if normal_input never ran: re-arm everything (buttons/firing/locomotion reset)
normal_input_stub(cmd, frametime)           // call site; after native keyboard/mouse movement
finish_move_stub(client, cmd)               // call site; write HMD view angles THEN run original
update_view_angles_stub(ps, cmd)            // call site inside Pmove; restore HMD pitch after native clamp
```

- **All-or-nothing install**: every hook site, byte window and the engine's command
  whitelist strings are verified with masked bytes first; any mismatch logs
  "command hook signature mismatch" and installs *nothing*. Optional features (auto-ADS,
  stance, remote-control axes) have their own byte checks and disable only themselves.
- **Never edit the command-history reader** (`CL_GetUserCmd`): prediction re-reads old
  commands. Edit only before insertion into the ring.
- Thread-local phase flags tell later stubs which sub-step actually ran.

### 6.2 Gating (every command consumer)

`gameplay = vr_controllers dvar && head enabled && head pose available && !recenter_pending
&& head.recenter_count == input.reference_generation && cgame initialized && no key
catchers && cl_paused == 0`, plus per-feature gates: story epoch (`allow_movement`,
`allow_turn`, `suspend_weapons`), climbing/ladder ownership, mounted turret / fixed sniper /
remote-control owner. Add the frame checks of 4.2 (fresh <= 150 ms, focused, sequence).

### 6.3 Rules for what you write

1. `cmd->buttons |= mask`; never clear a keyboard/native bit. Only exception: clear the
   attack bit when VR mechanics say "not ready" (mechanical veto, independent-fire owns the
   native shot, device not ready).
2. Axes are **added** to the keyboard axes: `clamp(int(signed char(orig)) + lround(axis*127), -127, 127)`.
3. **Do not rotate the stick by head yaw.** HMD yaw is already native view input
   (written at FinishMove), so the engine's own forward/right is already head-relative;
   rotating again applies the yaw twice and disagrees with keyboard movement.
4. Turning writes the **client yaw global** (`remainder(yaw + delta, 360)`), exactly like
   mouse turn, not the usercmd. Smooth turn is ordinary continuous camera motion and must
   not reset temporal-effect history each command; snap turn is discontinuous and requests
   a temporal-history reset.
5. View angles: `finish_move_stub` writes the game view at the final packing boundary
   (native CreateCmd pitch clamp and other input sources run after `normal_input`), independent
   of controller availability. `update_view_angles_stub` calls the original and then, for the
   exact HMD-tagged command only, restores `delta_angles[0]` and `viewangles[0]` so the ordinary
   pitch clamp is undone for that command alone (Pmove still owns prone/mounted/script
   constraints). `record_game_command(serverTime, packed angle, applied)` stores each command
   so prediction can later resolve the identical pitch (`resolve_game_pitch`). See ref 06.
6. State changes go through the engine's own command bindings (stance) or request bits
   (ADS); never write `pm_flags`, never synthesize keyboard state, never call the script VM
   from the command thread.

### 6.4 Locomotion and turning (`controller_locomotion.hpp`)

| Item | Value |
|---|---|
| Move deadzone | 0.2 (range 0.05..0.5), radial rescale `(min(m,1)-dz)/(1-dz)/m` |
| Smooth turn speed | **90 deg/s** at full deflection (range 15..360), turn deadzone **0.2** (0.05..0.5) with the same rescale |
| Smooth turn integration | **by timestamp, not publication**: `dt = clamp(now - last_consumed, 0, 50 ms)`; first armed frame dt = 0. Exactly 90 deg/s at 30/72/90/120/144 Hz; a 100 ms gap at full deflection yields 4.5 deg, not 9 |
| Turn sign | right deflection = negative yaw delta (the engine's yaw increases left) |
| Snap turn | default **30 deg** (range 5..90); fires when `|x| >= 0.7` and `|x| > |y|`; latch holds until `|x| < 0.25` (hysteresis); crossing straight to the opposite side does not re-arm |
| Frame validity | focused, both `move_active` **and** `turn_active`, `sequence != 0`, sample not in the future and <= 150 ms old, finite axes within 1.01 |

Neutral arming: after any reset (focus loss, menu, reference generation change, mode
change, sequence going backwards, > 150 ms consumer gap) movement **and** turning stay
disabled until both sticks are within their deadzones. Switching smooth<->snap while held
requires neutral. `vehicle_locomotion` adds both sticks (no view-turn channel when
driving) before the shared deadzone/normalization, and states that "opposite held sticks
cancel numerically but are not neutral". A held-forward "entry exception" lets boarding
cross once.

### 6.5 Stance gesture (`controller_stance.hpp`)

The right stick's vertical axis (the turn vector's Y; no new binding needed) requests
stand/crouch/prone **through the engine's own stance bindings**, never by writing movement
flags.

| Rule | Value |
|---|---|
| Arm | `|y| <= 0.30` (only the vertical axis; horizontal turning does not block) |
| First step | immediate at `|y| >= 0.90` (no dwell, diagonals allowed); stick down = lower posture |
| Second step | only if still held same direction, `|y| >= 0.75`, native posture == the expected posture (the first step was accepted), and >= **450 ms** after the first; abandoned after **1500 ms** unconfirmed; <= 2 requests per stroke |
| Stand + jump | stick click while crouched/prone requests stand and suppresses the VR jump bit for the entire hold of that click |
| Disabled/reset | pause, focus/tracking loss, reference change, ladder, mantle, mounted weapon, scripted body ownership; recovery needs neutral |

Observed posture is decoded from native movement flags (never from what was requested).
Requests use **absolute** bindings so a stale request can never toggle a standing player
down: to crouch `gocrouch`, to prone `goprone`, to stand `+togglecrouch`/`toggleprone`
only when the known current native request is crouch/prone. The binding dispatcher owns
its own reliable notification. A request the native layer refuses never fabricates a
posture and never repeats.

### 6.6 Fire request (`controller_firing.hpp`)

Fire only when `gameplay && muzzle_ready && owner.can_fire() && fresh input`, with the
gate reset on owner id / rear revision / rear hand / reference generation / continuity
change, and both grip **and** aim valid for the *firing hand's index*. A `discrete` mode
(scripted devices) additionally requires a new press. Mechanical readiness (break-action
closed, launcher, optics screen, scripted device) can veto and clear the bit. Shot
direction comes from the tracked muzzle pose, but ammo/cadence/damage stay native; a
separate native-ballistics entry with per-instance command-time clocks serves dual wield
(ref 08).

### 6.7 Auto-ADS = request only

The ADS bit is OR-ed in `create_command_stub` **after** FinishMove has packed the selected
weapon (and only if `owner.weapon == cmd->weapon`); native PM gates keep authority. The
policy compares center-eye to solved muzzle with entry/exit corridors + dwell:

| | depth (m) | lateral | height | facing cos |
|---|---|---|---|---|
| Entry | 0.15 .. 1.50 | <= 0.09 | -0.025 .. +0.16 | 0.866 (30 deg) |
| Exit | 0.10 .. 1.70 | <= 0.13 | -0.06 .. +0.22 | 0.766 (40 deg) |
| Thermal | as above | | | entry 35 deg, exit 45 deg |

Dwell 100 ms in/out (40 ms thermal). `far_extension = max(0, sight_to_muzzle + 0.60 -
1.50)` so suppressors/long barrels do not consume reach. Requires fresh muzzle+input,
both hands valid and a **support hand** (two-hand lease), not scripted/mounted/mantle/
ladder, and a byte-verified native ADS command site (`vr_autoAds`, default on). Counters
are labelled "request only, not native acceptance". The comfort translation of the gun
toward the eye is accounted for in the corridor so it cannot release its own request.

### 6.8 Other command producers in the same boundary

| Mode | What is injected |
|---|---|
| Vehicles / scripted branches that skip NormalInput | sticks -> same move axes (fallback after original returns), physical steering/throttle merged |
| Remote control (UAV/missile) | `remote_look` stick -> byte-sized remote-angle command (rates from native dvars); per-frame head angle beyond one signed-byte step stays pending; combined with native mouse axes without wrap; entry keeps native aim; exit restores native facing |
| Fixed scope / mounted turret | `fixed_sniper::command` / `mounted::command` own axes/fire; HMD commands suspended |
| Ladders / climbing owner | zero axes and clear jump/sprint/duck/prone bits while VR owns movement |
| Story scenes | `story_actions` (phase-dependent: same trigger = brake in one phase, melee in another; phase change invalidates both gates "so braking cannot become a buffered kill") |
| Sentry placement / C4 / notebook | attack bit at the CreateCmd boundary; "drive scripted logic through the same triggers the flat game uses" (never synthesize keyboard, set mission flags or send fake events) |

## 7. Script visibility: two different consumers

Scripts see player input in two ways; supporting only one silently breaks some scenes.

1. **Binding-notify waiters** (`notifyoncommand`, e.g. waiting for jump/sprint). They listen
   on the engine's reliable key-binding notification queue, **not** on usercmd bits. Each
   accepted new press also calls the engine's own notify routine once with the verified
   whitelist index (`+gostand` = 37, `+sprint` = 83; both identity strings are verified
   before install). Rules: one canonical notice per press (scripts that register both
   `+sprint` and `+breath_sprint` for one event still get one `+sprint`); a short tap is
   consumed once; held input resumed after menu/focus/stale/reference discontinuity is
   rejected; held movement keeps writing command bits without resending the notice. Do not
   call the script VM from the command thread. The `pressed` result is separate from the
   `held` mask for exactly this reason.
2. **Polled command state** (e.g. a script that checks melee/attack/ADS state every
   **50 ms**). The engine latches rising edges per command and clears the latch each frame,
   so a **one-command pulse is missed** at some poll offsets. Use a `polled` delivery mode:
   a physical hold keeps the bit held; a short tap stays visible for **>= 100 ms**
   (two polls) (`polled_minimum`). Sentry placement does the same (`until = now + 100 ms`
   on a held press; holding keeps attack held because the script's release loop needs it).
   Regression: simulate 90 Hz commands against all 50 ms poll offsets; the one-command
   implementation misses some, polled delivery covers all. A native "is shoot button
   pressed" builtin replaced with a VR predicate bridges a tap with a `fire_press{serial}`
   pending for 150 ms until a ballistic acceptance consumes it.

Counters for these paths count *submissions*, not script acceptance (label them so).

## 8. Menus and desktop mouse/keyboard arbitration

When a menu or key catcher is active, gameplay injection stops (`gameplay` gate above) and
a separate menu input owner takes over; neutral-arm all buttons/axes whenever target,
focus, generation or hand changes.

- **Pointer**: the `runtime_aim` ray hits the menu panel; hit UV maps to the native canvas
  coordinates `x = round(clamp(u)*(w-1))` and is delivered as the game's absolute cursor
  position; trigger = left-click (`K_MOUSE1`) only on a ray hit; primary = Enter (confirm);
  secondary = Escape (back). The right hand owns the pointer initially; the other hand's
  trigger **press** transfers ownership without clicking (hold keeps ownership, tracking
  loss cancels). Leaving the panel never transfers ownership.
- **Stick navigation** is secondary: arm when `|x|,|y| < 0.25` and no buttons; direction at
  `>= 0.65` along the dominant axis; first repeat after 350 ms, then 100 ms. "Stick mode"
  stays until a deliberate pointer move (> 0.025 in UV) or a controller action.
- **menu_recenter gesture** (`menu_button`): hold >= 1 s fires `recenter` once (release then
  does *not* toggle); a tap that ends before 1 s fires `toggle` (Escape). The separate
  `menu_toggle` action sends Escape. Never both from one press.
- Held trigger on an entry/briefing screen -> native Enter, independent of ray hits.

**Desktop mouse/physical key arbitration** (`input.cpp`, `native_ui_pointer.hpp`), needed
because the engine UI polls absolute desktop coordinates every frame even when nothing moved:

- Keep the desktop cursor baseline *independent* of coordinates delivered by the VR ray
  (`native_ui_pointer`): `native_position(x,y)` reports `moved` only if the polled position
  changed; real desktop movement releases VR ownership immediately (counts as physical
  activity); a **passive identical poll must not snap an owned VR pointer back** and must
  not count as activity.
- `acquire()` when VR delivers a pointer position, `release()` on any context change or when
  a GUI overlay captures input.
- Before a *physical* mouse button/wheel edge is delivered, restore the last desktop position
  if VR owned the pointer ("a stationary physical click owns its desktop position, not the
  last VR ray's").
- Any physical key event clears VR ownership of that key. VR synthesizes native key events
  through `vr_ui_key(key, down)` only if the key is **not currently held physically**
  (read the engine's key-state table); a held physical key cannot be borrowed and its release
  stays physical-owned. On release/pause/context change every VR-owned key is released.
- Developer overlays/console get first refusal on key/char/mouse events, and a menu-input
  trace (debug probe) counts polls, moves, suppressed polls and key events.

## 9. Movement mechanics layered on the engine's Pmove (H2-Mod `movement.cpp`)

Not VR-specific, but the pattern matters for ports with extra mobility rules: declare a
profile from scripts (`setpmoveprofile(player, profile)` atomic/validated;
`clearpmoveprofile`) and execute it by hooking the engine's walk/air move functions so
**predicted and authoritative movement run identical copied data without calling the
script VM**. Key details: "pressed" button edges are derived per command *server time*
(a second call at the same time returns the cached set; an older time recomputes from
`oldcmd`) so prediction replay is idempotent; runtime state keyed by player-state pointer
(8 slots) with a profile generation; local player only; a latched `input_projection`
(e.g. sprint request) and actions (`impulse`, `velocity_curve`, `acceleration`) with
cooldown/charge pools and `exclusive_group`s. When VR needs to relax a native rule (e.g.
sprint while aiming) patch the same predicate for **both** predicted and authoritative
paths (shared `allowed(ps)` policy; see ref 02 for the register-preserving bridge) or the
prediction desynchronizes. See `docs/pmove-scripting.md`.

## 10. Haptics

Semantic events (shot, magazine seated, hover, melee confirmation) flow:
`weapon_feedback::publish(event)` into a **16-entry** queue (overflow drops the oldest,
counted) -> a per-tick drain that re-validates freshness (owner/instance generation,
reference generation, age, gameplay state) and discards stale ones -> `controller_haptics::
request(hand, pulse)` into a mutex-guarded mailbox -> consumed at the next input sample ->
runtime call. Contract of the mailbox:

- **One pending pulse per physical hand; strongest wins.** A new pulse replaces the old only
  if reference generation differs, time regressed, the old one is > **100 ms** old, or its
  amplitude is >= the old one.
- Clamps: duration <= **0.1 s**, frequency **1..320 Hz**, amplitude <= 1; non-finite or <= 0
  rejected.
- `take()` discards (never delays) pulses when unfocused, `sequence == 0`, either pose of
  that hand invalid, reference generation changed, or sample older than 100 ms. No backlog,
  no replay after focus/tracking/recenter change.
- Delivery: OpenXR `xrApplyHapticFeedback(duration ns, frequency, amplitude)`; OpenVR
  `TriggerHapticVibrationAction(handle, 0, seconds, frequency, amplitude)`. Per-hand
  `bound`, `sent`, `failed`, `last_error` counters; haptic failure is reported, never fatal
  to rendering. Frame invalidation clears the mailbox. Binding files suppress the runtime's
  own click haptics (`haptic_amplitude 0`) so only mod haptics play. A user gain dvar scales
  amplitude; recoil goes to haptics + weapon presentation, never the head.

## 11. Diagnostics and tests for the input layer

- `vr_input_status` prints hook admission, frame age/sequence/focus, move/turn values,
  sprint/jump `active/down/presses`, notify counters, stance/ADS request counters (labelled
  "request only"), pose pipeline, calibration, game-view counters, and writes a status file
  automatically after the first input API failure (`input_history`: per-channel valid vs
  engaged, loss/recovery counts, first API failure retained across focus churn, 8-slot
  transition ring, "never available" distinct from "lost"). SteamVR setup report cross-checks
  the manifest against requested handles/types and captures binding info.
- Header-only suites with an injected `check` cover: neutral-arm list (4.1), counters, tap
  between reads, duplicate frame, smooth turn at 30..144 Hz, stance rules, haptics clamps,
  OpenVR binding JSON contracts (Python), calibration orthonormality/settling, tracking
  quality fencing, pose-pipeline adapter (bounded retry, ambiguous binding rejection).
  A WARP/CPU pass never establishes headset acceptance; analog thresholds still need
  real-device feel testing.
- Hardware probe: a synthetic-scene executable (left magenta/right cyan) that requires
  complete stereo pair counters, then defers acceptance to a human.

## 12. Porting notes and pitfalls

- Find the engine's per-tick command constructor and its three sub-steps first (ref 03);
  decide per feature whether the engine has a *binding dispatcher* (stance, notify) you can
  call rather than writing state.
- If scripts wait on key bindings, locate the reliable command/binding notification path
  early; usercmd bits alone will not wake them. If scripts poll at an interval, hold/extend.
- Treat `trigger`/`squeeze` thresholds as per-profile data with hysteresis; for SteamVR let
  the binding own them but keep a near-zero "any pull" trigger so the game-side discipline
  decides the real threshold.
- Plan a re-home for profiles that lack buttons (Vive: no recenter/face buttons); document it
  in the binding description and prompt text.
- Do not widen the 150 ms freshness windows to "make slow-motion work"; carry producer
  continuity instead.
- Do not reuse a firing/authority gate for pure presentation (a foregrip-only carry lost its
  held-magazine pose this way); keep `pose_ready` separate from `ready`.

## H2-only addresses and layouts (examples, never reuse on another build)

- Command path: `CL_CreateCmd` 0x1403D20F0 (entry bytes `48 89 5C 24 10 57 48 83 EC 40`);
  NormalInput call 0x1403D2223 -> 0x1403D13C0; remote-input call 0x1403D21CA -> 0x1403D1870;
  FinishMove call 0x1403D22A2 -> 0x1403D0300; UpdateViewAngles call 0x14068D680 -> 0x14068D7B0
  (pitch store 0x14068D909); command-time store 0x14068F81B; pack-angles 0x1403D03B0; sprint bit
  0x1403D108F; jump bit 0x1403CEF8E; ADS set/clear 0x1403D0F82/0x1403D0F79; notify-command
  function 0x1403D3A90 (call 0x1403CF21C); execute-binding 0x1403CF1E0; native stance request
  global 0x141E8A61C; client angles 0x141E8A65C, yaw 0x141E8A660; remote-rate dvars 0x14CE00E30;
  key-state table 0x141E87684 (stride 12).
- Desktop input hooks: `CL_KeyEvent` 0x1403D2AE0, `CL_MouseMove` 0x1403296F0, `CL_CharEvent`
  0x1403D27B0, LUI key event 0x140328F50.
- `usercmd` 64 bytes: buttons +4, angles +8, forwardmove +28, rightmove +29, remoteControlAngles
  +0x3e. `playerState`: commandTime 0x4c, pm_flags 0x54 (prone bit 0, crouch bit 1), delta_angles
  0x98, viewangles 0x108. Buttons: attack 0x1, sprint 0x2, jump 0x400, ADS 0x800. Whitelist:
  `+gostand` 37, `+sprint` 83, `+togglecrouch` 89, `toggleprone` 99, `goprone` 100, `gocrouch` 101.
  The engine's own native stance-request encoding (0/1/2 = stand/crouch/prone) differs from
  `pm_flags`; the two are deliberately distinct.
- Script-polled melee (Oilrig): command buttons copied to `client+0xe90c`, rising edges latched in
  `+0xe918`, cleared at 0x1404AD059.
- Bindings: action-set label "H2 VR"; SteamVR application manifests per config; env override
  `H2V_VR_BACKEND` selects `openxr`/`openvr`.
