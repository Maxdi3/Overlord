# 14 — Math and conventions

Everything numeric the port needs, with the numbers Overlord actually uses. Most
VR-port bugs are sign/handedness/unit mistakes that look plausible; write the
sanity tests in the last section **before** trusting any conversion. Pointers:
`head_pose_bridge.cpp`, `vr/engine_stereo_view.cpp`, `vr/engine_stereo_bridge.cpp`,
`vr/pose_filter.hpp`, `vr/spatial_math.hpp`, `vr/continuous_view_angles.hpp`,
`vr/game_view.hpp`, `vr/gameplay/hands/pose_solver.hpp`, `vr/spatial_panel.hpp`,
`vr/auxiliary_scene.hpp`, `vr/scene_compositor.cpp`. Behavioural context is in 05 (stereo),
06 (head/camera), 08 (IK).

## 1. Spaces and representations

| Space | Convention |
|---|---|
| OpenXR / OpenVR tracking | Right-handed, meters: +X right, +Y up, **−Z forward** (OpenXR LOCAL/VIEW/STAGE all the same) |
| H2 / Quake-family world | Units (H2: inches). Basis **rows**: `axis[0]` forward, `axis[1]` **left**, `axis[2]` up; +Z up. Screen-right = −`axis[1]`. Yaw increases to the **left** (a right turn is negative yaw); the usercmd pitch sign is **positive = look down** |
| H2 D3D view space | Left-handed, +Z forward, row vectors, **infinite reverse-Z**; view matrix columns are `(−right, up, forward)` (built from the axis rows) |
| Overlord portable core | forward/left/up row-vector basis, translation in **meters**; normalize into this at the adapter boundary and keep engine units out of policy code |
| Quaternions | `xyzw`, active rotation, column-vector (`pose_filter`, `spatial_math`); `from_axis` takes row basis vectors and transposes |
| Row-vector matrices | translation in row 3 (`vp[12..15]`); `clip[c] = vp[12+c] + Σ_axis p[axis]·vp[axis*4+c]` |

Rule of thumb: keep **every** engine-specific permutation/sign in one function pair
(`tracking_to_engine`, `engine_orientation`). Everything else works in the portable basis.

## 2. Tracking → engine conversion

```cpp
// vector: tracking (x right, y up, z back) -> engine (forward, left, up)
v_engine = { -v.z, -v.x, v.y };

// rotation: transform the basis vectors, not the elements
forward = tracking_to_engine(R · (0, 0,-1));
left    = tracking_to_engine(R · (-1, 0, 0));
up      = tracking_to_engine(R · ( 0, 1, 0));      // rows of the engine basis; det must be +1
// validate: rows unit within 0.05, pairwise |dot| < 0.05, |det − 1| <= 0.05
```

Relative to the recenter reference (`R_ref` = yaw-only rotation about tracking-up, `p_ref` = full position):

```
local_pos_m = tracking_to_engine( R_refᵀ · (p − p_ref) )
local_pos_u = local_pos_m · units_per_meter
local_rot   = engine_orientation( R_refᵀ · R )
```

World position of a tracked point (head or controller), given the camera's **yaw-only** base basis
`B` (rows forward/left/up) and world origin `W` (spatial frame):

```
world = W + Σ_i local_pos_m[i] · units_per_meter · B[i]          // vector rows, i = forward,left,up
world_axis[row] = Σ_i local_rot[row][i] · B[i]
```

Yaw-only base means pitch/roll come only from the HMD. Camera position uses the same sum with
`camera_base_origin` as `W`. `tracking_to_world` is **transactional**: invalid scale (≤0 or >10000) or a
malformed orientation leaves the output untouched.

Worked check (scale 10, recenter at (0, 1.6, 0) m): pose (0.1, 1.7, −0.2) m -> forward 2, left −1, up 1
units; yaw 90° in tracking maps to the matching engine basis.

## 3. Units and world scale

- `units_per_meter` is a physical unit conversion: **H2 = 39.3700787** (1 unit = 1 inch). Derive it from the
  engine's unit definition, not from eye height. (A heuristic "native standing eye height / ~1.65 m" is
  not used by Overlord; use it only as a tuning aid for engines with nonphysical units.)
- It scales head translation, hand/controller positions, IPD/eye offset, near plane clamp, interaction reach,
  shoulder/holster offsets and all authored offsets (authored in cm -> converted once with `cm/2.54` -> inches).
  Keep solver math in meters (`scale(pos, 1/units)`) and convert at the boundary.
- Setting `vr_worldScale`: range 0.01–10000 (bridge accepts (0, 10000]); launcher shows
  `percent = 100 · native/stored` (stored 39.37 = 100%; **higher % = world looks larger**). It applies on
  next game launch ("world size also scales tracked movement and reach"). Test at scales
  {1, 20, 25, 39.37, 40, 80, 100}.
- When it changes mid-session solvers `rescale(old/new)` or `rebase()`; carried ladders/anchors detach.

## 4. Eye offsets, IPD and the shared camera

From the runtime (one `xrLocateViews` sample, or equivalent):

```
head_pos   = (pos_L + pos_R) / 2
head_rot   = slerp(q_L, q_R, 0.5)
eye_offset_head_local[eye] = R_headᵀ · (p_eye − p_head)          // published with tan(angle) per eye
IPD        = |p_R − p_L|          // valid [0.02, 0.12] m or the family is invalid; default 0.064 m
half_eye_offset_units = IPD · 0.5 · units_per_meter   // valid (0, 1000] units
```

- Engine eye origin: `origin_eye = origin_center ± left_axis · half_eye_offset` (**view_eye 0 moves along +left**; the
  `output_eye -> view_eye` swap flag handles `swap_eyes`). Test: 63 mm at scale 100 -> eyes ±3.15 units (6.3 apart).
- Reject canted eyes: `|dot(q_L, q_R)| < 0.99999` (≈ 0.51° relative rotation) is fatal for a shared-camera engine.
  Separate locate calls can update prediction even at the same display time; VIEW-space disagreement is not cant.
- When the engine keeps a **relative** eye offset (frontend rebase), set
  `relative_eye = natural_rebase + (eye_origin − natural_primary_origin)`. Copying the absolute origin makes
  models subtract the world camera twice (a test caught the absolute form).
- Light descriptors computed camera-relative need the same per-eye rebasing (05).

## 5. Projection from runtime FOV

Inputs: `l = tan(angleLeft)` (negative), `r = tan(angleRight)`, `d = tan(angleDown)` (negative), `u = tan(angleUp)`;
`w = r − l`, `h = u − d`. Valid if `l<0<r`, `d<0<u`, `w,h ∈ [0.2, 20]` (write path accepts up to 40).

**H2 form** (left-handed +Z forward, row vectors, infinite reverse-Z; `n` = near distance in units):

```
m0 = 2/w     m5 = 2/h
m8 = −(r+l)/w   m9 = −(u+d)/h     // row 2: off-axis terms
m11 = 1      m14 = n               // all others 0
[ 2/w         0           0   0 ]
[ 0           2/h         0   0 ]
[ −(r+l)/w    −(u+d)/h    0   1 ]
[ 0           0           n   0 ]
```

**Standard OpenXR/D3D** (right-handed −Z forward, column vectors, finite depth [0,1]):

```
x: 2/w, 0, (r+l)/w, 0        y: 0, 2/h, (u+d)/h, 0
z: 0, 0, −f/(f−n), −f·n/(f−n)   w: 0, 0, −1, 0
```

- Always verify the sign of the off-center terms against a capture: each runtime frustum edge must map to
  exactly NDC ±1 (`ndc = tan·scale + center`, mirrored centers for left/right eyes; test: `scale = 2/w`,
  `center_x = −(r+l)/w`). A sign error converges/diverges the eyes and is uncomfortable instantly.
- **Read-back** the optics actually rendered from the finished slot: `l = (−1 − m8)/m0`, `r = (1 − m8)/m0`,
  `d = (−1 − m9)/m5`, `u = (1 − m9)/m5`; insist on the exact axis-aligned form (`m11 = 1`, `m15 = 0`, zeros elsewhere).
- **Legacy symmetric scalars** the engine keeps for culling/occlusion: `tan_half_x = max(−l, r)`,
  `tan_half_y = max(−d, u)` (write to every copy).
- **Inverse-projection ray constants** (fog/post reconstruct view rays as `(clip.xy · v.xz + v.yw) · linearDepth`):
  `{1/m0, −m8/m0, 1/m5, −m9/m5}`, recomputed from the eye's final matrix (including jitter).
- **Finalization contract:** the engine's finalized VP is built from rotation × projection and **excludes camera
  translation**; its inverse contains the eye origin. Derived eye slots are not valid until the engine's finalizer has
  produced `VP` and `inverse VP` for the eye; reject copied center/other-eye inverses, non-finite or all-zero
  matrices (energy = Σ|x| ≥ 0.1), mixed publications. Identical-FOV eyes (SteamVR/VDXR) have equal VP and differ only
  in the inverse — still valid.
- **OpenVR pitfall:** `GetProjectionRaw` vertical names are inverted: its `top` is already the engine's "down" tangent and
  `bottom` the "up" tangent. Pass through, do not negate.
- **Mono-capture fallback remap** (symmetric centered game frustum -> asymmetric eye, `scene_compositor.cpp`):
  `scale_u=(r−l)/(2·hx)`, `offset_u=(l+hx)/(2·hx)`, `scale_v=(u−d)/(2·hy)`, `offset_v=(hy−u)/(2·hy)` with
  `hx,hy` the symmetric half-tangents; clamp UV to a half texel.
- **Depth-hack (viewmodel) projection** has its own near term: restore the full current-eye asymmetric matrix but keep the
  dedicated near (`m14` only), gated on matching cache/input versions, finite floats, `proj[0] > 0`, `proj[5] > 0`,
  `|proj[11] − 1| ≤ 0.001`.

## 6. Near plane and culling unions

- **Near clamp:** `near = clamp(native_near, half_ipd/8, 1.0 units)` (never pushed outward past 1.0, never below
  `half_ipd/8`). Reason: scripted cameras request 0.01; at finite IPD that needs a near-180° shared culling frustum and
  would reject the pair. A fixed scope compositor keeps native near.
- **Culling union** (shared visibility frustum that encloses both translated eye frusta; keep center origin and basis):

```
expansion  = half_ipd / near                       // reject if > 20
tan_left   = min(l_L, l_R) − expansion
tan_right  = max(r_L, r_R) + expansion + 0.1       // 0.1 tangent right margin (culling only; optics stay exact)
tan_down   = min(d_L, d_R)   tan_up = max(u_L, u_R)
```

  Derivation: at forward depth z, the left eye's leftmost ray point is `−halfIPD + z·tanL`; `halfIPD/z` is largest at the
  near plane, so expanding by `halfIPD/near` encloses both frusta over the whole depth range. Test numbers:
  tan_left −4.526 / tan_right 4.626 with near-eye expansion. Apply to the **admitted production pair only**; proof/unarmed
  frames keep the natural view. A fullscreen scripted scope uses a narrow envelope instead (`±max(0.1, x) ± expansion`,
  `y = max(0.1, native y)`), because gathering unseen wide-angle geometry overflows native model lists.
- Legacy symmetric half-extents must enclose the longer optical side.
- **FX/particle camera planes** (engine's separate camera with side planes, plane count ∈ {5,6}, `valid==1`): slopes
  `{max(rR)+0.1, −min(l), −min(d), max(u)}` for (right, left, bottom, top); `n = slope·forward ± lateral_axis`, normalized;
  `D = n·center − |n·left_axis|·half_ipd`, then `nextafter(D, −inf)` (engine keeps points with `n·p ≥ D`, so supporting both
  eye origins needs the smaller D). Patch at the one verified builder call, before FX workers read it; never asynchronously.
  (H2: planes written at camera `+0x20`; a header comment says `+0x10` — the code is right.)
- **View-slot validation:** each basis row length² ∈ [0.81, 1.21], pairwise |dot| ≤ 0.1; origin finite.
- **Crop (scope/auxiliary view)**, window `(x, y, w, h)` normalized with y from the top, `sx=w, sy=h,
  cx = 2x + w − 1, cy = 1 − 2y − h`: for projection and VP rows `m[r][0] = (m[r][0] − cx·m[r][3])/sx`,
  `m[r][1] = (m[r][1] − cy·m[r][3])/sy`; inverse VP: `inv[0..3] *= sx`, `inv[4..7] *= sy`,
  `inv[12+c] += cx·inv[c] + cy·inv[4+c]` (so the engine's jitter finalizer never runs twice); then recompute legacy
  half-tangents and ray constants. Near-plane rescale: scale projection/VP column 2 and inverse-VP row 2 by
  `new_near/old_near` (ratio ∈ 1e-4..1e4).
- **Temporal history per eye:** previous VP and `previous_eye_delta = (origin_now − origin_prev)·engine_scale` come from that
  eye's *own* previous committed state; if no committed state for the same device generation and an earlier pair, seed with
  the current (zero motion). Commit only a complete, in-order pair.

## 7. Rotations, quaternions and Euler

- Normalize every runtime quaternion; reject `|q|² − 1| > 0.01`. `q` and `−q` are the same rotation: use `|dot|` for
  angle (`2·acos(|dot|)`), flip sign before slerp (shortest arc); a quaternion sign flip must not spin.
- Midpoint of two rotations: `slerp(qL, qR, 0.5)` (`d < 0.9995` -> sin-weighted, else nlerp).
- **Compose with matrices/quaternions, never by adding Euler angles.** Relative rotation of a change since last frame:
  `source_prev^T · source`; additive scripted rotation accumulates `full_base = full_base · prev^T · source`.
- Matrix <-> quaternion: standard trace branch with the largest-diagonal fallback (`pose_filter::quaternion/rotation`);
  round-trip test at 0/90/179.9/−179.9/180°.
- Pose rigid transforms: `compose(a,b) = (a.p + a.R·b.p, a.R·b.R)`, `inverse(p) = (−Rᵀp, Rᵀ)`,
  head-filter correction `= filtered · inverse(raw)` (applied to head and hands alike).
- Rigid delta for baked parts: `desired · inverse(source_bind)` applied exactly once.
- **Euler conventions (engine command only).** From rows forward `f`, left `l`, up `u`:
  `pitch = atan2(f.z, hypot(f.x,f.y))`, `yaw = atan2(f.y, f.x)`, `roll = atan2(l.z, u.z)`; at the pole (`hypot < 1e-4`) keep yaw and
  recover roll from the left axis (handles upside-down heads). Then compare with the alternate triple
  `(180 − pitch, yaw + 180, roll + 180)` and take the one nearest the previous value (squared wrapped distance). Use
  `std::remainder(x, 360)` for all wrapping. Axis from H2 angles (pitch p, yaw y, roll r):
  `[cp·cy, cp·sy, −sp]`, `[−sy·cr + sp·cy·sr, cy·cr + sp·sy·sr, cp·sr]`, `[sy·sr + sp·cy·cr, −cy·sr + sp·sy·cr, cp·cr]`
  (pitch positive looks down in this form; `continuous_angles` uses the opposite pitch sign).
- **Heading (everything that is not the command pair):** geometric, never Euler.
  `heading = atan2(f.y − l.x, f.x + l.y)` (fused / tilt-free yaw; keep the previous value when `hypot(...) < 0.01`, i.e.
  within ~8° of fully inverted); forward-projected `atan2(f.y, f.x)` with a **left-axis fallback** `atan2(−l.x, l.y)` when forward
  is vertical; recenter uses the horizontal forward projection only when upright. Why: after headset standby/resume the Euler
  branch (pitch ≈162, yaw ≈72–74, roll ≈−177) put shoulders 180° off the physical heading. Full rules in 06 §2.
- Horizon roll (diagnostics): with `r_up = normalize(world_up − (world_up·f)f)`,
  `roll = atan2((r_up × u)·f, r_up·u)`; undefined when `|r_up|² < 1e-4`.
- Same-axis heading delta for authored bones passing through a pole: for the axis `i` maximizing
  `min(|a_xy|², |b_xy|²)` use `atan2(a.x·b.y − a.y·b.x, a·b)` over the XY components.

## 8. Packed command angles

`degrees = packed · (360 / 2^24) + delta_angle` (H2: 24-bit turn in a 32-bit field; 0.000021457672°/count). Check the target
engine's scale (16-bit shorts are common). Final view angle = decoded + `ps->delta_angles`; subtract the delta before
injecting an absolute pitch. Right turn = negative yaw; positive command pitch looks down; clamps (±85° pitch in H2) are applied
by the engine after decode — see 06 §3 for the exact-command restore.

## 9. Recenter math

```
reference_heading = atan2(f.y, f.x) of the horizontal forward   (upright: up.z > 0 and hypot > 0.01; else keep the Euler branch)
R_ref = yaw-only rotation about tracking-up by reference_heading
p_ref = full tracked position (x, y, z): recenter zeros ALL three translation axes, including height
local_angles = { abs_pitch, 0, abs_roll }   // gravity-relative pitch/roll are preserved, even on the first pose
```

Overlord recenters height too (the camera lands on the engine's eye position); a port may keep floor-relative height by
holding `p_ref.y` fixed — document the choice. Only a FOCUSED + fully TRACKED sample may seed it (06 §1). Native facing is
preserved across a recenter: `reference_offset += old.heading − new.heading`.

## 10. Two-bone arm IK (`pose_solver.hpp::solve_limb`)

```
d      = |wrist − shoulder|
stretch= clamp((d + 0.001) / (U + L), 1, 1.25)        // allow up to 125% of native reach; scale both segments
U *= stretch; L *= stretch
reach  = clamp(d, |U − L| + 0.001, U + L − 0.001)      // inside (|U−L|+ε, U+L−ε)
dir    = d > 1e-5 ? (wrist − shoulder)/d : body_forward
along  = (U² − L² + reach²) / (2·reach)
elbow  = shoulder + dir·along + pole·sqrt(max(0, U² − along²))
wrist' = shoulder + dir·reach                           // limited = |reach − d| > 0.01
```

- **Elbow pole:** `rest = unit(left·(±0.6) − up)` (left arm +, right arm −; outward/down bend, mirrored), then **swing-transport**
  it from body-forward to the reach direction with a shortest-arc quaternion `from_to(body_forward, dir)`; project out `dir`.
  Exactly behind (`dot < −0.9999`) is an ambiguous half-turn: use the rest bend itself as the axis. Do **not** project the animated
  elbow or a fixed world hint directly: it is singular when the hand points along the hint. An explicit elbow hint (if
  length(projected) > 0.001) replaces the pole.
- **Orient both segments through the same bend plane** (`orient_arm`): carry upper and lower segments with frames built
  from `(direction, plane_normal)`; independent shortest-arc rotations spin the elbow through 360° when a segment circles the
  antiparallel direction (a 3.47° forearm change used to give 152.63° of elbow roll; now <4°). Straight arms use the authored
  anatomical frame (shoulder rotation Y, then Z), never a world axis.
- Validation (transactional, never write a partial pose): limb lengths in [0.1, 100], distance ≤ 200, body axes orthonormal
  (±0.001, right-handed), parents acyclic, norms ∈ [0.5, 1.5], arms are descendants shoulder->elbow->wrist.
- Shoulder estimate: center = `head − (forward·back + up·down)`, sides ± `left·half_width` (0.18 / 0.20 / 0.08 m, 06 §7).
- Wrist offset: a grip-local rigid translation (`inward, back, up`, ±0.5 m) rotated by the grip axes and scaled by units/m — a
  fixed world offset cannot cancel the controller-origin motion when rotating about the real wrist.

## 11. Pose filters

```
half  = half_life · (strength/100) / (1 + (v/v_ref)² + (ω/ω_ref)²)
alpha = 1 − 2^(−dt/half) = −expm1(−ln2 · dt / half)
```

Parameters (head 15 ms / 2 mm / 0.25°, hand 45 ms / 15 mm / 2°, desktop 120 ms / 0 / 30°, with speed references), reseed
rules and bypass behaviour: 06 §8. `v` = raw position change / dt / v_ref, `ω` = angle(prev_q, raw_q)/dt/ω_ref. Position and
quaternion both use `alpha`; the residual lag is then clamped. Filters are not prediction.

## 12. Color transfer and compositing math

Used in several places (display->linear conversion into the ring, desktop mirror, HUD compositing):

```
sRGB EOTF (decode): x <= 0.04045 ? x/12.92 : ((x + 0.055)/1.055)^2.4
sRGB OETF (encode): x <= 0.0031308 ? 12.92·x : 1.055·x^(1/2.4) − 0.055
```

- The engine's final PostFX target is **display-encoded**; submit **linear** (`ColorSpace_Linear`), so convert with the EOTF
  in a fullscreen-triangle pass (positions (−1,−1), (−1,3), (3,−1), `Draw(3)`, no vertex buffer, no diagonal seam).
- Desktop mirror reads linear eye textures and must apply the OETF because the UI shader expects display-encoded input.
- HUD captured as **premultiplied, gamma-encoded** layers: composite in encoded space (`fg + bg·(1 − fg.a)`), decode back to
  linear once; reinterpret `_SRGB` SRVs as `_UNORM` for encoded sources so hardware does not decode. Native additive ink = RGB with
  zero alpha; reverse-subtract borders need a separate subtractive texture.
- Pause dim: multiply linear color by `(1 − dim)` (OpenVR: `FadeToColor` alpha 140/255).

## 13. UI and overlay geometry

- **Projected quad:** `clip = vp[12..15] + Σ (world − eye_origin)[axis] · vp[axis·4 ..]` (subtract the *same* eye origin once because
  the finalized VP excludes camera translation); for HUD set `z = w·0.5` (no scene depth test) unless `preserve_depth`. Freeze the
  world corners **once per pair** so both eyes see the same quad (never billboard per eye). Do not reject vertices with `w < near` on
  large panels: let the GPU clip homogeneous coordinates (`clip_mode::hardware`) or the panel pops on big head turns. Billboard
  basis validated orthonormal (`|dot| − 1 < 0.002`).
- **Head-relative canvases:** panel width `= 2·d·tan(30°)` (tan 30° = 0.57735) for a 60° panel at distance `d` (fixed scope plane
  d = 2.0 m; narrative panels at 2.0 / 1.6 / 1.2 m); an 80° readable window uses tangent `tan 40° = 0.839099631`. Fixed visual size via a
  virtual tangent `tangent = viewport / (2·reference_m·pixels_per_meter)`.
- **Shared angular canvas** for full-screen overlays (blood, low-health): union of both eyes' tangent extents
  (`min(left)`, `max(right)`, ...), projected into each eye's clip rect so equal view directions sample equal UVs. Requires
  `tan_left<0<tan_right`, `tan_down<0<tan_up`, widths ≥ 0.001, |tan| ≤ 100.
- **Recording/crop transfer between asymmetric eyes** (tangent space, "capture at infinity"):
  `u' = (src.tl + x·src_w − tgt.tl)/tgt_w`, `v' = (tgt.tu − src.tu + y·src_h)/tgt_h`, clamp 0..1.
- **Menu surfaces:** plane 0.9 m wide at 0.5 m, each older menu pushed back 0.25 m (≤ 4), popup dark layer 0.05 m in front of its
  parent, theater cylinder radius 3 m with width `radius·π/2` (90° arc); OpenVR curvature `= width/(2π·radius)`, OpenXR cylinder
  central angle `= width/radius`, pose center `= anchor + forward·(distance − radius)`; layer rotation basis `(right, up, −forward)`.
  Anchor is yaw-only from the head (derive yaw from the right axis if forward projects to < 0.05). Pointer ray = controller aim
  forward = `−Z` column of the aim orientation; ray hit valid if distance ∈ (0, 20) m and UV ∈ [0,1]; ray–cylinder is a quadratic
  with `atan2` for the arc angle; letterbox UV mapping into 1920×1080 with 1e-5 tolerance.
- Optic lens (physical scope, `gameplay/optic_geometry.hpp`): lens quad billboard per eye (right = −lens.left); eye-to-lens depth
  along the lens axis must be 0.03–0.8 m (× units/m), lens radius 1 mm–10 cm (× units/m), magnification 1–12; lens UV =
  `(0.5 + δ·right/(2R), 0.5 − δ·up/(2R))`; visibility: `displacement = hypot(2(u−.5), 2(v−.5))`, ordinary scopes need
  `displacement < 2` and `displacement·(1−1/mag) + 1/mag ≤ 0.94` (1.0 when `preserve_depth`); thermal admits a partial image
  through a wider pupil (eye-box scale 2). The non-aiming eye gets a dark aperture, never the aiming eye's image.

## 14. Threshold and tolerance quick table

| Item | Value |
|---|---|
| Rotation validity (bridge) | rows unit ±0.05, pairwise dot < 0.05, det ±0.05 |
| Rotation validity (`pose_filter::valid`) | R·Rᵀ within 0.05 of I, det > 0.95, |p| ≤ 100000 |
| Body-estimator basis | dot error ≤ 0.002, det ≥ 0.998 |
| Quaternion norm² | within 0.01 of 1 |
| Eye-cant guard | `|dot| ≥ 0.99999` |
| IPD / half offset | [0.02, 0.12] m / (0, 1000] units |
| Tangent size | w,h ∈ [0.2, 20] (write ≤ 40) |
| Near clamp / culling expansion | [half_ipd/8, 1.0] / ≤ 20 (+0.1 right margin) |
| View-slot basis rows | length² [0.81, 1.21], |dot| ≤ 0.1 |
| World scale | (0, 10000] |
| Head freshness / filter gap | 150 ms |
| Body gap / jump | 150 ms / 0.75 m |
| Height reference | neutral height only from the first FOCUSED+TRACKED pose (tracking loss never re-seeds it) |
| Remote stick wire | int8 ±127, rate (0, 720] deg/s |
| Arm stretch / solver length limits | ≤ 1.25 / [0.1, 100] |

## 15. Sanity tests to write first

1. Conversion: known tracking offsets give the expected forward/left/up units (section 2 example); rotation `det = +1`.
2. A rolled/pitched head keeps physical pitch/roll after recenter; a yaw-only recenter does not touch gravity.
3. Projection: every runtime frustum edge maps to NDC ±1; off-center terms mirrored between eyes.
4. Heading survives pitch ±90…270° and the captured alternate-Euler orientation (1000×).
5. Eye separation = IPD·scale exactly along the engine left axis; left eye is +left.
6. Culling guard points just outside the right optical edge pass visibility at distances 1, 4, 10000.
7. Filters: disabled = bit-exact bypass; duplicate sequence id returns the cached result; hand jitter energy < 70% at
   72/90/120/144 Hz; head correction ≤ 2 mm / 0.25° per update.
8. IK: finite for the controller at the shoulder; elbow continuous over a 21×21 direction grid; reach restored without state.
9. Packed-angle round trip (`packed·360/2^24`) with a non-zero `delta_angles`.
10. All of the above at several world scales and refresh rates (45–144 Hz); NaN/inf/negative/oversized inputs rejected
    transactionally (`memcmp`-unchanged output).
