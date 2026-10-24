# 10 — Comfort, scripted scenes, vehicles and mounted weapons

Overlord files: `gameplay/campaign/scripted_sequences.cpp`, `campaign/sequences/<map>.cpp`
(airport, breach, estate, favela, gulag, oilrig, rappel, roadkill, …),
`campaign/sequences/camera_policies.hpp`, `gameplay/scripted_camera_reference.cpp`, `vehicles/`, `mounted_turret.cpp`,
`ladders`/`ladder_runtime.cpp`, `campaign/cliffhanger/*` (climbing),
`native_scripted_control.hpp`, `scripted_body.hpp`, `fixed_sniper.cpp`,
`notebook_runtime.cpp` (remote missile camera), `sentry_runtime.cpp`.
Docs: `vr-scripted-sequences.md`, `vr-scripted-control.md`, `vr-scripted-free-look.md`,
`vr-scripted-arms.md`, `vr-vehicles.md`, `vr-mounted-turret.md`, `vr-ladders.md`,
`vr-cliffhanger*.md`, `vr-player-death.md`, `vr-official-cheats.md`.

## 1. Classify every non-ordinary moment

For each mission, list the moments where the native game takes control: player
linked to an animated rig, `disableweapons`, vehicles, turrets, cameras on
remote objects, cutscenes, death, slow motion, and forced look-at.
For each, record:
- **identity**: how to detect it exactly (linked entity, animation name, script
  flag, vehicle type). The map name alone is never enough.
- **camera policy** (06 table),
- **input policy**: movement allowed? turning allowed? weapons? which buttons
  map to the scripted actions (e.g. melee/brake prompts),
- **hands policy**: hidden, scripted arms following the native animation, or
  free hands,
- **exit**: how control returns (restore command heading once).

Scene adapters publish small copied snapshots (epoch, stage, flags). The camera
and input cores consume them generically.

## 2. Native permission boundaries

- `disableweapons`/`enableweapons` intervals gate VR firing, carrying and
  hands (`scripted_control::predicted_allowed`).
- Death releases grips and hides hands.
- Script waiters listen for *binding notifications*, not command bits. Send
  the native `+gostand`/`+sprint`/`+melee` notify when the VR gesture fires.

## 3. Vehicles

Head rotation stays free. Native vehicle yaw changes rotate the tracking heading by
the same relative amount. Steering can come from a physical handlebar/wheel grab
plus the stick (`vehicles::controls`). Throttle is merged with the stick. Driver
weapons can be drawn from a chest rig.

## 4. Mounted turrets / fixed optics / remote cameras

- Turret: the hands grab the handles; turret yaw/pitch follow the hands, not the head.
  Camera policy: free head, seat-relative translation.
- Fixed thermal sniper/optics: native optic owns rotation (fixed). Show it on a
  panel or as a scope.
- Remote missiles/UAVs: camera = remote view with the head as a delta, aim from
  stick plus head (`apply_remote_control`).

## 5. Climbing, ladders, mantling

Ladders: the movement owner switches to hand-over-hand. Each hand grabbing a rung
moves the player by the hand delta (`ladders::owns_movement`, camera origin
adjust). Ice-pick climbing in Cliffhanger is a full physical-locomotion carrier.

## 6. Comfort checklist (defaults)

- [ ] Snap turn on by default, smooth turn optional (speed, deadzone).
- [ ] No camera bob, shake or view kick. Recoil only moves the gun.
- [ ] DOF, motion blur and fullscreen blur off in VR.
- [ ] Horizon never tilts from game scripts; physical roll only.
- [ ] Cutscenes: fade in/out at hard cuts, and keep the head free where possible.
- [ ] Vignette on fast artificial motion (optional).
- [ ] Seated/standing height offset, recenter on a long press.
- [ ] Left-handed mode (mirror poses: `pose_mirror.hpp`).
- [ ] Recording/streaming guide frame (`recording_frame.cpp`), desktop eye/FOV choice.
- [ ] Options: aim assist, quick reload, recoil, melee damage, cheats.
