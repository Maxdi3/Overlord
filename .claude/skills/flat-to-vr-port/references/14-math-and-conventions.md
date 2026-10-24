# 14 — Math and coordinate conventions

## Spaces

| Space | Convention |
|---|---|
| OpenXR / OpenVR tracking | Right-handed, meters: +X right, +Y up, **−Z forward** |
| H2 / Quake-family engine | Units (inches), axis rows: `axis[0]` forward, `axis[1]` **left**, `axis[2]` up (+Z up world) |
| Overlord portable core | Forward/left/up row-vector basis, translation in meters |

Tracking → engine-local (Overlord `head_pose_bridge::tracking_position`):
```
local = R_origin^T · (p_tracking − p_origin)          // relative to recenter origin
engine = { forward: −local.z, left: −local.x, up: local.y }    // then × world_scale for units
```
Rotation conversion uses the same axis permutation applied to the basis
vectors of the rotation matrix (convert all three columns/rows consistently, and
check `det = +1` afterwards).

World position of a tracked point:
`world = camera_base_origin + Σ_i engine_local[i] * world_scale * base_yaw_axis[i]`
(yaw-only base, so physical pitch/roll come only from the HMD).

## World scale

`units_per_meter = (native standing eye height in units) / (~1.65 m)`, or derive it
from engine documentation. H2: 39.3700787 (1 unit = 1 inch). Make it a saved
setting (`vr_worldScale`). If the world feels too big or small, check this first.

## Eye offsets

From OpenXR: `eye_offset_head_local = R_head^T · (p_eye − p_head)` where `p_head`
is the midpoint of the two eye positions (one `xrLocateViews` sample). For engines with a
single shared camera rotation, translate along the engine's left axis by ±IPD/2 ×
scale. Reject canted eyes (relative rotation > ~0.5°).

## Asymmetric projection from runtime FOV

Inputs: `l = tan(angleLeft)` (negative), `r = tan(angleRight)`, `d = tan(angleDown)`
(negative), `u = tan(angleUp)`.

D3D, left-handed, +Z forward, row vectors, **infinite reverse-Z** (H2 style; `n` = near):
```
[ 2/(r−l)        0              0   0 ]
[ 0              2/(u−d)        0   0 ]
[ −(r+l)/(r−l)   −(u+d)/(u−d)   0   1 ]
[ 0              0              n   0 ]
```
Standard OpenXR/D3D right-handed, column vectors, finite depth [0,1] (−Z forward):
```
x: 2/(r−l), 0, (r+l)/(r−l), 0
y: 0, 2/(u−d), (u+d)/(u−d), 0
z: 0, 0, −f/(f−n), −(f·n)/(f−n)
w: 0, 0, −1, 0
```
Always check the sign of the off-center terms against a capture. A sign error
shows up as eyes that converge/diverge wrongly and is uncomfortable instantly.
Also update any symmetric "tan half FOV" scalars the engine keeps for culling
(use `max(−l, r)` and `max(−d, u)`).

## Culling union frustum

Keep the center origin. Choose a projection whose left plane is the more
outward of (left eye's left plane translated) and whose right plane is the right
eye's right plane, including the eye translation. Overlord derives a conservative
union valid "for every point on or beyond the near plane" (`derive_culling_union`).

## Quaternions and rotation

- Normalize every runtime quaternion. Reject `| |q|² − 1 | > 0.01`.
- Quaternion sign equivalence: `q` and `−q` are the same rotation. Use `|dot|`.
- Midpoint: `slerp(qL, qR, 0.5)`.
- Compose rotations as matrices or quaternions; never add Euler angles.
- Euler extraction for the engine (pitch/yaw/roll): handle the pitch ±90° branch;
  keep yaw continuity across frames (`continuous_view_angles.hpp`).

## Recenter

`origin_yaw = yaw(R_head)` (yaw-only). Keep the physical pitch/roll and the
position origin `p_origin = p_head` (or keep height and recenter only xz for
standing play).

## Two-bone IK

See `08` for `solve_limb`. Rules: clamp reach to `(|U−L|+ε, U+L−ε)`; allow a small
stretch (≤ 1.25×); choose the elbow pole from a body-relative rest bend
swing-transported to the reach direction; orient both segments in one bend plane.

## Filters

One-pole exponential with speed-adaptive half-life:
`half_life = max_half_life * strength / (1 + (v/v_ref)² + (ω/ω_ref)²)`,
`alpha = 1 − 2^(−dt/half_life)`. Head max 15 ms, hands 45 ms, desktop 120 ms (Overlord).
Reset filters on recenter, teleport and reference change.
