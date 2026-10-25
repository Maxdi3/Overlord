---
name: flat-to-vr-port
description: Build a native VR mod (stereo rendering, 6DOF head tracking, tracked hands, physical weapons/interaction, VR UI) for an existing flat-screen PC game, following the proven architecture of Overlord (the MW2 Campaign Remastered VR mod, D3D11 + OpenXR/OpenVR). Use when the user wants to port, "VR-ify", or add VR support to another game; reverse engineer a game's camera/renderer/input for VR; hook a D3D11/DXGI swapchain and submit to OpenXR; render a game per eye; drive a viewmodel with controller IK; inject controller input into a game's command stream; hook a game's script VM or UI for VR; or adapt menus/HUD/cutscenes for VR. Also use when reading or extending the Overlord codebase itself.
---

# Flat-to-VR Port

This skill captures how Overlord (`github.com/Aeka0/Overlord`, fork `Maxdi3/Overlord`)
turns a flat single-player game into a native VR game by hooking the original executable.
It does not rewrite the game.
It was distilled from a complete read of the repository: 2,227 files, about 359k lines of code,
docs, tests and data. Each reference file leads with the transferable technique, then points at
Overlord's implementation. H2 addresses and offsets appear only as labelled examples; never reuse
them on another game.

> Scope and ethics: only single-player or offline games the user legally owns. Never help
> bypass DRM, anti-cheat, or online integrity checks. Overlord pins and verifies the exact
> executable it hooks and refuses others. Its `arxan.cpp` only *observes*; it defeats nothing.
> If the target has kernel anti-cheat or is online-only, stop and tell the user.

## The model in one paragraph

1. **Host.** A host process loads the **unmodified game binary** at its preferred base. In
   Overlord, a host EXE at `0x160000000` calls `LoadLibrary` on an ASLR-stripped, cached copy of
   the game at `0x140000000`. The host resolves the game's imports through a component system,
   bootstraps its TLS, and installs **byte-verified hooks at pinned addresses**.
2. **Graphics.** The hooks wrap the D3D11 device and swapchain and expose Present-pre/post
   callbacks.
3. **Runtime frame loop.** OpenXR runs inside the game's real Present. Wait, begin and locate run
   after Present; transfer and end-frame run before the next one. OpenVR is a manual backup.
4. **Stereo.** The game renders the scene **once per eye through its own renderer**:
   - Per-eye view slots are derived from the engine's finalized center view, with the eye offset
     and an asymmetric projection from the runtime FOV.
   - The render-thread world pass re-runs per eye: eye 0 on the engine's record, eye 1 on a
     private clone.
   - Per-eye resources are isolated by hooking D3D11 context binders.
   - Each eye's post-processed image goes into a texture ring and is submitted as a pair.
5. **Head.** The tracked pose is composed into the engine's final camera at its view-setup
   boundary. HMD yaw and pitch are also written into the command's view angles at FinishMove, so
   gameplay "looks" where the head looks.
6. **Input.** Controller actions are sampled once per frame into an immutable snapshot. Buttons
   use edge counters and generations. The snapshot is ORed and added into the game's own per-tick
   command *before* the command enters history.
7. **Hands and weapons.** After native animation, an IK solver overwrites viewmodel bones so the
   gun follows the controller. Shots originate from the solved muzzle. Physical mechanics such as
   reloads are pure planners that commit to native ammo by compare-and-swap; the conflicting
   native reload paths are blocked.
8. **UI and scripts.** Menus and HUD are captured from the native 2D command stream into
   compositor layers with ray pointing. Prompts are rewritten by native string key. Cutscenes and
   scripts are classified from native evidence and steered through guarded script-VM hooks.
9. **Authority stays native.** The game owns ammo totals, inventory, damage, scripts and saves.
   VR proposes, and verified adapters commit on the owning engine thread.

## How to use this skill

Work in phases. Each phase has an observable exit criterion
(`references/13-porting-checklist.md`). Don't start hands or weapons before stereo and head
tracking are solid.

| Phase | Goal | Read |
|---|---|---|
| 0 | Feasibility: engine, API, binary identity, legal/anti-cheat gate, stereo strategy | `00-feasibility-and-strategy.md` |
| 1 | Load the game under your host; component system; hook tiers; fail-closed installs | `02-injection-and-hooking.md` |
| 2 | Reverse engineer camera, renderer, input, viewmodel, fire, UI, scripts; observe-only census | `03-reverse-engineering-playbook.md` |
| 3 | D3D11 interception; OpenXR session plus test-color frame loop; OpenVR backup | `04-openxr-runtime.md` |
| 4 | True stereo: per-eye engine execution, culling, per-eye resources, post chain, capacity | `05-stereo-rendering.md` |
| 5 | Head tracking into camera and aim; recenter; world scale; camera policies | `06-head-tracking-and-camera.md` |
| 6 | Controller actions into the command stream; bindings; locomotion/turn/stance | `07-input-and-commands.md` |
| 7 | Hands IK, render binding, muzzle firing, mechanics, arbitration, equipment | `08-hands-weapons-ik.md` |
| 8 | Menus, HUD, video, prompts, launcher; scenes, vehicles, turrets, climbing, comfort | `09-ui-hud-menus.md`, `10-comfort-and-scripted-scenes.md` |
| 9 | Script VM (GSC/Lua/LUI), localization, custom asset packaging | `17-scripting-and-assets.md` |
| 10 | Diagnostics, crash evidence, build pitfalls, packaging, acceptance | `11-diagnostics-testing-release.md` |
| 11 | Test harnesses: mocks, WARP, synthetic layouts, hostile-callee bridges, fuzzing | `16-test-harness-recipes.md` |

These references apply in every phase:
- `01-architecture.md`: layers, authority, identity and generations, threads, shutdown order.
- `12-overlord-file-map.md`: where everything lives in Overlord, plus a labelled H2 hook table.
- `13-porting-checklist.md`: milestones with exit criteria and a pitfall table.
- `14-math-and-conventions.md`: coordinate conversion, projections, heading math, IK, filters.
- `15-starter-skeleton.md`: project layout, proxy and host-exe bootstrap, first-days plan.

All paths above are under `references/`.

## Non-negotiable rules (learned the hard way in Overlord)

1. **Pin, verify, then hook.** Pin the exact executable (size, hash, sentinel). Before patching
   any site, check its bytes with a mask and its call target. If they don't match, skip *that
   feature* and keep native behaviour (fail closed). For ABI-level patch sets, install all or
   none, with rollback. **Never scan-and-patch at runtime.** Pattern scanning is an offline
   RE/codegen aid: Overlord generates compile-time patch tables with an offline disassembly census.
2. **Hook boundaries; don't replay engines.** Wrap the engine's own functions (render owner, view
   finalizer, skeleton calc, fire, command build). Run the original once per eye inside its real
   transaction. Calling a second scene owner or replaying a finalized frame crashed Overlord;
   see the failed-experiment tables in 03 and 05.
3. **One sample, many consumers.** The runtime samples head, hands and buttons once per frame into
   an immutable snapshot with generation counters (recenter, continuity, binding, device, pair).
   Gameplay never polls the XR runtime. Use **per-consumer staleness limits**: 150 ms input/pose,
   100 ms haptics, 200 ms calibration settle, 50 ms turn dt cap, 250 ms captures and damage overlay.
4. **The game owns the totals; VR state commits by compare-and-swap.** VR may hold mechanical
   state such as an escrowed magazine or a chamber partition, but it must conserve rounds. It
   commits only by compare-and-commit against the native cells on the owning thread, and
   reconciles on external mutation. A failed write consumes the input edge and is never retried.
5. **Edit commands before history.** Inject into the per-tick command before it is stored. Never
   clear native keyboard bits. Never mutate on read: prediction replays old commands.
6. **Idempotent per eye and per replay.** Every per-frame consumer must be safe to run for the
   second eye and for a prediction replay. Latch rejections as well as successes per native epoch.
   A duplicate frame is a no-op.
7. **Never show the wrong image.** No half eye pairs, no desktop or other-eye fill, no "latest"
   fallback on cache eviction. Skip or black the frame instead.
8. **Thread discipline.** XR frame calls run on the real Present owner. Render hooks never block
   on the game thread. Hold no locks across native or GPU work. The one exception: join the
   engine's *own* GPU context lock, as Overlord does (100 ms), instead of adding a new one. Don't
   force `ID3D11Multithread` protection. Poll D3D queries on the renderer thread, not at Present.
9. **Arm from neutral.** A control held when a context opens (attach, recenter, resume, focus
   regain, menu close) needs a neutral sample and then a fresh press. Tracking loss is not a release.
10. **Identity over names.** Objects are identified by definition, instance generation and
    revision, plus exact topology for weapon admission (receiver model, bone count, attachments,
    live capacity). Scenes are classified from native evidence, never from names, health or time.
11. **Evidence labels.** Mark claims *confirmed*, *decision* or *pending*. CPU, mock and WARP
    tests never establish headset acceptance, so record those results separately. Labelled H2
    addresses are examples only.
12. **Bounded everything.** Fixed-capacity queues, traces and caches. No per-frame allocation,
    scans or file I/O on hot paths. Diagnostics drop instead of blocking.
13. **Comfort is configurable, physical pitch and roll are sacred.** Always show real head pitch
    and roll. Make turning, bob, blur, DOF and motion blur options. Overlord defaults to smooth turn
    and keeps movement camera bob *on* as a toggle. Convert screen-space effects per eye rather than
    silently dropping them.

## Quick start for a new target (minimum viable VR)

1. Run the 00 feasibility gate. D3D11 is easiest; D3D12 and Vulkan need different bindings.
   Check whether a generic injector already fits (UEVR for Unreal, REFramework for RE Engine).
2. Build the hook layer: proxy DLL or host-exe loader, detours plus rel32 call-site rewrites,
   register-preserving relays, masked-byte verification, a guarded memory reader, and typed native
   entry points (02, 15).
3. Hook `D3D11CreateDevice`/`CreateDXGIFactory1` and swapchain `Present`/`ResizeBuffers`
   (vtable slots 8/13). Promote the device/swapchain to active only after a real Present (04).
4. Create the OpenXR session on the game's adapter (LUID check). Run wait/begin/locate after
   Present and transfer/end before the next one. **Milestone: a solid color in the headset.**
5. Run an observe-only census to find the final camera/view slot and its finalizer, the
   scene-render owner, and the frame's per-view state (03). Derive eye slots, re-run the world
   pass per eye with isolated history resources, and copy each eye to the ring (05).
   **Milestone: correct stereo, no swimming, identical transparency and particles in both eyes.**
6. Compose the HMD pose at the camera boundary and feed yaw/pitch into the command at FinishMove,
   using geometric heading (06). **Milestone: aim follows the head; recenter works after headset
   standby.**
7. Inject sticks and buttons (07). Then add the skeleton IK and muzzle firing (08). Then UI,
   scenes and scripts (09, 10, 17), and diagnostics and tests throughout (11, 16).
