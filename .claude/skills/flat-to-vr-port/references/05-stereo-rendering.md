# 05 — Stereo rendering through the engine (strategy A in detail)

Overlord files: `vr/engine_stereo_renderer.cpp` (hooks), `engine_stereo_bridge.cpp`
(view family publication), `engine_stereo_view.cpp` (per-eye slot derivation),
`engine_stereo_binding.cpp` (scene claims), `engine_scene_completion.cpp` (CPU
input waits), `engine_stereo_owner_pass.cpp` (per-eye transactions),
`engine_stereo_eye_resources.cpp` (per-eye RT/history isolation),
`engine_stereo_tessellation.cpp`, `native_post_aa.cpp`, `native_render_session.cpp`.
Docs: `docs/vr-runtime-rendering.md`, `docs/vr-shared-tessellation-and-head-pitch.md`,
`docs/vr-transparent-shadow-stereo.md`, `docs/vr-dynamic-light-stereo.md`.

## 1. The pipeline in one picture

```
Game thread (frontend)                               Render thread (backend)
──────────────────────                               ───────────────────────
CG view setup ── head pose composed (06)
R_RenderScene(center view)   [hooked: observe only]
  allocate/init view slot    [observed]
  view finalizer             [validated]
  ► publish_generator_views: derive L/R eye slots from the
    finalized center slot + published view family; run the
    engine's pure finalizer on them; publish {pair, publication}
  generate draw surfaces     [culling uses a UNION frustum of both eyes]
  FX camera                  [side planes replaced by both-eye envelope]
                                                       backend executes world record:
                                                       target-prepare hook (owner):
                                                         claim publication for this record
                                                         await scene completion (CPU inputs)
                                                         begin owner pass (pair)
                                                         ├ LEFT : arena record + left eye fields → native owner
                                                         │        → end_view: copy final eye image to ring
                                                         ├ (AUX : optional scope/optic view)
                                                         └ RIGHT: private clone record + right eye fields → native owner
                                                                  → end_view: copy to ring
                                                         end pass → pair published
                                                       desktop: native tail continues normally
Present-pre → OpenXR submits pair (04)
```

## 2. Publishing views early, reading scene contents late

- Publish the finalized eye identity **before** calling the draw-surface generator,
  so the concurrently running backend can claim the same scene.
- Do **not** copy unfinished surface/FX lists at publication time.
- In the backend, before cloning or executing per eye, run the engine's own
  job-pumping waits (initial → surfaces → effects) and check frontend identity
  before and after (`engine_scene_completion::await`). These are CPU completion
  flags, not GPU fences.
- Reject when frontend identity, generation or the source camera changed between
  publication and consumption (stale publication over a different scene payload).

## 3. Deriving an eye view slot (`engine_stereo_view.cpp: derive_eye`)

```cpp
// natural = finalized center-eye slot bytes (copied), config = runtime family
out = natural;
float dir = (view_eye == 0) ? +1.f : -1.f;            // engine axis[1] points LEFT
float off = dir * half_eye_offset_units;               // = IPD/2 * world_scale
origin += axis_left * off;                              // translate along head-left
rebuild_view_rotation(out);                             // view matrix from axis
write_projection(out, tanL, tanR, tanD, tanU, near);    // asymmetric off-axis
// then call the ENGINE's finalizer (H2 0x14077F550) to build VP and inverse VP
```
H2 projection (row vectors, +Z forward, infinite reverse-Z):
```
m[0]=2/(r-l)  m[5]=2/(u-d)  m[8]=-(r+l)/(r-l)  m[9]=-(u+d)/(u-d)  m[11]=1  m[14]=near
(l,r,d,u are tangents: tan(angleLeft) < 0 < tan(angleRight), etc.)
```
Also write the engine's legacy symmetric scalars (tan-half-x/y = max of each
side) because occlusion/near-frustum code (Umbra) reconstructs a symmetric
frustum from them.

Validation (`validate_finalized`): matrices are finite and non-empty, and the
inverse matrices differ between eyes. Identical VP matrices are legal for symmetric FOVs
because the eye origin is folded only into the inverse. Reject slots that still
hold copied center-eye matrices.

Canted displays (Pimax, some wide-FOV headsets) report per-eye rotations.
Overlord rejects them in native mode. Supporting them needs a per-eye rotation
in the engine camera representation.

## 4. Culling and LOD must cover both eyes

- **Frontend culling**: use one conservative frustum whose volume contains both
  translated eye frusta (`derive_culling_union`). Keep the center origin and
  basis; widen only the projection.
- **FX/particle culling** has its own camera. Replace its side planes with the envelope
  (`apply_fx_culling_union`).
- **GPU-side culling/tessellation**: terrain tessellation was computed once from
  the left eye and shared, which left gaps at the right eye's outer edge. Fix: overwrite
  the tessellation view constants with the union view before Unmap
  (`engine_stereo_tessellation.cpp`).
- **Shadows/lights** are view-independent but may be culled by view. Check
  shadow casters near the eye-only frustum edges.
- **LOD** origin: use the head center for both eyes so models don't pop between eyes
  (`camera.cpp` writes the composed origin to the LOD origin global `0x14EEE0890`).

## 5. Per-eye execution (owner pass)

- Eye 0 executes the engine's **own arena record**, with the eye-local fields
  copied in (`copy_eye_local_record_fields`). Eye 1 executes a **private clone**
  of the complete record (`clone_scene_records`).
- Each eye has a transaction that saves and restores native state and tracks
  pair, eye, frontend, record, owner thread and device generation.
- The **dynamic uploads** (skinned vertices, particles, dynamic index buffers)
  produced during the left eye must remain valid for the right eye. Track
  upload phases and boundaries (`engine_stereo_dynamic_arena`, `dynamic_upload`).
- **Camera-relative model placement**: engines that render models relative to the
  camera (rebase origin) need that origin moved per eye
  (`rebase_camera_model_list_origins`, `synchronize_backend_model_state`). An
  unmoved placement cache gives "everything is at the center eye" artifacts.
- **Viewmodel depth hack**: the engine replaces the projection with a symmetric
  viewmodel FOV and a tiny near plane. For VR, restore the eye's full asymmetric
  projection and change only the near term (`restore_backend_depth_hack_projection`).
  Otherwise the gun appears at a different depth and size in each eye.

## 6. Per-eye resource isolation (the subtle part)

Anything temporal or screen-space with history must be **per eye**:
- **SSR / TAA / Filmic SMAA history**: natural, left and right are separate
  resource sets. Rebind on eye entry before the *first read*, restore on
  completion or cancel. Commit history only for a complete pair. Reset on
  discontinuity (recenter, snap turn, teleport, mode change).
- **Previous view-projection constants** (motion vectors/reprojection) must be the
  same eye's previous VP.
- **Frame-counter-indexed rings** (AA jitter index) should advance with a
  contiguous VR phase, not the engine frame counter
  (`native_post_aa_frame_thunk.hpp`).
- Never copy history from the other eye or from the natural view every frame
  to hide inconsistencies.
- SMAA edge detection reused the depth buffer, so preserve and restore depth
  before any later depth-tested composition.

## 7. Post-processing and screen effects

- Run native post-AA per eye **before** copying to the ring (`native_post_aa.cpp`).
- Screen-space effects that assume screen center = view center (lens flares, sun
  shafts, vignette, damage overlays, blur) need per-eye projection or must be
  disabled in VR (`native_flare.cpp`, `fullscreen_blur_source.cpp`,
  `damage_screen_component.cpp`, `vr_disableBlur`).
- DOF and motion blur: default off in VR.

## 8. Desktop mirror

Sample a completed eye texture (default right). Crop symmetrically around that eye's
optical forward to the window aspect, with a configurable FOV (`desktop_mirror.cpp`).
Use the projection metadata retained *with that image*, never newer runtime values.

## 9. Bootstrap and evidence gate

Overlord does not submit until a **census** proves that every execution context and
command list during the first stereo frame is known, and that a *distinct-eye GPU
readback* differs between eyes (`engine_stereo_gpu_census.cpp`,
`native_stereo_source`). A minimal port can replace this with:
1. Render L/R into separate textures.
2. Read back a small region and assert that the images differ by the expected parallax.
3. Only then arm submission.

## 10. Performance notes

- Two full world passes roughly double GPU time. Provide resolution scale
  (render-target size from the runtime's recommendation × scale) and native SSAA settings.
- Keep CPU work per eye minimal: clone records with memcpy and do no per-frame allocation.
- Measure owner/Present/WaitFrame/Submit phases separately (`region_capture`,
  `engine_stereo_gpu_timing`) and log GPU timestamps per eye.
