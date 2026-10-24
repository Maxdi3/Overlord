---
name: flat-to-vr-port
description: Build a native VR mod (stereo rendering, 6DOF head tracking, tracked hands, physical weapons/interaction, VR UI) for an existing flat-screen PC game, following the proven architecture of Overlord (the MW2 Campaign Remastered VR mod, D3D11 + OpenXR/OpenVR). Use when the user wants to port, "VR-ify", or add VR support to another game; reverse engineer a game's camera/renderer/input for VR; hook a D3D11/DXGI swapchain and submit to OpenXR; render a game per eye; drive a viewmodel with controller IK; inject controller input into a game's command stream; or adapt menus/HUD/cutscenes for VR. Also use when reading or extending the Overlord codebase itself.
---

# Flat-to-VR Port

This skill captures how Overlord (`github.com/Aeka0/Overlord`, fork `Maxdi3/Overlord`)
turns a flat single-player game into a native VR game. It works by hooking the
original executable, not by rewriting it. Use the method and contracts here for
any target game. Use Overlord's code as the worked reference implementation.

> Scope and ethics: only single-player or offline games the user legally owns.
> Never help bypass DRM, anti-cheat, or online integrity checks. Overlord itself
> refuses pirated copies and verifies the executable it hooks. If the target has
> kernel anti-cheat or is online-only, stop and tell the user.

## The one-paragraph model

A host process loads the **unmodified game binary** at its preferred base. It
resolves the game's imports through a component system and installs
byte-verified hooks at known addresses. Those hooks do several things:

1. **Graphics:** they wrap the D3D11 device/swapchain and expose Present-pre/post
   callbacks.
2. **VR runtime frame loop:** OpenXR (or OpenVR) runs *inside* the game's real
   Present: wait/begin frame after Present, submit eye textures before the next
   Present.
3. **Stereo:** the game renders the scene **once per eye through its own
   renderer**. Per-eye camera slots are derived from the engine's center view
   (eye offset plus asymmetric projection from the runtime FOV). The render-thread
   world pass is re-run per eye with isolated per-eye resources. Each eye's
   final image is copied into a texture ring and submitted.
4. **Head tracking:** the tracked head pose is composed into the engine's final
   camera at its view-setup boundary. HMD yaw/pitch are written into the
   player's command view angles so gameplay "looks" where the head looks.
5. **Input:** controller actions are sampled once per frame into an immutable
   snapshot. They are injected into the game's own per-tick command (move axes,
   buttons, angles) *before* it enters command history.
6. **Hands and weapons:** after native animation computes the viewmodel
   skeleton, an IK solver overwrites bones so the gun follows the tracked
   controller and the arms reach it. The game's fire function is wrapped so
   shots originate from the muzzle bone's pose.
7. **UI:** menus/HUD are captured from native 2D draw streams and shown as
   quad/cylinder compositor layers with controller-ray pointing.
8. **Authority stays native:** the game remains the only owner of ammo,
   inventory, damage, scripts and saves. VR systems propose; verified adapters
   commit through native functions.

## How to use this skill

Work in phases. Each phase has an exit criterion you can verify before
moving on. Don't start hands/weapons before stereo + head tracking are solid.

| Phase | Goal | Read |
|---|---|---|
| 0 | Feasibility: engine, API, binary stability, legal and anti-cheat check | `references/00-feasibility-and-strategy.md` |
| 1 | Load the game under your host; hooking infrastructure; component system | `references/02-injection-and-hooking.md` |
| 2 | Reverse engineer the camera, renderer, input, viewmodel and fire path | `references/03-reverse-engineering-playbook.md` |
| 3 | D3D11 interception, then OpenXR session plus a test-color frame loop | `references/04-openxr-runtime.md` |
| 4 | True stereo: per-eye engine rendering, culling, per-eye resources | `references/05-stereo-rendering.md` |
| 5 | Head tracking into camera and aim; recenter; world scale | `references/06-head-tracking-and-camera.md` |
| 6 | Controller actions into the game's command stream; locomotion/turning | `references/07-input-and-commands.md` |
| 7 | Hands IK, weapon pose, muzzle firing, physical mechanics | `references/08-hands-weapons-ik.md` |
| 8 | Menus, HUD, video, comfort, cutscenes, vehicles | `references/09-ui-hud-menus.md`, `references/10-comfort-and-scripted-scenes.md` |
| 9 | Diagnostics, offline tests, headset acceptance, launcher, release | `references/11-diagnostics-testing-release.md` |

Always-relevant references:
- `references/01-architecture.md`: layers, data flow, threads, ownership rules.
- `references/12-overlord-file-map.md`: where each mechanism lives in Overlord,
  with H2 addresses (examples only; never reuse addresses on another game).
- `references/13-porting-checklist.md`: milestone checklist with exit criteria.
- `references/14-math-and-conventions.md`: coordinate conversion, projection
  matrices, quaternion and IK math, world scale.
- `references/15-starter-skeleton.md`: project layout, proxy bootstrap,
  fail-closed install pattern, Present-driven frame loop, first-three-days plan.

## Non-negotiable rules (learned the hard way in Overlord)

1. **Verify before you hook.** Pin the exact executable (size/hash/sentinel bytes).
   Before patching a site, compare its bytes with a mask. If they don't match,
   *skip that feature* and keep the original game behaviour (fail closed). Never
   fall back to "another build's" addresses.
2. **Hook boundaries, don't replay engines.** Wrap the engine's own functions
   (render scene, view setup, skeleton calc, fire, command build). Prefer
   calling the original once per eye inside its real transaction over
   re-implementing rendering or game logic.
3. **One sample, many consumers.** The runtime samples head, hands and buttons once
   per frame into an immutable snapshot with generation counters. Gameplay never
   polls the XR runtime. Recenter, focus loss and reference changes bump
   generations, and consumers reject stale data.
4. **Native state has exactly one writable owner: the game.** VR keeps no
   second copy of ammo, inventory or health. Mechanical VR state (chamber,
   magazine) must reconcile with the native totals, and every mutation goes
   through a verified native call on its owner thread.
5. **Edit commands before history.** Inject into the per-tick command *before*
   it is stored in the command ring. Never mutate on read: prediction replays
   old commands.
6. **Never show the wrong image.** Never submit half an eye pair. Never fill a
   missing eye from the desktop backbuffer or from the other eye. Skip frames
   instead.
7. **Thread discipline.** XR frame calls run on the real Present owner thread.
   Render-thread hooks don't block on the game thread. No locks are held
   across native/GPU work. Diagnostics read published snapshots.
8. **Evidence labels.** Tag every claim as *confirmed* (observed),
   *decision*, or *pending* (needs runtime/HMD validation). CPU/mock tests never
   establish headset acceptance, so record them separately.
9. **Bounded everything.** Fixed-capacity queues, traces and caches. No per-frame
   allocation, scans or file I/O on hot paths.
10. **Comfort defaults.** Real head pitch/roll is always shown. Use snap or smooth
    turning, disable camera shake/bob/DOF/motion blur/vignettes in VR, and keep
    the horizon level.

## Quick-start for a new target (minimum viable VR)

1. Identify the engine/API (D3D11 is easiest; D3D12/Vulkan need a different
   swapchain/runtime binding; see 00).
2. Get a hook framework running: proxy DLL or host-exe loader, MinHook-style
   detours, byte verification and signature scanning (02).
3. Hook `IDXGISwapChain::Present`/`ResizeBuffers` (vtable slots 8/13) and
   `D3D11CreateDevice` to obtain the device (02, 04).
4. Create an OpenXR instance with `XR_KHR_D3D11_enable` on the game's adapter
   LUID, create a session/swapchains, and run wait/begin after Present and
   end before the next Present. **Milestone: a solid color in the headset.**
5. Find the camera/view struct and the scene-render entry (03). Render the
   scene twice with eye-offset cameras and asymmetric projections, then copy
   each result to the eye swapchain. **Milestone: correct stereo, objects at
   correct depth, no swimming.**
6. Compose the HMD pose into the final camera, then feed yaw/pitch into the
   command angles. **Milestone: aiming follows the head, recenter works.**
7. Inject the thumbsticks/buttons. Then do the viewmodel IK and muzzle fire.

Each step's full detail, pitfalls and Overlord code pointers are in the
references. When the user's target differs from Overlord (other engine,
API or threading model), read `00-feasibility-and-strategy.md` first to pick
the right stereo strategy.
