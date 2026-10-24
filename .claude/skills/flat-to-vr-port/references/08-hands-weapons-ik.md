# 08 — Hands, arm IK, weapon pose, firing and physical mechanics

Overlord files: `gameplay/hands/component.cpp` (skeleton hook), `hands/pose_solver.hpp`
(two-bone IK), `hands/pose_math.hpp`, `pose_library.hpp`, `pose_mirror.hpp`,
`hands/service.cpp`, `weapon_render_pose_component.cpp`, `viewmodel_visibility.cpp`,
`weapon_interaction.cpp` (fire + recoil adapter), `aim_assist.cpp`, `weapon_recoil.hpp`, `weapon_registry.cpp`,
`weapons/<name>/*.hpp`, `families/ar.hpp`, `physical_reload_runtime.cpp`,
`hand_interaction/runtime.cpp`, `interaction_coordinator.cpp`, `weapon_carry_runtime.cpp`.
Docs: `vr-weapon-interaction-architecture.md`, `vr-hand-interaction-redesign.md`,
`vr-hand-interaction-runtime.md`, `vr-weapon-registration.md`, `vr-weapon-carry.md`,
`vr-grenades.md`, `vr-melee.md`, `vr-weapon-recoil.md`, `vr-aim-assist.md`.

## 1. Driving the viewmodel from the controller

Hook the engine's **skeleton calculation** for animated objects (H2 `0x140657D80`):
```cpp
void calc_skeleton_stub(Object* obj, uint32_t* requested_bones) {
  if (in_hook || !is_viewmodel_or_owned(obj) || !weapons_allowed()) return original(obj, requested_bones);
  in_hook = true; scope_exit reset{[]{ in_hook = false; }};
  // 1. ask the engine for ALL bones (not just requested) so the solver has the full chain
  uint32_t all[8]; copy(requested_bones); set bits for every bone; original(obj, all);
  // 2. solve transactionally into a scratch array; write ONLY if the whole solve succeeded
  if (solve(rig, obj->matrices, targets, shoulders, body_axis, holding_hand, scratch))
      copy(scratch → obj->matrices);
  // 3. publish the solved muzzle/laser bones for the fire path and the render binding
}
```
Inputs to the solver:
- **Weapon target**: controller grip pose (calibrated wrist) × the authored
  *wrist-to-weapon* transform (measured from the native animation's bind pose) →
  where `tag_weapon` must be.
- **Support hand**: when the off-hand grips a foregrip/handguard region, the gun's
  orientation is solved from both hands (rear hand position + support direction).
- **Shoulders/body**: estimated from the HMD (head-locked or inferred body yaw)
  (`body_pose.hpp`).
- **Fingers**: native finger transforms survive as wrist-relative rotations. Grip
  poses (trigger index curl, magazine pinch) come from a pose library.
- **Native schema**: bone names and indices are bound once per asset generation
  (`native_schema.hpp`, `rig_builder.hpp`, `rig_cache.hpp`) and rebound when
  assets reload.

### Two-bone arm IK (`pose_solver.hpp: solve_limb`)
```
d = |wrist - shoulder|; allow stretch up to 1.25× (scale both segments)
reach = clamp(d, |U-L|+ε, U+L-ε)
dir = unit(wrist - shoulder)
pole = elbow_pole(dir, body_axis, hand)  // mirrored outward/down rest bend, swing-transported
        (or the projection of an elbow hint onto the plane ⟂ dir)
along = (U² - L² + reach²) / (2·reach)
elbow = shoulder + dir·along + pole·sqrt(U² - along²)
```
Orient both segments through one oriented bend plane (`orient_arm`) to avoid
360° elbow spins. Validate everything: finite values, orthonormal body axis, sane lengths.

### Visibility
Hide the native arms when they make no sense (no weapon, scripted body), hide
the gun's world-model duplicates, and show empty hands as a separate hands-only
object (`empty_hands_native`, `viewmodel_visibility.cpp` hooks the hidden-bone,
rigid and skin submission sites).

## 2. Firing from the muzzle

Wrap the engine fire function (H2 `0x140518A40`) and its shot-parameter builder (`0x140518880`):
```cpp
void fire_stub(Entity* e, int ev) {
  if (e != local_player || !vr_firing_enabled) return original(e, ev);
  muzzle = current_muzzle();                 // from the solved tag_flash bone (published by the skeleton hook)
  input  = controller_input::latest_interaction();
  if (!gameplay_active || !scripted_control::firing_allowed || stale(input, 150ms)
      || !input.grip[hand].valid || weapon mismatch) { ++rejected; return; }   // drop the shot, don't fire from the eye
  if (!trace_clear(eye → muzzle)) { ++obstructed; return; }  // a gun poking through a wall must not shoot past cover
  shot.geometry = {muzzle.position, muzzle.forward};
  aim_assist::apply(shot.geometry, strength);   // optional, bullets only
  active_shot = &shot;                          // thread_local; scope restores the previous one
  original(e, ev);                              // native damage/ammo/effects
  recoil::confirmed_shot(...);                  // VR recoil = visual/weapon kick from WeaponDef values
}
void parameters_stub(Entity* e, Params* p, int ev) {
  original(e, p, ev);
  if (active_shot && active_shot->entity == e) p->shot = active_shot->geometry; // every pellet/penetration segment
}
```
- Optional: remove hip-fire bloom by replacing the spread argument with the ADS
  baseline at the bullet call (`bullets_stub`). Keep intrinsic shotgun spread.
- Projectiles and grenades: same origin/direction override at their dispatch calls.
- ADS: there is no "ADS button" in VR. Auto-ADS when the sight is near the eye
  (`controller_ads.hpp`) lets native ADS accuracy and optics apply.

## 3. Weapon mechanics (physical reloads)

Original reload animations are **not authoritative**. A mechanical state layer is:
```cpp
struct ModuleState { FeedState feed; int feedRounds; ChamberState chamber; ActionState action;
                     InteractionState interaction; SyncState sync; uint64_t revision; };
```
- Invariant: `native loaded ammo == feedRounds + (chamber loaded ? 1 : 0)`.
  Never create rounds. Partition the existing native total on admission.
- Pooled magazines: removing a magazine returns its rounds to native reserve.
  Drawing a fresh one reserves up to capacity. Insertion transfers; cancel
  refunds exactly once.
- Empty action per profile: `LockOpen`, `StayClosed`, `OpenBolt`, `ManualCycle`.
- Transition table (shot, final shot, magazine out/in, action release,
  charging, shell insert, module select): each step requests an engine transaction
  and becomes committed only after the adapter confirms it.
- External mutation (pickup, checkpoint, script give/take) → pause and
  reconcile from a fresh native read.
- `BUTTON_RELOAD` injection is only a fallback for unsupported weapons.

Per-weapon recipe folder (`weapons/m4/`): `profile.hpp` (assembly, attachments),
`poses.hpp` (hand poses), `magazine_contacts.hpp` (well position/axis, seat
depth), `magazine_grasps.hpp`, `handle_poses.hpp`, `mechanics.hpp`
(family rules), `reload_interaction.hpp`, `feedback.hpp` (sounds, haptics),
`reload_profile.hpp` (registration). Families share rules (`families/ar.hpp`).
Registration is explicit (`weapon_registry.cpp`). Admission requires the full
native scene, attachment topology, capacity and ammo cells to match. A name
match alone is not enough.

## 4. Hand interaction arbitration

One system decides who owns each hand each frame:
- `InteractionFrame` (one calibrated input + body frame), `TargetHandle`
  (instance + generation), `Affordance` (touchable surface + input
  requirements), `InteractionSession` (an established grasp), `HandComposition`
  (which sessions may coexist on one hand, e.g. knife + magazine),
  `HandPosePlan` (one wrist target + finger recipe per hand).
- Batch order: report settled relations → collect proposals from domains
  → resolve once in the arbiter → settle grants per declared phase →
  reconcile handoffs → publish feedback.
- Avoid cross-module "is this hand busy?" queries and order-dependent
  presenters overwriting each other's bones.

## 5. Carry, holsters and body slots

Native inventory stays authoritative. VR assigns each owned weapon to a hand,
body slot (hip/back/chest) or overflow queue. Switching to the grabbed weapon uses
the native immediate switch (`switchtoweaponimmediate` through a script call)
and then cancels only the leftover raise timer. Body slots are positioned from an
estimated body anchor (head-relative, with an optional head-locked mode).

## 6. Grenades, melee, equipment

- Grenades: grip from a chest slot, pull the pin with the other hand (commits one
  native ammo debit), release with the tracked velocity → native projectile.
  No native throw animation.
- Melee: detect a physical swing (velocity, contact with an enemy volume) →
  submit a native melee hit with that target. Native damage decides the result.
- Night vision and other head equipment: hand-to-head gesture → native toggle.

## 7. Minimal path for a new game

1. Skeleton hook: move `tag_weapon` to the controller with a fixed offset; ignore arms.
2. Fire hook: muzzle origin and direction.
3. Two-bone IK for the arms; hide the arms if it looks wrong.
4. Support-hand two-handed aiming.
5. Only then physical reloads, one weapon family at a time, with the ammo invariant tests first.
