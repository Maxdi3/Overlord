# 00 — Feasibility and strategy selection

Decide these before writing code. Record answers in a `docs/target.md` in the
new project.

## 1. Target identification

| Question | How to answer | Why it matters |
|---|---|---|
| Exact executable build | SHA-256, file size, PE timestamp, a sentinel dword | All addresses are build-specific. Overlord hard-fails on size mismatch and checks `*(DWORD*)0x140123456 == 0xE465E151` (`src/client/main.cpp: verify_version`). |
| Bitness, ASLR, base | PE header (`ImageBase`, `DllCharacteristics & DYNAMIC_BASE`) | Fixed-address hooks need a fixed base. Overlord clears `DYNAMIC_BASE` in a cached copy and requires base `0x140000000`. |
| Packing / DRM / integrity | Entropy, section names, Arxan/Denuvo/Steam stub signatures, self-checksumming | Integrity checks may crash on patched code. Overlord's base (H2-Mod) has an `arxan.cpp` component that neutralises checks in the single-player build. Do not defeat DRM to enable piracy; only allow the owned, legitimately launched copy to run modded. |
| Anti-cheat | EAC/BattlEye/Vanguard drivers or services | Kernel anti-cheat means **stop**. Online-only means **stop**. |
| Graphics API | Imports of `d3d11.dll`, `d3d12.dll`, `vulkan-1.dll`, `opengl32.dll`, `d3d9.dll` | Chooses the runtime binding and how hard stereo is. |
| Engine | Strings, module names, asset formats (`.ff`, `.pak`, `.uasset`, `.pck`, `.bsp`) | Choose between a generic injector (UEVR for Unreal, REFramework for RE Engine) and a bespoke mod (Overlord style). |
| Render threading | Does the game have a frontend (game thread, builds draw lists) / backend (render thread, executes) split? | Determines where to publish eye views and where to re-execute per eye. |
| Scripting | GSC/Lua/Papyrus/Blueprint VMs | Cutscenes, scripted cameras, mission logic and UI may live in scripts that you can hook or read. |
| Units | Player eye height / known object sizes | World scale in units per meter (CoD: 39.37 inch units; Unreal: 100 cm; Unity: 1). |
| Projection convention | Look at the projection matrix in a frame capture: reverse-Z? infinite far? row or column vectors? LH or RH? | Every per-eye matrix you write must match exactly. |

## 2. Is a generic tool enough?

- **Unreal Engine 4/5** → try UEVR first (stereo via engine `IStereoRendering`
  emulation). Only go bespoke if you need physical weapons/hands beyond what
  UEVR plugins provide (and then you can still write a UEVR plugin).
- **RE Engine** → REFramework VR.
- **Source / id Tech / CoD-family / custom engines** → bespoke Overlord-style mod.
- **Unity (Mono)** → BepInEx/MelonLoader + Unity XR plugin injection is often
  simpler than native hooking.

## 3. Choose a stereo strategy

Ranked best-quality first. Overlord uses **A**.

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **A. Engine-native per-eye execution** | Publish two eye views from the engine's center view. Re-run the engine's world render pass (backend record) per eye with eye camera slots, isolated targets and history. | True stereo with full engine quality (lighting, particles, post FX). Synchronous, no judder. | Most RE work. Must handle culling union, temporal history, per-eye constant buffers, tessellation, depth-hack viewmodel projections. |
| **B. Call the scene renderer twice** | At the frontend, call `RenderScene(view)` once per eye. | Simpler when the renderer is re-entrant. | Many engines are *not* re-entrant: frame-global state is consumed or finalised by the first call. Overlord tried a "dual-record" variant and **it crashed** because the frame finaliser had invalidated transient state (see the comment in `engine_stereo_renderer.cpp: render_scene_stub`). |
| **C. Alternate-eye rendering (AER)** | Render the left eye on even frames, right on odd; reproject. | Easy and cheap. | Temporal disparity, motion sickness, halves the per-eye rate. Fallback only. |
| **D. Instanced stereo via shader patching** | Patch vertex shaders to output to a texture array with `SV_RenderTargetArrayIndex`. | One pass, fast. | Requires rewriting every shader (DXBC patching) and all screen-space passes. Very hard for D3D11 games with many shaders. |
| **E. Depth-based synthetic stereo** | Reproject the mono frame with the depth buffer. | Trivial. | Not real VR: disocclusion artifacts, wrong specular. Debug only. |

**Rule:** start with B only if a capture proves the render function is
re-entrant per frame. Otherwise go straight to A. Never ship E. Use C only as a
documented fallback.

## 4. API-specific binding notes

- **D3D11** (Overlord): OpenXR `XR_KHR_D3D11_enable`, swapchain images are
  `ID3D11Texture2D`. Copy or blit from the engine's eye targets on the same
  immediate context. Overlord validates adapter LUID and feature level, and
  enables `ID3D11Multithread` protection.
- **D3D12**: `XR_KHR_D3D12_enable` with the game's command queue. You must
  signal/wait fences around the copy; hook `ExecuteCommandLists` and Present.
- **Vulkan**: `XR_KHR_vulkan_enable2`. You must create the instance/device
  through the runtime's requirements, which is hard after the game created them.
  Intercept `vkCreateInstance`/`vkCreateDevice` to add required extensions.
- **D3D9/OpenGL**: use a D3D11 interop device (`WGL_NV_DX_interop2`, or a shared
  D3D9Ex surface) and submit through D3D11. Expect extra copies.

## 5. Runtime choice

Default to **OpenXR** with suggested bindings for Oculus Touch, Valve Index and
HTC Vive. Keep **OpenVR** only as a manually selected backup. Overlord shows that
both can sit behind one `runtime_backend` facade whose backend is fixed for the
process lifetime (no automatic switching after failure).

## 6. Feasibility gate (go/no-go)

Proceed only if all are true:
- [ ] Single-player/offline mode exists; no kernel anti-cheat.
- [ ] One executable build can be pinned (Steam/Battle.net updates will break
      addresses, so plan a version check and update procedure).
- [ ] You can locate (03): the swapchain, the final camera/view struct, the
      scene render entry, the per-tick input command builder, and the viewmodel
      skeleton/transform.
- [ ] A frame capture (RenderDoc/PIX/Nsight) shows a projection matrix you can
      reproduce bit-for-bit for the mono case.
