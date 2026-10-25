# 05 — Stereo rendering through the engine (strategy A in detail)

Scope: how to make the game's **own renderer** produce a left and a right eye image per frame
(strategy A in `00`), safely and without per-frame heroics. Everything transferable is stated as a
contract or technique first; "Overlord:" lines give the worked implementation; "H2-only" lines
give addresses/offsets as examples only (never reuse them on another build or game).

Overlord files (all under `src/client/component/vr/` unless stated):

| Concern | Files |
|---|---|
| Hook installation, frontend/backend orchestration | `engine_stereo_renderer.cpp/.hpp` |
| Published view family (IPD, scale, tangents) | `engine_stereo_bridge.cpp` |
| Eye-slot / record math (pure CPU, byte layouts) | `engine_stereo_view.cpp/.hpp` |
| Frontend→backend publication/claim table | `engine_stereo_binding.cpp` |
| CPU scene-input join | `engine_scene_completion.cpp` |
| Per-pair owner transaction | `engine_stereo_owner_pass.cpp/.hpp` |
| Per-eye RT/history isolation | `engine_stereo_eye_resources.cpp`, `engine_stereo_output_merger.cpp`, `engine_stereo_resource_ops.cpp`, `engine_stereo_pointer_cache.hpp` |
| Per-eye patches | `engine_stereo_light_views.hpp`, `engine_stereo_dynamic_arena.cpp`, `engine_stereo_dynamic_upload.hpp`, `engine_stereo_tessellation.cpp` |
| Admission / evidence | `engine_stereo_execution.cpp`, `engine_stereo_gpu_census.cpp` (diagnostic only), `renderer_evidence.cpp` (in `diagnostics/`) |
| Post chain | `native_display_contract.hpp`, `native_post_aa*.{cpp,hpp}`, `native_fullscreen_blur.cpp`, `native_flare*.{cpp,hpp}`, `native_display_backup.hpp` |
| Ring / handoff | `native_render_session.cpp`, `eye_composition.hpp` |
| Optic third view | `auxiliary_scene.{cpp,hpp}` |
| Native resolution | `engine_scene_resolution.cpp`, `engine_scene_extent.hpp` |
| Capacity patches (`src/client/component/`) | `scene_surface_storage*.{cpp,hpp}`, `scene_static_surface_*.{cpp,hpp}`, `scene_model_lighting*.{cpp,hpp}` |

Docs: `docs/vr-runtime-rendering.md`, `vr-shared-tessellation-and-head-pitch.md`,
`vr-transparent-shadow-stereo.md`, `vr-dynamic-light-stereo.md`, `vr-lens-flare-rendering.md`,
`vr-player-death.md`, `vr-optic-rendering-feasibility.md`, `vr-thermal-world-isolation-research.md`,
`vr-render-performance-review.md`, `scene-surface-storage.md`, `scene-static-surfaces.md`,
`scene-model-lighting.md`.

> Reality check before you commit to this strategy: Overlord's stereo layer is the largest and
> most evidence-heavy part of the mod. Roughly 60% of it is scaffolding that proves identity,
> ordering and isolation, not math. The math is the easy 10%. Budget accordingly (see §17 for a
> minimal path).

---

## 1. The pipeline in one picture

```
Game thread (frontend)                                  Render thread (backend)
──────────────────────                                  ───────────────────────
CG view setup ── head pose composed (06)                (publish_view_family at Present-post:
R_RenderScene(center view)  [hooked: OBSERVE ONLY]       IPD, world scale, per-eye tangents)
  reserve scene record      [observed]
  allocate view slot        [observed]
  initialize slot (engine fills the center slot)
  ► derive L/R eye slots from the center slot, run the engine's
    PURE finalizer on each, validate_finalized
  ► publish_generator_views: {pair_id, publication, eye slots, source camera}
    (BEFORE the draw-surface generator, payload NOT copied)
  [apply union culling widening to the center slot — only for an admitted pair]
  generate draw surfaces    [culling uses the UNION frustum]
  FX camera build           [side planes replaced by both-eye envelope]
                                                        backend executes the world record:
                                                        "target prepare" (the OWNER) hook:
                                                          claim publication  (acquire_current)
                                                          join CPU scene completion (3 waits)
                                                          begin pair: clone records, history, hooks
                                                          ├ LEFT : arena record + eye-local fields
                                                          │   native owner → end_view:
                                                          │   resolve final target → engine PostFX
                                                          │   redirected into VR target → post-AA
                                                          │   → blur → copy to ring (+composition)
                                                          ├ AUX (optional scope view, index 2)
                                                          └ RIGHT: PRIVATE CLONE record + eye fields
                                                                  native owner → end_view (same)
                                                          end pair → pair published to the ring
                                                        desktop: the native tail continues normally
Present-pre → runtime submits the pair (04)
```

Key structural facts:

* The **frontend runs once** (center view). It only *derives and publishes* eye identities.
* The **backend owner runs once per eye** (and once for an optional third view), each time on a
  complete, self-consistent record. Repeating the *frontend* scene entry is wrong (§15).
* The desktop path is left intact: the engine's natural tail still runs on the natural record, so
  the mod never needs to synthesise a desktop image.
* Everything fails closed to flat rendering: if any prerequisite is missing, the natural owner
  just runs on the natural record (`engine_stereo_renderer.cpp` ≈2122-2141 claim gate:
  record valid, `record_type == 4` world, bootstrap complete, claim acquired, scene resolution
  ready, owner pass ready).

### Where per-eye work sits in a generic engine

| Engine stage | Per-eye action | Overlord hook style |
|---|---|---|
| View setup (frontend) | derive eye slots from the center view, validate | wrap slot initializer; call the engine's finalizer twice |
| Scene entry (frontend) | observe only | wrapper that never mutates the natural scene |
| Draw-surface generation (frontend) | publish views just before; widen culling | call-site patch on the generator call |
| FX/particle camera (frontend) | widen culling planes | call-site patch on the FX camera builder |
| Record execute (backend) | run N times with per-eye record + isolated state | hook the *owner* of the world record |
| Final post chain (backend) | per-eye display transform, AA, blur | redirect the PostFX destination getter |
| Shared GPU constant buffers | patch the main view between Map and Unmap | Map/Unmap hook |

---

## 2. Identity and ordering contracts (the part that makes it safe)

### 2.1 Publish early, read late

* Publish the **finalized eye identity before** calling the draw-surface generator, so the
  concurrently running backend can claim the same scene before any consumer is enqueued.
* Do **not** copy unfinished surface/FX lists at publication time (they are still being produced
  by async jobs).
* In the backend, before cloning or executing any eye, run the engine's **own** job-pumping waits
  (initial → surfaces → effects), check frontend identity before and after each wait, and require
  all four completion flags non-zero (`engine_scene_completion::await`, a template over
  identity/wait/read_flags so tests share the control flow). Never invent a timeout, poll loop or
  worker; call the engine's predicates with the engine's wait function. No eye/GPU scope may be
  held across the wait (workers might need it). These are CPU completion flags, **not GPU fences**.
* Publication preconditions in the frontend (`publish_generator_views`): exactly one allocator and
  one generator call this scene; slot index equals allocator index; `selected == slot`; slot bytes
  equal the bytes captured at the initializer (`memcmp`); record arena pointer non-null.
* User-reported effect of the early-publish/late-read split: "very large" performance
  improvement in the heaviest level (no measured FPS in the docs).

### 2.2 Publication/claim table

`engine_stereo_binding.cpp`: fixed 64-entry cache-line-aligned table, entry states
`free → publishing → published → claimed → free`, all transitions CAS. Key =
`(record address, frontend pointer)`. `publish` validates both eye slots share `pair_id` and
`publication`, `output_eye` is 0/1 and `view_eye` differs.

**Pitfall (the big one): engines reuse record addresses. A record address is not a frame ID.**

`acquire_current(natural_record, frontend)`:

1. Read `ceiling = latest_publication_sequence` *before* reading the camera.
2. Drain all publications visible at entry for that address, keep the **newest**, release the rest.
3. Accept only if the published 12-float source camera (origin, forward, left, up) equals the
   current record's camera **and** the camera re-read after the claim (it did not change
   mid-claim).
4. Otherwise it is a miss. Never substitute an older publication or camera.

Related rules:

* **No implicit "next eye" API.** Natural scene-call order is not a stereo contract and can cross
  world snapshots. Every consumer presents its full identity.
* The pair identity tuple is `(pair_id, publication, output_eye, device generation, owner thread,
  context)` plus the source camera. A queued consumer (HUD, viewmodel placement, composition) must
  key by this whole tuple: the backend may render frame N after the frontend published N+1.
  Bounded rings of such data answer "not found" on eviction, **never "latest"**.
* A predicted view family stays immutable until both eyes consumed it
  (`published_eye_mask == 0x3`) so unrelated left/right images are never paired.
* After bootstrap, release the claim even when GPU admission is closed. Otherwise obsolete CPU
  publications accumulate and outlive the engine's owner.
* A retained token at the next record boundary is an incomplete prior transaction: close it with
  an explicit fault (`overlap`/`record_mismatch`). Never carry an eye pair across records or infer
  completion from elapsed time.
* Reset/test hooks refuse while a producer or claim is active.

### 2.3 Transaction object and failure policy

`engine_stereo_owner_pass`: `begin(claim, natural_record, graphics)` → `begin_view(eye)` →
engine renders → `end_view(eye)` (display transform + copy) → `end()`.

Admission order inside `begin`:

1. validate claim (record type, finalized views);
2. `engine_scene_completion::await`;
3. `clone_scene_records` (private right-eye record; a single CPU snapshot for both payloads);
4. in production ask the native ring `accepts_pair(pair_id)` **before** taking `production_active`
   (a valid but older pose family — OpenVR admits one immutable pose family at a time — is
   declined and the engine's original path runs without occupying a ring slot);
5. observe owner camera, save center origin, snapshot the shared culling view (for tessellation),
   prepare temporal history, install eye-resource hooks (idempotent re-check), plan auxiliary view.

Failure policy (do not just "skip frames"): an unpublished incomplete pair is **discarded** back
to the ring; a published pair (end_view(1) already published) is **quarantined** until external
Submit drains it; if discard cannot be proven safe, quarantine. Never reuse unproven slots.
Temporal history commits **only** for a complete, published, non-failed pair; reset is a request
flag applied at the next `begin` (so a console command never mutates an in-flight eye).
Control-plane requests (history reset, A/B toggles) are latched only at the next owner-pair
boundary. A `failure` enum (prerequisite, claim, record, thread, target_contract, source_contract,
resource_creation, copy, query, readback, content, timeout, device, model_state, dynamic_mesh,
eye_resource, display_transform) plus per-stage timings are recorded.

Thread discipline: each transaction is a thread-local value; observers read thread-local state, so
cross-thread draws are counted as `foreign_*` and ignored, never attributed. Reentrancy guard:
thread-local `observation_active` and a saved/restored `dynamic_upload_owner_scope` so nested or
foreign owners do not inherit the outer stereo lifecycle.

### 2.4 Eye order

Strict: eye 0, then (optional) aux (view index 2), then eye 1. `begin_eye` enforces
`eye == (completed_mask == 0 ? 0 : 1)`; `begin_auxiliary` requires `completed_mask == 1`;
`end_pair` requires mask 3 and a final all-clear scan. Output order (first/second replay) is
distinct from `view_eye` (the camera eye); `vr_engineSwapEyes` swaps the mapping.

---

## 3. Deriving an eye view slot (`engine_stereo_view.cpp: derive_eye`)

```cpp
// natural = finalized center slot bytes (copied), config = runtime view family
out = natural;
float dir = (view_eye == 0) ? +1.f : -1.f;       // H2 axis[1] points LEFT
float off = dir * half_eye_offset_units;         // = ipd_m * 0.5 * world_scale
origin += axis_left * off;                       // translate along head-left
rebuild_view_rotation(out);                      // rotation from axis rows
write_projection(out, tanL, tanR, tanD, tanU, near);
// then call the ENGINE's finalizer (H2 0x14077F550) to build VP and inverse VP.
// The engine applies temporal jitter exactly once, inside this call.
```

H2 projection (row vectors, +Z forward, **infinite reverse-Z**):

```
m[0]=2/(r-l)  m[5]=2/(u-d)  m[8]=-(r+l)/(r-l)  m[9]=-(u+d)/(u-d)  m[11]=1  m[14]=near
(l,r,d,u are tangents: tan(angleLeft) < 0 < tan(angleRight), tan(angleDown) < 0 < tan(angleUp))
```

Also write the engine's **legacy symmetric scalars** (`tanHalfX = max(-tanL, tanR)`,
`tanHalfY = max(-tanD, tanU)`, in both the primary and secondary copies) because occlusion
(Umbra-like) and near-frustum code reconstruct a symmetric frustum from them. Using the longer
side clips nothing the eye can see.

Round trip (`read_projection`): recover tangents from the exact rasterised slot as
`tanL = (-1-m8)/m0`, `tanR = (1-m8)/m0`, `tanD = (-1-m9)/m5`, `tanU = (1-m9)/m5`; insist on the
exact axis-aligned +Z perspective form (`m11==1`, `m15==0`, other off-axis terms zero). Use this to
describe the optics of the image that was actually rendered (desktop mirror, recording frame),
never the latest runtime FOV.

### 3.1 View family publication and thresholds (`engine_stereo_bridge.cpp`)

| Quantity | Rule |
|---|---|
| Eye positions → IPD | distance between published eye positions; accepted in **[0.02, 0.12] m**, else the family is invalidated. Defaults: IPD 0.064 m; world scale 39.3700787 units/m (H2 uses inches); scale accepted in (0, 10000] |
| Half eye offset (units) | `ipd_m * 0.5 * world_scale`; must be finite, in (0, 1000] |
| Tangents | finite, `tanL<0<tanR`, `tanD<0<tanU`; symmetric half-tangents in [0.1, 10] at publication |
| Per-eye tangents inside `coherent_configs` | width and height (`tanR-tanL`, `tanU-tanD`) in **[0.2, 20]** |
| Pair coherence | equal `pair_id`, `frame_id`, `publication`; output eyes exactly {0,1}; view eyes distinct |
| Axis validity (`valid_axis`) | each basis row length² in **[0.81, 1.21]**, pairwise \|dot\| ≤ **0.1** |
| `write_projection` | width/height up to 40, `reverse_z_scale > 0` |
| `validate_derived` | slot matrices finite, `proj[0]>0`, `proj[5]>0`, `proj[11]==1`, `proj[14]>0` |
| `validate_finalized` | VP and inverse-VP "energy" (sum of abs) ≥ 0.1 each; **inverse VP bytes differ between eyes** |

**Why the inverse must differ, not the VP:** H2's finalizer forms the camera-relative VP from
rotation × projection only and folds the eye origin only into the inverse. Symmetric-FOV runtimes
(SteamVR/VDXR) legitimately give identical VPs for both eyes. A validator that demanded different
VPs rejects a valid pair (issue #34); one that demanded nothing accepts unchanged center-eye
copies. Test both the symmetric and asymmetric case.

**Canted displays** (wide-FOV headsets with per-eye rotation): the engine camera has one
orientation plus per-eye origin and projection, so Overlord rejects them in the runtime adapter
(`openxr_runtime.cpp`, reason `stereo_eye_orientation_mismatch`), not in the view math. Supporting
them needs a per-eye rotation in the engine camera representation (the slot math above would then
also need a rotated axis per eye).

### 3.2 Near plane

`near = max(half_ipd/8, min(native_near, 1.0 unit))`; non-finite or ≤0 native near is returned
unchanged so derivation fails visibly (the clamp must not hide a bad value).

* Why a lower bound: scripted cameras can request 0.01-unit near planes (scope entry). With a
  finite IPD that needs a ~180° shared culling frustum (`half_ipd/near` blows up) and used to
  reject stereo publication entirely. The bound keeps `half_ipd/near ≤ 8`.
* Why an upper bound of 1 unit: keep shoulder-held geometry visible without moving an already
  closer native near plane outward. The same value feeds the render slot (+0x148), the reverse-Z
  term (+0x78) and the culling union.
* A fullscreen scripted **scope** scene (`screen_scope_aspect > 0`) keeps the native near
  (e.g. 100) and an envelope around the native optical rays; collecting unseen wide-angle
  geometry can overflow native model lists.

---

## 4. Everything that differs per eye in the scene record

Cloning the record and swapping the view slot is **not enough**. Audit every datum derived from
the camera. Overlord copies/derives these eye-local fields (`copy_eye_local_record_fields`,
`clone_scene_records`): view slot (rotation, projection, VP, inverse VP, origin, axis), the
inverse scene projection constant, the relative eye offset, history VP, previous-VP constant and
previous-eye-position constant. Eye 0 executes the engine's own arena record with these fields
copied in; eye 1 executes a private clone made from one CPU snapshot (the clone checks
`origin == natural camera`, otherwise rejects).

| Field | Rule |
|---|---|
| **Inverse scene projection constant** (post-opaque fog / SSAO / SSR ray reconstruction: `(clip.xy*v.xz + v.yw) * linearDepth`) | per eye `{1/sx, -m8/sx, 1/sy, -m9/sy}` from that eye's *final* matrix (asymmetric centre and jitter included). Replacing only the view slot leaves a mono / FOV-mismatched ray field. **Method:** grep the engine's shader constant table for every constant derived from the projection (view/inverse, VP, near-plane origin/dx/dy, eye offset, viewport dims, inverse scene projection, depth-from-clip, clip-space lookup, **all previous-frame constants**) |
| **Relative eye offset** for camera-relative models | `natural_rebase + (eye_primary - natural_primary)`. Copying the absolute eye origin makes rigid/skinned models subtract the world camera position twice. Sanity bound: per-eye displacement² in (0, 1000²] |
| **Temporal history** (history VP, previous-VP constant, previous-eye-position delta) | §8 |
| **Camera-model draw-list origins** | §6.4 |
| **Dynamic light descriptors** | §6.1 (lives outside the record) |
| **Backend placement cache, depth-hack projection** | §6.4, §6.5 |
| **Current vs history fields** | Current placement/origin (`+0x100`, current placement) are per-eye; HISTORY fields (`+0x19C`, `+0x2DC0`) must not be used as a current source. The clone step *repurposes* `+0x2DC0`, so a later observer reading it as "previous eye" is wrong (`ssr_history_probe`: `history_origin_reliable_for_ssr=false`). Observers must know which fields the preceding mutator reused |

An incoherent clone is rejected as a whole (`pair_id == 0`): no partial publication (singular
projection, non-finite values, wrong source camera).

---

## 5. Culling, LOD, and shared visibility data must cover both eyes

### 5.1 Frontend culling union (`derive_culling_union`)

Keep the center origin and basis; widen only the projection.

```
expansion = half_ipd / near                       // rejected if > 20
tan_left  = min(tanL_a, tanL_b) - expansion
tan_right = max(tanR_a, tanR_b) + expansion + 0.1 // 0.1 tangent right-side margin
tan_down  = min(tanD_a, tanD_b)
tan_up    = max(tanU_a, tanU_b)
```

Derivation: at centre-forward depth z the left eye's leftmost point is `-h + z*tanL` and the right
eye's rightmost is `+h + z*tanR`; `h/z` is largest at the reverse-Z near plane, so expanding the
tangent by `h/near` encloses both translated frusta for the whole visible depth range, including
near geometry. The union slot also goes through the engine's finalizer. Scope scenes use the
narrow envelope described in §3.2.

* **Apply only to an admitted production pair** (`apply_production_culling_union`; admission
  states unarmed/unavailable/accepted). Unarmed or proof frames keep the natural view; widening
  them contaminates the desktop path without fixing HMD visibility. A failed FX-camera contract
  also vetoes admission.
* Failed experiment: a virtual rearward apex (translate the apex back by
  `d = max(h/-L, h/R)`, forward distance `z+d`, depth numerator `near+d`) is mathematically tight
  but gave no practical improvement and was withdrawn (`docs/vr-equipment-visibility-capture.md`).

### 5.2 FX / particle camera planes (`apply_fx_culling_union`)

Engines often keep a **separate camera copy** for particles (H2: `FxSystem.camera` with origin,
plane array, axis, count). Patch it at the one verified builder call, before FX workers read it
("never patch the FX camera asynchronously").

* Require `valid == 1` and plane count ∈ {5, 6}.
* Slopes, in the engine's plane order (right, left, bottom, top):
  `{max(tanR)+0.1, -min(tanL), -min(tanD), max(tanU)}`.
* Normal `n = slope*forward ± lateral_axis`, normalised (right/left use the left axis, bottom/top
  the up axis; H2: screen right = -axis[1], screen up = +axis[2]).
* Plane distance `D = n·center − |n·left_axis| · half_ipd`, then `nextafter(D, -inf)`. The engine
  keeps points with `n·p ≥ D`; supporting both eye origins needs the smaller D. This also covers
  near-eye particles.
* H2: the planes are written at camera `+0x20` (the header comment in `engine_stereo_view.hpp`
  says `+0x10`; the code is authoritative).

### 5.3 GPU-side shared culling / tessellation constants

The engine fills one shared constant buffer once per frame from the frontend view (H2: 35 views
including shadows, 0x1600 bytes). If that view is the left eye, terrain tessellation/culling leaves
gaps at the right eye's outer edge. A CPU union frustum does **not** cover GPU-internal culling.
Quick diagnostic: toggling `r_tessellation` removed the gaps.

Fix (`engine_stereo_tessellation.cpp`): hook the fill call (snapshot the union/rendered pair into a
thread-local), the `Map` call (record the mapped pointer for slot 0 of the same context) and the
`Unmap` tail jump. At `Unmap`, `replace_main_view` validates that the native upload is
byte-identical to the expected rendered view (memcmp; finite source floats), then overwrites only
the main-view fields with the union view and calls the original `Unmap` exactly once.
H2-only ranges: VP 64 B at `0x20 ← 0x80`, projection 64 B at `0x8E0 ← 0x40`, origin 12 B at
`0x11A0 ← 0x100`; shadow entries, tessellation factors and `origin.w` untouched. Only when
frontend index/pointer satisfy `natural_record - arena == index * record_size` and owner thread
match. Status: `vr_tessellation_status` (prepared/applied/rejected; applied should rise, rejected
stay 0). General lesson: any shared constant buffer filled once per frame must be intercepted
between Map and Unmap, not patched afterwards.

### 5.4 LOD / streaming origin

Engines keep a "view origin for LOD/cull/streaming" separate from the render camera. Write the
composed head-centre origin there so both eyes pick the same LODs and models do not pop between
eyes (`camera.cpp` writes the composed origin into the H2 global `0x14EEE0890` when the head pose
was applied or the game is in free-move; condition: `cg_paused==2 || camera_was_applied`).

### 5.5 Other visibility notes

* Shadow casters near eye-only frustum edges: check them.
* **Lights are not view-independent in H2** (§6.1): shadows/lights computed camera-relative need
  per-eye rebasing.
* Model-index-keyed native tables (cull-radius floats, bit arrays) touched when models are
  submitted twice must be sized for the pool you actually use (`component-sub-game-2` note).

---

## 6. Per-eye patches during the owner pass

Each patch below is a validate-all → write → restore-at-eye-exit scope. Every one counts
*rewrites, foreign bypasses and contract mismatches*; the "foreign" counters prove you did not
touch unrelated work.

### 6.1 Dynamic light descriptors (not cloned with the record)

The array of camera-facing dynamic lights lives in frontend data outside the per-view record
(H2: up to 8 descriptors × 0x130 B; technique `+0x100` in `0x25..0x30`, view-record pointer
`+0x108`, origin `+0x110`). Instanced XModel passes read the origin from the descriptor;
ordinary passes read the backend eye origin. Left uncorrected the two disagree by ± half-IPD,
in opposite directions per eye: a detached translucent lighting layer (helicopter spotlight,
muzzle flash) that "moves outward in opposite directions in the two eyes".

`engine_stereo_light_views::scope`: for the first N descriptors, **validate all entries first**
(source record pointer == natural record, technique in the lighting family, origin bit-identical
to the natural origin, count ≤ 8, finite floats), snapshot, then write eye record pointer and eye
origin; all-or-nothing (a late mismatch leaves earlier entries untouched), refuses overwrite while
active, restores byte-exactly (twice-safe) at eye exit and on failed admission. Foreign records,
technique 0, shadow-map descriptors and entries beyond the count stay native.
General rule: **any frontend list that stores a viewer origin must be patched and restored per
eye**.

### 6.2 Dynamic geometry arenas (index-origin rewind)

Engines write per-frame dynamic geometry (particles, glass, decals/marks, sparks, "code" meshes)
into arenas consumed sequentially by cursor. Running the world twice advances the cursor, so eye 2
reads the wrong region (symptom: a **constant nonzero `StartIndex` delta** between eyes, e.g.
`0x00A46000`, or the right eye losing smoke/dust shadows because the transparent-shadow atlas
was 255 for the right eye).

H2: eight dynamic-mesh descriptors (stride `0x58`: 5 code_trans, glass, mark, spark), each with
index-buffer pointer, index count, **index base** and vertex pointer/payload bytes; the backend
advances the base as it consumes.

Technique (`engine_stereo_dynamic_arena.cpp`):

* Left eye: at each owned-geometry view-copy boundary `capture_backend` snapshots the index-base
  origins (up to **128 boundaries** per eye including light/shadow views).
* Right eye: `replace_backend` writes the matching left snapshot back (verify by re-reading),
  remembers the displaced natural post-left state once, and restores it at right-eye exit. Left
  and right boundary counts must be equal. `replace_backend` first proves the backend's own data
  block pointer equals the snapshot's data identity so an ordinal match cannot mutate a foreign
  arena. Failure paths roll back.
* `classify_view_copy` separates **camera view** authority from **owned geometry** authority:
  shadow/light subviews keep their own camera but consume the same dynamic geometry, so "owns
  geometry" is decided by data-block identity at the geometry boundary, not camera identity.
  Bug history: `note_backend_view_copy` returned early for sources other than the eye record, which
  skipped index restoration for owned shadow/light geometry (two authorities: eye camera/model
  state vs dynamic-index capture/replay).
* A fullscreen PostFX view copy must not extend the matched dynamic-index sequence after `end_eye`
  restored the natural cursor.

### 6.3 "Advance to next upload" runs once per pair

The engine's call that opens the NEXT dynamic upload with `D3D11_MAP_WRITE_DISCARD` (H2: owner call
site at `+0x7A8032`, nine descriptors) must run **once per engine frame, after both eyes consumed
the current allocation** — not once per replay. State machine
`awaiting_left → deferred → advancing → advanced` (`engine_stereo_dynamic_upload.hpp`):

* left call → **defer** (current allocation stays mapped/valid for eye 2);
* right call → **advance** (exactly one);
* an auxiliary (scope) view consumes the same unmapped allocation as a second deferral that never
  advances;
* on any early exit after left, `take_deferred` performs the original on the owner thread; a Map-all
  that was interrupted is **never retried** (partial mapping may have occurred);
* wrong thread, foreign arena, right-before-left, duplicate boundary → reject; a duplicate must not
  Map twice. `finish_dynamic_upload` demands exactly 1 left + 1 right call and final phase
  `advanced`.

Generic rule: any ring/arena advance in the engine happens per *engine frame*, not per replayed
eye. Test with a WARP device: for 128 families both eyes' `CopyResource` snapshots must equal the
*current* upload with exactly one `WRITE_DISCARD` advance per family.

### 6.4 Camera-relative model placement

* **Backend placement cache** (`synchronize_backend_model_state`): the engine caches the last model
  placement under a key that omits eye identity (H2: XModel placement cache qword at backend
  `+0x3270`); unfixed, eye 2 reuses eye 1's placement ("everything at the centre eye"). Fix:
  verify backend primary/rebase source pointers equal the expected record and `record+rebase`, and
  that backend primary origin and relative eye offset memcmp-equal the eye record; then **null only
  that single borrowed-identity qword** so the engine rebuilds the matrix and bumps its own
  version counters. Do not synthesise version fields. Registered per eye bitmask
  (`register_backend_state`), max 8 backend states.
* **Camera-model draw-list origins** (`rebase_camera_model_list_origins`): descriptor lists whose
  origin was filled from the centre camera: for a fixed subset (H2: 27 of 68 descriptors, indices
  0..25 and 29; stride `0x130`, active type at `+0x100`, origin at `+0x110`), if active and origin
  == natural centre, rewrite to the eye origin; if already eye → `already_eye`; anything else →
  `foreign`, untouched; return true only if the four buckets sum to the subset size. After the
  eye, `census_camera_model_list_origins` must report **zero "other"** origins or the eye is
  rejected (safe: native state already ran; it only keeps an incoherent eye off the HMD).
* **Mod-added models** must see the same placement in both eyes: sample one clock per native
  packing job and share it between eyes ("individual parts must not sample their own clock"),
  keep queued submissions immutable until the render drain (no slot recycling), pad per-model cull
  radius thread-locally instead of patching shared asset data. Late-bound placement at the native
  packing boundary is detailed in `08`.

### 6.5 Viewmodel depth hack

Engines render the viewmodel with a symmetric viewmodel FOV and a tiny near plane. For VR restore
the eye's **full asymmetric projection** and change only the near term; otherwise the gun has a
different depth/size per eye (`restore_backend_depth_hack_projection`, via a detour on the native
projection builder, post-return, gated by exact owner/eye/backend identity).

* Applies only if the depth-hack flag bit 0 is set.
* Preconditions: projection-cache version == input version; all 16+16 floats finite;
  `depth_hack_near > 0`; `proj[0] > 0`, `proj[5] > 0`, `|proj[11]-1| ≤ 0.001`.
* Action: copy the eye's 16-float projection into the cache and overwrite only `m[3][2]`
  (cache+0x38) with the depth-hack near.
* World-entity assemblies that should not use the depth hack clear **only bit 0** of the scene-entry
  flags at the common insertion boundary (H2 flat flags `0x1002441` → `0x1002440`).
* Fullscreen PostFX / view copies with `native_display_record` set forward straight to the
  original (no depth-hack fix-up, no target-select tracking).

### 6.6 Flares, damage overlays and other screen-space effects

Screen-space effects that assume "screen centre = view centre" must be **converted or re-run per
eye**, not simply disabled.

* **Lens flares are converted** (`native_flare*.cpp`): flare quads are pre-projected NDC (z=1)
  from the centre camera. At emit read the engine's already-written 4 vertices (verify allocator
  stride 48, count +4, bounds) and lift them onto the light's camera-depth plane:
  `world = origin + depth*(fwd - x*tan_x*left + y*tan_y*up)`, `depth = (light-origin)·fwd`;
  invalid data collapses the quad, never NaN. At draw, **per eye**: snapshot that eye's record,
  bake the absolute eye origin into the engine's eye-relative matrices (WORLD0 identity, eye-offset
  constant zero), RAII-swap the live source-state projection block and clear the depth-hack bit,
  **bump the dependent matrix-cache version stamps on entry and exit** (never WORLD0's own stamp,
  never rewind a GPU/cache-valid stamp), restore the four input matrices. A dvar disables only the
  VR draws; native emission remains for the desktop. All reads `__try`-guarded.
  Keep finite-distance flares distinct from directional sun flares and authored ghosts; no fixed
  pixel offset.
  Rule: when you temporarily swap an engine matrix input, find the version/stamp scheme of the
  derived caches (WV, WVP, code constants), bump dependents on entry and exit, restore only inputs.
* **Fullscreen blur is re-run per eye** (`native_fullscreen_blur.cpp`, `fullscreen_blur_source.cpp`):
  the native death/injury blur runs *after* the display transform VR copies from, so it would be
  missed. Block the unwanted *sources* (menu-blur radius zeroed at `CL_GetMenuBlurRadius`;
  injury blur dropped only when the VM program counter lies in the script's `blurview` function
  range with 2 args and local player, so story/death requests survive), then call the verified
  native Gaussian/compose routine per eye after the display transform and before VR HUD
  composition, using a **GPU-only backup of the raw HDR** (`source_backup::compose`: copy raw →
  saved, run `display → raw slot`, copy result → display, restore raw via RAII; source and
  destination must differ; reject on resource/signature failure rather than publish a
  half-processed eye). Don't infer injury from health thresholds. A comfort option disables it.
* **Blood/low-health overlays** (`damage_screen.hpp`): one shared angular canvas — union of both
  eyes' tangent extents projected into each eye's clip rect so equal view directions sample equal
  UVs; requires `tan_left<0<tan_right`, widths ≥ 0.001, |tan| ≤ 100; captured draw ≤ 250 ms old,
  same device/reference generation and context; left marks `left_drawn`, right draws only if left
  did.
* **Hit-direction / waypoint indicators** (`directional_ui.hpp`): rebuild each as a world
  billboard from the *natural (centre) camera*, then project per eye with that eye's VP and origin;
  edge-clamped arrows on a head-relative canvas at depth `2*units`, tangent 0.8391 (~80°).
* DOF and motion blur: default off in VR (native dvars written at next start, no hook needed).
  Use `r_dof_limit`/`r_mbLimit`-style engine options, not hooks.

Details of HUD/menu capture are in `09`.

---

## 7. Per-eye resource isolation via D3D11 context hooks

**Problem:** two world passes per frame through one native renderer share every persistent render
target: SSR history, TAA/SMAA histories. Eye 2 would read eye 1's history. **Fix:** make the
native renderer see *different resources per eye* by rewriting binds at the D3D11 immediate
context, not by editing the engine.

What needs isolating: only **history/temporal/ping-pong-with-persistence** targets. Scratch
targets can stay shared because the eyes run sequentially. H2: slot 0 = SSR scene-colour mip
target **91** (read through PS `t13`; effect timeline tracks ids 4, 80, 91), optional slots 1..6 =
native post-AA histories **16..21**; a third copy per target for the auxiliary view
(`view_count = 3`, `view_index = 2`, `no_view = 3`).

### 7.1 Hook set (vtable slots of `ID3D11DeviceContext`, standard D3D11)

| Purpose | Slots |
|---|---|
| OM binds | `OMSetRenderTargets` 33, `OMSetRenderTargetsAndUnorderedAccessViews` 34, `ClearState` 110 |
| Clears | `ClearRenderTargetView` 50, `ClearDepthStencilView` 53, `ClearUnorderedAccessViewUint/Float` 51/52 |
| SRV setters | PS 8, VS 25, GS 31, HS 59, DS 63, CS 67; `CSSetUnorderedAccessViews` 68 |
| Resource ops | `CopySubresourceRegion` 46, `CopyResource` 47, `UpdateSubresource` 48, `CopyStructureCount` 49, `GenerateMips` 54, `ResolveSubresource` 57 |
| Draw/dispatch (census) | `DrawIndexed` 12, `Draw` 13, `DrawIndexedInstanced` 20, `DrawInstanced` 21, `DrawAuto` 38, `…InstancedIndirect` 39/40, `Dispatch` 41, `DispatchIndirect` 42, `ExecuteCommandList` 58 |
| Constant-buffer probes | `xxSetConstantBuffers` VS 7, PS 16, GS 22, HS 62, DS 66, CS 71; `Map` 14, `Unmap` 15 |

Rules for the hook layer:

* **One detour owner per vtable entry**; features register observers/rewriters on it (`set_draw_observer`, `set_rewriters`,
  channels `gpu_census / ssr_consumer / effect_timeline`) instead of installing a second detour on
  the same method. A pre-flight rejects aliased targets (drivers may share entries between context
  modes), targets must be executable and unchanged across devices; install failure is permanent and
  stubs stay pass-through (never partially unhook a process-wide set). Retired hook generations keep
  their trampolines for late calls; DrawIndexed keeps up to 16 trampolines because immediate and
  deferred modes can have different entries that call each other.
* Every stub calls the original **exactly once**, rewrites arguments *before* it, publishes events
  to observers *after*. With no capture active and no observer mask bit set the stub is O(1).
* Redirect only when atomics (`pair_active`, `active_owner_thread`, `active_pair`, context,
  device generation) **and** a thread-local owner token all match
  (`owner_hook_token_matches`). Foreign-thread / foreign-context calls are counted and ignored.
* A rewritten view is found by `view->GetResource()` matching an isolated original texture; the
  replacement view is created lazily with the original view's own `GetDesc()` on the eye texture
  and cached. Hot-path discipline: hash index (`stereo_pointer_index<512>`) for known views plus a
  negative "foreign view" cache (`stereo_pointer_cache<1024>`, cleared per pair) because nearly all
  SRV binds are unrelated material textures; stack arrays only. Pointer caches never alias: a
  collision is a miss/eviction, never another view's mapping; `clear()` is mandatory at
  pointer-lifetime boundaries.
* Maintain a thread-local OM binding snapshot (`sequence`, RT count, engine target id, RT0, DSV)
  after the *effective* (rewritten) views are bound; the engine's own "select render target N"
  routine calls `note_target(id)` so draws can be tagged with the engine's logical target id
  without a device query. Invalidate on `ClearState` and on `ExecuteCommandList(FALSE)`.
  A "replay guard" asserts that a replayed dispatcher makes zero OM binds.

### 7.2 Eye lifecycle

* **Create** per-eye copies with the *same* `D3D11_TEXTURE2D_DESC` as the original; seed once with
  `CopyResource` from the natural texture; keep across pairs while texture identity + full desc
  `memcmp` are unchanged (so changing AA mode or the game publishing another **mip RTV of the same
  texture** does not throw history away). Rebuild on device/context/generation change. Validate
  sources: Texture2D, array 1, no MSAA, USAGE_DEFAULT, no CPU access, no misc flags, SRV+RTV/DSV
  binds, same device as the context, no duplicate ids. Optional history slots: id 0 + null view
  means "inactive", a partial binding is a failure. Unused AA histories are not allocated;
  changing AA modes preserves unrelated SSR images.
* **Rebind on eye entry** (`rebind_inherited_state`): read back everything currently bound (OM
  RTV/DSV/UAVs, CS UAVs, all six stages of SRVs), re-issue the Set calls so the rewriter swaps in
  eye views, then verify zero original-target views remain bound; else fail the pair. Isolation
  must be in place **before the eye's first read** — preparing at first *write* is too late.
  UAV subtleties: use `D3D11_KEEP_UNORDERED_ACCESS_VIEWS` initial counts; a UAV start slot below
  the RT count is a binding-contract failure.
* **Restore on exit/cancel** (`restore_inherited_state`): map eye views back to originals,
  **disable redirection (`current_eye = no_view`) before the inverse setters run**, re-set, verify
  zero isolated views remain.
* **Quarantine**: if restore verification fails, quarantine the generation, reject further pairs
  (`quarantine_rejections`), retry cleanup from `cancel_pair`; release the old COM graph only after
  an owner-quiescence boundary (`pair_active=false` published after every hook returned and
  borrowed views dropped). A generation change while a pair is active defers release; foreign-thread
  invalidation is deferred to the owner thread.
* When the aux history must reset, clear **every mip** of the aux texture via per-mip RTVs (SSR
  reads a different mip than the writer).
* Proof without altering anything: at the known SSR consumer draw read PS SRV slot 13 and classify
  its resource as null/natural/left/right/foreign per eye; "cross_eye" must be 0.

### 7.3 Policy for temporal data

* SSR / TAA / SMAA-T2x / Filmic histories: natural, left, right (and aux) are separate sets.
  Commit history only for a complete pair. Reset on discontinuity: recenter, snap turn
  (`request_temporal_history_reset`), teleport, mode change, resize (the OpenVR resize callback only
  stores the generation; the next Present-pre reinitialises and requests a reset), long gaps.
  Smooth turning does not reset.
* **Never** copy history from the other eye or from the natural view every frame to hide
  inconsistencies; isolating only the right eye or only the matrices still leaks.
* Frame-counter-indexed rings (AA jitter phase): replace the engine's global frame counter **only
  inside the AA call** by patching the two native readers with a register-preserving thunk
  (rax, rcx, rdx, r8–r11, flags, xmm0–5) and use a contiguous VR phase (advance 1, `& 0x3FFFFFFF`,
  reseed when previous pair ≠ last completed pair or age > 250 ms). Other native rendering still
  sees the real frame counter. This keeps T2x/Filmic usable when the engine renders more frames than
  the headset consumes.
* SMAA reuses the scene depth buffer as scratch; `depth_preserver` saves/restores it around the AA
  call so later depth-tested composition (optics, laser) still sees real scene depth.
* AA validation: SMAA **edge target is RG8 on AMD but RGBA8 on NVIDIA** (accept both for the edge
  role only, never for colour); blend/colour targets need 4 channels; never use the HDR scene as an
  AA destination; reject wrong extent/format/bind with named reasons.

---

## 8. Temporal history seeding and commit (matrices)

`prepare_temporal_history` / `commit_temporal_history`:

* State per eye: VP[16], origin[3], device generation, pair_id.
* If a committed state exists for the same device generation and `committed.pair_id < pair_id`,
  each eye's previous VP and eye delta come from **its own** previous committed state. Otherwise the
  pair is **seeded**: previous = current → zero motion. Never borrow the other eye's matrix.
* Written into the eye record: history VP copy, previous-VP shader constant, and previous
  eye-position constant `= (current_origin − previous_origin) * scale` with the engine's own scale
  in the fourth lane. Required: finite values, nonzero scale.
* Commit rejects stale/out-of-order pairs (`state.pair_id >= prepared.pair_id` → false) and only
  runs for a complete, non-failed, published pair.
* H2-only layout: history VP `+0x2D40`, previous-VP constant `+0xF90`, previous eye-position
  constant `+0xFD0`.

---

## 9. Post chain per eye: PostFX redirect, display transform, ring copy

Engines tone-map in a final PostFX pass. For VR you want the **eye image after tone-mapping and
AA, in a VR-owned target**, while the desktop tail keeps the raw HDR scene.

* **PostFX destination redirect** (`render_native_display`, `postfx_destination_stub`): set a
  thread-local `native_display_record` + route; the stub on the engine's "give me the PostFX
  destination" call returns the VR spare ping-pong target instead of the desktop target for exactly
  that record; invoke the PostFX owner directly with `dimensions_match=true` (H2: RDX unused, R8
  selects the target-scaled viewport; ping-pong targets 4↔5, format `R11G11B10_FLOAT`,
  SRV|RTV|UAV, 1 mip). Accept only if the destination getter fired **exactly once**
  (`native_display_routes == 1`). Then native post-AA, then fullscreen blur (order: PostFX → post-AA
  → blur). While `native_display_record` is set, the view-copy, depth-hack and target-select stubs
  forward straight to the original; a fullscreen PostFX view must not be treated as a second scene
  dispatch. The historical "record classifier" turned out to be this final PostFX owner (a
  fullscreen post chain), so replaying it as a scene is wrong; it only runs via the redirect.
* **Resolve which target holds the final scene** (`native_display_contract::resolve(final_target,
  ping_pong1, ping_pong2, selector)`); check the engine's image texture/SRV matches the
  RTV-derived texture (stale bindings can't authorise an input); run the engine's *original*
  display transform for this eye into the native-sized ping-pong destination (checks: not aliasing
  source, same device, supported descriptor, same extent). Preserve the raw HDR source for the
  natural tail. Borrow scene depth (registry slot 4 DSV) for compositor depth testing.
* **Ring copy / conversion** (`native_render_session`): ring of 2 pairs × 2 eyes of ordinary
  same-device textures (no sharing flags); pair states `free → rendering → published → release →
  free | quarantined`; `pair_id` = pose-family/frame id so pose family, view family and ring slot
  share one identity; admission gate `accepting_pairs + expected_pair_id` admits exactly one
  producer per pose family and closes when both eyes complete; the gate is suspended before any
  rebuild/format change. Conversion: display-encoded → linear (fullscreen triangle,
  `Load()` + sRGB decode `x<=0.04045 ? x/12.92 : ((x+0.055)/1.055)^2.4`) because submission is
  linear. Record once per (pair, eye, source-target-index) on a **deferred context** and replay
  with `ExecuteCommandList(list, RestoreContextState=TRUE)` so the game's immediate-context
  bindings are preserved with no manual save/restore. Tag the mod's own command lists and
  deferred contexts with `SetPrivateData(GUID, magic)` so the census/hooks recognise them (never
  mark the game's immediate context). Same-device check by `IUnknown` identity, not raw pointer.
  A replaced source texture in the same cache slot hard-fails the pair; any copy failure
  quarantines the pair and keeps the first failure as value-only evidence.
  Dimensions come from the runtime recommendation with **no clamping**: out of limits means fail.
* **Composition layers** (`eye_composition.hpp`): fixed ordered layer slots (14: world_equipment,
  weapon_guides, optics, weapon_display, screen_scope, spatial_hud, diagnostics, indicators,
  remote_hud, damage, narrative, interaction_diagnostics, menu_backdrop, recording_frame), each
  registered with an RAII `callback_registration`. `compose` runs inside `copy_eye` after display
  conversion and before the ring target is published, on the game's serialised immediate-context
  owner: no waits, no runtime IPC, no retained targets, all D3D state restored. World-anchored quads
  use the **current** world matrix/model origin of that eye's record (current placement − primary
  eye origin), never the previous-frame world; both eyes must report the same placement. Left
  prepares and freezes geometry, right draws only if left drew.
* The desktop mirror reads the **latest published pair** on the same owner thread, never a pair
  being rendered, incomplete or quarantined (§14).

---

## 10. Auxiliary optic / scope view (third scene pass)

Optics (magnified scopes, thermal, monitors) are rendered as a **third bounded scene** (view
index 2) between the eyes. Do not copy the other eye into a missing/dark lens, and never render a
native scene from a composition callback.

* Order **left → aux → right** (`begin_auxiliary` requires `completed_mask == 1`). Eye 0's display
  image + depth are captured into `pending_left_image` and its ring copy is deferred, because the
  aux pass reuses native scratch targets; `finish_pending_left` restores left via `CopyResource`
  into the native source identity and copies it. Eye 1 requires aux complete and no pending left.
  The aux colour/depth are copied first (scratch may be overwritten); the lens draw reads the saved
  image and reverse-Z geometry in front (hands, receiver) still occludes the lens.
* Request: source eye, normalised window (x,y,w,h), owner/generation tokens, optional
  `full_thermal`, `near_distance` override, `native_center` (camera at the published centre origin
  rather than either physical eye).
* **Exact clip-space crop** of an already-finalised record (`crop_record`): with
  `sx=w`, `sy=h`, `cx=2x+w-1`, `cy=1-2y-h`, for projection and VP matrices (rows i):
  `m[i][0] = (m[i][0] - cx*m[i][3])/sx`, `m[i][1] = (m[i][1] - cy*m[i][3])/sy`. Inverse VP
  updated algebraically: `inv[c]*=sx`, `inv[4+c]*=sy`, `inv[12+c] += cx*inv[c] + cy*inv[4+c]` so the
  engine's jitter finaliser never runs twice. Then recompute the legacy tan-half scalars and the
  inverse-projection ray constants `{1/m0, -m8/m0, 1/m5, -m9/m5}`. Window must be finite,
  `w,h ≥ 1e-4`, inside [0,1]. The crop keeps origin, near, jitter and orientation, so every ray is
  inside the source eye's frustum.
* **Near-plane rescale** (`apply_near`): scale projection/VP column 2 and inverse VP row 2 by
  `new_near/old_near` (ratio in 1e-4..1e4, near ≤ 10000) instead of re-running the finaliser
  (e.g. near override beyond the front housing; Dragunov example 31.4 cm clearance).
* `center_record`: relative eye offset moves by `center − eye`; origin moves to the centre; VP is
  camera-relative so needs no change.
* **Thermal**: never invent activation — requires the native heat bit in the record flags byte
  (`+0x204 & 1`), then sets bit 2. Native thermal ADS zeroes the derived SSR multiplier before either
  eye renders, so the two **world** records are normalised (thermal bits cleared, SSR field restored
  from the public scale, explicit 0 allowed, 5 bytes only, leased and restored without rolling back
  native changes made meanwhile) while the aux record keeps the untouched thermal snapshot. Never
  write the global dvar temporarily (frontend workers read it concurrently).
* **History**: aux has its own history copies (§7). `prepare_history` uses the previous pair's
  VP/origin only if device, pair order, owner/generation/eye/reference/revision/thermal/near/
  native_center and window size (within 0.8×..1.25×) are unchanged; else history = current. The
  previous-eye-position delta is `(origin_now − origin_prev) * scale`.
* `image_copy::capture`: owner thread only, immediate context only, same device/generation, 2D
  single-sample single-mip DEFAULT textures ≤ 16384, private SRV/DSV copy target recreated when the
  source desc changes (`memcmp`), never retains a native-borrowed DSV across frames.
* Dynamic upload: the aux call is a deferral that never advances (§6.3). Planners are pluggable
  atomics (screen/weapon-display plans); rejection reasons are recorded per stage.
* Pinhole relation for choosing the scope scene FOV: apparent lens angle `2*atan(r/d)`; scope scene
  angle `2*atan(r/(d*M))` (e.g. r=19 mm, d=100 mm, 6× → 21.52° vs 3.63°). An image magnifier cannot
  recover geometry behind its pixels; thermal needs the extra scene. Eye choice with hysteresis;
  the other eye gets a scope shadow; the extra scene must finish before the final two-eye
  publication. Cost: scope owner calls averaged 10.15 ms in one measured Gulag capture
  (`vr-render-performance-review.md`).
* A full fixed-scope compositor (scene completely covered) shrinks the culling union to the native
  optical envelope (§3.2).

---

## 11. Native scene resolution forcing

Goal: the engine's internal scene colour/depth RT is exactly the runtime's per-eye size (no
submit-time rescale). Resizing must go through the engine's own resize path at its own boundary.

`engine_scene_resolution.cpp`, `engine_scene_extent.hpp`:

* Hook all callers of the engine's "calculate scene dimensions" function and its main-loop
  "check video changes" call. After the original, overwrite `scene_base` and `scene` in the window
  parameter struct with the runtime extent (H2: `+0x20/+0x28`; display `+0x30/+0x34` untouched;
  supersample count at `+0x44`; struct 0x48 bytes). Idempotent; modifies only four u32 fields.
* **Require supersample = 1**: refuse (never silently multiply or override) a nonzero SSAA grid
  (H2 counts {0,2,4,8,16} rejected). The launcher's preflight treats `r_ssaaSamples > 1` as a
  blocking error and auto-repairs.
* Trigger the rebuild only at the **natural main-loop boundary** by setting the engine's
  "soft resize requested" flag, because that is where the engine drains the render thread and takes
  its GPU mutex before destroying/recreating targets. Never resize inside Present or a hook.
* State machine `awaiting_runtime → queued → rebuilding → ready | failed`. Verify after rebuild:
  video-config scene == requested, colour and depth textures of the scene target (read via
  `GetResource` on registry views) have exactly that size and sample count 1, and the desktop
  target equals the display size.
* Limits: width/height ≤ 16384 (D3D11) and engine viewports are signed 16-bit; 0 or >16384 is
  rejected; **never clamp** a runtime recommendation to a lower size; a different recommendation
  later → refuse and require a process restart (never resize an in-flight ownership ring); scene
  contract independent of desktop size.
* The runtime side takes the **max `recommendedImageRect` over both views**, requires ≤ each
  view's `maxImageRect`, uses it for both eyes and hands it to the engine (`04`).
* Stereo claim is gated on `engine_scene_resolution::ready()`.
* Launcher "risk settings" preflight: find early which native render options break per-eye
  rerendering and enforce them (H2: SSAA ≠ 1× blocking; shader preloading, sun/spot shadow caches,
  fill-remaining-VRAM acknowledged warnings with one-click repair; unknown state is reported, not
  assumed safe; shader preloading was tied to a `Create2DTexture` crash).
* The runtime's optical aspect never equals the texture aspect: projection comes from runtime
  tangents.

---

## 12. Engine capacity limits become visible in stereo

Stereo roughly doubles per-frame packing demand against fixed arenas tuned for one view, and
VR-only models (hands, weapons, magazines) add more. Symptoms in Overlord: equipment vanishing when
turning, TV textures flipping; native log lines such as
`MAX_SCENE_SURFS_SIZE((1 << (16 + 2))) exceeded - not drawing surface` (captures: cursor 313592 and
301376 bytes vs 262144 limit, 780/855 failed packings), `Too many visible static model surfaces`
(warning 20), `Too much model lighting used - not drawing static model` (warning 18).

**Acceptance criterion:** during stereo tests grep the engine's log for `exceeded`, `not drawing`,
`too many`, `too much`; any hit is a failure.

Fix pattern (all-or-nothing instruction contract tables, `src/client/component/scene_*`, generated
offline by `tools/generate_scene_surface_contract.py`; see also `02`/`03`):

1. **Whole-image census, not a literal search.** Disassemble a clean dump of `.text`/`.pdata`
   offline (Capstone), audit every operand that references the structure by immediate or
   `base+field` displacement, preserve addends, exclude same-valued RVAs/strings outside the
   renderer. A literal-base search missed 24 sites and crashed twice (null brush-surface pointers
   via `+0x62b708/+0x62b728`). Assert exact census counts so a different build or an incomplete
   audit fails generation.
2. **Change the ABI unit coherently.** The native `uint16 surfId` is a scaled **byte offset** (4-byte
   units), not an ordinal; raising only the byte limit wraps IDs. H2: unit 4 → 8 bytes, capacity
   `65536*8 = 512 KiB` per frontend, **242 instruction patches** (e.g. hidden-record stride 4→8,
   `cmp eax,0x40000 → 0x80000`, `shr rax,0x12 → 0x11`, masks `0x3fffc → 0x7fff8`, brush prefix
   `0x1c → 0x20` and surface index advance 6→3, the diagnostic string digit) plus relocation of
   every arena base reference to a mod-owned bank. Patch kinds: base-only, arena+field,
   folded/divided, modular-inverse (stack-base-minus-offset). Do not touch the neighbouring
   motion-history arena (different 4-byte unit).
3. **Bank layout.** Two native frontends are consumed concurrently with in-flight backend workers,
   so the replacement bank keeps the native frontend stride (`banks[native_frontend_stride +
   capacity]`, one displacement selects the matching bank from either frontend pointer; positive,
   ≤ INT32_MAX, multiple of 8). Do not compact the gap or share a scratch buffer (aliases an
   in-flight frame N-1). A static `.bss` bank costs nothing for the unused gap.
4. **Transactional patching.** Verify ALL sites first (masked bytes), compute all replacements,
   write (`VirtualProtect` RWX + memcpy + restore + `FlushInstructionCache`), re-verify each
   written site, rollback (restore `before`) in a scope guard unless committed. Mixed 4/8-byte
   indexing is corruption: no partial install, no runtime toggle. Gate on the verified-image probe.
5. **C++ consumers are format consumers too.** The mod's own readers of the native stream must use
   the shared `record_bytes()`; a stale hidden stride of 4 broke original-magazine hiding. A native
   patch census cannot catch C++ consumers; test all 65,536 IDs, last slot, hidden entries at
   begin/middle/end.
6. **Check what is serialised.** Savegames archive derived handles (H2 `DynEntityClient::
   lightingHandle` includes the static partition offset), so enlarging the lighting cache
   (4096 → 12288 static of 16384 total, via two minimum-quota compares through flags-only relay
   bridges) needs a range guard (`static < handle ≤ total`, using only the register native code
   clobbers next) on the dynamic-reuse site; out-of-range falls to the allocation path.
7. Static transparent list quota: per-region byte quota 1024 → 4096 (lit-transparent region), backing
   bank ×4 (`0xD2000 → 0x348000`), 15 operand sites; budget table `0x140990B78`.

Verified byte match proves the *listed* sites match, not that the list is complete; completeness is
a separate census test. Other cheap resource headroom: bigger engine memory pools
(`component/memory.cpp` doubles the hunks/physical reserve) for stereo and extra assets.

---

## 13. Evidence gate: execution census, distinct-eye readback, diagnostic census

Three distinct mechanisms (do not conflate them):

| Mechanism | File | Role |
|---|---|---|
| **Execution census** | `engine_stereo_execution.cpp` | Production gate: proves single context/thread, no opaque deferred work, every draw attributable to the scene-owner record to be replayed |
| **Distinct-eye readback (bootstrap proof)** | `engine_stereo_owner_pass.cpp` | First stereo pair runs in non-production mode; both final eye textures are read back and hashed; arms production |
| **GPU census** | `engine_stereo_gpu_census.cpp` | Post-bootstrap, **report-only**, one-shot left-vs-right draw-stream diff scheduled on demand; cannot publish or cancel the native pair |

### 13.1 Execution census

* Hooks 9 draw/dispatch/ExecuteCommandList slots (Draw 13, DrawIndexed 12 via a separate observer
  that forwards, DrawIndexedInstanced 20, DrawInstanced 21, DrawAuto 38, indirect 39/40, Dispatch 41/42,
  ExecuteCommandList 58) plus OMSet 33/34 and ClearState 110. Targets validated executable,
  pairwise distinct, unchanged across devices.
* **Admission gate = one 64-bit atomic**: bit63 open, bit62 frame-scope, low 32 bits = writers in
  flight. Writers CAS-increment; a closer clears the open bit and cannot finalise until writers == 0
  (the linearisation point). **17 states**: five `classifier_*`, seven `frame_*`/present
  states (`frame_pending … frame_closing`, `awaiting_end_present_post`, `end_present_arming`),
  `failure_closing`, `finalizing`, `resetting`, `complete`, `failed`.
* Scope 1 ("classifier") = one-shot capture around the engine's per-record classifier call
  (historical name; it is the PostFX owner); scope 2 = one complete Present-pre → Present-post frame.
* Pass criteria (`finalize_report`): classifier record seen and ended after it began;
  `frame_end_present_pre == frame_start_present_post + 1`; `end_present_post == end_present_pre`;
  Present HRESULT ≥ 0; start/end thread equal; no event overflow (32768-event ring), no admission
  collision, no identity truncation.
* Per event: QPC, context, caller (`_ReturnAddress`), thread, args, OM binding snapshot, TLS "backend
  record context" and "scene owner context" (pointers never dereferenced), up to 16-frame stack,
  de-duplicated by (binding seq, route, caller, op). Distinct contexts/threads/records tracked in
  lock-free open-addressing CAS sets (256 slots); overflow = failure.
* The mod's own conversion helpers (UI canvases, ring conversion) are exempt only if provably tagged
  (command-list/context private-data marker, recording-context check, TLS flag) because WARP
  re-enters Draw/Dispatch while executing a list. Unmarked lists, `RestoreState=FALSE` and foreign
  contexts remain failures.
* **Production gate** (`strict_execution_contract_ready`): state complete, no error, events
  recorded, scene-owner events > 0, `distinct_scene_owner_records == 1`, 0 scene-owner thread
  mismatches, 0 foreign-context events, 0 opaque ExecuteCommandLists, not deferred-opaque, not
  truncated. Readiness must come from the *current process*; files from another run never count.
* A fuller acceptance (`accepts_complete_execution_evidence`, `renderer_evidence.cpp`) is the
  conjunction of six: prerequisites, backend target frame, output merger, DrawIndexed, execution
  census, terminal device identity consistency (single device generation and context across probes;
  attempts==1, completions==1, failures==0; report/status field-by-field consistent; every error
  counter zero). Terminal evidence is immutable: replacing devices never mutates a completed report.
* Minimal port: for one frame count draws per (context, thread) and assert 1/1, no deferred contexts.

### 13.2 Bootstrap readback

The first pair is non-production: both final eye textures are copied to STAGING textures
(`copy_source_to_staging`; if the source is currently bound as RT, unbind with
`OMSetRenderTargets(0)`, `CopyResource`, rebind), a `D3D11_QUERY_EVENT` ends the pair,
`on_present_pre` polls it (`D3D11_ASYNC_GETDATA_DONOTFLUSH`, ≤ **240** polls), then `Map`s and
FNV-1a hashes each eye. Pass only if `content_hashes[0] != content_hashes[1]` and both have
non-zero bytes; only then `state=complete` and production begins (production does zero
readbacks). Staging format must equal the source format for a legal `CopyResource`.
OpenVR arming additionally requires this proof to show both eyes completed, distinct and
non-zero on the current context/generation/owner thread.

### 13.3 Joining the engine's GPU mutex (the Rule-7 exception)

H2 serialises every immediate-context use (Present, `GetData`, the backend renderer) through a
recursive kernel mutex behind a global HANDLE (H2-only: `0x140C05740`). The owner pass waits on that
same handle (**100 ms** timeout) around eye-resource begin/end, display copy and pending-left
restore instead of enabling D3D11 multithread protection or adding a competing lock. Only
`WAIT_OBJECT_0` counts as success (`WAIT_ABANDONED` still releases but is a failure); wait
total/max/last are exported. Overlord merely **queries** `GetMultithreadProtected`; forcing it did
not help. For another game: find the engine's own device lock (or confirm single-thread use) and
join it. The mod's own `state_mutex` is still held across D3D calls inside eye-resource begin/end;
the principle is "no mod-private lock across native work except in tightly scoped probe code;
join the engine's own context lock".

Related thread rules:

* D3D11 immediate-context queries (`End/GetData` for copy completion) must stay on the **renderer
  thread** that produced the copy: polling from Present raced `CopyResource/End` on the game's
  single-threaded device and surfaced as an NVIDIA driver fault inside Present. Overlord polls in
  the `R_EndFrame` hook (`scheduler.cpp`); that hook runs *after* the frame's records are built, so
  it cannot define the pose for them — freeze the per-eye pose at the owner, not here.
* GPU timestamp queries (`engine_stereo_gpu_timing`) are **compiled out of production** (the
  automatic one-shot at the eighth retired pair added GPU commands to a stable NVIDIA frame);
  production timing is CPU `steady_clock` counters per phase.
* Preflight rejects a game device created `D3D11_CREATE_DEVICE_SINGLETHREADED` for the OpenVR path
  and requires the proven scene-owner thread == Present thread (`openvr_runtime.cpp`).

---

## 14. Desktop mirror

Sample a completed eye texture (default right, `vr_desktopEye`) inside the game's existing GUI draw
callback (no independent render loop, shared texture, GPU wait or runtime call), via ImGui draw-list
callbacks that bind a custom pixel shader and restore state. Refuse unless the current RT equals the
engine's desktop target and its format is `R8G8B8A8_UNORM`.

* Crop symmetrically around that eye's optical forward to the window aspect:
  `half_x = min(tan(fov/2), available_x, available_y*aspect)` with a half-texel inset; flagged
  `limited` when the request can't be satisfied. Defaults: 95° horizontal FOV (30–120).
* Use projection metadata retained **with that image** in the ring, never newer runtime values
  (missing/invalid metadata skips the draw: `invalid_projection_or_extent`).
* The eye texture is linear; the mirror shader (`D3DCompile`, ps_4_0) applies the exact sRGB OETF
  (threshold 0.0031308, 12.92, `1.055*pow(x,1/2.4)-0.055`) because the UI shader expects
  display-encoded input.
* Optional spectator stabiliser: low-pass the camera orientation, compute the relative rotation to
  the source eye, bisect the slerp fraction (16 iterations) to the largest correction whose four
  corner rays (0.85 of the crop half-tangents) stay inside valid UV bounds; the shader applies a 3×3
  homography.
* Menus/console stay usable on the desktop without VR. Mirror failure never affects HMD transport.
  The mirror reads only the latest *published* pair, never one being rendered.

---

## 15. Failed experiments and pitfalls (do not repeat)

| Experiment / mistake | Outcome | Do instead |
|---|---|---|
| **Dual-record**: synthesise a second scene record before `R_EndFrame` and invoke a second frontend scene owner | After the frame finaliser invalidated transient state, same-process A/B showed the mutation caused the engine's fatal "Tried to use (null)" drop; calling the owner a second time corrupts the surface builder's transient state | Derive per-eye slots from the **pure** finaliser (called twice per family) and run both eyes inside one backend owner transaction. A/B in the same process before trusting any replay strategy |
| Repeat `R_RenderScene` per eye | Frame-global state consumed/finalised by the first call | Per-eye backend owner transaction |
| Replay the "classifier" command stream as a second pass | It was the fullscreen PostFX chain; replaying adds GPU work and contaminates the census | Redirect its destination (§9) |
| Bootstrap pointer-swap proof (`engine_stereo_backend_view`): swap the source pointer of the backend's view-slot copy to an eye slot for one call | Valid as a proof gate (post-conditions: pointer restored, destination bytes equal the eye slot, natural record byte-identical after dispatch), not the production mechanism | Production runs the owner on the real eye record (§2.3) |
| Command-stream replay evidence: hash the engine's recorded command stream before/after dispatch (cap 16 MiB / 1,000,000 records, FNV-1a; must be identical) and compare natural vs replay DrawIndexed shapes (same caller/index_count/base_vertex; `start_index_location` identical or all shifted by one constant = "dynamic index append") | Proves replay does not consume the stream and exposes arena advances | Keep as an offline/probe aid |
| Per-eye culling with a virtual rearward apex | Tight in theory, no practical gain, withdrawn | `half_ipd/near` union (§5.1) |
| Isolating only the right eye, or only matrices | History still leaks | All history resources, per eye, before first read |
| Copying history from the other eye/natural each frame | Masks bugs, wrong motion vectors | Seed once, then per-eye commit |
| Preparing history at first write | Too late (SSR read first) | Rebind on eye entry |
| "Shadows/lights are view-independent" | Wrong in H2 (§6.1) | Per-eye rebasing with all-or-nothing restore |
| Absolute eye origin as relative eye offset | Models subtract camera twice | `natural_rebase + displacement` |
| Identical-VP check in `validate_finalized` | Rejects symmetric-FOV runtimes | Require distinct **inverse** VP |
| Widening the frontend culling for unarmed/proof frames | Contaminates desktop | Only admitted pairs |
| Patch `FxSystem.camera` asynchronously | Races FX workers | Patch at the builder call |
| Temporarily write a global dvar for thermal | Frontend workers read it concurrently | Normalise the records |
| Rewrite-based hooks as runtime toggles | rel32 rewrites aren't atomic | One-shot loader install (see install-time rule below) |
| Auto GPU-timing one-shot in production | Added GPU commands to a stable NVIDIA frame | Compile out; CPU counters |
| Polling D3D queries from Present | NVIDIA driver fault | Poll on the renderer thread at `R_EndFrame` |
| Disabling `vr_engineProbe` as "diagnostics" | It controls production stereo publication (historical name) | Leave it; gate real probes with startup-frozen options |
| Private compositor device / cross-device copy / shared fences | Obsolete experiments (TDRs from double full-res copies) | Same-device, non-shared ring |
| `+0x10` FX plane offset from a header comment | Code writes `+0x20` | Trust the code |
| MSVC v142 `/GL /LTCG` WPO | Placed mutable publication buffers into `.rdata` / dropped writes (level-load AVs) | WPO off; publish/readback test + PE/PDB section audit |

Install-time rule for every call-site patch (`install_hook`): compare the 5 bytes against the
expected `E8 rel32` with a full mask, decode the displacement and require it resolves to the expected
callee, require the callee executable, require the stub within rel32 range, capture page protection,
write (`VirtualProtect` RWX, memcpy, `FlushInstructionCache`, restore), re-read and re-verify. Order:
**inner observers and consumers first, the record producer last**, so a producer never publishes into
a half-installed consumer. On any failure roll back every site with the captured protections,
re-verify, and `throw` if rollback is incomplete ("refusing to start with partially patched code").
This is a one-shot loader operation before the renderer thread starts. To observe a mid-function value
(e.g. a `GetData` result), replace a 6-byte `mov ecx,[rip+x]` with `call relay; nop` through a
register/flag-preserving near relay that replays the displaced load and reserves stack space for an
aligned call.

---

## 16. Debugging method: find which engine state must be per-eye

When per-eye artifacts appear, instrument at the D3D boundary and **diff the two eyes**; fix cause by
cause (SSR history, CB content, dynamic arena cursors, effect index rebasing). Build a small version
early. In order of cost:

1. **Cheap**: counters for rewrites / foreign bypasses / contract mismatches per patch; identity
   mismatch reasons; the evidence enums below.
2. **Lockstep call census** (`engine_stereo_gpu_census`, one-shot, report-only, scheduled by a
   control-thread token that the render thread applies at the next exact owner boundary): record the
   ordered GPU calls for each eye (API, caller, args, VS/PS/CS/GS/HS/DS identities, RTV/DSV
   resources, depth/blend/raster state, input layout, topology, index buffer, vertex buffers,
   viewport/scissor hashes, FNV hashes of CB/SRV/sampler/UAV/output bindings; 20-bit mismatch mask),
   pair by global ordinal and family-local ordinal, compare. A well-behaved stereo pass has the
   **same call stream in both eyes**; divergence points at engine paths using stale per-eye state.
3. **Shared-resource hazard**: record written resource identities per eye;
   `right_first_read_before_write_candidate` = eye 2 reads a resource only eye 1 wrote → a resource
   that needs isolation. Only left writes populate the table so material textures don't consume
   capacity.
4. **Constant-buffer provenance**: hook `Map/Unmap/UpdateSubresource/CopyResource/Set*ConstantBuffers`
   (per stage), hash contents, record upload generation/source/caller/thread; classify slot use via
   shader reflection (`D3DReflect` on bytecode the D3D11 wrapper stashed as private data at
   `CreateShader`; disassembly `dcl_resource tN`/`dcl_constantbuffer cbN` fills what reflection
   misses) so "left bound / right null but unused" isn't noise; byte/word-diff with variable-name
   attribution. Tag buffers/shaders with `SetPrivateData(GUID, serial)` (D3D addresses are
   recycled; shader load-definition pointers are reused). Content with no observed upload since
   tracking began is `content_unknown`, never "equal". `CopyResource` propagates source content only
   if sizes match; `UpdateSubresource` is known only with subresource 0, null box, full size.
   H2's 1088-byte material buffer is `CopyResource`-fed from a structured UAV buffer (stride 16): a
   material probe stages those copies per eye and an experiment snapshot/replay hook tests
   "stale per-view material data".
5. **Dynamic-FX family comparison**: classify draws into families by caller address range from exact
   x64 `.pdata` extents; compare per-family call counts, IndexCount/StartIndex/BaseVertex, programs,
   buffers, strides, state, CB content, SRV/sampler descriptors. A **constant StartIndex delta**
   means a shared dynamic arena advanced between eyes. A particle/index/vertex GPU readback
   comparator (staging copy per eye; poll; compare only the exact indexed range; CPU reference
   to separate "GPU copy mismatch" from "CPU data differs") answers "does the engine upload
   different data per eye?".
6. **Scene-batch census**: snapshot backend scene-batch descriptors at each view-copy ordinal and
   hash streams (4 sampled 1 KiB windows), normalising only exact current-view lighting metadata,
   to compare eye-0 vs eye-1 engine-side descriptors and local executor consumers (SSR, code_trans,
   glass, spark) with per-consumer sample quotas (so one consumer cannot starve others).
7. **Effect timeline**: freeze one pair; per tracked target per eye: invocation counts, first/last
   clear, `clear_before_first_inv`, `resource_shared`; lockstep walk with 4-step resync lookahead
   classifying the first divergence (`tail_count`, `extra_output0/1`, `replacement`). Watch who last
   wrote SSR-read textures.
8. **Shader discovery without source**: find which pixel shader reads a resource at a required SRV
   slot using bytecode reflection + a write-provenance ring (16384 records at the D3D API boundary;
   `ExecuteCommandList` is a hard opaque barrier unless the list is tagged as the mod's own).
   Discovery runs 2..4 sampled pairs, only candidates seen in **both** eyes of ≥ 2 pairs count,
   then a focused capture ≤ 8 attempts; ordering violations fail the probe.
9. **Binary-diff evidence**: one-shot immutable captures (slot before/after initializer, left/right
   eye slots, record before/after owner) with SHA-256 and `delta=name changed_bytes first last`
   between stages prove exactly which bytes your derivation changed. A same-pair GPU readback was
   what exposed the transparent-shadow bug where draw counts, shader and resource identity looked
   identical.
10. **Write the template probe for a native effect** (flare probe style): reads engine memory via a
    destructor-free SEH leaf, calls the original exactly once with no observer lock, asserts layout
    assumptions after the call (e.g. vertex count +4).

Discipline for every probe: fixed static storage with explicit overflow counters; evidence is an
enum (`unfinalized / not_observed / invalid_shape / observed / truncated / incomplete`), never a
boolean — **truncated/incomplete/unfinalized are not-passed**; seqlock trace rings (CAS even→odd,
drop on contention, versioned reads); status getters try-lock and print `snapshot_busy`; heavyweight
probes are explicit commands gated by startup-frozen options and never enabled implicitly;
observers never issue GPU work (a diagnostic copy uses an "unobserved" helper so it doesn't perturb
what it measures); safe reads of engine memory are `VirtualQuery` committed + non-guard, then
`__try memcpy` in a destructor-free leaf. Counters measure admission, not visible output
("submissions are not proof of visible stereo pixels"); record offline/WARP evidence separately from
headset acceptance (`11`).

---

## 17. Minimal-port path for strategy A

1. Find the frontend/backend split. Capture one frame; record the view struct, its finaliser, the
   scene-record owner, and the call that consumes the record.
2. Hook as observers only; prove record address reuse (don't assume FIFO) and the completion flags.
3. Derive eye slots from the centre slot using the engine's pure finaliser; validate (§3).
4. Publish before the surface generator; claim with a camera-equality check (§2).
5. Run the owner once per eye on a private clone (eye 2) + eye-local fields (§4). Rewind arenas,
   defer the upload advance (§6.2–6.3).
6. Isolate history by rewriting D3D11 binds (§7). Start with SSR/TAA only.
7. Add the lockstep census early (§16 steps 2–3) — it is the fastest way to find the remaining state.
8. Gate submission: execution census (single context/thread, no opaque lists) + distinct-eye readback
   (§13). Until then, fail closed to flat.
9. Union culling, FX planes, tessellation, LOD origin (§5). Then lights, placement cache, depth hack
   (§6). Then post chain/AA/blur/flares (§6.6, §9). Then capacity audit (§12).

Exit criteria: correct binocular parallax; no per-eye popping or detached layers; no engine
"exceeded/not drawing" logs; execution census clean; identical call streams between eyes in the
lockstep census; headset acceptance recorded separately.

---

## 18. Performance notes

* Two full world passes roughly double GPU time. Provide resolution scale (runtime recommendation ×
  scale) but keep the engine's own SSAA at 1× (§11).
* CPU: clone records with memcpy, no per-frame allocation; the dynamic arena and placement probes
  use a bounded working-set query (`K32QueryWorkingSetEx`, `VirtualQuery` fallback) and never cache
  page permissions across native calls; observer interest masks avoid constructing per-draw
  arguments; pointer indexes with collision fallback.
* Measured example (Gulag/M14 EBR capture, all launcher diagnostics off): 21.94 ms average per owner
  transaction, ~6.00 ms in nested dynamic-view synchronisation, scope aux owner calls 10.15 ms (not
  a before/after number). Left-on debug probes cost real CPU (16-frame stack capture per transaction;
  hashing 0x501C8 bytes × 256 transactions ≈ 160 MiB; `VirtualQuery` per diagnostic call; watchdog
  every 2 s): gate them behind startup-frozen selections. A Debug build spent 35–44 ms per eye pair
  (JMC/RTC helpers): use RelWithDebInfo for performance work.
* **Do not parallelise native eye owners** that share the native context and dynamic arenas.
* Measure phases separately (CPU `steady_clock`): transaction, per-eye, owner invoke, dynamic view
  copy, native copy, dynamic restore, context-lock wait; optional wall-time wait probes
  (`vr_debugWaitTiming`). Submit interval is not HMD FPS/reprojection.

---

## Appendix A. H2-only layout and address examples

All addresses assume the H2 MW2CR SP image at base `0x140000000`. Examples of *kinds* of data to
find in another game, never values to reuse. Probes are fingerprint-gated (exe SHA-256, PE
timestamp, image size).

**View slot (0x170 bytes)** and **scene record (0x8090 bytes, current view slot first):**

| Offset | Content |
|---|---|
| slot `+0x00` / `+0x40` / `+0x80` / `+0xC0` | view matrix / projection / VP (camera-relative) / inverse VP |
| `+0x100` | origin (3 floats) |
| `+0x10C` | axis 3×3 rows forward/left/up (screen right = −axis[1]) |
| `+0x140/0x144`, `+0x150/0x154` | legacy tan-half x/y (primary/secondary) |
| `+0x78`, `+0x148` | reverse-Z near (record, slot) |
| record `+0x190` | current model placement origin (`+0x19C` = previous-frame world; history) |
| `+0x204` | thermal/stencil flags byte (bit0 native heat, bit1 forced) |
| `+0x570` | command stream; `+0x1F68` frontend; `+0x1F70` record type (world = 4) |
| `+0xF90` / `+0xFD0` | previous-VP constant / previous-eye-position constant (w = scale) |
| `+0x1270` | inverse scene projection constant (4 floats) |
| `+0x2C90/94/98`, `+0x2C9C` | target ids, ping-pong selector |
| `+0x2CA4` | SSR scale float |
| `+0x2CC0` (+0x100 = `+0x2DC0`) | frontend rebase view; relative eye offset used as XModel eyeOffset |
| `+0x2D3C` | backend depth-hack near |
| `+0x2D40` | history VP |
| `+0x2FC8` | draw-list descriptors: 68 × `0x130`; active type `+0x100`, origin `+0x110`; camera-model subset indices 0..25 and 29 |

**Backend state:** projection cache `+0x0100`; dynamic data pointer `+0x2BE8` (dynamic-light
descriptors); eye projection `+0x2C30`; copied view `+0x2BF0`; primary origin `+0x2D60`; relative eye
offset `+0x2D70`; projection cache version `+0x3160`, input version `+0x31EA`; primary source pointer
`+0x3238`, rebase source `+0x3240`; XModel placement cache qword `+0x3270`; depth-hack flags `+0x3340`.
Dynamic arena descriptors at data `+0x5409C0` (code_trans ×5), glass `+0x540B78`, mark `+0x540BD0`,
spark `+0x540C28`, stride `0x58`; fields: index count `+4`, index buffer ptr `+8`, index base `+0x18`,
vertex payload bytes `+0x24`, vertex buffer ptr `+0x28`.

**Hook sites (examples):** reserve record `0x140779E00`; `R_RenderScene` `0x14077B820`; allocate slot
`0x14076D5D0`; initialize slot `0x14077E340`; **finaliser `0x14077F550`**; generate draw surfaces
`0x140778E60`; FX camera build `0x140463310` (camera ABI: size `0xC0`, valid `+0x0C`, planes written
at `+0x20`, axis `+0x70`, plane count `+0x94`); backend target-prepare **owner** `0x1407A7DA0`
(extent `0x1407A7DA0..0x1407A8306`); dynamic upload `0x1407A8A80` (call `0x1407A8032`); post-bind
`0x1407A7320`; backend record `0x1407A6BE0`; view bind `0x14079F6F0`; final PostFX owner
`0x1407B0740`, PostFX destination `0x1407B09A0`; view copy `0x14078B5D0` (0x170-byte copy); view
setup `0x14078A340`; depth-hack projection builder `0x140786D30`; target select `0x1407892C0`; command
dispatch `0x14079EAE0`; tessellation fill `0x14029E9D0`, map `0x14029F930`, unmap tail `0x1402A0D30`;
query publish `0x14074BB90`.
Globals: frontend pointer `0x150F91188` (producer side) / `0x151A7F3B0` (backend side); LOD origin
`0x14EEE0890`; GPU context mutex handle `0x140C05740`; video config `0x14EEE0CF0`. Frontend offsets:
slot base `+0x540400`, record arena pointer `+0x540FA0`, current record index `+0x540F90`;
completion flags `+0x541BE8..0x541BF4`; dynamic light descriptors `+0x541C08` (count `+0x542588`, max 8).
**Target registry** base `0x150FD1700`, stride `0x40`, capacity 304; entry = GfxImage* `+0`, RTV `+8`,
DSV `+0x10`, width/height u16 `+0x30/+0x32`; id 4 = main scene colour/depth, id 1 = desktop, 4↔5
display ping-pong, 91 = SSR colour mips, 16..21 = AA histories, 13 = display input, 14/15 = SMAA
edge/blend, 12 = filmic scratch. **Dynamic-FX consumers** (`.pdata` extents): code_trans
`0x1407B9780–0x1407B9AFB`, glass `0x1407B9B00–0x1407B9D76`, mark `0x1407B9D80–…`, spark
`0x1407BA6B0–0x1407BA9DB`. **Post-AA:** dispatcher `0x140296EB0`, frame counter `0x14EEE0C9C`
(readers `0x1402970DF`, `0x14029737B`), mode at record `+0x20A4` (0 none, 1 fxaa, 2 smaa, 3 smaa_t2x,
4 filmic, 5 filmic_t2x, 6 internal). **Blur:** native routine `0x1407B0550`, radius `record+0x234`,
menu blur `CL_GetMenuBlurRadius 0x1403D8790`. **Flare:** emit `0x14042E816→0x140432510`, draw
`0x140295118→0x140294E90`; matrices at source state `+0x2BF0`, depth-hack flag `+0x3340` bit 0, version
stamps `+0x31E8/31EA/31EC/31F6/31FC/31FE` (never WORLD0 `+0x31FA`). **Lights:** technique `+0x100`,
`0x25 spotlight, 0x26 muzzle-flash point, 0x28 shadow/cucoloris spot, 0x2D fog variant`; shadow-map
draw technique `0x06` is not a light.
