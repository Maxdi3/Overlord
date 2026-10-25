# 08 — Hands, arm IK, weapon pose, firing, physical mechanics, interaction

Phase 7 of the port. Do not start before stereo, head tracking and command injection are solid
(refs 05-07). This is the largest subsystem in Overlord. Read section 0 first; sections 1-4 are the
minimum viable "gun in my hand" path, sections 5-9 are the layers that make it feel physical.

Paths: unless stated, files are under `src/client/component/vr/gameplay/` (written `gp/`).
Scene-model files live in `src/client/component/` (`scene_models.*`, `scene_rigid_part.hpp`).
H2 addresses and offsets are examples only (appendix at the end); never reuse them on another build.

## 0. Layer model and the rules everything below obeys

| Layer | Owner | Overlord files |
|---|---|---|
| Native state: ammo, inventory, damage, scripts, animation timers | the engine (single writable owner) | adapters `gp/native_*.cpp` |
| Mechanical projection: chamber, magazine, held rounds, action state, hinge/cover/lever progress | a pure planner + server-thread adapter; compare-and-commit against native | `gp/detachable_magazine.hpp`, `tube_feed.hpp`, `cylinder_feed.hpp`, `break_action_feed.hpp`, `belt_feed.hpp`, `closed_bolt.hpp`, `open_bolt.hpp`, `manual_bolt.hpp` |
| Intent: who holds what with which hand | one arbiter, simulation thread only | `gp/hand_interaction/*`, `gp/interaction_coordinator.cpp` |
| Pose: wrists/fingers/arms/gun | skeleton hook, render thread | `gp/hands/*`, `gp/weapon_render_pose_*` |
| Presentation: haptics, sound, HUD, guides, FX | derived, never creates gameplay state | `gp/weapon_feedback.*`, `weapon_hud_*`, `chambering_guide_*` |

Rules (each was a real bug somewhere in Overlord):

1. **Native owns the totals; VR owns a validated projection plus escrow.** (The "VR keeps no second
   copy" wording in SKILL.md is shorthand.) VR does hold `held_rounds`, live/spent chamber masks,
   belt/cover state, but every change to native ammo is a compare-and-swap against the engine's own
   cells, on the server thread, and any divergence faults the projection instead of guessing (section 6).
2. **Presentation never feeds back into mechanics or input.** Snapped/constrained hands, ADS comfort
   translation, recoil, smoothing and look-ahead are render-only. Raw controller poses drive contact tests
   (`tracked_wrist(..., raw)` vs stabilized visual wrist, `gp/hands/position_offset.hpp`).
3. **Never reuse an authority gate for presentation.** `pose_ready` (holding hand, grip identity, tracking
   and camera freshness, geometry) vs `ready = pose_ready && owner.can_fire()`. A foregrip-only carry lost its
   held magazine when the render path used the firing predicate (`docs/vr-foregrip-reload.md`).
   Render-only identity checks must be *looser* than authorization checks (`gp/weapon_render_owner.hpp`
   compares only the fixed rear-carrier identity; an inserted-magazine blink across 92 assemblies came from
   comparing the full owner incl. support grip revision at skin time).
4. **Identity = definition token + instance generation + revision; never a name or a bare token.**
   `gp/weapon_identity.hpp {token, generation}`; `hold` has `revision` (any topology change) and
   `rear_revision` (changes only when the *control* hand changes; "support changes must not rearm the firing
   hand"). Entities are keyed `{entity, generation}` (generation array bumped in a hook on the engine's
   free-entity function, `gp/native_carry.cpp`). Every deferred action re-validates its key.
5. **Fail closed, with a reason string.** Wrong prologue bytes, unregistered receiver, duplicate script
   relation, unknown attachment, mixed-bone triangle: feature off, native behaviour intact, one console line.
   Every subsystem has a `vr_*_status` command that prints counters and the last decision string and writes
   `minidumps/overlord-*.txt` atomically on the async pipeline (ref 11).
6. **Freshness idiom.** Every cross-thread sample carries `reference_generation`, `continuity_generation`,
   `sequence`, `sampled_at`. Consumers reject `now < sampled_at`, age > 150 ms (100 ms for feedback events and
   visuals tied to the latest hand), or any identity/revision mismatch. Counters (`presses`, `releases`,
   `generation`) not booleans, so a short tap and a resumed-held button both work. Per-subsystem values:

   | Value | Where |
   |---|---|
   | 150 ms | pose/input freshness for fire, muzzle, carry, support, scripted arms, render caches, attachments |
   | 100 ms | feedback events (`weapon_feedback.hpp`), part-hand look-ahead bounded to 50 ms |
   | 200 ms | `grab_intent` press-before-arrival memory (`body_equipment.hpp`); melee history reset on calibration change |
   | 250 ms | FX refresh/queue expiry, dropped-item preview handoff (250 ms) |
   | 300 ms | magazine insertion guard after an impact release; belt cover-push cooldown |
   | 400 ms | player-wide melee hit gate |
   | 1.0 s | holster dwell for quick reload (`quick_reload.hpp`); 1500 ms native-selection retry |
7. **Thread/pipeline discipline.** `scheduler::pipeline::server` (game tick: all VM access, entity writes, ammo
   commits; guard with `scheduler::is_executing(server)`, not a sticky thread id), `main` (assets, FX, sound,
   250 ms slow prep), `async` (file writes). Render/eye callbacks read copied snapshots and never call the engine.
   Never hold a mod lock across native work: copy under a short lock, format/write outside. Cross-thread
   hand-off is a small mutex-protected *published* struct plus generation/epoch.
8. **Idempotent per eye and per prediction replay.** Skeleton hook, hand solve, render bind and gesture update
   run once per native pose/input `sequence` even though both eyes (and prediction replays) ask again.
9. **Edge/arming rules.** A press only counts if the button was seen up (`armed`); entering a region with the
   button already held is not an acquisition; a failed native write *consumes* the edge (no retry from the same
   squeeze); tracking loss or focus loss is never a release (retain state, cancel pulls, defer release);
   a proven physical release ends the session even when pose is lost; per-hand admission before edge
   consumption ("a missing opposite hand must neither swallow this hand's release nor replay it on recovery").
10. **Bounded everything.** Fixed arrays/rings (15 weapon instances, 128 pose ring, 4096 tickets, 8 sessions /
    64 candidates). Overflow rejects new work; it never evicts live state or lets truncation pick a winner.
11. **Evidence ladder.** Every weapon README opens with a status line (offline candidate / default-enabled /
    HMD-accepted + date) and a "Verification boundary" section. Offline tests and CPU mocks never imply HMD
    acceptance. Poses are tuned by headset feedback loops; keep dated tuning logs (`weapons/m9/README.md`).

## 1. Skeleton hook: driving the first-person skeleton from controllers

Hook the engine's **skeleton calculation for animated objects** (H2 `0x140657D80`, 19-byte masked prologue
check) and overwrite bones after the engine computed them. Overlord: `gp/hands/component.cpp`
(`calc_skeleton_stub`, ~2200 lines), `hands/pose_solver.hpp`, `rig_builder.hpp`, `rig_cache.hpp`,
`native_rig.hpp`, `native_schema.hpp`.

### 1.1 Hook skeleton (what the real stub does)
```cpp
void calc_skeleton_stub(Object* obj, uint32_t* requested) {
  // 0. dispatch by owner; first match wins. Foreign objects: just original().
  if (scene_models::skeletal_model::owns(obj))   { original(); apply(obj); return; }  // mod-created scene props
  if (ending::owns_pose / scripted_arms::owns / mounted::owns_model) { /* each: all bones, original, apply(obj, rebuilt) */ }
  empty = empty_native::owns(obj); independent = owned_native::owns(obj);
  if (in_hook || !alive || !obj || !requested ||
      (!empty && !independent && obj != client_object(primary_viewmodel_handle))) return original();
  // second viewmodel present (dual wield / special prop): conservative reject, never drive half
  in_hook = true; scope_exit reset;                       // thread_local re-entrancy guard
  lock(per_object_solver_mutex);                          // serializes solver contexts ONLY; caller owns the DObj lock
  select solver context: empty_solver | native_solver | owned_solvers[i] keyed by object
  if (same (object, matrices ptr, timestamp/epoch, resource gen, asset gen, model ptrs,
            duplicate_parts, counts, calculated bits) as last call) return original();   // epoch latch
  ... validate tracking (150 ms, spatial.generation == input.reference_generation), build rig, targets ...
  all[] = requested | every bone bit;  original(obj, all);                                // FULL bone request
  verify (obj->calculated[w] & all[w]) == all[w] for every word, else reject "incomplete skeleton"
  solve transactionally into scratch; copy to obj->matrices only if whole solve succeeded
  publish muzzle/laser/hand solves (section 3); remember(epoch) in a scope_exit (success AND rejection)
}
```
Requirements, each with its reason:

- **Request all bones, then verify the calculated bits.** Native callers ask for partial bone sets (muzzle query,
  render). A later child calculation would otherwise "inherit an already corrected parent twice". The bit array
  is 8 x 32-bit words (256 bones), MSB-first (`0x80000000 >> (i % 32)`).
- **Epoch latching includes rejection.** The solver remembers `(object, matrices, timestamp, resource gen, asset
  gen, model pointers, duplicate_parts, model/bone counts, calculated bits)`. If identical, call original and
  return, success *or* rejection. "Tracking recovery between partial queries must not change one eye only."
  A recreated hands-only DObj reuses storage, so its own lifetime counter is part of the key.
- **Binding cache, not per-frame string matching.** `binding_identity()` fingerprints raw pointers
  (names/bind/boneinfo/material/surface), bounds as exact bits, counts, an asset epoch (bumped by the
  pre-unload observer so recycled addresses never match) and admission bits. `rig_binding_cache` stores
  accept *and* reject. Rebuild only on fingerprint change.
- **Per-object solver contexts.** `native_solver`, `empty_solver`, `owned_solvers[N]` (one per mod-owned object).
  Each carries last-epoch key, binding, twist state, ADS transition, grip presenter. The solver mutex serializes
  contexts only; no native call from another component may wait on it. Context reset on asset or resource change.
- **Rig resolution by semantic bones inside the first-person DObj, not by model names** (`rig_builder.hpp`):
  hand model = unique `j_shoulder_{le,ri}`, `j_elbow_*`, `j_wrist_*` + `tag_weapon`; receiver = unique `j_gun`,
  muzzle `tag_flash`, optional `tag_laser`. Kinds: firearm, hands_only, shield, melee, scripted_body.
  Merge parent lists from each XModel's `parentList` (distance-to-parent encoding), root bones attach through
  the DObj attachment table (255 = none), apply the `duplicate_parts` alias table. Reject: multiple hand rigs,
  multiple gun roots, non-topological parents (parent must precede child), overlapping arm/weapon ownership,
  extra models not rooted under the hand model, > 254 bones, composite flag.
- **5-degree muzzle admission.** The muzzle bone's bind-space forward axis must satisfy `dot >= 0.9961947`
  (cos 5 degrees) with a positive offset > 0.01, unless a reviewed per-weapon contract exists
  (`native_muzzle_contract.hpp`; H2 javelin base has a 10.3-degree axis, validated to 0.1 deg / 2 cm). Reason:
  firing along a mis-modelled muzzle axis is worse than not firing. "A hidden prop cannot supply a firing origin."
- **Side-suffix matching at token boundaries.** Searching for `_ri` also matched `ring` and produced duplicate
  `j_leng*` bones (M200, M1887, shield): match `_ri_` / `_ri` exactly. A hand model missing two optional leaf
  bones (`j_webbing_*`) rejected the whole profile and dropped the held gun: allow exactly the named optional leaves.
- **Transactional write.** Validate finite values, unit axes, topology (shoulder<elbow<wrist, no overlap between
  arms and weapon), write into scratch, copy once. Child subtrees move rigidly (`transformed()`: wrist subtree by hand
  delta, elbow subtree by lower-arm delta, shoulder subtree by upper delta). Gun/weapon bones follow the rear-grip wrist.
- **Build mode is not a feature switch.** The skin-submission hook that prepares hands must exist in every build:
  a Debug-only install gate made optimized clients fall back to native first-person rendering (foreground
  depth, equip/ADS visibility) (`docs/vr-empty-hands.md`).

### 1.2 Solver inputs
- **Wrist target** = tracked controller grip pose + a **grip-local rigid offset to the physical wrist**
  (`position_offset.hpp`): `wrist = grip_pos - scene_view_offset + R_grip * offset`, with the inward sign flipped
  per hand, scaled by `units_per_meter`, clamped to `max_hand_offset`. The offset must be *grip-local*: rotating
  about a real wrist moves the controller origin and `R_grip * offset` cancels exactly that; a fixed world offset cannot.
  Position calibration never touches aim calibration, the native empty-hand orientation basis, authored fingers or the
  wrist-to-weapon transform. Defaults are zero ("None" preset). Tuning instruction for users: rotate your hand about the
  physical wrist; the virtual wrist must stay still. Negative result worth knowing: estimating an OpenXR "palm" from
  model wrist+knuckle midpoints was rejected in the headset, and SDK joints from controllers are simulated, not
  measured (`docs/vr-hand-alignment.md`). Do not report automatic anatomical calibration.
- **Weapon placement**: the rear wrist is authored in gun-local space (section 7.4); the gun root is solved from the
  rear wrist; with a support hand on a foregrip, `aimed_rotation(two_hand, rear_rot, rear, support, authored_span)`
  swings the authored grip-to-grip vector onto the real hand-to-hand vector, keeping rear-hand roll (no world-up
  singularity, no assumption the foregrip lies on +X; skip if either span < 0.5 units).
- **Shoulders** are estimated from the HMD, yaw-only body frame: half width 0.18 m [0.10-0.35], down 0.20 [0.05-0.45],
  back 0.08 [-0.10-0.30] (dvars). A body estimator (`vr_bodyPoseMode` inferred | head_locked: neck pivot 8 cm behind the
  eye, 7 cm tolerance then 120 ms follow, 35 deg yaw allowance, history reset on recenter / gap > 150 ms / jump > 75 cm)
  feeds holsters and chest slots; never infer torso yaw from hands.
- **Native view offset**: native global bone translations include the render-view offset (H2: vec3 at view + 0x58);
  subtract it going in, add it coming out. One copy of tracking reference + camera origin + world scale per solve.

### 1.3 Two-bone arm IK (`hands/pose_solver.hpp: solve_limb`, `orient_arm`, `elbow_pole`)
```
d      = |wrist - shoulder| (reject non-finite or > 200)
stretch= clamp((d + .001)/(U+L), 1, 1.25)            // both segments scaled; max_arm_stretch_ratio = 1.25
reach  = clamp(d, |U-L|+.001, U+L-.001)
dir    = unit(wrist - shoulder)   (d~0: body forward)
pole   = elbow_pole(dir, body_axis, hand)             // rest bend = unit(left*(+-0.6) - up), SWING-transported
                                                      //   body-forward -> dir (exact 180: the bend itself is the half-turn axis)
         or unit(projection of an explicit elbow hint onto the plane perpendicular to dir)
along  = (U^2 - L^2 + reach^2) / (2 reach)
elbow  = shoulder + dir*along + pole*sqrt(max(0, U^2 - along^2));   wrist' = shoulder + dir*reach
```
- Validate inputs: lengths in (0.1, 100), body axis orthonormal to 0.001, right-handed.
- `orient_arm` carries *both* segments through one oriented bend plane (`arm_plane_normal`, fallback to the authored
  shoulder frame when straight). Independent shortest-arc rotations spin an elbow 360 degrees while the hand barely moves.
- Projecting an animated elbow or a fixed world hint directly is singular when the hand points along the hint; swing
  transport is not, in the front/side/overhead domain.
- No stretch on skinned full bodies (scripted arms use `stretch = 1`; "reach stretching pulls the arm/chest seam apart").
- **Arm clearance** (`arm_clearance.hpp`, belt-fed guns): swivel search +-24 steps of 5 deg around the shoulder-wrist axis
  minimizing penetration of a box obstacle (11 samples along elbow-wrist) + posture preference + smoothness; never "escape by
  lifting the elbow"; 6 rad/s rate limit; 0.10 s contact blend; clavicle compensation moves the virtual shoulder up to 0.06 m
  (0.6 m/s) when fully extended and still blocked. Visual only; recomputed each frame from the base pose, never feeds input.
- **Forearm twist** (`forearm_twist.hpp`): two-bone IK leaves "wrist twist" deformation bones (siblings of the wrist, children
  of the elbow, weights on thousands of vertices) unrotated, so the cuff does not follow wrist roll. Finalization pass reads
  the FINAL anatomical wrist after all weapon/equipment/empty-hand overrides (all three commit paths share it), takes the
  residual twist about the elbow-wrist axis (swing/twist decomposition against the elbow-transported bind basis), applies the
  *full* axial rotation to the helper (skin weights distribute it), excludes flexion, keeps the last twist when the forearm
  axis is undefined near 180 degrees, and resets on reference/model change, rejected or untracked pose, gap > 150 ms. Bind helpers by
  semantic name + parent + model ownership, never synthesize bones from animation channel names.
- **Bar/foregrip grasps** (`bar_hand_pose.hpp`): derive the grasp centre from posed *finger joints* (partial wrap curls) and
  move the wrist around it; no guessed world offset; applied after weapon posing so only the selected arm branches change.

### 1.4 Mod-owned skeletons (empty hands, per-weapon objects, stowed weapons)
- A native DObj with no animation tree has collapsed joint translations. For **empty hands**, create the mod's own
  hands-only DObj via the engine's object-create function, seed from the model's full *bind pose* rebased at the native
  view anchor (`tag_view`) + current head position, wrist basis = `inverse(weapon_tag_bind_rot) * wrist_bind_rot`, then run the
  same arm IK with absolute wrist targets (`gp/empty_hands_native.cpp`, `docs/vr-empty-hands.md`). Borrow the current
  character's hand model, never hard-code it.
- **Per-weapon objects** (`hands::owned_native`) for dual wield, holsters and drops: one DObj per carried weapon (hands model +
  gun model). Skin each on the engine's frontend boundary with the engine's own pose function
  (`lock -> calc_pose(all bits) -> unlock`), accept only if the skeleton hook acknowledged that exact `(matrices, epoch)`
  (`solved_matrices == matrices && solved_epoch == epoch`), then submit via the engine's viewmodel submit. Failed solve: not
  drawn ("tracking failure never renders a bind pose"). Recreate only on asset change, after `R_SyncRenderThread`.
- Scene entity ids: reserve ids beyond the ordinary entity range (H2: 3999 empty hands, 4000+ stored weapons, capacity 4032) with a
  `static_assert`. Native firearm viewmodels are suppressed by wrapping the add-view-weapon function with thread-local flags and
  classifying the submitted object by its model list vs personal-firearm WeaponDefs (bounded BFS through composite models).
- **Per-object hide tags**: each owned DObj copies bone-hide bitmasks from *its own* WeaponDef `hideTags` bound against its
  assembled bone-name list (not from the selected weapon). An independent DObj with an empty mask shows both sight states.
- **Stowed (holstered) weapons**: the hook poses a non-held instance at a holster pose by writing matrices and hiding all
  non-weapon bones through the native hidden-bone bit array; no hand/fire authority.
- **Selection transition**: while the native model still shows the old gun, match the DObj receiver to the held instance whose
  WeaponDef models match, keep tracked hands for that old owner, and publish no muzzle/fire authority ("visual continuity is not
  firing authority").
- **Gestures for empty hands**: grip/trigger -> open/point/pinch/fist, time-based blend `w = -expm1(-ln2*dt/0.045)` (about 90% in
  150 ms), reverse transitions continue from current progress, frozen on tracking loss, never advanced twice by two eyes.
- **Scripted full-body arms** (`scripted_arms_*`, `scripted_body.cpp`): run after the primary hook completed all bones; `wants_pose`
  only when the requested bit mask includes an arm bone (so tag queries cannot pre-complete the skeleton before native
  controllers run); `once_per_pose{object, matrices, timestamp}`; blend LOCAL transforms in parent space after IK; authority
  from native animation weight (0 while any authored clip has weight, smoothstep of the takeover clip's normalized time,
  200 ms envelope), never a wall timer or a transient flag.
- Wrap IK output failures conservatively: scene-prop and multi-viewmodel cases fall back to `original()`; the muzzle mailbox
  is invalidated (`invalidate_muzzle`) and a support grip is dropped when the hook cannot publish.

## 2. Hand and finger posing from authored data

- Finger poses are **parent-local quaternions** per joint name (18 joints per hand: 15 finger segments + `j_pinkypalm`,
  `j_ringpalm`, `j_webbing`), applied through a `pose_library` (`hands/pose_library.hpp`, `pose_math.hpp`, `pose_schema.hpp`).
  Only rotations are stored; live glove segment lengths stay authoritative, so different hand models fit.
- **Manipulation poses must override all 18 joints of the free hand**, otherwise the palm leaks from the previous pose.
  Unanimated webbing uses the normalized reference-bind rotation.
- **Mirroring for the other hand** (`hands/pose_mirror.hpp`): reflect across the gun's XZ plane
  (`position: y -> -y`, `rotation: (x,y,z,w) -> (-x, y, -z, w)`), then multiply by the hand's per-bone mirror basis
  (`library.mirror_basis`, `local_rotation(parent_basis^-1 * reflect(q) * bone_basis)`). Rigid objects held in a mirrored hand are
  re-derived with `object_in_wrist` (mirror the hand around the attachment, recover the object pose); never mirror or rescale the mesh.
  Right-hand finger rotations are never assigned to left joints. "Mirror the hand around the hardware, not its wrist": side-mounted
  handle contacts stay on their real side; only the hand posture is anatomically reflected. Where asymmetry matters, author both facings.
- **Finger blends**: grip transitions interpolate bone-local poses over `blend_seconds` (0.10 s default, or `clamp(t/0.12 s)` for
  hand-presented grips) so fingers never pop; part-hand transitions use a 90 ms smoothstep on a controller-*relative* offset so
  wrist motion continues immediately (`part_hand_transition.hpp`, `part_return_transition.hpp` advances the OLD segment to now
  before re-targeting).
- **Trigger discipline** (`trigger_discipline.hpp`, 46-receiver direction/curl table): the free index finger extends off the trigger
  using capacitive `trigger_touch`; weight smooths with half-lives 0.035 s (open) / 0.012 s (close); a pressed trigger wins over touch.
- **Secondary motion** (`secondary_motion.hpp`): spring-damper chain for dangling parts (<= 4 links, stiffness 90, damping 11,
  force gain 4, substeps `clamp(ceil(dt*240),1,12)`, joint limits +-1 rad, reset on dt > 0.05 / root jump > 0.25 m / regressing
  sequence, one step per input sequence).

## 3. Binding the solved pose to the exact render record

Problem: bones are solved on one thread (skeleton hook); the muzzle, HUD, optics, held magazines and laser must match the pose the
renderer actually draws, per eye, with no "latest pose" substitution. Overlord: `gp/weapon_render_pose_component.cpp`,
`weapon_render_pose_cache.hpp`, `weapon_render_pose.hpp`, `weapon_model_anchor.hpp`, `hands/attachment_pose.*`.

1. The skeleton hook **publishes a solved pose keyed by `(object, matrices, epoch)`** (H2: matrices at object+0xA8, epoch at +0xB0)
   with the muzzle (and laser) bone and the solve context.
2. Three call-site detours (each verified as `E8 rel32` to the expected target and reachable by rel32): **skin**, **submit**, **viewmodels**.
3. `skin_stub` reads the muzzle bone (and laser bone) from the matrices **before** the native skin, calls native exactly once, reads
   again, and requires **bit-identical bytes** (`memcmp`) both times: proof the skeleton was not recomputed between solve and skin.
   It then links `entity -> surface` (entity+0x68) and, via a thread-local record from the viewmodels stub, to the render record and its
   12-float camera.
4. `submit_stub` binds `record + camera + entity + surface + object` to a scene entry. A second conflicting bind marks the entry
   `conflict` and the pose is refused (`pending / ready / conflict`).
5. Lookup (`for_scene`) needs the complete **stereo pair identity** (both eyes' `pair_id`, `publication`, `output_eye`, equal natural
   camera). "A record address alone is not a frame ID" (addresses are reused, ref 05).
6. Consumers (HUD, optics, attachments, FX, magazine placement) receive a snapshot `{muzzle, control_grip: model_anchor}`; none resamples
   tracking. `model_anchor` stores MODEL coordinates plus the solve origin: never reconstruct the model position by subtracting a later
   view offset from a world pose.
7. `hands::attachments::publish` keeps a **128-entry ring of committed hand solves** (object, matrices, epoch, wrist bones, head in
   model-origin space, units, mirror bases, render origin, reload tokens). Render-time props find the solve for *their* record with
   `for_record(record, camera12)` (record pointer + exact camera matrix + <= 150 ms + `matches()` re-reading epoch and both wrist
   bones from memory), or `for_current(reference, lease)` for skeletal props evaluated before skin. "Body attachments share the exact skin
   record's body and origin; never combine its relative head with a later global camera origin."
8. Props are `omit`ted when the committed hand solve for their record is older than 150 ms or has another reference generation: the
   hand analogue of "never show the wrong image".

## 4. Showing mod-built geometry through the engine's own scene pipeline

Everything held in a hand that is not the animated viewmodel (magazines, shells, bolts, knives, grenades, chest items, rockets,
laptops, falling cases) is a separate model submitted through the native scene filter and placed *late*.
Overlord: `src/client/component/scene_models.{hpp,cpp}`, `scene_rigid_part.hpp`, `scene_skeletal_model.*`, users in
`gp/*presenter.cpp`, `reload_item_runtime.cpp`, `weapon_carry_runtime.cpp`, `grenade_runtime.cpp`, `chest_equipment.cpp`.

### 4.1 Submit coarse, place exactly
```cpp
// main/frontend, once per frame per part, at server/frame rate:
scene_models::submit(model, &coarse_placement, flags /*e.g. no_cast_shadow 0x40*/,
                     &lighting[slot], lit, unlit, emissive, cull_padding /*0.2-0.25 m in units*/);
// later, while the engine packs rigid surfaces for the render job (both eyes consume it):
placement_result on_prepare_placement(const preparation& job, const void* entry,
                                      GfxPlacement& current, GfxPlacement& previous) {
  // identify our entry: lighting-handle address stored by the engine at entry+0x68 -> slot/part by pointer arithmetic
  // frozen snapshot (submitted) vs live published snapshot: id, pose_revision, holder, state, reference gen, age <= 150 ms
  pose = hands::attachments::for_record(record, camera, ...)   // the EXACT skeleton skinned for this record
  current = hand_pose * part_in_hand + record's model-placement origin; previous = current (kills motion-blur ghosts)
  return replace | omit (mismatch/stale) | retain (owned, keep native) | unchanged (foreign entry);
}
```
- `submit()` wraps the native filter function (H2 `R_FilterXModelIntoScene`) and a generator/prepare/build triple hooked in
  `scene_models.cpp` (all call sites byte-verified). `register` callbacks in `post_unpack` only; competing replacements of one entry
  are *omitted*, not composed.
- **Identity token**: the engine retains the lighting-handle pointer (it "copies" the placement but keeps the address). Use a stable
  static array (`unsigned short lighting[N]`), never a reallocating vector; the slot address identifies the part.
- **One time sample per packing job**: `preparation.at` is shared by both eyes and all parts; free-flying items evaluate their analytic
  flight at `max(preparing.at, lease.at)` so pieces never diverge.
- **Frozen tickets**: `submission_pool<ticket, 2048..4096, 13>` freezes identity per submission; the callback looks up by entry, never "latest
  occupant of a slot". Falling items: lifetime 1.2 s, gravity `9.81*units`, rail departure follows the live attachment until its deadline then
  continues analytically ("a later simulation publication must not move an already displayed departure").
- **World-item handoff**: when the hand releases a world item keep submitting a preview until the engine's own submission for that entity
  (same id, generation, first model) is observed; expire at 250 ms.
- Extra culling padding for independently held models: per-submission radius (0.25 m) at the engine's frustum-admission call, scoped per
  model/thread and restored.

### 4.2 Creating the meshes from engine assets (no extracted art ships)
- `scene_models::rigid_part::create(source, bone)`, `create(source, anchor, bones)`, `create_face_partition(model, bone, face_ranges, include)`,
  `create_skin_partition(source, bones)`, `create_instances(...)` (<= 5 instances: body, follower, three rounds) build XModel descriptors from the loaded
  viewmodel; `register_runtime_models({descriptor, source})` makes the engine treat them as legal models (call once per immutable batch, on the
  main asset owner thread, only after the whole set validated).
- **Validate against fingerprints before building**: exact model name, bone count/names, per-surface `{vertex_count, triangle_count}`,
  bounds (+-0.02), LOD/surface counts; M67 grenade lever additionally an FNV-1a hash of the index buffer
  (`frag_triangles_hash`, "M67 lever topology changed" disables it). Reject duplicate bone names (bind by name, never cached indices or script
  string ids).
- **Lifetime**: retain every published descriptor until queued rendering drained: caches keyed `(source model ptr, surfaces ptr, profile ptr)`
  detect reloaded zones; release only after the pre-unload observer (`fastfiles::on_pre_unload`) bumps the asset epoch and nulls availability.
  Retry throttled (250 ms loop, 1 s per attempt). Exhausted caches close admission (restart required, reported in status); descriptors are never
  recycled while async skin jobs or both eyes may hold them.
- **Save/load**: if you invent engine asset objects that live particles or entities can reference, handle the serializer and the zone-unload barrier
  (Overlord names runtime FX `__overlord_fx/{world|follow|casing}/<hex>/<source>`, appends to the engine's FX name dictionary at the save-prepare hook,
  resolves at load, rejects the whole checkpoint if unmappable; `native_fx_checkpoint*.cpp`). IDs are append-only (one-based, never renumbered).
- **Baked deformation** for animated props (`cliffhanger/model.cpp`): decode the blend stream, bake N deformation steps (30 Hz keys interpolated to
  120 Hz) into immutable CPU-skinned meshes; never edit a queued GPU buffer or the source asset during playback.
- **Fail-closed asset surgery** (`spent_case_geometry.hpp`): recipe verifies face counts and extents, welds vertices, splits components, requires
  manifold edges and one closed mouth loop, builds a hollow interior; any mismatch returns false and the effect is skipped.
- **FX variants** (`native_followed_fx`, `native_fx_world_space`): copy the immutable effect def, flip `RUN_RELATIVE_TO_EFFECT` on chosen elements so flames
  follow a VR pose (the wrong enum pins flames to both eyes); clear the engine's "draw with viewmodel" element flag on shell-casing graphs (it maps to the
  scene depth-hack bit, so world-space casings drew in front of independently rendered guns).

### 4.3 Hiding parts of native viewmodels without editing assets
Selected per frame by `part_visibility { surface, rigid_groups, skinned_groups }` (`gp/viewmodel_policy.hpp`, `viewmodel_visibility.cpp`,
`rigid_part_visibility.hpp`, `skinned_part_visibility.hpp`):

| Mode | Mechanism |
|---|---|
| `surface` | hook the call that fills the native hidden-bone bit mask; OR in the module's mask |
| `rigid_groups` | hook the rigid surface-packet builder; replace the packet's surface pointer with an immutable metadata copy whose hidden rigid group has `triCount = 0`. `plan_rigid_visibility` verifies groups contiguous (`first_triangle == running`, sums equal totals, `boneOffset & 63 == 0`, <= 32 groups) |
| `skinned_groups` | hook the skin packet builder BEFORE CPU/GPU skin-job publication; `plan_skin_packets` validates the whole packet stream before any write; `partition_skin` parses the engine's blend stream, **fails if any vertex mixes selected/unselected bones or any triangle straddles**, builds a subset index buffer (new buffer + SRV); if all triangles go, keep one zero-area triangle so counts/strides stay legal |

Rules: never hide a whole shared surface to remove one moving part; variants immutable and never evicted until pre-unload; cache keys include a `memcmp` of
the original surface (asset swaps invalidate); per-frame masks live in a 128-entry ring keyed `(object, matrices, epoch)`. The mod's own stream parser must follow
the engine module's ABI too (a hard-coded 4-byte hidden stride rejected 8-byte streams and left the magazine visible). Other hide techniques: ADS-mesh subtree
masks (`tag_scope_ads_on`) so the physical scope stays; silencer cap bones by presence; animation-only duplicate magazine subtrees hidden only while physical ownership
is validated; equipment hidden by bone mask (vehicle/scripted bodies publish masks into the same pipeline).

### 4.4 Magazine/round "rigid-subset" fill recipes (`weapons/*/magazine_fill.hpp`)
Show the magazine as an independent held object with 0..N rounds by reusing the *receiver's own geometry* (the world magazine differs, fit error up to 5 cm):
per asset record source-model SHA-256, per-surface vertex/triangle counts (live runtime check), `surface_face_range {surface, first_tri, last_tri}` lists for body-only
and body+1/2/3 rounds (each higher state must contain the lower: `valid_magazine_fill`), a round-stack offset table (staggered cartridges), optional follower faces,
low/high AABB of the magazine body, `.bones = N`. "Never take a whole multi-round bone as one round." Only meshes rigid per bone group are eligible. `bolt_partition.hpp`
carves a disconnected bolt surface by face ranges + AABB when handle and bolt share one bone. `folding_handle.hpp` rotates only the end faces of a handle bone.
A falling spent-round subset hit a native resource-index crash until each subset registered its real loaded source identity; transient viewmodels are excluded from ordinary
scene counts, so use a record-scoped skin readiness check.

### 4.5 Per-eye overlays (guides, debug, laser, scope screens)
`eye_composition::register_consumer(present, layer)`: callback receives `{eye, pair_id, view-projection bytes, per-eye model origins, scene depth, device generation}`.
**On eye 0 build both eyes' draw lists** (verify each eye's pair id/output eye), store in a thread-local pair; **eye 1 draws only if pair/publication/device match and eye 0 drew**
(never half a pair). Depth-test against native scene depth; world beams keep native reverse-Z. Used by the chambering guide (opaque tinted mesh of the real action parts, 2 s pulse,
`tint = .10 + .08 wave`, phase reduced with `fmod` before `sin`), laser/designator beams (trace on the engine's frontend thread into a 16-entry ring, endpoint re-projected to the nearby
collision plane only within 0.5 units), the on-gun ammo HUD billboard (section 9.3), debug overlays and the fixed-sniper/Javelin screens (section 10.7). Failure policy: black or skip, never a stale or unmasked image.

## 5. Firing

Two paths exist; a port starts with the first.

### 5.1 Path A: shots through the engine's own fire event (selected weapon)
Wrap the engine fire function and its shot-parameter builder (H2 `0x140518A40` / `0x140518880`, call sites `0x1404AD1ED`, `0x140518B91`, bullets `0x140518C2E`, ADS-spread query
`0x140518BB9`; each masked-byte verified; failure disables only that feature). Overlord: `gp/weapon_interaction.cpp`, `shot_geometry.hpp`, `weapon_fire_delivery.hpp`.
```cpp
void fire_stub(Entity* e, int ev) {
  prev = active_shot; active_shot = nullptr; scope_exit restore;     // nested NPC events never inherit the muzzle
  if (independent_fire::owns_native(client)) return;                  // second path owns this weapon
  if (e != local_player || !firing_enabled) return original(e, ev);
  owner = sample_native_equipped();  input = controller_input::latest_interaction();
  if (!gameplay_active(input) || !scripted_control::firing_allowed || stale(input,150ms) ||
      !grip/aim valid for driver hand || !pose ready || entity_weapon(e) != owner.weapon ||
      delivery == unsupported)                    { ++rejected; return; }   // drop the shot; never fire from the eye
  if (!muzzle_path_clear(e, pose)) { ++obstructed; return; }                // gun poking through cover cannot shoot past it
  shot.committed_geometry = geometry(pose);            // exactly the first 0x30 bytes: forward, right(=-left), up, origin
  if (delivery == bullets) aim_assist::apply(shot.committed_geometry, strength, entity, route::projected);
  active_shot = &shot;  posture = native posture captured BEFORE native damage/scripts run
  original(e, ev);                                     // native damage, ammo, effects, scripts, notifies
  recoil::confirmed_shot(owner, reference, now, posture);
}
void parameters_stub(...) { original(...); if (active_shot && active_shot->entity == e) p->shot = active_shot->committed_geometry; } // every pellet/penetration
void bullets_stub(..., float spread, ...) { if (hip-fire-spread-removal) spread = engine_ads_spread(ps, weapon); original(...); }
```
- `muzzle_path_clear`: point-capsule `G_TraceCapsule` eye (origin + viewHeight) -> muzzle with the engine's bullet content mask (H2 `0x280e831`); refuse if
  `fraction < 1`, allsolid/startsolid, or eye-muzzle distance > 200 units; shield clearance first. The same helper serves independent fire, the underbarrel path and dropped-item physics.
- **Delivery classes** (`weapon_fire_delivery.hpp`): bullets (engine weapon type 1), native projectile (type 3: rocket/rifle grenade; native +attack stays, same muzzle geometry is
  injected at the shared parameter builder; guided launchers keep native target selection), thrown (type 2, different input contract), mod-defined `scripted_device` (laser designator).
  Aim assist applies to bullets only.
- **Trigger -> attack bit** (`controller_firing.hpp`, `controller_component.cpp`): fire only when `gameplay && muzzle_ready && owner.can_fire() && fresh input`; reset on owner id, rear
  revision, rear hand, reference or continuity change; both grip and aim valid for the *firing hand's* index; `discrete` mode additionally requires a new press. `+attack` (0x1) is OR-ed into the command; mechanical
  readiness (break-action closed, launcher, near-eye Javelin, scripted device) may veto and clear it. Request only, native PM keeps authority.
- **Hip-fire bloom removal** (`vr_disableHipFireSpread`, default on): replace the spread argument with the ADS baseline from the engine's own spread function, keeping intrinsic dispersion and shotgun pellets;
  a signature mismatch disables only this policy. H2 weapon class numbers differ from shared SDK labels (rifle 0, sniper 1, MG 2, SMG 3, shotgun 4, pistol 5, GL 6, RPG 7): verify enums from live definitions.
- **Aim assist** (`gp/aim_assist.cpp`, `aim_assist_geometry.hpp`): runs only on the native server fire thread on frozen geometry; any failure leaves the shot untouched. Candidates from the engine's own
  AI query re-read every shot (`getaiarray("bad_guys")`, plus `"neutral"` on two museum maps); per candidate `isalive`, `getshootatpos`, within `max_distance = 1300` units and a cone of
  `aim_assist_degrees(strength)` (`vr_aimAssistStrength` 0-100, default 0 = off, -> half-angle 0-10 deg linear: 50 -> 5). Rank by alignment cosine (computed in **double**, so tiny cones do not widen when the float cosine rounds to 1.0),
  tie -> nearer, tie -> lower entity id; always scored against the ORIGINAL muzzle. Visibility trace only for candidates that can beat the current best ("an occluded target must not hide the next visible one"):
  `G_TraceCapsule` origin -> target, accepted if `fraction == 1` or hit entity == target. Civilians excluded by script `type == "civilian"`, a named drone or classname substring, never by team
  (`ignoreme/noncombat` are not universal protection). A mission whose objective fails on torso/head hits disables assist until a script flag (read live; the user's saved strength is never changed).
  Applied as `from_to(forward, best_direction)` rotation of the whole shot frame, origin unchanged. Fixed scopes instead correct the native command angles at the existing command boundary.
- **VR recoil** (`weapon_recoil.hpp`, `weapon_recoil_tuning.hpp`): moves only the gun; the HMD never rotates. Native hip gun-kick values are pitch *velocities* with arbitrary endpoint order:
  expected |speed| = `(lo^2 + hi^2) / (2 (hi - lo))` if `lo < 0 < hi`, else `(|lo| + |hi|)/2`. `degrees = min(speed * 0.05 s, 8) * posture (stand 3 / crouch 2 / prone 1) * weapon_scale (0.25 SMG .. 5.0 pistols)
  * one-hand penalty x4` (modes all/long/off; only when the *actual* support hand is absent and not prone). Accumulate `stored = min(stored * exp(-dt/0.22) + shot, 55)`, zero after 2 s. Applied to a
  freshly solved pose by rotating gun + firing wrist + fingers about the firing wrist pivot (gun -Y axis) and re-solving the support arm with the support contact exact; never incrementally. State keyed by
  weapon identity + rear revision + reference generation (16 entries). An earlier bug rejected values > 30 or descending ranges and disabled recoil on most guns. Recoil is only added when the engine
  actually emitted the shot.
- **Auto-ADS** (no ADS button in VR; `controller_ads.hpp`, `ads_alignment.hpp`, `ads_comfort.hpp`): the mod OR-s `BUTTON_ADS` (0x800) after the engine's command pack, so native PM keeps authority. Corridor from the
  solved muzzle, hysteresis: entry head 0.15-1.50 m behind, lateral <= 0.09, height -0.025..+0.16 to the barrel axis, facing cos 0.866 (30 deg); exit 0.10-1.70, 0.13, -0.06..+0.22, cos 0.766 (40 deg);
  thermal 35/45 deg; dwell 100 ms in/out (40 ms thermal), advanced only on *new solved pose samples*; both hands required; support release cancels immediately; long assemblies extend the far limit to
  rear-optic-to-muzzle + 0.60 m. **Comfort translation** rigidly moves gun, muzzle and optic toward the eye along the barrel axis (max 0.05 holographic, 0.10 ACOG, 0.15 scope, 0.45 thermal, eye relief
  8 cm behind the rear lens / 5 cm thermal; weight full within 0.30 m, zero at 0.60 m; quintic smoothstep) with a **critically damped spring + speed ceiling** (10 Hz, 0.25 m/s; thermal 24 Hz, 0.9 m/s; tail snapped
  under 0.05 mm and 1 mm/s): a first-order filter started a 15 cm move at 1.25 m/s. Translation is subtracted in the corridor test to avoid self-oscillation. Native sprint-during-ADS predicates are patched
  by asmjit bridges (ref 02: register-preserving bridges; verify all sites before installing any; the same `allowed(ps)` policy must apply to authoritative AND predicted movement or prediction desyncs).

### 5.2 Path B: independent fire (dual wield, carried guns, underbarrel, per-hand)
Used when a gun is not the engine's selected weapon. Overlord: `gp/native_ballistics.cpp` (`fire_owned`), `independent_fire_runtime.*`, `independent_fire_clock.hpp`, `underbarrel_native.cpp`,
`native_launcher_fire.cpp`, `docs/vr-independent-weapon-fire.md`. Never swap the selected weapon or spawn a fake player/NPC.
Order of operations inside one reentrancy-guarded transaction (`thread_local emitting` covers scripts too):

1. validate: not emitting, server pipeline, cgame initialized, `ps->commandTime == command_time` (the real field, not faked per hand), no mount flags, `scripted_control::firing_allowed`, not paused, no key catchers
2. validate geometry; weapon type is bullet; `loaded > 0`; eye->muzzle <= 200 units; shield clearance; `G_TraceCapsule` eye->muzzle with `muzzle_clearance_mask = 0x280e831 & ~actor_contents(0xc000)`
   (cover blocks, actors do not)
3. build the engine fire packet (0x50 bytes: shot geometry, weapon, alternate flag, definition pointer; bytes 0x30..0x3b zero), ADS spread via the engine's accessor
4. freeze the real firing weapon's full name and script args BEFORE the debit (native selection may belong to the other hand)
5. project the instance into the native ammo cell (`native_ammunition::project`), restored by scope exit
6. **debit via compare-and-commit** (skipped when `player_sustainAmmo`), re-checking player pointer/commandTime/permission right before
7. **settle the mechanical owner via callback before** native damage/scripts can run (the observer never writes native ammo)
8. **send `weapon_fired(name)`** with the engine's own notify (the direct bullet call skips the event; mission scripts wait on it). Engine order: prepare payload -> compared debit -> settle -> `weapon_fired` -> ballistics.
   "Never refund or retry after this boundary: scripts may have observed it."
9. re-check context/timeline, then call the engine's bullet function (or class-6 projectile fire for underbarrel/launcher with the same packet)

- **Shot identity** = (ownership generation, instance, module, sequence); frozen binding per request; one transaction at a time in stable instance order; revalidate player/commandTime/ownership between guns.
- **Per-instance shot clock in simulation command time** (`independent_fire_clock.hpp`): decoded native timing `{mode 0 auto / 1 single / 2-4 burst / 6 double-barrel, interval, delay, burst_pause}`; interval >= 20 ms
  (>= 10 for mode 6); the burst-pause accessor already returns **milliseconds** (M16/FAMAS/M93R all 200, do not multiply by 1000); mechanical-cycle weapons use interval 0; no backlog firing after a stall
  (cancel on command-time jump > 150 ms or regression); a trigger press must precede; losing authority cancels intent, not the cooldown; `rejected()` consumes the attempt without a cooldown; handover requires a neutral trigger.
- **Suppress stock attack at both boundaries**: command build *and* the simulation/prediction fire boundary (merely ceasing to OR the controller attack bit is insufficient; the keyboard/mouse attack bit still fires the selected weapon).
- **Prediction replay**: PM weapon code replays old command times. `native_shot_history.hpp` records the admission decision per `(weapon, instance_generation, command_time)` (128-entry ring) on the server; replays
  consult the record, not the current chamber; `spent` is set after the real shot. Without it prediction desyncs from the mechanical state.
- **Cross-feed settlement**: `settle()` calls the manual-feed, magazine, cylinder, tube and break-action trackers and requires each active one to agree with post-shot native loaded.
- Brass origin comes from the authored ejection transform or the world model's muzzle/brass markers, never from the support-hand side. Hit-scan "high explosive round" add-on (`hei_round.cpp`) is an example of a scoped
  bullet-trace observer + native radius damage.

## 6. Physical reload mechanics (the reusable architecture)

The original reload animations are **not authoritative**. Block/reshape the native reload, keep a pure mechanical planner, and commit to the engine's ammo cells by compare-and-swap.

### 6.1 Pipeline
```
gesture (raw hands, simulation thread)  ->  plan(rules, state, request) -> transaction{accepted|rejection, next, before, after, feedback, rounds_spent, item flows}
   -> adapter: ammunition::commit(tx, write)  where write = native_ammunition::compare_commit_owned(ps, id, tx.before, tx.after)  [server pipeline]
   -> success: state = tx.next, publish; failure: state AND escrow retained, counter++, input edge consumed (no retry)
```
- **Pure planner** never touches the engine. `state` carries `weapon`, `instance_generation`, `revision`. A request is rejected if any of the three differ, if hands are wrong (rear-hand ops vs off-hand ops are
  disjoint: `rear_action ? actor == rear : actor != rear`), or if `revision == UINT64_MAX`. Planners must not be called from render hooks, command polls or prediction replays.
- **Conservation proof inside the planner**: after every op `revision++`, `valid(next)`, and
  `total(current) + rounds_from_item == total(next) + rounds_spent + rounds_to_item`, else `invalid_state` (`detachable_magazine.hpp`, `tube_feed.hpp:117`, `cylinder_feed.hpp`, `break_action_feed.hpp`). Global form:
  `usable = reserve + live weapon contents + held payloads + falling/unsettled payloads`, `usable_after = usable_before + grants - accepted_shots - mechanical_extractions - discard_loss`; spent cases never count.
  Test with randomized thousands-of-request sequences and exhaustive short sequences, checking conservation after every commit.
- **Escrow**: a magazine/loader/round pulled from the waist leaves native reserve into `held_rounds` (hand-owned object) *before* it becomes active (reserve first, compare-commit must succeed, a rejected write cancels the
  reservation). Cancel/discard/cleanup returns rounds exactly once via `ammunition::dispose(rounds, reason, penalty)`; reasons `deliberate_discard` (optional penalty dvar `vr_discardAmmoPenalty`, off by default), `forced_cleanup`
  and `waist_return` never penalize. Tracking/focus loss, pause, missing assets, teardown are not deliberate discards. Read the policy at the irreversible commit. A faulted instance returns escrow
  (`release_fault`) and the native fallback remains available; VR state is never persisted while faulted.
  An empty used speedloader stays a real held item (existence is not inferred from rounds > 0). Refund of a dropped magazine goes to a weapon sharing the same ammo pool (`reserve_identity`), never by guessing caliber/capacity.
- **Reconcile every server tick** (external pickup/script grant/checkpoint): native `loaded` went up behind our back -> `granted_reserve` accepts only a *positive budget credit* (box pickup, +1 clip truncation) into reserve, never as a fake
  physical load; went down -> accept without refund and mark chamber unconfirmed; unexplained -> `fault`. Reserve-only changes update reserve and bump revision. Disposable launchers fold loaded back into reserve so grants cannot re-arm them.
- **Admission/bootstrap** (`update_instance`): only with a fresh authored scene (<= 150 ms), native name + live capacity matching a registered profile (`matches_native(name, capacity)`; name alone is not enough; capacity re-read from the
  native WeaponDef so a modded capacity is rejected), native weapon state idle. `import_native` partitions conservatively: closed bolt `N -> (N-1) + 1`; cylinder imports missing rounds as *spent cases* (a native count cannot tell fired
  from never loaded; one gravity clear establishes known-empty); break action `live = (1<<loaded)-1`; open bolt: all rounds in the magazine, chamber never loaded.
- **Lifecycle**: drop the whole inventory and `generation++` when the player-state pointer changes, game time goes backwards, or the native timeline token changes (checkpoint/restart); suspended while a native script owns
  weapons (`scripted_control::allowed(ps)`); `interrupt()` plans a `cleanup` op returning held rounds. `prepare_transfer / restore_transfer` re-admits only if native name/capacity/loaded still match.
- **Stale-cache lesson**: snapshots read outside the scheduler's server scope returned the ledger's old count (1 vs native 0); read actual native state through a read-only identity-bound helper; write authority unchanged.

### 6.2 Block or reshape the native paths that would fight the mechanic (not `BUTTON_RELOAD` injection)
`gp/reload_boundary_component.cpp` is the single owner of four detours: **fire** (H2 `0x140699240`, `manual_feed::allow_fire`), **consume** (`0x14069E430`, observes before/after, calls `manual_feed::consumed`), **reload start**
(`0x1406946D0`, `blocks_reload`) and **`PM_BeginWeaponReload`** (`0x140693D00`, earliest begin; an empty automatic reload can bypass the can-reload check, so it must be stopped before its events, camera animation and state change; blocking
only the later refill produced repeated start/end cycles). Lesson for a new game: find the *earliest* function that starts a native reload/animation. Production hooks stay installed with diagnostics off.
- Façade `manual_feed` (`manual_feed_boundary.hpp`) chains per-family `blocks_reload`, `allow_fire`, `consumed` using copied snapshots, no lock across families. `allow_fire` also rejects during a cheat-inventory transfer, when another
  independent-fire owner holds the weapon, and when the carried weapon is not the held one. `consumed` checks `after.loaded == before.loaded - (sustained ? 0 : 1)` with reserve unchanged; mismatch faults. Never call `commit_owned` from the consumed callback.
- Manually cycled guns (pump, lever, bolt) return false from `allow_fire` so they never enter native re-chamber scheduling.
- **Chambered +1** without patching the shared WeaponDef ("chamber is an instance-owned state"): redirect only the **two call sites** of the capacity query inside the reload admission and refill functions
  (H2 `0x1406965A1`, `0x14069E244`; the capacity function is also used by HUD/pickups/scripts) and have a `thread_local context*` set by detours at the outer functions return `capacity + 1` when the weapon currently has a chambered round
  (tactical reload animation 20 vs empty 21 read from the player state); `max(loaded, ...)` so a late callback never gives a negative transfer; native alone moves reserve to loaded, the adapter only provides the limit
  (`native_closed_bolt_component.cpp`, `_policy.hpp`). "Redirect call sites, not the function, when the function has other meanings."
- Prevent auto-switch/auto-drop of empty physical launchers by wrapping the no-ammo event call sites and the PM empty-removal call while a spent tube is still held (script takeweapon and drop still reach the real remover).
- Native animation state is not the VR transaction clock: native timers (H2 tactical ~1.99 s, transfer ~1.36 s) must never become insertion cooldowns; ammo transfer, mechanical readiness, hand animation and sound notetracks are distinct boundaries.

### 6.3 Ammo cells and the clip ledger
- The engine stores ammo in **sparse keyed tables**: clip cells (24 B at `ps+0x4a4 + i*24`) and reserve cells (12 B at `ps+0x3f0 + i*12`), 15 entries each; an absent key means zero. Key identity is 5 bytes (clip) / 6 bytes (reserve)
  of an 8-byte word, padding not compared; keys come from the engine's own key builders, never guessed. `storage::commit(expected_clip, expected_reserve, new_loaded, new_reserve)` re-reads both cells, rejects if either differs from the plan,
  rejects if a needed vacant cell is missing (never overwrites another weapon's cell), preflights both before the first write, keeps zero sparse, bounds loaded <= 1001, reserve <= 1,000,000. `commit_reserve_pair` validates both pools before any write
  (supply exchange). `native_ammunition_storage.hpp`, `native_ammunition.cpp` (SEH leaf reads, no destructors or engine calls under `__try`).
- **Clip ledger** (`weapon_clip_ledger.hpp`, `weapon_clip_projection.hpp`): the engine keeps ONE projection per weapon definition, so two physical copies of one gun need a server-owned ledger `id = {weapon token, generation}`, `projected`
  flag, `select()` swapping which instance is written to the native cell, `begin_pickup / finish_pickup / retire`, `ownership_epochs[512]`, `timeline_epoch` bumped on level start/shutdown, staged copy-validate-memcpy restore. A non-projected same-model
  instance must not inherit another gun's debit. Inventory sanity: exactly one inventory record for the token, WeaponDef layout through a decoder with `static_assert` offsets.
- Native auto-pickup/scavenging can consume a ground gun's reserve and clip: guard `Touch_Item` and the use-list admission with one shared `admit_duplicate_pickup` (`world_pickup_policy.hpp`).
- **Save/restore of VR state** (`weapon_inventory_snapshot.*`, schema 1): `capture` first settles every escrow, reads shared ammo pools once, persists only stable logical fields (not pointers, timestamps, leases, ammo keys, generations);
  `validate` is a read-only preflight against current native assets; `restore` stages ledger + native ammo + inventory atomically; hand/trigger leases require fresh input; a faulted family fails closed.

### 6.4 Rule struct and invariants (real field names; the design-doc enum is not in the code)
`mechanics::rules` (`detachable_magazine.hpp`):
```cpp
struct rules { int magazine_capacity; magazine_release release{button|physical_pull}; bool last_round_lock, release_control, plus_one;
               feed_type feed{closed_bolt|open_bolt|manual_bolt}; bool manual_catch, belt_fed, belt_bridge, physical_catch_release, discard_penalty; };
struct state { weapon; instance_generation; revision; magazine_inserted, chamber_loaded; magazine_rounds, reserve_rounds, held_rounds;
               hand magazine_hand; action_state action; manual_bolt::state bolt; belt_feed::state belt; };
enum class action_state { closed, locked_open, held_open, cocked_open, latched_open, manual_unlocked };
```
`action_state` meanings: `locked_open` = follower lock (closed-bolt, legal only with `last_round_lock` and no chambered round); `held_open` = held by the hand; `cocked_open` = open-bolt sear caught (ready to fire, a
closed-bolt lock is not); `latched_open` = manual catch (`manual_catch`); `manual_unlocked`. Other families have their own rule structs: `tube_feed::rules {capacity, drive (pump|lever|automatic), layout (tube|fixed_drum), chamber_bonus}`,
`cylinder_feed::rules {capacity, discard_penalty}`, `break_action_feed::rules {capacity}`, `open_bolt::rules {magazine_capacity}`, `closed_bolt::rules {magazine_capacity, last_round_lock, plus_one, manual_catch}`.
Family rules (`closed_bolt::valid`): `magazine_rounds + chamber <= capacity` unless `plus_one`; open states legal only without a chambered round; partial pull never extracts (only a full stroke + release calls `cycle`); `release` with an inserted empty
follower keeps the lock; `slap_release = unlatch + finish_cycle`; `reload_capacity = cap + (plus_one && retained chamber)`; inserting a full magazine into a chambered non-plus-one gun moves one round to reserve; a magazine inserted into a closed empty chamber still
needs a full cycle.
**The invariant is "total rounds conserved", not `loaded == feed + chamber` in every family**:

| Family | Native `loaded` maps to |
|---|---|
| closed bolt | magazine + chamber (N -> (N-1)+1; tactical may reach N+1 if `plus_one`) |
| open bolt / belt (Mini Uzi, MG4, RPD, M240) | magazine only; chamber always false; `plus_one` false; cocking never extracts/feeds; last accepted shot closes the action; dry trigger with empty/absent mag releases the sear |
| tube | stored + chamber (`chamber_bonus`); fixed drum has no chamber |
| underbarrel M203/GP-25 | `loaded == int(chamber)`; shotgun module up to 4 = 3 tube + chamber (`shotCount` 8) |
| cylinder | 6 cells with spent-case tracking, one gravity clear establishes empty |
| break action | live/spent bit masks |

`native_shot_history`, `blocks_reload`, `allow_fire`, `consumed` are the per-family triple; "one accepted shot consumes one shell regardless of pellet count".

### 6.5 Mechanism families (write 4-6 state machines; guns are tables)
| Family | Files | Key mechanics |
|---|---|---|
| Detachable magazine, closed/open bolt | `detachable_magazine.hpp`, `closed_bolt.hpp`, `open_bolt.hpp`, `manual_bolt.hpp` | ops `pull_magazine` (escrow the actual old magazine even if empty; reinserting never tops it up), `knock_magazine`, `quick_load` (waist supply into an absent magazine), `insert_external_magazine`, `release_catch` (dedicated physical paddle never falls through to eject) |
| Charging handle / bolt | `charging_handle_bolt/catch/fold.hpp`, `rotating_bolt_gesture.hpp`, `handle_catch.hpp`, `bolt_partition.hpp`, `folding_handle.hpp` | non-reciprocating handle bone separate from the mechanical lock (`locked_travel = 0` for AR handles; never hold the handle back because the feed is locked) |
| Belt/box | `belt_feed.hpp`, `belt_profile.hpp`, `belt_cover_push.hpp` | cover/bridge/box state with clearance gating (`bridge_clearance 0.9`); `settled_cover` snaps < 0.03 -> 0, > 0.97 -> 1; removing the box withdraws the whole belt; palm-push requires a continuous exterior approach (arming gap 25 mm, contact gap 12 mm, penetration limit 25 mm, speed 0.06-6 m/s, inertia only for committed strokes, 300 ms cooldown) |
| Tube/pump/lever/drum | `tube_feed.hpp`, `tube_gesture.hpp`, `tube_runtime.cpp` | shell loading broad contact (3.5-5 cm; M1887 7.5 cm with alignment -0.5), `port_radius`, rack stroke/full stroke; lever spin 36-sample wrist track + 648 finger quats; striker rack disabled by feed policy |
| Break action | `break_action_feed.hpp`, `_gesture`, `_presenter` | chambers as masks (`live`, `spent`); open only when latched; load needs `hinge >= 0.98`, alignment cos >= 0.25 and a `sweep_well` contact; `load_armed` only after the hand was outside the contact volume; barrel grasp accumulates `std::remainder` angle / open_angle; snaps < 0.04 to 0; close by wrist flick |
| Cylinder | `cylinder_feed.hpp`, `_gesture`, `_runtime` | gravity clear only after full open facing world-down (down cos 0.5, up 0.35, hysteresis 0.08); loader contact in the cylinder frame (radius 0.09, behind 0.16, inside 0.04, alignment 0.55); `fill` only on an empty cylinder; twist-to-close gate; swing -90 deg about X |
| Launcher | `launcher_feed.hpp`, `launcher_runtime.cpp` | waist draw (<= 0.18 m) debits one round into hand escrow; insertion crosses a front approach (`delta.x` in [0.09, 0.5]) before the seat window `[-0.04, 0.035]`, cos >= 0.4226, lateral <= 0.075; release refunds via compare; guided launchers convert the script's target delta into the muzzle basis before the native projection leaf |
| Underbarrel | `underbarrel_*.hpp/.cpp` | identity classified from native WeaponDef AND a reviewed name family AND exactly one host owning the clip key; palm-facing grips from controller orientation in the gun frame (support palm-up >= cos 75 deg = 0.2588 acquire, firing grip facing 0.766 / 0.643 retain); stroke tolerances 0.10/0.25/0.12 m (shotgun 0.16/0.40/0.18); smart supply picks secondary vs primary from native need (XOR grip to override) |
| Melee prop, shield | `special_melee.hpp`, `shield_*` | `defense`/`melee` capabilities, never dummy ammo or a muzzle |

### 6.6 Gesture detectors and starting numbers
- **Swept contacts, not corner probes** (`swept_impact.hpp`, `swept_box_contact.hpp`, `magazine_well_contact.hpp`): box-vs-sphere by *conservative advancement* (bound = translation + 2*half_angle*radius, step by `(distance - radius)/bound`, <= 128 iterations,
  0.05 mm tolerance); follow the *same material point* across samples; entering contact consumes the approach even if the transaction is rejected; contact re-arms only after separation (4 cm contact / 7.5 cm rearm AK family, 2.5/6 cm others, slaps 2.5/7 cm);
  continuity if `dt <= 0.15 s` and per-point step <= `max(25 cm, 8 m/s * dt)` (a real 68 ms / 31 cm step was wrongly rejected by a fixed 25 cm limit); use raw wrist, never snapped IK. Hands as 21 contact points (15 joints + 5 tips + palm) plus a 6x6x4
  palm grid (8/10/12 mm skin, max 0.16 x 0.14 x 0.06 m) (`palm_contact.hpp`).
- **Magazine latch strike**: material-point velocity along `latch_direction >= 0.15-0.18 m/s`, travel >= 1.2 cm, point behind the latch plane before; strike and insertion are separate transactions (clear and re-enter the well; 300 ms guard). Release impulse 80% of strike velocity
  capped at 3 m/s total, tumble from lever arm capped at 6 rad/s.
- **Well insertion**: swept segment vs a finite cylinder (radius, depth, capture_below), not a plane; alignment cosine; `max_contact_step 0.25 m` (teleport rebases history and forces withdraw-then-fresh-contact); margins `well_release 0.04`, `well_withdraw 0.015`.
  Defaults: rifle well radius 0.045-0.055 m, depth 0.06-0.07, capture-below 0.06, insertion cosine 0.0872 (85 deg); pistol capture radius 0.035, 6 cm above/below the *mouth* (not the protruding baseplate), 85-95 deg; big box mags 0.055-0.065/0.07.
  Magazine must leave the well volume + 15 mm before re-insertion (no timers).
- **Magazine pull**: grab radius 0.06-0.08 (fresh trigger press on the body; holding while entering does not grab), pull travel ~0.055 m along a per-gun `pull_axis`, lateral limit 0.12. Spare held, ammo conserved.
- **Grasp style selection** (`magazine_grip_selection.hpp`): palm roll about the hand forward axis (palm-up 0, inward 90, down 180, outward 270; 190-350 = bottom pinch, else body wrap; near-vertical pointing -> wrap), latched at press, never re-evaluated while held;
  capture assist rotates toward the authored grasp by <= 60 deg total; `choose_part_grip` picks facing first, distance second; palm facing decided from the physical palm before animated wrist offsets.
- **Wrist-flick/twist gate** (`cylinder_gesture.hpp: twist_gate`): raw runtime matrix (independent of camera, snap turn, offsets, IK); bounded window (64 samples, 0.22 s) sums roll about the aim axis; fires at `angle >= 0.35 rad`, rate >= 12 rad/s, `fast_angle >= 0.25`, `angle >= travel*0.5`;
  rearm after total motion < 0.8 rad/s; discard on continuity change, dt <= 0, gap > 0.5 s (0.15 s without continuity); reject > 60 rad/s. Break action 6 rad/s / 0.20 rad; lever 5 rad/s / 0.30 rad; lever catch needs <= 0.30 rad, <= 1.8 rad/s for 0.06 s. Tolerates
  uneven server cadence (33-68 ms observed) and slow-motion (producer continuity up to 200-500 ms).
- **Rotating bolt** (`rotating_bolt_gesture.hpp`): lift = `atan2(cross, dot)` about a pivot, active only with both vectors > 1.5 cm; snap < 0.04 -> 0, > 0.96 -> 1; then pull travel = delta X / stroke; release assist uses gun-space velocity (moving the rifle is not closing intent), speed 0.25-8 m/s within 35 ms, remaining lift <= 40 deg.
- **Hinge** (`hinge_travel.hpp`): accumulate `std::remainder(angle - previous, 2pi)/stroke` (no atan2 seam); retain overtravel until grasp ends; `hinged_attachment` open > 0.94, closed < 0.06, cancel on > 0.20 m jump or > 0.12 m path deviation.
- **Stroke tuning**: `locked_travel` = native last-fire settled slide position; `slide_stroke` = measured native peak + 3-6 mm overtravel; `full_stroke` ~ 0.90-0.97 x stroke (a stroke counts only on full travel; deliberate overtravel distinguishes "pull off the follower lock" from "grab the open handle").
  Examples (m): M9 0.061/0.055, USP 0.060/0.056, DE50 0.087/0.082, AK 0.094/0.087, FAL 0.144/0.135, UMP 0.085/0.079.
- **Support-hand lease**: new squeeze within 0.10 m acquires (holding while entering never auto-grabs); 100 ms blend; retained against the *two-hand solved* anchor with 0.22 m breakaway (rear roll cannot release); hands closer than 0.08 m cannot define a rifle axis and release;
  release is logical-immediate with a 100 ms return to rear-only aim that does not follow the freed hand; launchers wider (AT4 0.24/0.34, Javelin 0.14/0.26); acquire < release always (hysteresis). `vr_supportGripLock` keeps support attached; initial acquisition still needs a fresh press. Lease acquired at the
  one-hand pose, retained at the solved two-hand pose (`grip_presenter.hpp`).

## 7. Weapon recipes, admission and the authoring pipeline

### 7.1 Weapons are immutable data over a few mechanism families
Per weapon folder `weapons/<name>/` (real file set; some weapons omit optional files; `weapons/README.md` is the contract):

| File | Owns |
|---|---|
| `profile.hpp` | `profile` (receiver identity, `aiming`, wrists, finger table, acquire/release/blend, `equip_rest`, `suppress_equip`, feed capability pointers) + `select()` admission + skin/variant arrays |
| `poses.hpp` | idle wrist anchors for both hands, 30/36 finger quaternions, `equip_rest` part poses |
| `mechanics.hpp` | `mechanics::rules` (or inline `reload_rules`), capacity, release, chamber/feed policy |
| `reload_interaction.hpp` | `physical_reload::profile` (metres): strokes, radii, contact steps, grasp counts |
| `reload_poses.hpp` | magazine/slide/handle grasp anchors, contact bounds, finger poses per grasp |
| `reload_profile.hpp` | assembled `reload_profile`: bones, geometry, sound hookup, exact native-family admission |
| `feedback.hpp` | sound notetrack-key map only |
| `magazine_fill.hpp` | rigid-subset recipe (section 4.4) |
| `actions.hpp` | equip-suppression predicate (`suppress_equip`) |
| optional | `magazine_contacts.hpp` / `magazine_collision.hpp` (strike OBBs), `magazine_grasps.hpp`, `handle_poses.hpp`, `slide_grips.hpp`, `stock_support.hpp`, `foregrip_poses.hpp`, `bolt_partition.hpp`, `folding_handle.hpp`, `secondary_motion.hpp`, `knife_slide_grip.hpp`, `hud.hpp`, `geometry.hpp` (shield) |
Shared: `weapons/families/ar.hpp` (M4/M16/ACR closed-bolt rules + `charging_handle()` factory), `weapons/hand_poses/` (`edge_handle`, `left_handle`, `right_handle`, `side_pinky_up`, `magazine {bottom_pinch, body_wrap}`, `shell_grasp`),
`weapons/attachments/` (`rifle.hpp`, `precision.hpp`, `heartbeat_poses.hpp`, `underbarrel_poses.hpp`). Registration: `gp/weapon_registry.cpp` joins `register_profiles(selector, profiles...)` into one `constexpr` catalog
(`static_assert(size <= 256)`); `profile_capabilities<T>` derives pointer-deduplicated views per capability; capacity overflow empties the view (fail closed).

Data discipline (copy it): designated initializers in field order; configuration headers include only standard-library data (no game types/addresses/controller code) so offline tests are trivial; explicit units (reload profile distances metres, latch speed m/s,
rotating-bolt angles radians, blend seconds; receiver poses, bone contacts and mesh bounds stay in the game's NATIVE model units, converted exactly once at authoring); select shared defaults explicitly (`physical_reload::defaults`, `profile_defaults {acquire 0.10, release 0.22, blend 0.10}`);
"matching numbers do not establish that two mechanical families can share admission rules"; keep readability cleanups separate from tuning; `static_assert(grips.size() == slide_pose_count)` style cross-checks.

### 7.2 Admission: a name match is only a candidate
`select(models, receiver, rig, bones) -> profile_match {profile*, reason, hidden_mask, muzzle}` or `{nullptr, "why rejected"}` (`weapon_profile.hpp`, `weapon_profile_binding.hpp`, `weapon_registration.hpp`).
- Selection first requires the model list to be contiguous and sum exactly to the rig bone count; the receiver model containing `rig.gun` must have an exact known name (per skin) AND an exact **bone count** (H2 examples: AA-12 14, M4 24, AK 25, M16 28, M240 38, G18 10, MG4 37, RPD 51).
- Full hierarchy check where relevant (expected bone names + parent indices; e.g. AA-12 exactly one `j_reload` parented to gun and one `j_reload_end` parented to it; riot shield 17 names with parent table).
- **Attachments**: `bind_attachment_set(policies, models, receiver, rig, bones)` validates every attachment as `{model name, root bone, parent bone on the receiver (unique), root alias, expected bone count, role}`; roles `foregrip, launcher, cover, silencer, optic, sensor, laser, shotgun, bipod` with
  cardinality 1 and mutual exclusions (AK: GP-25 XOR shotgun; M16: cover XOR M203; M4: one of foregrip/M203; AUG: foregrip required + rail XOR scope); non-overlapping ranges, parent precedes child; the muzzle bone must be a descendant with forward cos >= 0.98; a validated suppressor supplies the muzzle
  (`tag_flash_silenced`). Unknown models, wrong parents, duplicates or conflicting roles reject the whole assembly ("attachment grip variant not authored"; the gun keeps native behaviour). One missing attachment alias (`attach_h2_m203_vm_digital`) rejected an entire scene, so every real variant must be registered.
- **Native admission is a second, independent gate**: exact native instance name (boundary-aware family match `native_weapon_family(name, "m4")` accepts `m4`/`m4_...`, with explicit exclusions for alternate feeds), **live WeaponDef capacity** (G18 and TMP bugs: assuming 33 / 15 silently blocked admission, the real value was 32),
  primary (not alternate) mode, unique owned ammo cells, idle entry, prepared visibility assets. Model/animation names often differ from native names (H2: M82 -> `barrett`, TMP -> `mp9`, PP2000 -> `p2000`). A key in a WeaponDef sound map is NOT evidence an attachment exists. "Discovery is not write authority."
  Launcher/alternate feeds are excluded from rifle authority (alternate mode seen via a player-state bit; the launcher def has no reverse alt link: do not invent a symmetric graph).
- Retry admission only at new asset/rig/context generations (an incomplete scene load is not proof of absence); cache exhaustion rejects, never evicts in-flight metadata.
- **Skins/variants are data clones**: `auto p = base; p.id = "ak47_arctic"; p.receiver/rigid_magazine_source = ...;` selected through `skins[]` and `receiver_skin(name)`; assembly array layout `assemblies[skin*N + variant]`; grenadier/shotgun variants overwrite only the support wrist, the finger overlay and set
  `free_hand_reference = &wrists` so the rear hand and free-hand basis stay identical when the support grip changes. Register a skin only after capturing its bone hierarchy and normalized bind rotations (matching to ~3e-6 cm) and keep the captured hierarchy as a test fixture. Do not admit silenced/tactical/akimbo variants by name prefix.
- `aim_rule { rear_hand (pistols, revolvers, knives, shield, devices), two_hand }`; rear hand always owns firing; non-firing items use `defense`/`melee` capabilities.

### 7.3 Equip-animation suppression and rest poses
`suppress_equip(animation_name)` (profile-owned, exact family prefix such as `base_equip_action(name, "h2_wpn_smg_mp5k_")`) redirects the engine's **animation-slot selector** at the five CLIENT call sites (never server callers) so pullout/putaway/inspect clips become the weapon's idle clip *before* notetracks and
curves are read (`native_equip_presentation.cpp`, `weapon_actions.hpp`; clip names read from the player-state xanim table; indices < 202 are idle). "Change the input to a native function, not its output." State, timers and sounds remain native ("presentation suppression, not instant equip"). While suppressed, `equip_rest`
(parent-local transforms of MOVING parts only, sampled from idle frame 0) overwrite bolt/trigger/clip/rounds/rings so native right-hand cocking clips do not disturb the tracked grip; static mount tags are excluded (names recur on attachments); alternate-feed actions are not suppressed. Remove sampled slide travel before storing a
rest pose; the second hinge of a nested part must not receive its gun-local translation as parent-local. A selection-finish hook zeroes the raise timer only for physically carried weapons on both simulation and prediction (section 9).

### 7.4 Pose authoring pipeline (offline numbers, no assets shipped)
1. **Source**: sample native animation frames offline from exported animation data: idle frame 0 for rest grips, a reload frame for the magazine hand, pullout/reload_empty frames for the charging-handle hand, pickup/inspect frames for slide/knife co-grasps. Each header cites the animation, frame, and the **SHA-256** of the clip and receiver;
   raw exports, meshes and parsers live in removable ignored storage and are never build or runtime inputs.
2. **Units**: export centimetres / 2.54 **once** -> native units (inches); quaternions (x,y,z,w).
3. **Frames**: wrist anchors stored **gun-local** (position + rotation relative to the gun root bone) after composing the receiver's additive root translation (`j_gun` offset (-1.844, 0, -3.266) cm on the M4; using `tag_weapon` as the origin misplaces both wrists). Native translation channels are ADDITIVE, rotation channels
   REPLACE parent-local rotation (do not multiply bind rotation again; `part_pose_mode::fixed_attachment` for attachment-local rest rotations). Fingers: parent-local quaternions keyed by joint name; only rotations transplanted.
4. **Retargeting**: native rifle pullout/reload clips use the RIGHT hand on the charging handle; the physical LEFT hand gets it by anatomical reflection (section 2), with explicit fits for both facings so the opposite wrist stays outside the receiver (`opposite_pose`, `edge_handle::fit(pose, opposite)`). Borrow only hand *shape* from donor animations, then re-fit the wrist to the target mesh
   and keep target lengths/ammo/sounds authoritative (M9 overhand slide grip uses MP5K reload_empty frame 9; USP uses H1 clips matched by joint name, omitting H1-only fingertip joints).
5. **Fit to the mesh, not a number**: place wrist/contact by measured finger-skin-to-mesh distance (sub-mm to ~1.7 mm), target the *foremost finger-skin edge* rather than a wrist coordinate, translate the whole grasp (not just curls) and rebind the acquisition contact to the real surface. Hook-style grasps for protruding tabs open intermediate/distal joints ~30 deg
   (`edge_handle`: `index_side` / `pinky_side` latched at acquisition). Fingers: four fingers share MCP/PIP flexion with coupled DIP, MCP rotates only about its flexion axis (no splay), retarget through bind frames (never copy a quaternion between differently oriented rigs).
6. **Authoring contacts**: acquisition boxes as `slide_grab_low/high` AABBs in gun-local units (often enlarged after headset tests: +4 cm outward on AK, etc.); `part_capture_halfspace {normal, offset}` bounding capture outside the receiver wall so assist cannot reach through the gun to the magazine; magazine strike boxes as OBBs enclosing the whole magazine body + 0.05 mm padding.
7. **Headset loop**: mesh proximity does not establish comfort. Expect several small user-driven corrections per weapon (the USP slide grasp took iterations: 2 cm, 2.34, then +1.5 cm, a third returned) and record each in a dated tuning log. Defaults to start from: acquire 0.10 m, release 0.22 m, blend 0.10 s, minimum hand separation 0.08 m.
8. **Variant checklist**: capture hierarchy + normalized bind rotations, register the model alias, add a fixture, add a regression case (synthetic permutation counts like ACR 3,024 / AK 8,064 assemblies are resolver coverage, not observed combinations).

### 7.5 Sounds via notetrack keys (`native_weapon_sound.*`, `weapon_sound_*.hpp`, `native_melee_feedback.*`)
- Address sounds by **semantic notetrack key** (`magazine_out/take/in`, `action_rear/close`, `bolt_unlock/lock`, `case_eject`, `cover_open/close`, `belt_laid`, `dry_fire`) resolved through the LIVE weapon's notetrack map (key -> script-string value -> alias) at event time, then play through the engine's positional sound entry
  (H2 `0x140380430(entity, origin, alias)`). Missing key = silent: never guess an alias after a failed lookup, never borrow another gun's map implicitly; sharing a donor is explicit (`sound_reference{name, kind notetrack|alias, part whole|second, notetrack_weapon, window}`).
- Verify alias existence/count and `!IsXAssetDefault` for mod-supplied aliases; never retain asset pointers across zone unload.
- Compound clips play once at completion (`cycle_notetrack`, `retain_close`, `split_removal`); a split recording (clipout containing out+in) uses the second part for insertion. Partial-sample playback ("slice") hooks channel queue/create, the loaded-voice decoder and fill: a thread-local `requested_alias` marks the next voice, cue windows are keyed by
  file stem + sample rate + frame count + split ms so another recording never inherits a cut; a slice budget trims the tail of the decode ring (in source frames, before pitch) so the mixer cannot expose the next action's audio.
- Feedback ownership: the accepted mechanical event owns the cue; rejected/cleanup/stale/paused events are silent; native shot audio stays native; haptics added on native shots. Haptic patterns e.g. rear-hand shot 0.65 / support 0.16, 45 ms, 170 Hz; magazine-in 0.18/0.40 45 ms 140 Hz; action close 0.32 35 ms 150 Hz; global `vr_weaponHapticScale` 0-2.
  Event queue fixed 16, drop-oldest, each event revalidated against the CURRENT presentation (generation + definition) before playing.

## 8. The hand arbiter as implemented

One system decides who owns each hand every simulation frame. The design doc (`docs/vr-hand-interaction-redesign.md`) uses different names (InteractionFrame, Affordance, HandComposition); the **code** is below
(`gp/hand_interaction/core.hpp`, `runtime.cpp`, `frame.hpp`, `pose_plan.hpp`, `access.hpp`; scheduling in `gp/interaction_coordinator.cpp`, `interaction_schedule.hpp`).

### 8.1 Types
- `domain { none, carry, magazine, cylinder, tube, hinge, underbarrel, knife, sensor, world, tactical, gesture, grenade, special, launcher, reload_item, vehicle, ladder }`; `role { control, support, foregrip, firing, part, supply, knife, sensor, world, tactical }`;
  `button { grip, trigger, secondary }`; `capability { aim=1, action=2, fire=4, melee=8 }`; `recipe { single, knife_magazine, knife_part }`.
- `target {provider, object_identity{value, generation}, component, binding}` (validity is provider-local: world targets use entity index in (0,4000) and may have generation 0; head gestures have value 0 and generation = tracking reference; everything else needs both nonzero).
  `grasp {destination, purpose, maintained button, recipe, abilities, pose}`; `session {id, reference, started, event, actor, held, settling}`; `candidate {actor, desired grasp, event id, priority, distance, orientation, intent, exclusive_target, replaces}`;
  `decision {request, result, session_id, blocking_session}` with `rejection { none, invalid, no_edge, incompatible, occupied_target, lower_priority, capacity, commit_failed, wrong_route }`; `edges {press, release, down, armed, event}`; `input_history` (per hand and button cursors over the runtime's
  generation/presses/releases; each press gets a monotonically increasing event id); `frame` (one calibrated input + body frame + hand objects); `pose_plan` (one wrist target + finger recipe + attachments per hand). `arbiter` holds `std::array<session, 8>` and `std::array<decision, 64>`.

### 8.2 Per-frame batch (`interaction_coordinator::run`)
`begin(frame)` (copy frame, update input_history, clear candidates) -> `report_domains(acquisition)` (supply observed first) -> `synchronize(true)` -> exchange underbarrel supply -> each domain **collects offers** (`schedule::collections`, nightvision first) -> carry contributes ->
**`resolve()`** (single arbiter, both hands together) -> each domain **settles** granted offers (`schedule::settlements`: nightvision, grenades, mission equipment, detachable, cylinder, tube, break-action, launcher, reload_items, ladders) ->
support/magazine handoffs -> underbarrel settle -> equipment/knife settle returns a `claimed` hand mask -> heartbeat -> `carry::apply_grips(claimed)` -> world use gets `~claimed` hands and edges -> publish -> `report_domains` again -> `finish()`.
The `claimed` bitmask threads through so one grip edge is never consumed twice. When no weapon batch is admitted: idle-weapon lifecycle (feed settlement), scripted-use frame, or a **reconciliation-only** repeat (repeated input must still settle escrow but never starts another acquisition nor swallows deferred releases).
- **Resolution**: sort deterministically (finite first, **priority ascending** = lower number wins, distance ascending, orientation descending, then ids); at most one commit per hand per maintained button per batch (`used = true` *before* the commit: "a refused native request must not turn into a different action");
  an already-consumed event id is `no_edge`; if the candidate buffer overflowed ALL candidates are rejected ("never let truncation choose an arbitrary winner"); a better new candidate never steals an existing session.
  Seen priorities (not a spec): pin 10, ladder 5, world throwback 15, reload item 10, knife/part 20, carry control/support 20, supply 30, carry holster 30, world 20 direct / 50 remote. Distances are normalized by the contact radius ("<= 1 means inside").
- **Witness rule (the key invariant)**: a grant is real only if the provider *reports it afterwards*: providers read `granted(hand, domain, id, button, role)`, perform the native transaction, then call `observed(hand, grasp)` (the relation that actually exists) or `completed()` for one-shots. `finish()` marks every accepted
  decision that was not witnessed `commit_failed`; `synchronize()` drops sessions not witnessed this frame and admits newly witnessed relations (counted `unplanned` if refused); `publish()` copies sessions under a mutex for render consumers. Engine-side inventory changes arrive as `observe_carry`, never as presses.
- **Composition** (`compatible(a, b)`): only explicit combinations (knife in grip + magazine pinch on trigger with recipe `knife_magazine` / `knife_part`); `support_magazine_transfer` allows exactly one replacement (rear-hand release exchanging its support lease for its own magazine); `supply_exchange` swaps magazine <-> underbarrel supply of the same
  object; `exchange_supply` is an atomic escrow transfer. `compose_pose` picks the hand's pose driver by rank (part/supply 0, firing/foregrip 1, carry 2, knife 3, other 4) and denies melee while a grenade/special/vehicle/settling/part/supply grasp exists.
- **Provider contract** (five entry points per domain, declared in a `provider{id, report, collect, settle, lifecycle}` table): `collect_interactions(frame)` (propose), `update_interactions()` (act only on `hi::granted`), `report_interactions()` (`hi::observed` leases still held), `lifecycle(suspended)`, `present(...)`/pose plan.
  Phase orders are `constexpr std::array` lists with `static_assert(valid_bindings(providers))`: every non-null callback must appear in exactly the right phases; `valid_phase` forbids duplicates, so ordering regressions are compile errors. Fourteen providers: reload_items, detachable_reload, cylinder_reload, tube_reload,
  break_action_reload, launcher_reload, underbarrel, equipment, grenades, mission_equipment, nightvision, heartbeat, world, ladders.
- Fixed arbitration for a fresh Grip press: authored support-grip contact > occupied body slot > ground pickup / native use. A matching higher-priority attempt consumes the press even if its commit fails. Mission providers share one abdominal entry (cliffhanger -> flare -> notebook).
- Input consumption: `Grip`/`Trigger` are physical facts; the system interprets Select/Activate/DrawSupply/Release. The existing session always receives its own release even if pose is invalid; a candidate query result is cached per (hand, input sequence, tracking reference, ray) including empty results.
- Pitfalls that led to this design: cross-module `busy` / `allow_knife` queries made outcomes depend on update order; presenters overwriting bones in sequence made the last writer win; a *targetless script grip notification* was recorded as entity occupancy and blocked the first ammo draw.
- Helpers: `shared_slider` (`constraints.hpp`: foregrip that follows a rifle AND a longitudinal slider, `travel = sqrt(r^2 - perp^2) - along`, rigid-motion invariant), `mechanical_contacts.hpp` samplers (each verifies same assembly as the live carried object, holding hand tracked, units > 0). A grab press slightly before arrival is remembered 200 ms (`grab_intent`).

## 9. Carry, holsters, body slots, instances

- **Native inventory stays authoritative.** `reconcile_instances(native.owned)` each tick; unknown weapons are placed right waist -> left waist -> back -> overflow queue (extraction-only, never re-enqueued by rescanning); surviving guns are never moved by reconcile. Capacity 15 = engine array bound.
  Model: `instance {id, rules{waist, promote_support, abdominal, right_control}, location{absent, held, left_waist, right_waist, back, overflow, abdominal}, hold owner, overflow_order}` with `invariant()` (no shared hands, unique slots, `right_control` guns rear hand right only). Permits per class:
  sidearms {waist, promote}, compacts {waist}, launchers `right_control`, mission device `abdominal` ("a native SMG category is not a waist permit").
- **Selection**: VR requests `native_carry::select(token)` (which calls the engine's script method `switchtoweaponimmediate` rather than reproducing timers), records `expected_selection`, waits for acknowledgement, retries after 1500 ms, accepts external selection changes as superseding; a script pause retains the pre-script selection only if the post-script selection is unchanged
  ("a changed final native selection is authoritative, including empty hands"). The selection-finish hook defers a stale draw command and zeroes the resulting raise timer (850 ms witnessed) only for physically carried weapons on both simulation and prediction.
- **Grip transactions** (`claim_grip`, `release`): priority support-on-held-gun > control-on-carried-gun > holster draw; a matching attempt consumes the press even if the target changed. Both hands' release bits resolve for one frame together BEFORE slot/drop work: rear released -> support promoted to control or kept carry-only; both released -> stow to the slot under the hand
  (`hit(slots, hand_pos)`), else a physical drop through a callback that must confirm native removal AND a real world entity (match entity generation); failure retains the last holding hand (`reject()`; retry requires regrip + release). Eligible empty slot -> stow; occupied -> atomic exchange; wrong class -> reject and retain (never falls through to a world drop).
  Tracking/focus loss and pause are never deliberate drops; on death grips release but native death/checkpoint code owns the world weapon (no live drop). Grip hold vs toggle (`vr_weaponGripMode`): toggle only after a *committed* carry, a release press cannot also acquire.
- **Drop** uses the engine's own drop-item function (so physics/ammo copying are native) but places the entity BEFORE physics creation (otherwise it spawns at the flat muzzle); drop pose = gun pose composed with `muzzle * inverse(model.muzzle)` because world-model origin differs from the viewmodel; clearance precondition; velocity from a hand history clamped to `8*units` m/s
  with dt in (1 ms, 150 ms); mechanical state copied into a 128-record ledger and restored on pickup. Duplicate pickups: akimbo world pickups split (`akimbo_world_split.hpp`), entity free is the single retain point.
- **Holster geometry** (`weapon_holsters.hpp`, metres from the yaw-only head frame, scaled by `units_per_meter`): waist width 0.23, down 0.60 from head, back distance 0.22, back down 0.24, radii 0.16 / 0.20; muzzle-down holster canted 8 degrees inward; overlapping volumes resolve by normalized distance to the authored anchor, not iteration order; volumes are
  `body_reach_volume` (oriented box dilated by a sphere radius; `waist_reach` extends only behind (-0.22 m), outside and below so forward hand motions never become inventory actions). Chest slots (`locate_chest`): centre = head + 0.045 forward - 0.35 down, slots 0.18 apart (knife, tactical, lethal); abdominal slot = chest anchor - 0.24 m body-up, 20 cm x 14 cm + 10 cm tolerance.
  Holstered scene entries get the no-cast-shadow flag. Supply layout default half width 0.21, down 0.62, height 0.20 with per-weapon overrides.
- Display equipment with the engine's own world model scaled to a target diameter (`stowed_scale` clamp 0.05-1) rather than a clone; the chest item identity comes from player-state offhand slots, not the "currently primed" slot.
- Quick reload (`quick_reload.hpp`): dwell 1 s at a waist holster with a whitelisted feed, restart on any owner/instance/slot/reference/continuity change or > 150 ms gap.

## 10. Melee, grenades, equipment, optics, turrets

### 10.1 Melee (`melee_motion.hpp`, `native_melee.cpp`, `equipment_runtime.cpp`)
Detector: up to 9 contact points per tool + hand pose in the calibrated body frame with `sequence/reference/identity/continuity/pose_revision` (any change clears the stroke); valid `dt` in [0.001, 0.12]; per-point useful displacement integrated along a direction (reversal resets, so a fast tip cannot arm a slow stock);
hand-motion sanity instead of a tip-speed cap (reject hand translation > 16 m/s or rotation > 35 rad/s). Minimum speed / travel: bladed 1.0 m/s / 0.04 m, fist 2.8 / 0.08, firearm 3.5 / 0.12, shield and other 3.0 / 0.10. Knife speed <= 12 m/s, per-sample distance <= 0.45 m. Player-wide 400 ms hit gate and per-hand "must withdraw before re-hit".
Hit resolution: the engine's locational trace (same as native melee) with its mask, reject non-entity hits, clearance capsule head -> point free unless it hits the target; **damage by the engine** (`G_Damage(..., MOD_MELEE)`), preceded for bladed/shield by a native temp-entity hit event (type 0x3c) so impact audio/FX play; never subtract health or force animations; non-bladed damage scaled `max(1, (native + 1)/3)`;
blood FX at the physical socket via the engine's oriented-FX function. Free fists need grip squeezed and no claimed hand. Lethal VR melee can claim an "EXO punch" ragdoll boost through an embedded virtual script (`exo_punch.cpp`). Enemy melee damage scaled in the engine's damage filter only when player target + actor attacker on different non-zero teams + melee means of death (default x0.5, 0.1-2.0); dog pounce replaced by a script-branch bytecode redirect (exact 126-byte function check).

### 10.2 Grenades (`grenade_*.hpp/.cpp`, `native_grenade*.cpp`)
State machine `stowed -> safe -> prepared -> cooking -> release_pending`; behaviours per kind (pin gesture / cook / timed). **Pin pull requires `debited == true`**: the caller commits exactly one native ammunition debit first (`native::debit`, with a caller-held `committed` flag so retries after spawn failure never double-debit), then `pull_pin`. Two-hand pull: acquire 0.10 m,
stroke 0.08 m along the grenade model +Y, lateral motion cannot arm, offset > 0.5 m aborts. **Mouth pull**: mouth = head + (0.07 forward - 0.07 up), one explicit trigger attempt acquires within 0.045 m, then head-space delta must not go behind the head (> 2 cm tolerance) and per-frame jump <= `0.02 + 3*dt` m. `holder_trigger::consume` is evaluated against the
phase at the *start* of the transaction (a pin pulled this frame cannot reinterpret the same press as cook). Throw velocity from an 8-point ring over <= 80 ms (newest sample <= 100 ms old), speed clamped `min(speed, units*15, 9000/multiplier)` after gain (`vr_grenadeThrowSpeedScale` default 2.5 [0.25-10]; footballs multiply by `vr_footballThrowSpeedScale` 1.5 [1-3]; sample history resets on gap > 150 ms or > 0.75 units jump).
Launch: clearance capsule from eye to release point sized to the projectile; blocked release shortens to `fraction - 0.02` and zeroes velocity (world) but keeps it if the blocker is an NPC; position > 200 units from eye clamps to the eye; refuse while the native cook timer is non-zero; call the native projectile spawner (parent, pos, vel, weapon, fuse ms). A failed launch keeps the committed grenade and retries without resetting the fuse.
**Throwback**: the engine's HUD hint state identifies the in-flight grenade, VR tests a forgiving aim (70 deg half-angle or within 0.12 m), calls the native pickup, transfers the three stock hand-off cells (owner, offhand, cook timer) and clears the hint; expired-in-hand grenades respawn with correct blame scoped to one synchronous spawn.
Skinned M67 split into body/pin/lever partitions verified by vertex/triangle count and index-buffer hash. No native throw animation.

### 10.3 Placed equipment, claymore, use (`native_claymore.cpp`, `special_equipment_*`, `native_use.cpp`, `world_interaction.*`)
- Claymore: forward trace, surface normal.z >= 0.7071, head-to-ground visibility >= 0.97, a 5-point support footprint (+-0.07/0.14 m, each within 0.3 of mid, headroom trace), debit one, spawn the native projectile into the ground. Recover: ownership via script `owner`, credit ammo first (compare-commit), `notify death` to wake the native cleanup thread, then free; roll back the credit if the entity survives.
- **Use through the engine's own use list**: hook the engine's get-use-list and player-use plus the inner calls; a thread-local `ray* querying` swaps eye position, forward vector, area-query bounds and per-type range (via a register-preserving bridge) so native scoring, script orientation and volume tests run from the HAND; the result list can be forced to a single admitted target keyed `{entity, generation}` and a command lease
  `{target, held, reference, time}` (150 ms). Reach 2.2 m (0.5-3), cone 12 deg; `score_model` treats <= 0.12 m as direct contact, otherwise analytic closest point between ray and AABB; tie-breaks contact > cosine > distance > entity id so candidate order cannot flicker; visible-surface test on up to 14 extremal mesh vertices with a shared trace budget of 64.
  Grip with an empty hand drives native `+activate`: the engine's reliable command-notify with binding indices verified from its string table (press / release), admitted only when the native reliable queue would accept it; a tap still sends one press (`notification_gate`).
- Mission devices (laser designator, flare, notebook/UAV laptop): grasp states with preview/trigger-armed, held pose from the authored wrist `root = wrist * inverse(authored_wrist)` (two-hand carry via `from_to`), placement preview rendered with the engine's own objective material, native activation through the mission script's own waiters (private notify names), recovery of a mid-sequence save.
  Hinge devices accumulate angle (`advance_hinge_travel`), commit only on explicit release; the request goes through the native action-slot activation.

### 10.4 Night vision and "boolean toggle" equipment (`nightvision*.hpp/.cpp`, `native_action_slots.cpp`)
Head gesture: grip point in head-local `(forward, left, up)/units` (frame validated to 0.02); separate acquisition envelopes for raising from the forehead vs lowering from the face; drag travel >= 0.12 m along up and a final height threshold; abort if the head turns (`dot(forward0, forward) <= 0.8`), > 2 s, or a hand jump > 0.30 m. Use the physical grip point, not the cosmetic wrist.
The effect is requested through the engine's own **input binding dispatcher**: validate the bind table (index `19 + slot*2` equals `"+actionslot N"`), key-down now, key-up 50 ms later on the main pipeline; guarded by an `authorized` lambda re-evaluated on dispatch, timeline and 250 ms age. Native presentation is hijacked by detouring the toggle transition: detect on/off change from the player-state flag, run a device clock (fade,
power-on time from dvars, command serverTime dedupes prediction replays, epoch fences checkpoints), override the vision/visibility queries for the local client, defer native sounds until the renderer acknowledged full black (250 ms timeout), shorten the weapon-state timers. Generalization: "make the game's boolean visual toggle follow a VR device with its own timeline."

### 10.5 Physical shield (`native_shield.cpp`, `native_bullet_trace.*`, `shield_geometry.hpp`)
Hook the engine's bullet-trace once; it exposes ONE interceptor + <= 4 observers (startup-only registration). The shield overrides the result to a fabricated hit on the player (fraction, normal, surface, entity 0, hit location) when the tracked panel (16 triangles, finite segment-vs-triangle, front-only for incoming, both faces outgoing) is intersected, stamping it in a 16-entry thread-local ring keyed
`(result*, params*)` + memcmp so the later damage hook recognises its own unchanged result; the damage hook emits the shield-impact event and stops that pellet. The fraction is upper-bounded by the nearest original obstruction (a shield cannot protect through a wall). Only ballistic means of death. **The shotgun cone-damage path is separate** and needs its own call-site intercept (verified by argument-setup instructions).
A gun entirely behind the shield fires normally; barrel/forearm/upper-arm penetration (capsules 3.5 / 2.5 cm) blocks; a stale shield pose blocks firing and removes protection rather than falling back. Forearm mount: two-bone IK so the shield is constrained like a strapped object (`forearm_mount.hpp`).

### 10.6 Optics (`optic_*.hpp`, `optic_runtime.cpp`)
Catalog of physical lenses `{model, root bone, lens/reticle materials, lens centre (m), radius, magnification, pupil radius, scene clearance, thermal}`; variants by suffix list; binding scans the viewmodel for lens/reticle surfaces (rigid, <= 32 rigid lists, reticle texture semantic 2). Per eye: project the lens quad, eye-to-lens depth 0.03-0.8 units-per-metre, radius 0.001-0.1 m, magnification 1-12,
eye-box test `displacement*(1-1/mag) + 1/mag <= 0.94` (thermal 2x eye box). Independent-scene mode renders ONE extra narrow view for the eye closest to the optical axis (hysteresis 0.1), with a sampling window (+1% guard, reject rather than clamp) and a private near plane; thermal needs a verified native thermal contract.

### 10.7 Auxiliary-camera screens and fixed scopes (Javelin CLU, fixed sniper)
Second scene render of a narrow FOV from the muzzle pose (`auxiliary_scene::request` with a crop from the native eye projection) + the engine's own 2D HUD captured as an ink texture, composited on a head-space-anchored plane per eye; every layer checked against epoch, device/reference generation, owner revision and age (100-150 ms), else black. Optical half-tangent: engine ADS FOV is horizontal at 4:3, so `tan_y = tan(fov/2)*0.75`.
Near-eye entry is a physical gesture (rear aperture within 0.14 m, exit 0.20, facing cos 0.819/0.643; trigger must be seen released after opening). Native lock logic is bent with two minimal interventions (a UI-watch getter call-site hook and a script-method hook active only at one verified bytecode call site). **Hand-clutch aiming for a fixed turret**: squeeze = clutch; hand translation in the mount's head frame
maps to angles with gain `2*atan(tan_half_y)*57.2958/travel` (travel default 0.20 m spans one vertical FOV so zooming with still hands cannot pan), spatial dead band 0.00035 m with no temporal filter, regrip never moves aim, tracking jump > 0.25 m requires release.

### 10.8 Mounted turret / vehicle gun (`mounted_turret*.{hpp,cpp}`)
Detect mount via player-state entity flags (`& 0x3000` turret, `0x100000` vehicle gun) + linked entity number; identify the assembled DObj by model-name list and bone count. **Aim**: detour the engine's turret-aim function; compute desired angles from the controller, write them into the player's `viewangles`, call original (it only clamps/normalizes), restore the head angles in a scope exit; keep requested local aim separate from the converged/slewed solved angle. The fire-parameter builder
re-reads view angles after head look is restored, so give it the turret as the geometry source (non-client branch) while the player keeps damage attribution. **Block native fire in all three places the engine reads it** (attack bit in the current usercmd and two playerstate copies) unless the gripped hand's trigger generation matches. **Camera**: patch the two camera tag-query call sites to run `camera_geometry`
after the native result (seat from bind poses + current tag matrices; eye lowered relative to bind geometry so animation never moves the eye); compose the neutral bind of the player tag onto a stable root before HMD composition. **Independent presentation pose**: copy the client pose struct, write local angles (turret floats; vehicle SHORT2ANGLE yaw/pitch), call the native controller with the copy so spin/belt animation survives. Yaw from the aim-pivot tag, pitch from the aim tag
(using one pivot for both reverses yaw). Controller state machine: takeover = squeeze press then release while calibrated (never a handle grab), grip a handle within 0.12 m, hand >= 0.1 m from both axes, per-sample angular delta <= 45 deg, `firing` needs gripped hand + `fire_armed` set after a trigger-up since grip + same trigger generation; free-look stick yaw/pitch 60 / 90 deg/s with 0.2 deadzone. Wrist calibration: first frame of the native hands-on-gun animation
captures `gun^-1 * wrist` per hand (reject if > 12 units from the authored handle), then hands follow the controller and re-attach to handles by IK when gripped.

### 10.9 Script-driven weapons and equipment
When the engine's *scripts* own the fire loop (vehicle guns, designator), bend them minimally: `gsc::intercept_builtin("magicbullet" | "playfxontag" | "bullettrace" | ...)` accepted only inside the known script function's bytecode range and for the player's frame, with matching argument types; replace the position with the controller-driven muzzle, require a cover trace head->muzzle, call the original with stack isolation, deduct ammo by writing the script object field with an expected-value compare
(skip on `player_sustainAmmo`), acknowledge the XR press once; on refusal clear out-params and suppress so the native loop sees no shot. Native shoot-button polling is replaced by a builtin that returns 1 only when fire is armed; a tap bridges the native 50 ms poll via a pending press (150 ms). Script machinery (virtual sources, hooks with guards, bytecode inspectors) is covered in refs 03/10.

## 11. Minimal path for a new game

1. **Skeleton hook** (section 1.1): move the weapon bone to the controller with a fixed offset, request all bones and verify the calculated bits, latch per epoch, ignore arms. Gate on byte verification; publish nothing else yet.
2. **Fire hook** (5.1): override shot origin/direction from the solved muzzle bone with the thread-local scoped shot context; drop (never eye-fire) shots when pose/input is stale; eye->muzzle cover trace.
3. **Render binding** (section 3) as soon as anything else must sit on the muzzle or hand (HUD, magazines, laser).
4. **Two-bone IK** for the arms with elbow pole + oriented bend plane; hide native arms if it looks wrong; then forearm twist and hands-only objects for empty hands.
5. **Support hand** two-handed aim (`aimed_rotation`, 0.10 / 0.22 m hysteresis, fresh-squeeze rule).
6. **Authoring**: sample one idle frame per weapon, gun-local wrists + parent-local fingers, cm/2.54 once, SHA-256 in the header, left hand by reflection; tune in the headset.
7. **Admission** (7.2): exact receiver/bone-count/attachment/native-name/capacity gates with reason strings *before* any ammo write; fail closed to native behaviour.
8. **Physical reloads, one family at a time** (6): block native reload at the earliest function; pure planner + conservation test first; compare-and-commit adapter; escrow; reconcile. Add the chambered +1 by redirecting capacity call sites only if the game has a chamber bonus.
9. **Arbiter** as soon as two systems want the same hand (8): provider table, deterministic resolve, witness rule, compile-time phase validation.
10. Then carry/holsters, melee, grenades, optics, scripted devices. Budget per weapon: about 8 headers of measured numbers once its mechanism family exists; the per-receiver authored tables (poses, contacts, fills, 46-entry finger table) dominate effort, not code.

Skip until needed: independent fire (dual wield), clip ledger (two copies of one gun), mesh partition/skinned subset visibility (needed once a held magazine must disappear from a merged viewmodel), FX variants and save integration.

## 12. Pitfall checklist (all observed)
- Partial skeleton query corrected twice -> request all bones, latch epoch, verify bits.
- Rejection not latched -> second eye sees a different pose. Presentation gate reused firing gate -> magazine vanishes. Render check stricter than authorization -> blink.
- Fixed world wrist offset instead of grip-local -> wrist swims during rotation. Fitted pivots are not anatomical truth.
- Right-hand finger quaternions on left joints; `_ri` matching `ring`; whole profile rejected for two optional leaf bones; `tag_weapon` used as gun origin; rotation channels multiplied twice.
- Native timers used as transaction cooldowns; reload hook placed after the begin (repeated start/end); blanket capacity patch hits HUD and pickups.
- Capacity assumed from the model name (32 vs 33/15); one unregistered attachment alias rejected a whole scene; alternate-feed names sharing a prefix admitted by prefix.
- Refund/retry after the `weapon_fired` notify; stock attack not suppressed at the prediction boundary; commandTime faked per hand; burst pause converted twice.
- Comparison of definition tokens made two copies of one gun fight for the muzzle mailbox; stale ledger reads outside the server scope.
- Latest-pose fallback for held parts; lighting-handle storage reallocated; descriptors recycled while async skin jobs live; cache keyed by address across zone unload.
- Per-frame hand-busy queries and last-writer-wins presenters instead of one arbiter; grips acquired from a held button after tracking loss; a failed write retried by the same squeeze.
- Presentation (comfort translation, snapped hands, recoil) allowed to feed input geometry.

## Appendix A: H2-only addresses and offsets (examples; never reuse)
- Skeleton calc `0x140657D80` (prologue 19-byte mask); client DObj lookup `client_object_address` with handles 4000 (primary) / 4001 (secondary); lock `0x140659010`, unlock `0x1406596C0`, calc pose `0x14038C9E0`, viewmodel submit `0x140776E10`, empty-hand create `0x140653180` / free `0x1406539D0`,
  add-view-weapon `0x1403B49A0`; render view pointer `0x141E39D30` (+0x58 world view offset). DObj: models `+0xD8`, model count byte `+15`, bone count `+0x10`, calculated bits `+0x80` (8 words), matrices `+0xA8` (32-byte `DObjAnimMat`: quat4, trans3, weight), epoch/timestamp `+0xB0`, hidden-bone mask `+0xB8`.
- Render binding: skin site `0x14071F167 -> 0x14071F190`, submit `0x14071FF74 -> 0x140774F10`, viewmodels `0x140779439 -> 0x14077F690`; hidden-bone `0x14071F434 -> 0x140658AB0`, rigid packet `0x14071F7C1 -> 0x14071E680`, skin packet `0x14071F879 -> 0x14071E020`; scene entry `+0x68` lighting/surface handle, entity `+0x98` object.
  Scene models: native filter `R_FilterXModelIntoScene`, generate `0x140778E60`, prepare `0x140723380`, build call `0x140723C04`, radius call `0x140724A56`.
- Fire chain: `G_FireWeapon` call `0x1404AD1ED`, fire `0x140518A40`, param builder `0x140518880` (call `0x140518B91`), bullet call `0x140518C2E`, ADS spread query `0x140518BB9` / accessor `0x1406A1670`, projectile dispatch `0x140518C6E` (type 3), class-6 launcher fire `0x14051CD40`; bullet trace `0x1404AC4E0`;
  `G_TraceCapsule` masks `0x280e831` (cover/shot), actors `0xc000`; G_Damage `0x1404BD2E0`; temp entity `0x1405183E0`; locational trace `0x1404C91B0` (mask `0x280E091`).
- Reload boundary: fire/PM `0x140699240`, consume `0x14069E430`, reload `0x1406946D0`, `PM_BeginWeaponReload` `0x140693D00` (automatic begin call `0x140698433`, segmented query `0x140693E53`); capacity redirect call sites `0x1406965A1`, `0x14069E244` (refill `0x14069E1B0`); clip key fn `0x14069D250`, reserve key fn `0x14069D0B0`; weapon defs table `0x14CE01580` (token & 0x1ff).
- Player state: weapon `+0x3bc` (& 511), flags `+0x3c0` (0x80 weapons disabled, 0x40 night vision on, 0x4000 alternate/"tracker open"), commandTime `+0x4c`, pm_flags `+0x54`, weapon state `+0x2c0`, weapon timer `+0x2b4` (raise 850 ms witnessed), animation action `+0x2b0` (tactical 20, empty 21), viewangles `+0x108`, owned weapon tokens `+0x2f8` x15,
  per-slot flags `+0x335 + 8i`, clip cells `+0x4a4 + i*24`, reserve cells `+0x3f0 + i*12`, action-slot types `+0x1fa0` / weapons `+0x1fb0`, cook timer `+0x64`, throwback owner `+0x1c`, offhand `+0x3b0`, usercmd attack copies `+0xe860 / 0xe90c / 0xe918`, entity flags `& 0x103000` busy gate.
- WeaponDef: weapClass `1488` (H2: rifle 0, sniper 1, MG 2, SMG 3, shotgun 4, pistol 5, GL 6, RPG 7), hipGunKickPitch min/max `3196/3200`, hipViewKickPitch `3232/3236`, clip capacity `0x6f0`, noPartial `0xe95`, segmented `0xe96`, hideTags `0x60`, worldModel `0x470`, fuse accessor `0x1406A4080` (SDK labels reversed: player fuse at `+0x7D4`).
- Fire packet 0x50 bytes (shot geometry first 0x30; weapon `+0x3c`, alternate `+0x44`, definition `+0x48`); bullet trace results 0x50 (target `+0x30`, location `+0x26`); bullet fire params start `+0x1c`, direction `+0x34`.
- Input/notify: notify-on-command `0x1403D3A90`, execute binding `0x1403CF1E0`, bind-name table `0x140BF84E0` (73 `+activate`, 74 `-activate`, `+actionslot N` at `19 + slot*2`), sound play `0x140380430`, FX oriented play `0x140453F50`, mounted aim `0x140534250`, turret fire params `0x140534500`, GetUseList `0x140526670`, Player_Use `0x140527380`, finish weapon change `0x140698AE0`, night-vision toggle `0x140697780`.
- Assets: hand model `viewhands_us_army` (68 bones, `j_{thumb,index,mid,ring,pinky}_{le,ri}_{0,1,2}`, `j_wrist_*`, `j_elbow_*`, `j_shoulder_*`, `j_wristtwist_*`), receivers `h2_viewmodel_<gun>_base` with bones `j_gun`, `tag_flash`, `tag_clip`, `tag_brass`, `tag_view`; authored units native inches; weapon README status snapshots (M9/USP HMD-accepted Sept 2026, most others offline candidates).
