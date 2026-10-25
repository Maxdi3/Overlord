# 09 - Menus, HUD, video, prompts and the launcher

Goal: every 2D thing the flat game shows (menus, pause, loading, video, HUD, subtitles, damage
overlays, prompts) becomes a world- or head-anchored VR surface, **without rebuilding the game's UI**.
Overlord never re-implements a menu or HUD. It *captures the engine's own 2D output*, re-presents it,
and feeds VR input back through the engine's own event functions.

All Overlord paths are under `src/client/component/vr/` unless they start with `src/` or `data/`.
Addresses/offsets are H2-only examples. Constants were spot-checked in the cited source files
(`menu_surface.hpp`, `native_hud_quad.hpp`, `native_hud_blend.hpp`, `native_menu_commands.hpp`,
`bridge_protocol.hpp`, `preflight_policy.hpp`, `risk_settings_config.hpp`, `presentation_options.cpp`,
`settings.hpp`, `data/ui_scripts/vr_gameplay/__init__.lua`); anything tagged `(report-only)` comes from
a reader report and was not re-verified.

## 0. Pick a strategy per UI element

| Element | Cheapest correct strategy | Overlord |
|---|---|---|
| Menus, pause, popups, briefings | Capture native draw commands -> texture -> compositor layer; ray pointer -> native mouse/keys | `native_menu*.{cpp,hpp}`, `menu_surface.hpp`, `openxr_menu.cpp`, `menu_overlay.cpp` |
| Fullscreen video / loading | Copy the swapchain image into a lease pool, show on a big curved layer, no world layer | `movie_presentation.hpp`, `openxr_menu.cpp`, `menu_overlay.cpp` |
| Gameplay HUD | Capture -> head/world-anchored quad, or re-home a few elements (ammo on the gun) | `native_hud_capture.cpp`, `weapon_hud_*`, `eye_composition.hpp` |
| Damage/blood, scope overlays, flares, blur | Convert per eye (not "disable") | `damage_screen*`, `native_flare*`, `native_fullscreen_blur.cpp` |
| Subtitles/announcements | Capture by native producer flags, 60-degree head-relative canvas | `narrative_ui*` |
| "Press X to use" prompts | Rewrite by **native string key** at the lookup function | `hud_prompts.*`, `component/game_text*`, `component/ui_scripting.cpp` |
| Settings/first-run/preflight | Separate launcher process (WebView2), never inside the game | `src/client/launcher/`, `src/launcher-ui/` |

Rules that hold for all of them: (1) never put flat 2D in screen space inside the HMD; (2) every
captured image carries identity (device generation, context, owner, reference generation, timestamp)
and is rejected, not guessed, when any of it mismatches; (3) a missing/stale/invalid capture shows
*nothing* (or black for optics), never another eye's or an older frame's pixels; (4) the native UI keeps
running natively (timers, animation, focus) - VR only observes and injects input.

## 1. Capturing the engine's 2D draw stream (implementable detail)

### 1.1 Pipeline

```
UI owner thread:  LUI/menu tree callbacks emit 2D commands into a per-frame command ARENA
   |  (tag [arena.used_before, arena.used_after) with the owning surface; hash bytes)
render thread:    per-view HUD dispatcher  +  inlined GLOBAL 2D loop (frontend/menus)
   |  walks the arena, calls per-opcode handlers (quad/text/stretch/rotated/lines/compass)
   |  >>> handler hooks: if this command pointer is selected -> flush tessellation, mark "copying",
   |      call original, flush again
D3D11 DrawIndexed observer: while "copying", retarget the draw to a private RGBA8 texture
   |  (same pipeline state, translated viewport/scissor, rewritten blend) then restore state
publication:      immutable `frame` lease {texture, generation, sequence, timestamp, context, ref gen}
consumer:         menu presenter / eye_composition layer / narrative / damage / scope / remote HUD
```

### 1.2 Command stream format (what you must discover for your engine)

Overlord (H2, `native_hud_quad.hpp`, `native_menu_commands.hpp`): header `{u16 size, u8 op, u8 flags}`,
op 0 terminates. Walk by `size`; bound the walk (<= 2048 commands and 0x100000 bytes per stream,
0x400000 absolute for menu streams); reject `size < 4`, truncated and zero-size records so a corrupt
stream can never loop. Known formats: op 10 StretchPic / op 12 rotated (56 B), op 16 XY quad (72),
op 17 LUI XYUV quad (104), op 18 compass (120), op 20 text (>= 233, UTF-8 at +232), op 25 line batch
(`8 + 32*count`). The decoder checks `size` against the expected size per op and rejects non-finite or
|v| > 32768 coordinates. Pitfalls found: StretchPic's +52 is padding (may be NaN), not an angle; colour
sits at +48 (ops 10/12), +64 (16), +96 (17); line batches **grow after emission** (size and count
fields change), so the fingerprint skips bytes 0-1 and 4-5 of a growing batch.

For another engine the equivalent is: find the point where UI draw requests are *recorded* (command
buffer, display list, or immediate calls) before they become GPU draws. If the UI is a render-to-texture
system (Scaleform, Coherent, Unity canvas) you can usually skip all of this and take its target texture.

### 1.3 Selecting commands: ownership, not geometry

- **Select by exact command pointer**, never by screen overlap. Proven producer ownership outranks
  overlapping coordinates. Scopes: weapon HUD, story text, progress text, announcement, damage, scope
  layers, warnings, markers, remote HUD, menu surfaces. Selection may additionally use material name
  (read by guarded pointer chase), geometry bounds, flags.
- **Ownership by command-buffer diffing** (`native_menu.cpp render_owned`, `native_waypoints.cpp`):
  wrap the element's native draw callback, read the arena cursor before/after, tag the byte range with
  the owning surface. The owner is found by walking the element's parent pointer chain (<= 64 hops)
  against known menu roots/background/cursor/backdrop elements. Also hook the **cached-draw path**: a
  cache hit emits commands without calling the primitive callback.
- **Fingerprint + lifetime**: store an FNV-1a hash of the tagged bytes; re-hash at consumption and drop
  on mismatch (arena reuse). Bounds: 3 arena frames, <= 2048 records, group lifetime 250 ms; hook the
  frame-reset function and invalidate only the arena that was reused. Ranges must not overlap; select
  by arena *containment* (adjacent arenas can be only 0x30000 apart), not a guessed span.
- A finalized native batch must fall entirely inside one surface (partial coverage rejects the stream).
  An unowned stream must never publish an empty menu over a valid one.
- **Classify by native flags and producers, not text**: subtitle flag 0x100, typewriter/pulse mask 0xC0,
  0x4000 = exclude from mirroring, frontend style 0x400 is not a selector; observed ammo text 0x2014 is
  not narrative; a hint with 0x10 is not a subtitle. Never classify by translated text, OCR or screen
  position. Script-owned HUD text is classified through the producer hook (section 5).
- Fade/letterbox detection: only a full-viewport quad whose four vertices cover the viewport counts as a
  fade (an AABB of a rotated diamond is not coverage; a letterbox bar is not a fade); read alpha at the
  right offset per format; black image needs a white tint, white image a black tint; overlapping fades
  compose in native order `(1-a1)(1-a2)`.

### 1.4 Hooking the handlers safely

1. Verify the function prologue bytes **and** that the dispatch-table slot really holds the handler
   address (`native_hud_capture.cpp` ~1506-1550). Each optional boundary (legacy quad, rotated, lines,
   compass) is verified separately and degrades independently; any failure logs once and leaves the
   native path untouched.
2. In the stub: if `cursor` is a selected command -> flush pending tessellation (`RB_EndTessSurface`),
   set the "copying" flag, call the original, flush again, clear the flag.
3. **Global UI loop**: frontend/menus bypass the per-view dispatcher via an inlined loop. Bracket it with
   asm relays (`native_ui_dispatch_bridge.hpp`): save flags + volatile regs + xmm0-5, call the observer,
   restore, execute the displaced instruction, `jmp` to the continuation. Install through
   `create_preserving_near_jump` when the JIT allocation could be out of rel32 range; NOP leftovers; if
   relay allocation fails keep the native loop. Same technique for the LUI render bridge
   (`native_menu_hook.hpp`).
4. Mid-function asm adapters must preserve every register the native code still needs (a plain C++ detour
   clobbering RCX/RAX crashed Overlord at load; see 02).

### 1.5 Redirecting the real draw into a private texture

At the actual `DrawIndexed` (observer registered with the single owner of that D3D11 vtable entry,
never a second detour): save RT/UAV/DSV/blend/depth/viewports/scissors, reject unsupported state,
retarget to the private RGBA8 target with `offset = bounds - atlas` translated viewport/scissor, issue the
same draw via the original, restore everything (`copy_draw_to`).

**Rejection rules** (all must hold): exactly one viewport; only RT0 bound; no UAVs; no depth/stencil
effect (a bound but disabled DSV is fine); viewport integral, top-left (0,0), 16..8192 px; area <=
8,388,608 (menus) / 16,777,216 (other). Fade quads must cover the whole viewport (all four corners
verified, scissors checked).

**Blend rewriting** (`native_hud_blend.hpp`): the target is a *premultiplied coverage layer*, so edit
only the **alpha** blend factors and leave the RGB operation alone.

| Native blend | Rewritten alpha blend | Notes |
|---|---|---|
| SrcAlpha/One + InvSrcAlpha, ADD | `SrcBlendAlpha=ONE, DestBlendAlpha=INV_SRC_ALPHA, ADD`, enable alpha write | ordinary source-over |
| Additive (Src One/SrcAlpha, Dest One) | alpha factors `ZERO/ONE` | RGB = emitted light, coverage unchanged |
| REV_SUBTRACT (SrcAlpha/InvSrcAlpha) | convert to ADD, alpha `ONE/INV_SRC_ALPHA` | subtractand stored separately, composited later as `max(D*(1-A)-S, 0)` |
| REV_SUBTRACT black unlit quad | flatten to source-over | only after verifying the exact material/shader (`unlit_2d.hlsl`, CRC, `$black`); nonblack reverse-subtract is **rejected** (cannot flatten over arbitrary scenery) |
| Opaque / replace-mode | original RGB with colour mask 7, then a second alpha-only pass with a compiled constant `float4(0,0,0,1)` PS | native replace-mode quads are opaque even if texture alpha is 0 |
| Fade-attenuated text | constant-colour PS fed by a dynamic CB; `E*(1-f)+C*f*A` | fade is captured on its **own** plane so it is never applied twice |

Unit-test the algebra (brute-force factors, assert layered form == native within 1e-12) and reject any
unproven mode instead of approximating it.

### 1.6 Pools, leases, publication, freshness

- Each target kind has 3 `shared_ptr<frame>` slots; a slot is reusable only when `use_count()==1`.
  Consumers hold immutable leases; a `frame` carries generation, sequence, timestamp, context pointer,
  reference generation and an owner/(session, revision) tag.
- Publication replaces the pointer under a mutex. **An empty publication is meaningful** ("previous
  subtitle is gone"). Never fall back to "latest".
- Freshness: 250 ms for menu/narrative/damage/backdrop captures (`damage_screen::current`,
  `menu_backdrop_component.cpp`), 150 ms for pose/input/remote HUD; consumers also require the same
  device generation and context. Every reject reason is a counter + state string
  (`menu_state_stale`, `menu_image_session_mismatch`, ...).
- Premultiplied, **display-encoded** UI is composited in encoded space onto an encoded copy of the eye
  and decoded to linear once (section 2.5). Source formats are reinterpreted `*_SRGB -> *_UNORM` in the
  SRV so hardware does not decode twice.

### 1.7 Pitfalls

- Re-running the UI dispatcher or cloning the tree: native LUI pools are finite and callbacks are
  stateful. Observe, don't replay.
- Lua/UI element pools exhaust (section 4.3): leased private copies must be released before the pause
  menu builds.
- A diagnostic/readback capture mode must be compiled/gated off (`readback_diagnostics=false`): no
  readback or waits in the capture path.
- Capture memory dies on fastfile/asset unload; every retained reference is leased with a retire hook.

## 2. Presenting surfaces

### 2.1 Two valid compositors

1. **Runtime layers** (menus, movie, cursor): one XR quad/cylinder layer (or OpenVR `IVROverlay`) per
   surface. The runtime reprojects them, so UI stays sharp and stable at any frame rate. Menus use this.
2. **In-eye composition** (HUD, damage, narrative, scopes, diagnostics): per eye, after the display
   transform and before the ring target is published, draw captured textures as projected quads into the
   eye image (`eye_composition.hpp`). Used where the element must be depth/FOV-correct or share the
   eye's post-processing look.

`eye_composition` has 14 ordered, atomically-registered layers (RAII `callback_registration`):
`world_equipment, weapon_guides, optics, weapon_display, screen_scope, spatial_hud, diagnostics,
indicators, remote_hud, damage, narrative, interaction_diagnostics, menu_backdrop, recording_frame`.
Rules: callbacks run inside `copy_eye` on the serialized immediate-context owner (no waits, no runtime
IPC, no retained destination targets, restore all D3D state); HUD hiding is evaluated once per pair
(`thread_local` keyed by pair id, re-evaluated at eye 0); a "weapon display" scene plane allows only
screen-like layers; a diagnostic layer must never replace the spatial HUD. Consumers build **both** eyes'
draw lists on eye 0 (verifying each eye's pair id) and eye 1 draws only if eye 0 succeeded ("never half
a pair").

### 2.2 Layout constants (verified in `menu_surface.hpp`)

| Item | Value |
|---|---|
| In-game menu plane | 0.9 m wide, 0.5 m from anchor, flat |
| Older menus (stack) | each pushed back +0.25 m (`layer_gap`), physical size unchanged, max 4 |
| Popup dim layer | 0.05 m in front of its parent (`dark_layer_gap`), never crosses the active menu |
| Theater (frontend/movie) | cylinder, radius = distance = 3 m, width = radius * pi/2 (90-degree arc) |
| Surface slots | 0-3 menus, 4 background/theater, 5 backdrop, 6 cursor |
| Depth animation | `approach_depth`: 2 m/s, dt clamped to 0.1 s, starts from the current depth, no queued animations, drift-free over 1000 push/pop cycles |
| Anchor | head position + **horizontal heading only** at open; if forward projects to < 0.05 use the right axis; recreated on menu session / recenter / device change |
| Pointer ray limits | hit distance in (0, 20) m, UV in [0,1] |

Native canvas -> VR canvas (`ui_canvas.hpp`): fit any source UI into a fixed 1920x1080 target with
**transparent padding** (scale = min); the extent+fit transform travels with each immutable frame;
`source_uv()` inverts a ray hit to native coordinates (1e-5 tolerance), and a hit in the padding is
rejected (no click, no pointer transfer). Honest limitation: the native UI lays itself out for the desktop
aspect; do not claim aspect independence.

**Constant active depth, receding ancestors**: the active layer is always at D; ancestor k sits at
D + k*gap; depth is recomputed from the current menu stack each frame; only the active layer takes input;
a click that misses it must not click through to the parent; a restored parent is non-interactive until it
is back at D and input has been re-armed; retarget interrupted animations, never chain them; page
replacement hides old ink immediately and starts at active depth. Popup **backdrop blur** is a world-space
plane behind the active popup, blurred per eye (radius 14) through the `menu_backdrop` layer; it requires
pair id, device generation, context, 250 ms freshness and matching spatial-frame generation or it skips.

### 2.3 OpenXR layers (`openxr_menu.cpp`)

- One XR swapchain per surface, sized to the captured image, recreated on size/format change. Menu
  surfaces prefer 8-bit sRGB (R8G8B8A8_SRGB, B8G8R8A8_SRGB) falling back to the eye format.
- Require `maxLayerCount >= surface_count + 1` at init. Cylinder layers need
  `XR_KHR_composition_layer_cylinder` (enabled only when present); otherwise quads.
- Cylinder: pose centre = `anchor + forward*(distance - radius)`, radius = distance, central angle =
  `width/radius`, aspect = `width/height`; quad: `{width,height}`. Pose rotation from basis
  `(right, up, -forward)`. Flags: `BLEND_TEXTURE_SOURCE_ALPHA_BIT` for UI, 0 for an opaque movie
  background.
- Back-to-front order: background slot, menus (oldest first), backdrop inserted just before the active
  (top) menu, cursor last. Menu-only frames carry **zero projection layers** and are counted separately
  from world frames in diagnostics.
- Finish acquire->wait->release for each surface swapchain before destroying it; invalidate layers on
  `session != current`, device generation change, or stale (>250 ms) state.
- UI textures are display-encoded and premultiplied; the blit has three colour modes: `linear`,
  `encoded_premultiplied` (un-premultiply -> sRGB decode -> re-premultiply in linear) and
  `encoded_opaque` (movie frames, alpha forced 1). Pause dim multiplies linear colour by `(1-dim)`.

### 2.4 OpenVR overlays (`menu_overlay.cpp`, backup backend) (report-only)

One `IVROverlay` per slot, key `overlord.native.menu.<pid>.<slot>`; flags `IsPremultiplied`,
`NoBackside`; width via `SetOverlayWidthInMeters`; curvature `= width / (2*pi*radius)`; sort order theater
0, menus `20+2*i`, backdrop `19+2*active`, cursor 100; transform via
`SetOverlayTransformAbsolute(TrackingUniverseStanding)` with columns (right, up, -forward, centre). Push
only state differences. A texture the SDK rejected or whose clear was not acknowledged is quarantined
(kept alive until `ClearOverlayTexture` is acknowledged or the runtime is detached). Hit-testing uses
`ComputeOverlayIntersection` (v is measured from the bottom: flip it). `menu_surface::intersect` is the
runtime-independent equivalent (plane quadratic / ray-cylinder with `atan2` arc angle) and is what unit
tests cover.

### 2.5 Compositing a captured HUD onto an eye

- The engine's finalized view-projection excludes camera translation: subtract the **same** eye origin
  once, then multiply (row-vector layout). Z forced to `w*0.5` for HUD (no scene depth test) unless the
  layer asks `preserve_depth`.
- **Freeze world corners once per pair** so both eyes see the same quad; never billboard per eye.
- Do **not** reject vertices with `w < near` for large panels; use hardware clipping of homogeneous
  coordinates or the panel pops on big head turns. `billboard()` validates an orthonormal right/up.
- Composite in **encoded** space: copy the eye target, encode, `fg + bg*(1-fg.a)`, decode to linear
  once. Additive ink is RGB with zero alpha; reverse-subtract borders live on their own texture. You
  cannot sample the RT you write, hence the copy.
- Panel blur backing: 3x3 tent or 5x5 binomial with mip prefilter; radius clamps 8 px (24 px mask mode);
  globally disabled when `vr_disableBlur`.
- State preservation for all of this: record into a deferred context and
  `ExecuteCommandList(list, RestoreContextState=TRUE)`; tag your own command lists with a private-data
  GUID so your own hooks ignore them (see 04/05).

### 2.6 Frontend, movie, loading, pause

- **UI-only frames**: for the frontend and fullscreen video, admit no world pair (discard any prepared
  pair, suspend native acquisition) but keep sampling tracking/input and keep presenting layers. Head
  tracking stays alive, so the user can still look around the theater.
- **Is a video fullscreen?** `native_video_playing = cinematic_started || (named && !(pending &&
  requested != acknowledged))`. `movie_theater(enabled, video, frontend, scene, fresh_menu, menus,
  briefing, native_fullscreen)`: a fresh briefing menu wins; frontend with menus is not a movie; with a
  loaded scene the engine's own fullscreen-cinematic draw gate decides (a monitor playing video in the
  world is *not* fullscreen playback); fullscreen config alone never invents a video.
- Movie frame: copy the swapchain buffer into a 3-slot lease pool (<= 8192x8192, <= 16M px, single
  sample, RGBA/BGRA, immediate context, same device), show opaque on the theater cylinder with no LUI and
  no input; hide on exit.
- **Pause**: dim the world *as the native pause does* (menu `scene_dim = 140/255` when `cl_paused` and the
  pause-menu flag): OpenXR multiplies the eye blit by `(1-dim)`; OpenVR uses
  `FadeGrid` + `FadeToColor(alpha)` (alpha sent only on change). Head tracking, hit-testing and the
  pointer must keep running during pause - measure that Present/UI keep ticking first.
- **Loading**: dark compositor backdrop (SteamVR_LoadLevel pattern) removed only after the first accepted
  new stereo pair; never admit a stale gameplay pair.
- Death/loading quote text is captured like any other producer (`CGDeadQuote` ownerdraw range, own
  evidence per producer; coverage of every failure screen is *not* claimed).

## 3. Menu input (state machines)

All VR menu consumers share five rules (they are the single most repeated invariant in the repo):
1. A control that was already down when the context opened/changed is **ignored** until a neutral sample.
2. Then a **fresh press** (counted through the runtime's `presses` counter so a tap between two samples is
   not lost).
3. Re-arm after: focus loss/return, reference (recenter) generation, continuity generation, binding
   generation, target/context id change, stale sample (> 150 ms: `now - sampled_at`), sampling hitch,
   tracking loss. A 660 ms gap must not turn a short press into a long press.
4. Duplicate frames (`sequence` not advanced) never re-fire.
5. Ownership moves between hands only on a deliberate fresh trigger edge.

```
pointer_owner:  owner = right hand initially
   each fresh trigger edge on the OTHER hand (and none on the owner) -> owner = other
   hit/miss never transfers ownership; simultaneous edges keep owner; tracking loss cancels

navigation (per target/hand): NOT armed until |stick|<0.25 on both axes and no buttons
   -> armed: click (trigger edge, needs a valid ray hit), confirm (primary), back (secondary)
   stick direction needs max(|x|,|y|) >= 0.65; first step 350 ms repeat delay, then every 100 ms
   reset() on target/focus/hand/generation change

menu_button (left primary, "menu_recenter" action):
   press->release in < 1 s  => toggle pause/back, fired ON RELEASE
   held >= 1 s              => recenter (once), release afterwards does NOT toggle
   armed only after the button was seen up in the current context/generation

held-trigger accept ("no clickable target" screens: briefings, entry screens):
   held trigger -> native Enter delivered on the UI owner thread, per-hand neutral arming
   (menu_acceptance.hpp); native hold-to-skip timers stay native and accept a 1.6 s hold
```

Pointing: ray = the controller **aim** pose (origin, `-Z` column of orientation); intersect the active
surface; UV -> `ui_canvas::source_uv` (letterbox) -> native cursor coordinates
`lround(u*(w-1))`, `lround(v*(h-1))`. Pointer is "ready" only if the active surface is at its target
depth, interactive and input-focused. Input focus loss keeps the menu **visible** but disables input.

### 3.1 Injecting into the engine's input path

Inject at the engine's **event functions** (`CL_KeyEvent`, `CL_MouseMove`, LUI key event) so every
downstream consumer (UI scripts, key catchers, bindings) behaves natively
(`component/input.cpp`, `component/native_ui_pointer.hpp`):

- `vr_ui_key(key, down)` tracks `vr_owned[256]`. Refuse a VR press if the *physical* key is already held;
  otherwise mark ownership and call the **original** handler directly (bypassing your own stub so VR
  presses don't count as physical activity). A physical event clears VR ownership ("physical wins").
  Release only keys still VR-owned; release everything on focus loss/menu close.
- `vr_ui_pointer(x, y)` calls the original mouse-move with menu-space coordinates and never moves the
  Windows cursor.
- **Native-mouse arbitration**: the native UI polls the desktop cursor every frame even when nothing
  moved. Keep (a) a native baseline, (b) a VR-ownership flag, (c) restore-after-click. If the polled
  native position *changed*, ownership returns to the physical mouse; if not, forward nothing. A physical
  click restores the desktop coordinates before delivering the edge. 2,700 identical polls (30 s @ 90 Hz)
  must not steal VR ownership; a 1 px real move must reclaim; extreme coordinates must not overflow.
  A deliberate pointer motion > 0.025 UV or any controller action returns the UI to VR mode.
- Menu root events can also be delivered by taking the UI critical section and calling the root's
  `processEvent({name=..., dispatchChildren=true, ...})` (`ui_scripting/` bridge).
- **LUI observation** (`native_menu.cpp`) (report-only): read the Lua tables (`LUI.roots.UIRoot0/UIRootFull`,
  `menuInfoStack`, `m_blockButtonInput`, `m_blockMouseMove`, `persistentBackground`, `mouse_cursor`)
  from raw memory with guarded reads each tick, hold nothing across calls; property lookup walks the hash
  part and `__index` metatables (depth <= 6). Stack visibility: only popup entries keep parents visible.
  Session/revision counters bump on VM change, frontend toggle, first menu, any menu id/element change;
  published images are tagged `(session, revision)` and rejected on mismatch. `reset_vm()` runs before
  the native Lua shutdown so no VM objects survive `vid_restart`.
- Diagnostics: bounded CSV traces (512 rows/window, 100 ms interval, 30 s windows at startup and after
  >= 500 ms unavailability), opt-in (`menu_input_trace.hpp`).

## 4. HUD strategies

Ordered from least to most work. Choose per element.

1. **Hide it**: `vr_hideHud` hides the native HUD and the spatial HUD while preserving menus/subtitles.
   Gate the **readers**, not the dvar (`presentation_options.cpp`): verify all 8 `cg_drawHUD` read sites
   by an exact 11-byte masked pattern (`mov rax,[rip+disp32]; cmp byte [rax+0x10],0`, displacement
   recomputed per site), prepare every relay first, then patch all-or-nothing with near-jump relays + NOP;
   each relay replaces only the displaced `mov rax,...` with a conditional load (flags saved with
   `pushfq/popfq`, no calls, no stack realignment) so the native test/branch remains. Publish
   `atomic_bool hide_hud/disable_blur` from the main scheduler so renderers never read native dvars. The
   dvar is untouched (saved game settings unchanged). Failure keeps native behaviour.
2. **Capture -> head-locked or lazy-follow quad** at a comfortable distance (1.2-2 m, <= 60-80 degree
   canvas). Fastest way to a readable HUD.
3. **Re-home elements into the world** (best): ammo on the gun, compass/objectives on the wrist, hit
   direction as world-anchored cues, waypoints as world billboards.
4. **Crosshair**: off in VR (aim comes from the gun/hand).

| Element | Overlord technique |
|---|---|
| Ammo | LUI adapter builds private native ammo widgets per hand (4.3), captured and drawn as a camera-facing billboard at the weapon (`gameplay/weapon_hud_*`). Shown while the weapon-HUD button is toggled; chamber prompt blinks on a 500 ms phase |
| Damage / blood / low health | Capture native blood draws (materials `h1_fullscreen_lit_bloodsplat_01`, `overlay_low_health(_alt)`, `h1_screen_blood`) preserving RGBA verbatim (R channel = intensity); one **angular canvas at infinity** = union of both eyes' tangent extents so equal view directions sample equal UVs (`damage_screen.hpp`); 250 ms expiry; rejected after device/context/reference change; suppressed in menus/pause |
| Hit-direction, grenade indicators, waypoints | World billboards from the natural centre camera re-projected per eye, 80-degree edge-clamp canvas, tiles 512x256 in a 2048 atlas, <= 32 waypoints (`directional_ui*`, `native_waypoints.cpp`). Hook the native projection to capture world position + screen anchor, the clamp function to detect edge-clamped markers, then re-place the cropped tile in the world |
| Subtitles / announcements / DSM progress | Head-relative canvas spanning **60 degrees** (`width = 2*d*tan(30)`) with source aspect: story/dialogue 2.0 m, announcements 1.6 m, progress 1.2 m (lowered 0.20 of canvas height). Each eye projects the billboard corners through its own VP; fade captured on a separate plane; tutorial borders (reverse-subtract) via the subtractive texture: blur -> `max(bg*(1-a) - s, 0)` -> text. Bounds: 2048 commands/1 MiB, 128 primitives, 8192 px/16 M px, 3 leased textures, 8 backdrop rects (`narrative_ui*`) |
| Remote (UAV/missile) HUD | Classify captured pieces by material allow-list (incl. `_colorblind` variants) into target markers and instruments; instruments on an 80-degree readable canvas (`half_width tan = .839099631`), markers re-projected at optical infinity through the same tangent window as native FOV; lease needs same remote-camera epoch, generation, context, <= 150 ms (`remote_hud*`) |
| Scopes / screens (Javelin CLU, fixed sniper) | Auxiliary scene render + captured native HUD ink composited on a head-anchored plane 2.0 m away subtending 60 degrees (`screen_scope_layout.hpp`); failure policy: black, never an unmasked or stale image |
| Kill/hit feedback (no crosshair) | Lua + GSC pair (`data/cdata/scripts/killfeed/`), dvar `vr_killfeedStyle` (default `mw2019`) |
| World prompts | Label 14 cm above model bounds centre, fixed visual size (48 cm at 2 m for weapon prompts, 28 cm others), explicit inter-run spacing; submitted at the UI scheduling boundary before native streams are sealed; falls back to stock cursor-hint text when the VR variant is unavailable |

### 4.1 Weapon-hand HUD placement details
Mirror for a left-handed gun by working on a **private scratch copy** of the command: mirror the quad X
coordinates about `border.x + border.right` and text via measured width (skip text flagged 0x4000), then
let the native handler draw the scratch; the submitted stream stays immutable.

### 4.3 The LUI per-hand ammo widget adapter (reference pattern for engine-UI-script adaptation)

`data/ui_scripts/vr_gameplay/__init__.lua` (the only VR-specific Lua in the repo; "no extracted game
scripts or decompiler dependency": adapt by *wrapping*, never ship derived game code).

Mechanics worth copying:
- **Gate by dvar**: return immediately unless `vr_weaponHudNativeSource` is set; `enabled()` also needs
  `vr_enable`. The script ships in the normal data folder and is a no-op in flat mode.
- **Hook the definition registry, with late-registration fallback**: wrap `LUI.MenuBuilder.registerDef`
  and, if `definitions.WeaponInfoHudDef` already exists, patch it in place (some sessions register SP HUD
  definitions only when a level loads). Keep wrappers idempotent with flags.
- **Native -> Lua data bridge**: C++ publishes a plain table `vr_weapon_hud` with `active()`, `count()`,
  `source(i)` (returns valid, weapon, loaded, reserve, capacity, maximumReserve, clipType, lowThreshold,
  name, definition, feed) and `commit(i, visible)` (Lua tells native which instance actually rendered, so
  capture knows which sources to composite). Read-only snapshot + acknowledgement; no HUD drawing by C++.
- **Per-hand private copies**: build one copy of the game's own ammo widget per hand through the game's
  factory (`builder.buildItems(original(), {}, slot)`), and run its `init`/`onRefresh` with the module's
  `Game` table temporarily swapped for a proxy (`setmetatable({...}, {__index = nativeGame})` via
  `rawset(environment, "Game", proxy)`, restored in `pcall`). The proxy answers `GetPlayerClipAmmo`,
  `GetPlayerStockAmmo`, weapon name, clip type, low-ammo threshold, `IsWeaponReloading=0`, etc. from the
  C++ snapshot for *that hand's* weapon. Remove the copy's own refresh timer. Native materials, fonts and
  pip geometry stay owned by the game's factory.
- Hide the original widget under a mod-owned zero-alpha container (native `animateToState` overwrites
  alpha on the same element), and force the ammo widget "on" by wrapping `animateToState`
  ("off" -> "on") because the native idle fade is wrong in VR.
- **Resource-pool pitfalls** (all in comments as evidence): the engine UI animation pool is finite, so the
  pause-menu constructor is wrapped to release VR copies first; `close()` only detaches, so cancel running
  animations and clear `transition_complete*` handlers before closing; weak-key tables for owners so a
  previous level's HUD is not retained; `collectgarbage` only on ownership transitions; `pcall` every
  build with a `failed` latch so a bad native build cannot loop.
- **Defensive metatable for off-by-one native data**: native pip graphic allocates `clipMax` pips while
  the closed-bolt +1 chamber legitimately reports clip+1; an `__index` fallback returns the last pip only
  for index capacity+1 and only when the VR chamber dvars are on (other nil indices still surface real bugs).
- **Test**: `tests/vr/weapon_hud_lui_tests.cpp` loads the *shipped* Lua file against a fake LUI host.
  Unit-test UI adapters by running the real script against a mock host.

### 4.4 Engine menu scripting hooks (only for engines with embedded Lua/Scaleform)
- Wrap `LUI.MenuBuilder.m_types_build[name]` once to run callbacks after a menu is built
  (`LUI.onmenuopen`), add buttons to native menus (`LUI.addmenubutton`: also rebuild the scrolling list
  and hint box). Builders take only `(menu, event)`, so pass popup arguments through a shared `stack`
  table. Temporary monkey-patches (e.g. replacing `LUI.Options.InitScrollingList` during one native build)
  must be restored in `pcall`/`error(menu, 0)` so a failed build never leaves a wrapper stacked.
- **Profile-ready boundary**: apply profile-backed settings only when `main_campaign` opens (profile
  loaded), guard with `Engine.InFrontend()`, apply once per *process* (`applied` flag in C++, not the Lua
  VM), run the native commit commands, then **reload the value and verify it** (`error()` if the profile
  did not accept it). Never trust a set; read back after commit. A VM reload or revisiting a submenu must
  not overwrite a later user choice.
- Launcher -> game one-shot requests: command-line flags (`-campaign-unlock 0|1`, consumed once via
  `take()`) and persisted dvars (`vr_subtitles`).
- Sandbox exposed Lua: stub `io/os/debug/coroutine/serialize` and replace with path-restricted wrappers;
  module discovery picks the highest-priority overlay *before* executing any initializer.
- Lua VM lifetime: clear all C++-held references on VM shutdown; `Engine.InFrontend()` guards scripts that
  belong to one context.

## 5. Prompt rewriting: by key, at the lookup function

Problem: the game says "Press [F] to use"; VR needs per-controller, translated text.

**Do not edit string tables and do not match displayed prose.** Intercept at the function that resolves a
string *key* and key the replacement by the native identifier.

- **Producers** (the registry is keyed by `(producer, exact native key, map scope, required feature)`):
  - `lui`: wrap `Engine.Localize(key)` in the Lua VM (`component/ui_scripting.cpp` ~237-256). Order:
    product text/branding aliases first, then `vr::hud_prompts::replace(source::lui, key)`, else the native
    function.
  - `script_hud` (report-only): hook the script HUD element draw (`CG_DrawHudElem`), resolve the element's
    label/value index through the native localized config-string table (`index + 0xF5`), then replace. Progress-vs-
    story classification uses complete native labels with `&&1` numeric placeholder matching, rebuilt on
    language change.
  - `world`: hint index in the entity (1..31) resolved against the hint config-string range (`+0xD5`);
    wrap the original ownerdraw call and, if the VR family is incomplete, fall back to the stock text.
- `hud_prompts.hpp` proves at compile time (`static_assert`): no overlapping map scopes, no duplicate
  keys, every semantic key has a complete argument contract. ~150 native keys in H2.
- Replace **only when**: VR is on, map/producer/feature all match (features are queried lazily from the one
  owning module: no VM reads, no cross-module dependencies), and the **exact current locale** is complete.
  Otherwise return nothing so the **original native text and formatter run**. Never mix partial VR labels
  with native cursor hints; never cross-fall into another mod language; no substring matches ("Ctrl",
  "[{+stance}]"); oversized or NUL-embedded keys fail early; un-reviewed interaction chains get no invented
  text. Native hint timing/fade/lifetime stay native. Loading a localized string or precaching it is not
  evidence a prompt is shown.
- **Catalogs** (`component/game_text*`): `enum class key` (74 semantic keys); per-locale `constexpr`
  arrays built with named entries (order irrelevant, duplicates fail compilation); 16 locales; templates
  use `{button}`/`{item}` plus `<em>...</em>` emphasis; a constexpr parser rejects unknown placeholders,
  stray `<`,`>`,`}`, nested/unbalanced `<em>`, > 4096 bytes; the placeholder *set* must equal the
  language-independent `parameters(key)` contract (word order may change). Missing translation = **omit
  the entry** (never copy English: that would mark it present and suppress the native fallthrough).
  Formatter caps: 4096 B, 8 args, 16 runs; no disk I/O, VM calls or mutable cache; do not cache translated
  text (language change applies on the next draw). Runtime locale = the engine's language dvar snapshotted
  once per UI transaction; unknown -> defer to native. Arguments are never interpreted as templates.
- **Controller-family keys**: Index/Knuckles lack X/Y, so some keys exist twice
  (`button_menu_recenter` "X on the left controller" vs `knuckles_button_menu_recenter` "A ..."). The
  runtime publishes a presentation-only atomic (`hud_controller::set_knuckles`) used solely for glyph
  choice; only controller-specific entries follow it.
- Native strings contain tokens you must understand when rewriting: `&&1` positional args, `^1..^7`
  colour codes, `[{+attack}]` bind placeholders, private-use glyph characters for gamepad icons,
  `@KEY` = "localize this key" in Lua text fields. Escape third-party text (`&&`, `@`) before passing it
  as an argument (`string.el`).
- Branding/product text is rewritten only for an explicit key allow-list; never rewrite dialogue,
  credits, URLs or identifiers.
- Script updates cannot rely on new localized keys unless the matching zone ships; use literals in scripts.
- The key list doubles as an inventory of adapters a campaign port ends up needing: pickup/use/resupply
  holds, rappel attach+brake, equipment draw/detonate, night-vision gesture, breach charge, laptop/
  notebook, fixed sniper, snowmobile board/drive/fire, M203 reload, grenade cook, chamber/empty HUD words,
  cinematic skip, knife pull.

## 6. Fullscreen effects: convert per eye, don't blanket-disable

| Effect | What Overlord does |
|---|---|
| **Lens flares** | **Converted**, not disabled (`native_flare*.cpp`). At emit read the engine's already-written 4 vertices (stride 48) and lift them onto the light's camera-depth plane: `world = origin + depth*(fwd - x*tan_x*left + y*tan_y*up)`; invalid data collapses the quad (never NaN). At draw, per eye, bake the absolute eye origin into the row-vector matrices, RAII-swap the live projection block + clear the depth-hack bit, **bump matrix-cache version counters on entry and exit** (except the WORLD0 stamp) so cached GPU constants rebuild. `vr_disableLensFlare` (default false) disables only the VR draw. Directional sun flares and finite-distance flares stay distinct |
| **Menu blur** | Zero its **source**: wrap `CL_GetMenuBlurRadius` to return 0 in VR; the engine still sums the gameplay/HUD/death terms (`sqrt(g^2+h^2+m^2)`) |
| **Damage/injury blur** | Reject only script calls whose VM program counter lies inside `maps/_gameskill::blurview` (range bound at level start, <= 8192 B, exactly 2 args, local player) so story/death blur is never dropped. Never infer injury from health |
| **Death / remaining native blur** | Run per eye *after* the display transform and before HUD composition with a GPU-only source backup (copy raw HDR, run native chain display->raw, copy result back, restore raw via RAII; no readback, no waits). Failure rejects the eye rather than publishing a half-processed one |
| Depth of field / motion blur | Native quality dvars (`r_dof_limit`, `r_mbLimit`; 0 off, 1..3 quality), written to the profile and applied at next start (launcher shows a bool and keeps an existing nonzero level). No hook |
| Blood / low-health | Section 4 (shared angular canvas) |
| Scope overlays / lens shadow | Physical lens quad with eye-box/exit-pupil mask; lens shadow is REV_SUBTRACT so its subtractand is captured on a separate surface |
| Thermal / SSR / post AA | Per-eye isolation and byte-verified contracts: see 05 |
| `vr_disableBlur` | Skips per-eye Gaussian, weapon-HUD blur, hint backdrops and menu blur; gate through the published `atomic_bool` |

Pitfall (cache stamps): after overriding a cached matrix, advance dependants' version stamps both on
entering and leaving the override and never restore old stamps (`native_flare_projection.hpp`).

## 7. Fonts and text

- **Reuse the game's own font** for VR-drawn captions: read the TTF bytes from the engine asset DB
  (`fonts/bank.ttf`, `0 < len <= 4 MiB`, verify name), `AddFontMemResourceEx` to register a private GDI
  face (family `BankGothic Md BT`), hold a `shared_ptr<face>` lease, retire it from the pre-unload hook,
  poll once per second until loaded (`native_caption_font*`).
- **Caption rasteriser** (`overlay_text_texture.cpp`): GDI `DrawTextW` into a DIB, take `max(R,G,B)` as
  coverage (kills ClearType fringes), optional 2 px outline by 5x5 alpha dilation on coverage only, tint
  *after* dilation, immutable RGBA8 upload. Bounds: 128 chars, font 8-128 px, texture <= 4096x256; RTL via
  `GetStringTypeW(CT_CTYPE2)` + `DT_RTLREADING`; invalid UTF-8 rejected; failures cached per
  `(device, text, size, outline, face, tint)`.
- Any text-heavy VR UI needs a **CJK/Cyrillic/Arabic font strategy** or it renders blank boxes
  (the mod ships per-language font zones and patches button text offsets for non-Latin languages).
- Diagnostics text should not depend on game fonts: `spatial_line_label.hpp` is a built-in 14-segment
  stroke font (A-Z, '-', <= 32 chars).

## 8. Recording / streaming guide
A guide frame drawn in **both** eyes showing exactly the desktop spectator crop (border outside a
one-texel guard, exterior dimmed, default 65%, 100 hides it). Transfer the crop between asymmetric eyes in
tangent space (`u' = (src.tl + x*src_w - tgt.tl)/tgt_w`), expanded +-1 px for bilinear; the desktop
capture itself stays clean (`recording_frame*.hpp`). The desktop mirror samples the completed selected
eye (default right eye, FOV 95, range 30-120, half-texel inset).

## 9. Launcher (outside the game process)

Why: settings, preflight, first-run guidance and diagnostics need a real UI, but WebView2/JS must never
enter the hooked game process, and init-only settings must be fixed *before* the game binary loads.

### 9.1 Architecture
- **One executable, two modes** (`src/client/main.cpp`): no `-singleplayer` argument -> launcher mode,
  which never starts game components, graphics hooks or tracking runtimes; with the game argument the
  host loads the game. After approval the launcher `CreateProcessW`s itself with the original args minus
  any stale `-campaign-unlock` plus `-singleplayer -launcher-start -campaign-unlock 0|1`, re-quoting with
  exact CRT rules (`quote_argument`: backslash doubling before quotes/at the end), command length <= 32767.
- UI: React/Vite bundle in `src/launcher-ui`, embedded into the exe as RCDATA resources and served from a
  virtual origin `https://launcher.invalid` by `WebResourceRequested` (`Cache-Control: no-store`,
  `nosniff`). Navigation off that origin is cancelled, new windows swallowed, permissions denied, host
  objects off, devtools/context menu/zoom/script dialogs off. The dev URL flag (`-launcher-dev-url`,
  only `http://127.0.0.1:5173`) is `_DEBUG`-only. CSP: `default-src 'self'; script-src 'self'; connect-src
  'self'; object-src 'none'; base-uri 'none'; frame-src 'none'`.
- Frameless window (custom `WM_NCHITTEST` with 6 px resize border, DPI-aware, 8 invisible resize grips
  that emit `window.control {resize<Edge>}`), dark theme forced. Skippable polish for a port.

### 9.2 Bridge protocol (`src/client/launcher/bridge_protocol.hpp`, `src/launcher-ui/src/bridge.ts`)
- Request `{id, session, method, params}`, reply `{id, session, result}`. Each document load generates a
  `crypto.randomUUID()` session; replies/events from another session are dropped.
- **Limits (verified)**: 64 KiB per message (`max_message_bytes`); nesting <= 16, checked by a
  quote/escape-aware pre-scan **before** JSON parsing; message must be an object with exactly 4 keys;
  `id` integer in (0, INT_MAX]; `session` `[A-Za-z0-9-]{1,64}`; source must be exactly the origin (or
  `origin/...`, so `launcher.invalid.evil` and `file://` are untrusted); `method` must be in a fixed
  **14-name allow-list**: `bootstrap, settings.save, settings.disableRisk, preferences.language,
  languages.scan, languages.save, languages.poll, game.preflight, game.fix, game.launch, window.control,
  window.state, links.open, renderer.ready`. Each handler re-validates its params (exact keys/types).
  `links.open` takes an **id** ("project", "releases", "issues"), never a URL.
- Client side: <= 64 outstanding requests (`launcher.busy`), 30 s timeout (`launcher.timeout`), errors
  cross as localization **keys** not native text, `dispose()` rejects pending with `launcher.closed`,
  missing host -> `launcher.hostRequired`. Long native work (language scan) returns `pending` and the UI
  polls (100 ms) with a generation guard.
- Threading: UI thread owns the window and a 30 ms timer draining a response queue; one **ordered worker
  thread** performs file mutations; COM callbacks capture a `weak_ptr` guard; errors are posted to the
  message loop (never a modal MessageBox inside a COM callback).

### 9.3 Schema-driven settings (single source of truth)
- `src/client/component/vr/settings.hpp` is one constexpr table: `number_setting{name, default, min, max,
  step, display_scale, display_reciprocal}`, `choice_setting` (null-terminated values + launcher label
  keys; `static_assert` that every default is valid and every value labelled), `boolean_setting` with
  `max_enabled_value` (quality-limit dvars: Off=0, 1..3; the launcher shows a bool but **keeps an existing
  nonzero level** so an edit never silently upgrades Low/Medium to High). The same table drives native
  dvar registration, launcher validation (exact key set, finite numbers in range), UI limits/defaults
  JSON and offline tests. `src/client/resources/launcher/settings-schema.json` (71 entries: 36 toggles,
  22 numbers, 13 choices; categories basics/calibration/gameplay/debug/cheats/other/personalization)
  adds `{category, group, label, type, unit, row, visibleWhen: [key, value], inverted}`.
- Display transforms: `displayScale` multiplies; `displayReciprocal` shows percent-of-scale (world scale
  stored as 39.3700787 units/m shows as 100%); HTML step is `"any"` for reciprocal fields; unrelated saves
  must preserve console-set precision. Numeric text is strictly validated (trimmed, <= 64 chars, regex,
  finite, in range: `0x20`, `NaN`, `1e999` can never become a saved number). `inverted` booleans round-trip
  tested: the stored dvar `vr_recoil` (true = recoil on) is shown as the checkbox "No recoil"
  (category `cheats`), so the checkbox is the logical inverse of the stored value.
- **Revision-guarded saves** (`useSettings.ts`): each edit bumps a per-key revision; saves go through a
  promise queue; a save captures the revision map at start and the canonical result only overwrites keys
  whose revision is unchanged; `revert(key, expected, previous, revision)` rolls back only if the field still
  holds the failed value *and* revision. Numeric fields debounce 180 ms and flush on blur; toggles/choices
  save immediately; a save with unchanged values and an existing config skips the native call.
- UI mirrors the console: calibration rows show the exact `seta vr_xxx <value>` command so users can tune
  live. Controller presets edit only fields that are actually displayed (smoke test asserts it).
- **Init-time vs runtime**: anything that decides which hooks install is resolved in the host *before*
  `load_binary` (runtime backend via env `H2V_VR_BACKEND`, controller-pose pipeline, debug-probe selection
  frozen at startup); precedence: direct start + existing env var keeps the override (diagnosis), launcher
  start lets the config win. UI marks "applies at next launch": world scale, pose pipeline, DOF/motion blur,
  subtitles, all debug probes.

### 9.4 Config merge (editing the game's own config)
The launcher edits the game's native saved config (one `set`/`seta name "value"` per line), so launcher and
in-game console share one source of truth (`vr_settings_config.hpp`, `vr_settings.cpp`):
- Tolerant line tokenizer: <= 3 tokens, <= 128 chars each, rejects `;` and `\` (config-injection guard),
  honours `//` and quotes, returns `{}` for anything odd so unknown lines are preserved.
- Replace only **complete recognised assignments**; keep binds, comments, BOM, unrelated cvars byte for
  byte; append `seta name "value"` with CRLF; re-read immediately before the merge; 4 MiB read cap
  (`gcount()` checked); NUL byte => refuse; atomic write; payload cap 4096 B; saving twice is a no-op.
- **Hashed dvar names**: the engine may write names as `0x%08X` hashes; match either form
  (`native_name` via `dvars::generate_hash`), last recognised assignment wins, remove both forms on save.
- Enum values are validated against the choice list (a value like `snap"; quit` is rejected); malformed
  saved values fall back to safe defaults (comfort effects and aim assist stay off).
- One-shot launcher->game requests ride the command line (`-campaign-unlock`), owned by the *process*, not
  the UI VM, consumed on first frontend read, strict parse.

### 9.5 Preflight and risk settings (`preflight_policy.hpp`, `risk_settings_config.hpp`)
Find early which native graphics options break per-eye re-rendering and make the launcher enforce them.

| Check | Severity | Auto-fix |
|---|---|---|
| Supported game exe (size sentinel) / required DLLs exist and start with `MZ` | error | restore from original install |
| OpenXR loader beside the host (when backend=openxr) | error | extract embedded copy (<= 16 MiB, atomic) |
| Config readable, text, <= 4 MiB | error | - |
| `r_ssaaSamples` != 1 (SSAA) | **error** (blocks launch) | write `seta r_ssaaSamples "1"` |
| `r_preloadShadersFrontendAllow`, `r_preloadShaders` (shader preloading; texture-creation crash) | warning | write `"0"` |
| `sm_cacheSunShadow`, `sm_cacheSpotShadows` (enums, not the derived `*Enabled` bools) | warning | write `"Disabled"` |
| `r_fill_texture_memory` (fill remaining VRAM; hurts streaming/other VR software) | warning | write `"0"` |

Severity model: only **errors block**; warnings can be launched through with explicit acknowledgement
**per launch request** (ids `shaders`, `shadows`; newly appearing warnings need fresh acknowledgement;
`launch_allowed`), and there is deliberately no "don't show again". Risk state is tri-valued
(safe/enabled/unknown); unknown is reported, never assumed safe. `game.fix {id}` and
`settings.disableRisk` ("disable all risk settings" without launching) remove existing assignments
(hashed and named, preserving BOM/binds) then append the safe value; repair targets are a fixed allow-list;
the report is recomputed after each fix (fixing the loader revealed a follow-on `game.binary` issue).

Launch flow: UI `game.preflight` -> save dirty settings -> re-run preflight -> (issues: modal table with
Fix buttons) -> (game language unavailable: language page) -> `game.launch {campaignUnlock,
warnings[]}`. `game.launch` is queued on the **same ordered worker** as settings writes, runs a *fresh*
`preflight::check()` and returns `approved`; only then does the UI thread spawn the process. While a launch
is pending all non-window requests are rejected (`launcher.busy`). Escape is ignored while busy.

### 9.6 Everything else worth lifting
- **First-run guide** only when the exe is supported and the profile has no VR assignments (graphics-only
  config is not onboarding); steps: language, device, comfort, assistance, challenge; each step saves only
  its own keys; skip writes base values so the file exists.
- **i18n**: UI stores *keys*, 9 locales + separate `shell-locales.json` / `preflight-locales.json`;
  lookup order locale catalog -> locale shell -> English -> raw key; tests enforce key parity (341 keys)
  across locales. Locale parity is not translation quality. Launcher language and game language are
  separate; game-language scan opens the local store offline only and proves files exist by reading span
  bytes.
- **Honest-status UX**: mandatory notice (beta, saves may break, run `vr_diagnose` and attach a minidump)
  and a banner for Debug vs optimised builds.
- **Support bundle / paths**: launcher prefs in `players2/h2-mod/launcher.json` (<= 4096 B, atomic).
- **Smoke test**: `-launcher-smoke <report.json> [-launcher-probe-size WxH] [-launcher-probe-page ...]`
  opens a hidden WebView2 with an isolated profile in a directory with **no game binaries** (enforced),
  20 s timeout writing `{"ok":false,...}`, runs layout inspections in-page (fits at 1040x720, 760x560,
  640x480; every resize grip is the `elementFromPoint` target; fonts loaded; animations interruptible),
  posts `renderer.ready`, native writes the report atomically. `tools/verify_launcher.py` checks
  `version` equals `version.json`, `settingNames == settings-schema.json` (catches a stale embedded
  bundle), presets subset of controls, and cleans only temp dirs under the output dir.
- **Build**: `tools/build_launcher.py` (file lock, pinned WebView2 SDK, Node >= 22.12, `npm ci` only when
  the lockfile is newer, embeds <= 512 files / 16 MiB into `launcher_assets.rcinc` + a constexpr asset
  table; WebView2 linked statically). Premake prebuild generates UI and native schema from one source.
- Skip for a port: scramble-text effects, parallax home scene, help-article database (75 articles with
  `sources`), auto-update UI (dormant), MOTD/achievements, credits.

## 10. Porting checklist for UI

1. Find where UI draws are recorded (command buffer / display list / render-to-texture). Decide capture
   vs native RTT.
2. Identify surfaces by native ownership (producer + range + fingerprint), not by look.
3. Capture to private premultiplied RGBA8; verify state; rewrite alpha blend only; reject the unprovable.
4. Present menus on runtime layers (quad/cylinder) at 0.5 m/0.9 m, theater 3 m/90 degrees; anchor yaw-only.
5. Pointer = aim ray -> UV -> letterbox inverse -> engine cursor via the *engine's* mouse function; neutral
   arm everything; own-vs-physical arbitration.
6. Movie/loading/pause: UI-only frames, dim as native, tracking never stops.
7. HUD: hide option first, then capture quad, then re-home ammo/indicators; convert (don't disable) blood,
   flares, blur; keep `diag` layers below the real HUD.
8. Prompts by key at the lookup function with exact-locale fallthrough.
9. Launcher as a separate process with an allow-listed bridge, schema-driven settings, preflight with
   risk-settings auto-fix, config merge that tolerates hashed names, smoke test in CI.
10. Evidence: every surface has a `vr_*_status` command with last-reject-reason strings and age.
