# 10 - Scripted scenes, vehicles, turrets, climbing, death, and comfort

Single-player games constantly take control away from the player: animated rigs the player is linked to,
disabled weapons, vehicles, turrets, remote cameras, death, slow motion, forced look-at. A VR port lives or
dies on how it handles those moments. The Overlord method: **identify each moment exactly from native
state, then publish a small copied policy** that the camera, input and hands systems consume generically.
Camera maths lives in 06, input plumbing in 07, hands in 08; this file is the scene layer on top.

Overlord paths are under `src/client/component/vr/gameplay/` unless stated. Addresses/offsets are H2-only
examples. `(report-only)` = taken from a reader report, not re-verified in source. Spot-checked in source:
`scripted_camera.hpp`, `camera_policies.hpp`, `scripted_sequences.cpp`, `scripted_sequence_policy.hpp`,
`free_climb.hpp`, `vehicles/steering.hpp`, `mounted_turret_policy.hpp`, `controller_locomotion.hpp`,
`settings.hpp`, `ladder_runtime.cpp`, `cliffhanger/physical*.{cpp,hpp}`. Other numbers in sections 4.2 (ladder
rung extraction, top exit), 6 (fixed sniper, remote look) and 8 (ending gestures) come from reader
reports/docs.

## 1. Scene identity: `classify(evidence)` from native flags

### 1.1 The contract
Every special scene has three parts (`campaign/sequences/<scene>.{hpp,cpp}`):

```cpp
struct evidence { bool alive, linked, intro_controller; unsigned parent_id, body_id; string_view animname; ... };
inline view classify(const evidence&);          // PURE, header-only, unit-testable without the engine
view observe();                                  // the ONLY code that touches the script VM; server pipeline only
bool supported();                                // map-name gate - necessary, never sufficient
```

- `evidence` holds only copied bools/ids/string_views gathered from **native state**: level flags, linked
  parent entity and its identity/animname/classname, weapon-permission bits in the player state, object
  variables. **Never time, never health thresholds, never elapsed-since-X.** (`oilrig_sequence_tests`:
  "gaze alone cannot unlock an early action; a sentinel flag alone cannot capture later animations".)
  Death is the one exception that *is* health-based, because the engine's own life state is the authority:
  `player_life::dead` = movement type 5..7, or type <= 1 and health <= 0, signature-verified before use.
- `classify` returns a `view`; it never reads the world.

### 1.2 Identity rules (each fixed a real bug)
- **Compare entity identities, not names.** Gulag: the legacy camera (level field 0xCC0B) and the
  remastered rig (0xC438) are both animated `player_rappel`; the adapter compares the linked parent's
  object id to both fields. Roadkill: `player_gets_in` stays set after boarding, so identity of vehicle vs
  rig anchor decides. Airport requires `parent == body stored in a level field` plus rig animname
  `player_ending`; sliding uses `player.slidemodel == linked parent`; dcemp matches `player_rig` to the
  specific wake-up body stored in a player field because the name is reused later.
- **Prefer persistent fields to transient flags.** Breach uses the persistent `breaching` field because
  notetrack flags "are consumed and cleared by another script thread in the same tick"; gulag keeps the
  downed body's camera until the link ends instead of ending at a transient `player_falls_down` flag.
- **The exit condition is usually native unlink/death**, not a flag edge (favela, sliding). Unknown entry
  points -> stage none (no VR takeover): allow-list known start points.
- Wait for the *last* script write before resetting: dcemp's script links the carrier twice 0.35 s apart,
  restores angles, then raises `emp_back_from_whiteout`; reset on that flag or the script overwrites you.
- Keep one heading across hand-offs (trainer pickups interpolate a `script_origin` then link to the
  animated `worldbody`).
- Let the native weapon-enable bit decide combat vs cinematic where stow/draw can transiently empty the
  selection (estate). Do **not** infer a cinematic boundary from health or time (airport notetrack raises
  `escape_player_shot`).
- Names used to *recognise assets* (models, bones) are hints; identity is `definition + instance
  generation (+ revision)`, never a name (07/08 invariants).

### 1.3 The coordinator (`campaign/scripted_sequences.cpp`)
Runs on the **server pipeline** every tick; publishes one immutable `view` under a mutex; consumers
(camera, input, hands, HUD prompts) copy it.

Order of adapters: airport -> death -> breach -> vehicle -> estate -> favela -> gulag -> ending -> trainer
-> rappel/oilrig; then, only if the player is linked and alive, the linked-parent adapters (oilrig
evacuation, cliffhanger, roadkill, dcemp, sliding); a linked scene with no adapter gets
`linked_gameplay` (or `physical_ladder` if the VR ladder owns the carrier). First match wins; each
adapter owns its own control policy (a camera-only vehicle scene must not suppress stance notifications).

Bookkeeping that matters:
- `epoch` increments on scene change, player-state pointer change, or command-time rollback.
  `position_epoch` is a **separate** generation for camera-translation ownership, preserved across a
  helper whose VM wrapper is recreated (`same_position_owner`: "recreating that wrapper is not a new
  camera shot").
- `for_player(ps)` expires only on `commandTime < started_time - 150`, never on slow frames: only native
  exit/lifecycle evidence may restore hands.
- Load/rollback (`player != current || commandTime < last_time`) bumps a load generation; **one recenter
  request** is issued after a load/rollback, only once live script *and* tracking observations are valid
  (`game_view::load_recenter::consume`). A normal first frame consumes the load too, so a later story
  sequence cannot trigger an unrelated recenter.
- **Fail-safe**: an exception while reading script state publishes `phase::release` with movement, turn and
  world-use denied, weapons suspended, authored arms. "Loss of a script observation cannot grant weapons or
  a QTE."
- **Input contract guard**: `native_input_contract()` byte-verifies the two native sites that read attack
  and melee state before the component is enabled (a flat capture showed an E-key edge set usercmd bit 2
  before the native states appeared).

### 1.4 What a `view` carries
`scene, stage/phase, camera (policy), rotation_tag, epoch, position_epoch, command_time, allow_movement,
allow_turn, allow_world_use, suspend_weapons, retain_weapon, hide_body_arms, independent_hands,
block_carry, hide_chest_equipment, hidden_entities[4], melee_mode (press|polled), arms (request)`.

Typical choices:
| Situation | Policy |
|---|---|
| Cinematic body linked (airport, trainer, vehicle intro) | camera only + weapons suspended, authored arms |
| Mixed (favela, sliding, roadkill, oilrig input) | camera free, native weapon/locomotion retained |
| Authored arms drivable by controllers (cliffhanger, ending) | "tracked arms", `independent_hands`, authored body hidden |

## 2. Per-scene policies

### 2.1 Camera policy (portable value type `scripted_camera.hpp`; math in `camera_rig.hpp`, see 06)

| Dimension | Values |
|---|---|
| owner | `player`, `script` (script owner = HMD yaw is *not* written into native command angles) |
| head_rotation | `fixed`, `pitch_roll`, `limited` (per-axis clamp from entry baseline, unwrapped), `free`, `command_delta` (head is relative input, remote vehicles) |
| head_translation | `fixed`, `attenuated` (gain 0.1, 5 cm spherical limit), `tracked` |
| script_rotation | `ignore`, `override_view`, `additive` (relative change of the source since last frame) |
| script_axes | `yaw`, `all`, `yaw_roll` |
| rotation_source | `view`, `tag` (named bone: `tag_player`, `tag_aim`, `tag_origin`, none) |
| camera_entry | `preserve`, `align` (one-shot "cut": armed on request, waits for a valid anchor, consumed once per native entry lifetime) |

Named profiles (`camera_profiles`): `gameplay, free, aligned, locked, yaw, vehicle, authored,
authored_aligned, authored_full, authored_yaw_roll, align_on_entry(p), bounded, fixed_scope,
remote_control`. Scene table (`campaign/sequences/camera_policies.hpp`) examples:

| Scene | Policy |
|---|---|
| ladder (VR-owned) | player-owned, free, **tracked** |
| generic linked gameplay | player, free, attenuated |
| airport shot | script, **fixed** head, override_view all axes from tag |
| breach | `yaw` (authored yaw, drop authored pitch/roll) |
| rappel, trainer, roadkill, dcemp wakeup | `free` |
| gulag intro | `authored_yaw_roll` (new rig) / `authored` (legacy helper); gulag evacuation: free head, fixed translation, additive yaw_roll from tag |
| cliffhanger physical climb | script + free + tracked; rescue: additive all axes from tag, attenuated |
| museum credits | additive yaw from view, tracked, align |
| oilrig execution/recovery | fixed head, attenuated, override_view all |
| death | `aligned` |

Priority when several apply: remote_control > thermal/fixed scope > mounted turret > scene.
Rules:
- Compose by **matrices**, never by adding Euler angles; heading for entry/hand-back/body is geometric
  (`atan2` on forward/left axes, see 06/14), Euler is only for packing native commands.
- Only identity/phase admission lives in adapters; "all camera math stays in the portable camera_rig".
- A player-owned fully-tracked camera keeps `position_epoch = 0` so entering a native link does not
  re-anchor and shift the entry. Gulag's intro shot uses the scene epoch so alignment is consumed once.
- A new DObj/tag = new rotation baseline; an invalid source holds the current heading with no catch-up.
  Do not seed from a helper tag whose raw roll (119 degrees in one case) would create a sideways basis, nor
  from clamped rear-facing player angles.
- Tag choices: `player` (animated camera bone; yaw sampled from the unmodified bone, not clamped player
  angles), `origin`, `aim` (legacy helper, never a camera bank), `none` (keep heading).
- Animated camera bones near the vertical pole flip forward-XY heading: compare the SAME axis (right/
  forward/up) between consecutive poses, choose the axis whose horizontal projection is largest in both
  (`score = min(|a_xy|^2, |b_xy|^2)`), add only that axis' `atan2` delta (`scripted_view.hpp`).
- Scripted translation: `vr_scriptedHeadScale` default 0.1, range 0..0.25, 5 cm limit; translation baseline
  has its own epoch separate from rotation; translation never grants or revokes weapon permission.
- Hand-back: restore the command heading **once**, from the *rendered* heading (including partial head
  constraints). Time moving backwards resets the rig.
- Camera smoothing is render-only (ladders: exponential half-life 18 ms on the collision-approved body;
  head tracking is never filtered).

### 2.2 Input policy

**Native permission boundary** (`native_scripted_control.hpp`):
- `disableweapons`/`enableweapons` set/clear bit `0x80` in `ps+0x3c0` (paired instructions byte-verified).
  `scripted_control::allowed(ps)` = not dead AND weapons-disabled bit clear (or the verified airport
  opening) AND not mounted AND not owned by a scripted body/arms scene. It gates VR firing, carrying and
  hands on both prediction (`predicted_allowed`) and the server.
- It is *not* a proxy for animation, linking or vehicle occupancy; mounted state can have native weapon 0
  with the bit set, so it cannot distinguish turret from cinematic. Death is separate (the bit alone did
  not cover it). Mounted ownership uses entity `e_flags`, not `pm_flags`.
- **Death releases grips and clears manipulation/render leases** without a native drop; ordinary scripted
  suspension *retains* grips. Releasing the last hand in empty space during a playable scene consumes the
  release (no queued drop).

**Scripted button bridge** (`campaign/scripted_sequence_policy.hpp`, `sequences::actions`): the same
physical trigger maps to different native semantics by phase - descent: brake (`brake_button=1` via
`digital_button_gate`), melee: press (`melee_button=4`). A phase change invalidates both gates "so braking
cannot become a buffered kill". Admission requires focus, a fresh sample (<= 150 ms), a settled
orientation, valid grip+aim for the hand, and the scene epoch.

**Script waiters and the polling rule** (the old "listen for binding notifications, not command bits" was
half true). Identify which receiver the script uses and deliver accordingly:

| Script waits with | Deliver by |
|---|---|
| `notifyoncommand` (binding notify) | call the engine's own notify function with the verified binding index once per accepted press (`+gostand`, `+sprint`, `+melee` ...); do **not** call the script VM from the command thread |
| polls command-derived state every 50 ms (`attack`, `meleebuttonpressed`, `ismeleeing`, sentry placement, oilrig melee) | the engine's rising-edge latch clears every frame, so a one-command pulse is missed at some poll offsets: **hold the bit while the physical control is held and keep a short tap visible >= 100 ms** (`polled_minimum`); tests simulate 90 Hz commands against all 50 ms offsets |
| native weapon *state* (`ismeleeing` = weapon states 14..16) | produce the real command bit that makes the native state machine reach it; never write the state |

Also: held-at-entry input is not a fresh edge (a trigger held when a scripted state starts must be released
and re-pressed); phase change, focus, tracking, reference, binding or checkpoint change discards pending
pulses; no re-arm while still held.

**Movement/turn**: `allow_movement`/`allow_turn` per phase (oilrig: native yaw limits are widened +45
degrees each side in the opening segment by a register-exact asm bridge on the float-getter call sites
*after* the original `Scr_GetFloat` ran, so argument errors are preserved; non-VR, other scenes, NaN and
out-of-envelope values pass through unchanged).

### 2.3 Hands policy

| Mode | When |
|---|---|
| hidden | cutscene body without usable arms; `hide_body_arms`, hidden_entities hide the story body/knife/NPC |
| authored arms | native animation drives the arms (`arms.mode=authored`), controller-driven interaction suppressed (death keeps native arms visible) |
| scripted arms blended by **animation progress** | `scripted_arms_*`: IK only the requested chains, weight from the authored animation clips' normalized time (0 if any automatic clip has weight > 0), smoothstep over the takeover clip, per-hand envelope `min(authored, smoothstep(elapsed/200 ms))`; never infer completion from a flag or wall timer |
| free hands | mixed scenes; `independent_hands` |

Rules: hook after the main skeleton hook completed all bones, under the native DObj lock; one solve per
native pose (`once_per_pose{object, matrices, timestamp}`), rejected poses are not retried by the second
eye; `wants_pose` returns true only if the requested bone mask includes an arm bone (so camera/tag queries
cannot pre-complete the skeleton before native controllers run). Blend LOCAL transforms in parent space
after IK (blending wrists alone snaps native elbows); elbow pole from the authored torso frame, not the
HMD; no limb stretch on a skinned full body. Props attached to a wrist are posed inside the same skeleton
after IK. Authored body arms hidden via visibility masks keyed `(object, matrices, epoch)`, restricted to
the one linked instance (client object handles are shared between adapters).

**Exit**: control returns on native unlink/lifecycle evidence; restore the command heading once; hands
restored only by that evidence, never by a timer.

## 3. Script-VM bridge (how scenes get physical interaction)

Pure input injection cannot make a scripted loop physical (climbing, pulling a knife, crawling). The
reusable idea of this repo: **replace the minimum script boundary with a native-controlled one**, keep the
story authoritative.

1. **Inject script text**: `gsc::register_virtual_source(name, source, map)` (name must start `scripts/vr_`,
   `[A-Za-z0-9_/]`, <= 128 chars, source <= 64 KiB, optional per-map gate, startup-only, never replaces a
   native asset). Compiled at load into the engine's persistent script arena, freed on script shutdown.
   Pair it with native builtins callable from it (`gsc::add_function("vrphysicalclimbbegin", ...)`, ~9
   per scene: prepare/seed/begin/state/milestone/fallside/entryready/end). Builtin table enlarged (0x1000
   slots), custom builtins run under try/catch converting C++ exceptions into script errors.
2. **Redirect native functions** at level start (`scripting::on_level_start`): look up entry points by
   (file, id) and `notifies::set_gsc_hook(native_pos, replacement_pos, guard)`. Only the minimum boundary
   is redirected (climb: the two native climb *loops*; entry, first-wall exit, jump/slide/hang, failure and
   final exit stay native). The replacement calls the same native helpers (fall, cleanup) so flags,
   notifies and saves behave identically. Hooks are unbound at shutdown and re-bound each level; any
   failure leaves native behaviour untouched.
3. **Guards decide per call**: configuration guard (`eligible()`: VR on, independent hands on, trace
   signature verified, map name, linked parent animname) and **call-site guard** (`called_from(range)`:
   `scr_VmPub->function_frame[-1].fs.pos` must lie inside the native caller's bytecode extent, found as
   `script_function_extent` = next greater address in the sorted function table, rejecting > 65536 B;
   frame depth capped at 31).
4. **Claim ownership from configuration, not transient readiness**: transient asset/HMD readiness after
   load must wait *inside* the bridge (`wait 0.05` polling a builtin) rather than fall through into an
   unhookable, already-running native loop.
5. **Adapt notetrack handlers instead of re-implementing**: rewrite `left_stab/right_stab` to `stab` and
   wrap the native hit callback to capture the real impact point/normal first.
6. **Declare checkpoint fields in the replacement script** (`body.vr_climb_entry = undefined;`) so field ids
   are registered before native save loading or C++ access.
7. **Text-only replacement**: UI hints come from a builtin returned into the native hint call, keeping native
   layout/blink/cleanup.
8. **Whole-statement bytecode surgery with exact-shape inspectors**: never skip a single opcode. Parse the
   function's bytecode range, verify the exact opcode pattern and counts (`inspect()` rejects unless e.g.
   exactly 8 weapon statements with expected names; vehicles: exactly 1 reload site, 1 refill, 1 consume, N
   `waittillmatch` sites), then redirect each whole stack-neutral statement to the resume point. A
   redirect executes its target directly without chaining another hook, and the VM decodes the target
   opcode immediately: hook the NEXT instruction. Locate a block by masked signature (259-byte block +
   mask, relocated string ids/call operands masked, search inside the function extent <= 2048 B, verify every
   `GetString` operand against live interned strings, exactly one match else fail closed). To replace a
   return value use a continuation stub `{OP_CallBuiltin1, func_lo, func_hi, OP_Ret}` after the original
   parameter prologue.
   **Pitfall**: never insert/skip parameter-binding opcodes. Bypassing `OP_clearparams` handling made it
   release values up to `SCRIPT_CODEPOS` (type 7) while the new call had `PRECODEPOS` (type 8), crossing
   frames and releasing the caller's live references (unrelated crashes in input/lighting/render). Validate
   and execute the original prologue, redirect only the body.
9. **Intercept builtins narrowly**: `gsc::intercept_builtin(name, cb)` (<= 8, startup-only); accept a call
   only if the VM position is inside the specific native function (`scr_function_stack->pos` range) AND
   the frame's local owner is the player AND arg count/types/names match; otherwise the original runs.
   Return true means "I supplied the result through an isolated call". Examples: `magicbullet`/
   `playfxontag` for vehicle guns, `vectordot` at one call site for gaze, `bullettrace` in the designator
   trace. Never re-enter `VM_Execute` from a builtin already running inside the native loop: compute the
   needed object on the server coordinator and return it.
10. **Observe, don't consume**: `notifyoncommand` interceptors return false (native continues); notify
    aliases deliver the original event plus one VR event; waiters can be rewired by patching `GetString`
    operands (`vr_flare_ignited`). Restore bytecode edits on shutdown while keeping control-flow guards
    when loading a save; never retain VM string refs through shutdown.
11. **Trigger the story's own waiter instead of faking its outcome**: boarding a snowmobile calls
    `scripting::notify(trigger, "trigger", {player})` on the exact `script_origin` use target after
    checking level flags (`player_snowmobile_available == 1`, `player_rides_snowmobile == 0`, ...), the
    native prompt text/key and classname - "never set mission flags or mount a vehicle directly".
12. **VM call contract** (calling the script VM from mod code): arguments pushed in **reverse order**; wrap
    with *stack isolation* (swap `top/maxstack/inparamcount/outparamcount` to a private 512-slot array,
    `Scr_ClearOutParams` and restore on exit) because hooks fire mid-script; contain script errors with the
    **game's own `_setjmp`** + SEH around the call; every wrapper does AddRef/RemoveRef hygiene (strings via
    `SL_GetString`); guard hook re-entrancy when a detoured function is called back. Restore the engine's
    stripped script-error diagnostics before building on the VM.
13. **Persist plugin state in the VM** so native saves carry it (section 4.3), and keep story milestone flags
    set by the replacement *script* from physical progress, never by C++.

## 4. Carrier / proxy locomotion (climbing, crawling, ladders)

### 4.1 Mechanism
A hidden, non-solid `script_model` with model `tag_origin` is spawned by the replacement script; the
player is linked to it (`playerlinktodelta(carrier, "tag_origin", 1, 0,0,0,0, 1)`), ground reference set
(`playersetgroundreferenceent(carrier)`), authored body hidden. A native builtin ticks at 50 ms
(`vrphysicalclimbstate`) and returns state. C++ moves the carrier by writing the entity `origin` via the VM
after a **swept collision test**:

- `free_climb::slide(from, delta, 0.005*units, sweep)` where `sweep` = `G_TraceCapsule(..., bounds, 0,
  0x280e831)` with a "hanging" capsule derived from the player's standing bounds (lower body shrunk to
  <= 0.22/0.22/0.45 m half extents, head clearance kept). Up to 4 trace iterations with plane clipping
  (`clip_motion`, crease handling) and a skin gap.
- Re-read the entity origin after writing; error if > 0.02 units differ ("native carrier movement was
  rejected"). On a blocked slide, clip hand goals against collision planes or report `obstructed()` so the
  solver goal does not wind up.
- Link-to-delta preserves the native eye offset: observe it only after the link advanced (`wait 0.05`),
  reject if > 128 units.
- Hand-back: `body.origin = body_start + carrier.origin - carrier_start`, unlink, relink to the body's
  `tag_player`, show body, delete carrier, play the native closing animation. Failure uses the native fall
  helper with the side of the last released hand.
- Ladders use the same mechanism but a *full* capsule and `playerlinktodelta(carrier,"tag_origin",1,
  180,180,85,85,1)` (the link grants 180 degrees yaw / 85 pitch; zero limits were removed). The native
  automatic ladder admission is disabled by hooking the ladder-move check (call the original for trace
  bookkeeping, then clear the ladder flag) *only while VR owns the player*.

### 4.2 Solver contract (`free_climb.hpp`)
Everything in metres (`pos / units_per_meter`); `rescale(old/new)` on world-scale change, `rebase()` whenever
tracking is lost, a generation changes, a sequence repeats or resources are missing.

- Each hand plants on a world contact `{point, normal}`; the body goal moves by **minus** the hand's tracked
  motion, averaged over planted hands; body step bounded (`bounded_step`, 2.5 m/s; ladders 2.0 m/s); `dt`
  from native command time, rebase if `dt <= 0 or > 0.15`.
- **Attach** only when the control is "fixed" (`fixed = !trigger_active || !grip_active || trigger_down ||
  grip_down`), tracking is continuous (< 0.5 m jump), the tip moves >= 0.08 m/s *into* the surface normal
  (and by > 0.5 mm), and a native trace supplied a valid unit-normal contact.
- **Detach** on a non-fixed release plus withdrawal >= 0.025 m along the normal (ice-pick extraction);
  ordinary hand grips release immediately. `topology_changed` rebaselines on 2->1 hands.
- Contact discovery: while a hand is free, trace (same trace and contents mask as the authored picks) from
  the previous tip to current tip + 0.025 m when travel > 0.02 units; valid only if fraction in (0,1), no
  entity hit, surface **collision material** in the whitelist (ice/plaster/rock/snow; "some authored cliff
  faces are collision-labelled plaster or rock") and normal length > 0.9. A fallback head->tip trace admits
  a tip within 0.06 m of a wall hit so a pick is not buried behind the wall.
- What the player sees is what is sensed: while latched the IK wrist is replaced by the world support
  (`wrist = support + normal*withdrawal - R*tip_local`); when a wall lies between head and tip, push the
  wrist to `outside_surface` offsets (0.07 m then 0.002 m); the rendered wrist is published back and used
  next tick.
- Ladders: detect by **asset family** (model name patterns), not map name; extract real rung geometry from
  the LOD0 indexed mesh's horizontal edges (length 8..80 units, tilt <= max(0.4, 3.5%), unique-edge dedupe
  before the 8,192-edge limit, `>= 3` distinct heights on one plane, budgeted scan: 512 static models + 128
  entities + 4096 triangles per tick); a contact is accepted only if a native capsule trace along the outward
  normal hits a face flagged `SURF_FLAG_LADDER` (mask 0x10000) and a separate solid trace shows visibility.
  Grab = nearest rung within 0.10*units of the palm and 1.4*units of the head. Top exit: pull >= 0.08 m
  above the start at the top contact -> landing search (0.4/0.65/0.9 from the face, floor normal z >= 0.7),
  route swept in two segments, lift 4.5, land 1.5. Release: unlink, restore ground ref, outward velocity
  0.6*units, set the ladder-jump lockout unless grounded. Render-only "late pull preview" is clipped to a
  server-swept clear segment (<= 12 cm). Ground contact consumes blocked pull accumulation.

### 4.3 Surviving checkpoints and loads (`cliffhanger/physical.cpp`)
- Keep plugin-only state **in the VM**: `persist()` writes one script array (body, start, finish, final,
  milestones, result, fall_side, hands[2]{valid, point, normal, rotation}) onto the proxy entity
  (`vr_climb_save`) on every attach/detach/milestone change - native saves serialize it; process globals
  and tracker baselines are lost. Store only world contacts, never tracking baselines.
- Two restore paths: `restore_current()` from entity fields after a load (proxy `animname ==
  "vr_climb_carrier"` -> climbing; `worldbody` with `vr_climb_handoff` -> authored exit) and
  `restore_script_context()` when the engine resumed the serialized script thread (arguments of the state
  builtin). Milestone bits recovered from level flags; "missing historical contacts do not manufacture a
  release/fall" (wait for real support).
- Validate on restore: ids bounded (< 56320 script slots, `type == SCRIPT_ENTITY`), entity numbers < 4000,
  `|start-finish| <= 1600`, world scale in [0.01, 10000], saved normals unit within 0.01, quaternions valid,
  bits masked, result clamped. Roll-back detection: command time going backwards, entity key (entity +
  generation) comparison each tick; rebase trackers after any restore.
- A save containing a stale carrier (`vr_ladder_carrier`) is released at load, **never other story rigs**.
- Milestones for story flags derive from physical progress (rise > 4, >= 70, >= 100 units set bits 1/2/4,
  the script converts them to the story flags/notifies). Crawl: milestones computed as dot(animation start
  offsets, direction); "never advance a guessed animation index".

## 5. Vehicles

- **Identity**: living linked player + `player.vehicle == parent` + animation identity (`zodiac_player`,
  `snowmobile_player`).
- **Camera**: `camera_profiles::vehicle` (script-owned, free head, attenuated, additive yaw from the view,
  entry align); head rotation stays free; native vehicle yaw changes rotate the tracking heading by the same
  relative amount.
- **Locomotion**: driver axes have no view-turn channel; both sticks are *summed* before the shared radial
  deadzone, then capped at length 1 ("opposite held sticks cancel numerically but are not neutral").
- **Handlebar steering** (`vehicles/steering.hpp`):
  - Steering plane = physical tracking-up plane; pivot = midpoint of the two hand contacts at grab; one-hand
    mode uses a lever from stored contact offsets (rotates about the calibrated centre), two-hand mode uses
    the hand-span direction with the neutral set at the **first bilateral grasp** (not a fixed axis).
  - `angle = -atan2(axis . (a x b), a . b)` between base and current projected vectors, clamped +-90
    degrees; shared speed-adaptive pose filter on the common angle.
  - Response: dead zone 4 degrees, engage hysteresis 4.5 on / 4 off, full deflection at 30, curve
    `x^1.5`, slew 8 units/s. Rebase on any grab/release/invalid direction (span < 0.06 m two-hand,
    < 0.025 m one-hand) **without releasing the grip**; a collapsed angle suppresses output and rebases on
    recovery.
  - Input is the *raw physical wrists* in the transported tracking reference
    (`compose(inverse({world_origin, yaw axis}), wrist)/units`), not the stabilized visual wrist, so the
    vehicle's own translation never becomes hand motion. Secondary button re-centres. Throttle = trigger on
    a held hand, merged with the stick (negative stick wins).
  - Hands are drawn on the *native* handlebar: the skeleton hook reads the `handle` bone matrix, publishes
    it relative to the tracking reference (<= 150 ms), `constrain_hands` places wrists at
    `handle * authored wrist offsets`.
- **Driver weapons** (`vehicles/native.cpp`): intercept `magicbullet`/`playfxontag` only inside the specific
  native shot function and player scope; replace the position with the controller-driven muzzle, require a
  cover trace head->muzzle (accept a hit only if it is the vehicle itself), call the original under stack
  isolation + safe execution, then deduct ammo by compare-and-set on the script object variable (refuse if
  it changed; skip when `player_sustainAmmo`), acknowledge the XR press once, emit haptics. On refusal
  `Scr_ClearOutParams()` and suppress so the native loop sees no shot. The native shoot-button poll builtin is
  replaced to return 1 only when `firing(s)` (fresh snapshot, grip+trigger, magazine inserted, rounds > 0);
  a tap bridges the native 50 ms poll via `fire_press{serial, at}` for 150 ms. Targeting reproduces the
  native candidate logic from the gun frame (zodiac yaw < 20 degrees, |pitch| <= 15, range 1300; snowmobile
  forward dot > 0.94, range 750). FX are queued and emitted only once a rendered gun pose newer than the
  request exists (`effect_state`), so they appear at the visible gun even when the server position trails a
  fast boat by metres. Aim assist is a separate muzzle-relative adaptation of native policy; driver quick-
  load is always on.
- **Auto-boarding** (`snowmobile_boarding`): evidence must hold a 250 ms dwell; any interruption (identity,
  reference, continuity, 150 ms gap, vehicle moved > 0.5 unit, speed > 1, distance > 1.6 m) restarts it; a
  consumed trigger key is remembered until level reset.
- Bytecode inspectors bind **fail closed** (`++binding_failures`) when counts differ.

## 6. Mounted turrets, fixed optics, remote cameras

**Mounted turret** (`mounted_turret*.{hpp,cpp}`, `vr-mounted-turret.md`):
- Detection: player-state entity flags `e_flags & 0x3000` (turret) / `0x100000` (vehicle gun) plus
  `ps+0x1e` linked entity == turret entity; the assembled model identified by profile (model list + total
  bone count), not ids.
- Aim: **scoped view-angle substitution** around the native turret-aim function: compute desired angles from
  the controller, write them to the player's `viewangles`, call the original (which only clamps/normalises),
  restore the head angles in a scope guard. Keep the *requested* local aim separate from the native
  converged/slewed angle or convergence accumulates into the next input. The fire-parameter builder gets
  the turret as the geometry source (the player keeps damage attribution) so the shot matches the visible
  gun. Block native fire in every place the engine reads attack unless the gripped hand's trigger generation
  matches.
- Yaw comes from the yaw pivot joint, pitch from the aim joint (using one pivot for both reverses yaw).
- Camera: patch the two camera-tag query call sites to re-derive the seat from bind poses + current tag
  matrices after the native result; compose the neutral bind of the moving `tag_player` onto a stable root
  before HMD composition; the eye is lowered a fixed offset from gun-local geometry (Blackhawk: +0.35 m *
  world scale) so animation never moves the eye. Refresh the skeleton after the native pose calc under the
  original DObj lock before bounds/cull/skin jobs.
- Controller state machine: takeover = squeeze press-then-release while calibrated (never doubles as a handle
  grab); grip a handle within 0.12 m; hand >= 0.1 m from both axes; per-sample angular delta <= 45 degrees;
  `firing` needs the gripped hand + `fire_armed` set only after a trigger-up since grip + same trigger
  generation; free-look stick adds yaw 90 / pitch 60 degrees/s after a 0.2 deadzone with re-arm. The camera
  and gun share one pose; acknowledged server stick yaw is stripped before adding the continuous total (no
  snap-back).
- Independent presentation pose: copy the client pose struct, write local angles, call the native
  controller with the copy so spin/belt animation survives while aim differs. Wrist calibration captures
  `gun^-1 * wrist` per hand on the first frame of the native hands-on-gun animation (reject if > 12 units
  from the handle).

**Fixed thermal sniper / fixed optics** (`fixed_sniper*.hpp`, `screen_scope_layout.hpp`): the native optic
owns rotation. HMD rotation is removed from the aim command and scope camera; display plane 2.0 m ahead
subtending 60 degrees, anchored to the head orientation captured at epoch start, head translation attenuated
(gain clamp [0, 0.25]) so head motion only moves the viewer; one extra native scene pass at the original
vertical FOV; zoom changes real sampling. **Hand-clutch aim**: squeeze = clutch, hand translation (in the
mount's head frame) maps to angles with gain `2*atan(tan_half_y)*57.2958 / travel` (travel 0.20 m default,
0.05..1.0; spans one vertical FOV so zooming cannot pan), spatial dead band 0.00035 m with
`1 - noise/length` gain and **no time filter**, re-grip never moves aim, teleport > 0.25 m requires
release. Sticks: left = zoom, right = coarse aim at 0.1 scale. Failure: black, never stale.

**Remote missiles/UAV** (`remote_look.hpp`, `remote_view.hpp`): camera = remote view (`remote_control`
profile: `command_delta`), head treated as a virtual mouse/stick. `remote_look::consume` returns logical
rates only if focused, sample fresh (<= 150 ms), epoch set, deadzone in [0.05, 0.5], stick finite and
|x| <= 1.01, **armed** (stick first returns inside the deadzone after the scene starts), dt <= 50 ms.
`remote_view::apply` adds the head delta (wrapped +-180) to native pitch/yaw; `apply_control` encodes it
on the native int8 stick wire (`pack(x) = floor(clamp(x,-1,1)*127+.5)`) with a carried `remaining_`
accumulator so quantisation error is not lost; rate must be in (0, 720] degrees/s; a 64-entry command-time
history lets render layer only the *unconsumed* head delta over native aim.

## 7. Death

(`docs/vr-player-death.md`, `scripted_sequences.cpp`)
- Detect via the engine's own life state (type 5..7, or health <= 0), not weapon permission (that bit was
  not set in the captured death). Gate every feature on `!dead` on both prediction and server.
- Camera: the native falling camera and body animation stay intact; shared `aligned` policy aligns heading at
  entry then composes full tracked yaw/pitch/roll independently of authored constraints; death owns its
  own physical-head translation baseline even when not linked (`vr_scriptedHeadScale`).
- Hands: grips released, leases cleared, **no native drop and no ammo removal**; native body arms stay
  visible, controller arms suppressed; stick locomotion/turn and world use disabled. Airport's authored
  shooting scene is the exception where the verified linked ending body outranks the death camera.
- Fullscreen effects: death blur re-run per eye (section 09 s6); death quote text captured from its ownerdraw
  producer only; checkpoint restore uses the normal timeline reset; world-space FX graphs get bounded
  presentation names added to the native name dictionary before save streams so checkpoints can resolve them.

## 8. Other scene patterns worth copying

- **Gesture-based scenes** (ending, `ending/interaction.hpp`): stage = `classify(scene_evidence)` from level
  fields and an allow-list of entry points. Crawl: eye height target 0.35 m above the traced floor with a
  650 ms smoothstep before locomotion ("never jump upward on first input"), hands plant by trace (accept
  within 0.07 m, normal.z > 0.65). Knife pull: grasp only on a squeeze press edge with the wrist within 10
  cm of the authored anchor; progress = physical wrist rotation (`work = angle/5` when angular speed in
  [0.6, 18] rad/s and angle <= 0.8 rad per sample, max of the two hands), drained by the script loop which
  advances `setanimtime`. Throw: gaze cone cos >= 0.819 (35 degrees) toward the target's eye tag, and BOTH
  absolute and head-relative hand displacement toward the target >= 0.14 m after >= 120 ms, abort > 0.65 m.
- **Physical equipment leases**: identity `lease = (revision+next)*2 + side`; authorization derives from
  native flags and the item actually selected, not installation flags; VR carry cannot steal the story's
  selection (`preserve_native_selection`); native attack is suppressed for cosmetic tools; detonation is
  *observed* (`on_notify_alias`), not executed.
- **Render consistency without locks**: published / submitted / now snapshot triplet; a scene model's
  placement callback compares the submitted snapshot against the current one (mission generation, holder
  revision, lease revision, reference generation) and **omits** on mismatch. Never show a prop at a stale
  pose (props are omitted when the committed hand solve is > 150 ms old or from another reference).
- **Aim assist** is a per-shot cone on the server firing thread (strength 100 = 10 degrees), protected
  phases, failure never cancels a shot.
- Some scenes re-label state for presentation only (`mechanical_display_seconds`: `min(0.05, now - sample)`,
  zero when older than 150 ms; presentation never commits ammo/sound/ownership).

## 9. Cross-cutting scene invariants

1. Every consumer is **idempotent for the second eye and prediction replay**; a duplicate frame is a no-op.
2. **Held-at-entry/resume needs a neutral sample and a fresh press.** Tracking loss is not a release
   (deferred release); a proven physical release ends its session even when pose is lost.
3. Freshness is **per-subsystem**, not one constant: input/pose 150 ms; haptics pulse 100 ms; calibration
   settle 200 ms; smooth-turn dt cap 50 ms; remote look dt 50 ms; captures 250 ms; magazine insertion and
   cover-push guard 300 ms; grab-intent window for a press that starts just before the palm arrives 200 ms (report-only); boarding dwell 250 ms. (The old claim
   of a damage-overlay 350 ms window is not in the source: it is 250 ms.)
4. Identity = definition + instance generation (+ revision). Queued render data is keyed by pair/
   publication; eviction yields not-found, never "latest".
5. Fail closed: a failed byte check disables only that feature and logs once; an observation exception
   releases rather than grants.
6. Everything a scene adapter reads from the script VM happens on the server pipeline; the camera/command
   code reads only the published copy (never the VM from command construction or render threads).
7. Plugin state that must survive saves lives in script-visible variables, never C++ globals.

## 10. Comfort checklist (corrected, with real defaults)

Defaults below are from the shared settings table `component/vr/settings.hpp` and the registration sites;
the Launcher shows the same values.

| Item | Overlord default | Notes |
|---|---|---|
| Turn mode `vr_turnMode` | **smooth** (not snap) | `smooth`/`snap`; offer both; a port may prefer snap-by-default (recommendation, not Overlord behaviour) |
| Smooth turn speed `vr_turnSpeed` | 90 deg/s (15-360) | deflection ramps after a 0.2 deadzone; per-tick dt capped at 50 ms |
| Snap angle `vr_snapAngle` | 30 deg (5-90) | fires at \|x\| >= 0.7 and \|x\|>\|y\|, re-arms below 0.25; snap triggers a temporal-history reset |
| Move/turn deadzone | 0.2 (0.05-0.5) | radial rescale; **no axis rotation by head yaw** (HMD yaw is already native view input) |
| Neutral arming | always | after focus loss, menu, reference/generation change, mode change, > 150 ms gap |
| Camera bob `vr_cameraBob` | **ON** (toggle) | seven call sites + two `MULSS` sites via a register-preserving bridge; off removes positional bob and the two ADS angular terms only (idle sway, recoil, viewmodel bob stay native); weaponless native factors default to 0, so policy uses 1.0 only when the native scale is exactly 0 in camera scope |
| Recoil | gun-only, `vr_recoil` ON, penalty `long` | recoil moves the weapon pose, never the camera/view kick |
| Depth of field / motion blur | launcher default Off (`r_dof_limit`, `r_mbLimit`; quality 0..3 preserved) | native dvars, next start |
| Lens flares | converted; `vr_disableLensFlare` default false | toggle disables only VR draws |
| UI/fullscreen blur `vr_disableBlur` | false | menu blur is zeroed at source *always*; injury blur suppressed at its script call; death blur per eye |
| Hide HUD `vr_hideHud` | false | |
| Subtitles `vr_subtitles` | true | applied at profile load |
| Head / hand / desktop stabilization | all **OFF**; strengths 30 / 40 / 50 | 0 bypasses; per-subsystem half-lives (see 14) |
| Scripted head translation `vr_scriptedHeadScale` | 0.1 (0-0.25), 5 cm limit | scales physical HMD translation, not native camera animation |
| ADS comfort / ADS sprint / auto-ADS | true / true / true | eased whole-gun pull toward the eye, spring-damped |
| Handedness | right (left supported by anatomical reflection) | |
| Quick reload / support grip lock / physical ladders | true / true / true | |
| Aim assist | 0 (0-100 -> 0-10 degree cone) | |
| Enemy melee damage x | 0.5 (0.1-2.0) | |
| World scale | 39.3700787 units/m (shown 100%) | next launch |
| Recording guide dim | 65% | |

Checklist (what to do in a new port, with what Overlord does):
- [ ] Real head pitch/roll always shown in ordinary gameplay. Overlord's explicit exceptions: authored
      `fixed`/`locked` policies (airport shot, oilrig execution/recovery, fixed scope) where the *script*
      owns the camera. Treat each as a comfort risk to justify and test on the HMD.
- [ ] Locomotion: smooth + snap turning, deadzones, neutral arming; vertical stick = stance with
      firm-push semantics (arm at \|y\| <= 0.30, step at >= 0.90, second step needs >= 0.75 and the native
      posture confirmed, 450 ms repeat, 1.5 s expiry).
- [ ] Camera bob: expose a toggle (Overlord ships it ON; for a new port default OFF is the safe comfort
      choice - say so explicitly), and make the weaponless case work.
- [ ] No camera shake/view kick; recoil only moves the gun. DOF/motion blur off by default; blur effects
      converted or removed per eye, never double-applied.
- [ ] Never tilt the horizon from scripts (roll comes from the head). Roll/horizon metrics are computed
      (`horizon_roll_degrees`) for diagnostics.
- [ ] Cutscenes: head free wherever possible (`free`/`aligned`/`authored` with entry alignment); hard cuts
      are native cuts - Overlord adds **no** comfort fade/vignette of its own (a port may add one; none is
      shipped). Cinematic pause/loading uses a dark compositor backdrop.
- [ ] Attenuate scripted translation (gain 0.1, 5 cm) instead of hard-locking position.
- [ ] Seated/standing: recenter sets the full position **including height**; heading from the horizontal
      forward; hold the menu-recenter button ~1 s (tap = pause).
- [ ] Left-handed mode (`pose_mirror.hpp`; mechanical hardware stays on its real side, only posture mirrors).
- [ ] Recording/streaming guide frame (`recording_frame.cpp`), desktop eye/FOV choice, desktop
      stabilization (reserves 15% of tangent extent).
- [ ] Options: aim assist, quick reload, recoil, melee damage, throw speed, world scale, cheats; all
      persisted in the game's own config and shown in the launcher.
- [ ] Every comfort/blur/DOF claim is labelled *confirmed / decision / pending HMD*; CPU/mock tests never
      establish headset acceptance.
