# 09 — Menus, HUD, video and 2D presentation

Overlord files: `vr/native_menu.cpp`, `native_menu_hook.hpp`, `native_menu_stack.hpp`,
`native_hud_capture.cpp`, `menu_surface.hpp`, `menu_overlay.cpp` (OpenVR),
`openxr_menu.cpp` (OpenXR layers), `menu_backdrop.cpp`, `spatial_panel_renderer.cpp`,
`native_hud_quad.hpp`, `hud_prompts.cpp`, `narrative_ui_component.cpp`,
`directional_ui_component.cpp`, `damage_screen_component.cpp`,
`native_waypoints.cpp`, `gameplay/weapon_hud_component.cpp` (+ `weapon_hud_*.hpp`), `data/ui_scripts/vr_gameplay`.
Docs: `vr-native-menus.md`, `vr-weapon-hud.md`, `vr-narrative-ui.md`,
`vr-damage-screen.md`, `vr-presentation-options.md`, `game-text-i18n.md`.

## 1. Capture, don't clone

Don't rebuild the game's menus. Instead:
1. Observe the native UI tree roots (main menu, pause menu) on the UI owner
   thread and tag each element's draw command range at its render callback.
   Also cover the cache-hit path that emits cached commands.
2. At the 2D draw dispatch (the per-view HUD dispatcher *and* any inlined global
   2D loop), capture the quad/text/line commands that belong to a surface into an
   offscreen texture. Use fingerprints and bounded lifetimes to reject reused
   command memory.
3. Present that texture as a compositor layer (quad, or cylinder if supported)
   in front of the user, placed from the head pose when the menu opens. Popups
   push their parents backward.

## 2. Pointer input

Ray-cast the `aim` pose against the layer plane/cylinder → UV → the game's
virtual cursor coordinates → feed the native mouse position/click. Hit-testing is
pure math (unit-tested). Stick navigation maps to the native up/down/accept/back.

## 3. Frontend, cinematics and video

- Frontend (main menu) and fullscreen video: submit **UI-only** frames (no world
  layer), as a "movie theater" panel (`menu_surface::movie_theater`).
- Pause: dim the world (blit with a dim factor) and show the menu panel.
- Loading screens: a panel, or a black world with head tracking still active.
- Never put flat 2D content in screen space in the HMD. Everything 2D is a
  world- or head-anchored panel.

## 4. HUD in VR

Options, from simplest to best:
1. Capture the native HUD to a texture and show it as a head-locked or lazy-follow
   quad at a comfortable distance (Overlord: `native_hud_capture`, `remote_hud`).
2. Re-home specific HUD elements: ammo on the gun (`weapon_hud_component.cpp`: shown when
   the weapon HUD button is pressed), compass/objectives on the wrist, hit/damage
   indicators as directional cues (`directional_ui`), waypoints in world space
   (`native_waypoints.cpp`).
3. Option to hide all HUD (`vr_hideHud`).
- Crosshair: off in VR (aim comes from the gun).
- Subtitles/narrative text: a panel below the view, localized
  (`narrative_ui`, `game_text` catalog).
- VR action prompts: replace the native "Press [F]" strings with VR equivalents
  through the localization layer (`game-text-i18n.md`).

## 5. Fullscreen effects

Damage blood overlays, blur, vignettes, flashbangs and scope overlays drawn in
screen space must be converted (e.g. damage as an edge tint per eye) or disabled
(`vr_disableBlur`, damage screen component, `fullscreen_blur_policy.hpp`).

## 6. Launcher (outside the game process)

A separate process (`overlord.exe` with no args) shows a WebView2 + React app
(`src/launcher-ui`). An allow-listed JSON message bridge
(`launcher/bridge_protocol.hpp`: `bootstrap`, `settings.save`, `game.preflight`,
`game.launch`, …) handles settings, a first-run guide, control help, preflight
checks (game files, runtime installed) and launching. Assets are embedded
in the EXE. The game process never loads WebView2.
