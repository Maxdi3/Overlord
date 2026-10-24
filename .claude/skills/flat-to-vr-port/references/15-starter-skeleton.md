# 15 — Starter skeleton for a new VR port (proxy-DLL variant)

A minimal layout to start a new game port in the Overlord style. It is meant to
be grown, not shipped as is. Fill in the `TODO(RE)` items from the
reverse-engineering playbook (03).

```
mygame-vr/
  premake5.lua            # or CMake: x64, C++20/latest, static CRT, /DYNAMICBASE:NO not needed for proxy
  deps/                   # minhook, OpenXR-SDK (loader), gsl, asmjit (optional)
  src/
    dllmain.cpp           # proxy exports (dxgi.dll/version.dll) + bootstrap thread
    loader/component_loader.{hpp,cpp}
    common/hook.{hpp,cpp} hook_validation.{hpp,cpp} native_memory.hpp signature.{hpp,cpp}
    game/
      target.hpp          # build pin: size/hash/sentinel; RVAs (constexpr) + static_asserted structs
      symbols.hpp         # typed native functions: using fn_t = ...; inline auto f = fn<...>(rva)
    component/
      console.cpp         # log + command registry (vr_status, vr_reinit, vr_recenter)
      d3d11.cpp           # device capture, Present/Resize detours, events, generation
      vr/
        runtime_openxr.cpp      # 04
        head_pose_bridge.cpp    # 06
        controller_input.cpp    # 07
        stereo_renderer.cpp     # 05 (engine-specific adapter)
        eye_ring.cpp            # per-eye textures + blit
        camera_hook.cpp         # final camera boundary (06)
        command_hook.cpp        # usercmd injection (07)
        viewmodel_hook.cpp      # skeleton IK + muzzle (08)
        fire_hook.cpp           # 08
        menu_layer.cpp          # 09
        diagnostics.cpp         # 11
  tests/                  # CPU tests for math/policies; mock OpenXR
  data/vr_input/          # OpenVR manifests if supporting the backup
```

## Bootstrap (proxy)

```cpp
BOOL APIENTRY DllMain(HMODULE m, DWORD reason, LPVOID) {
  if (reason == DLL_PROCESS_ATTACH) {
    DisableThreadLibraryCalls(m);
    load_system_proxy_target();                 // forward exports to System32 dxgi.dll
    if (!target::verify_build()) { log("unsupported build; VR disabled"); return TRUE; } // fail closed
    MH_Initialize();
    component_loader::post_start();
    // Install hooks that must precede device creation (D3D11CreateDevice) now;
    // defer game-code hooks until the game finished unpacking (first Present or a known init call).
  }
  return TRUE;
}
```

## Fail-closed hook install pattern

```cpp
struct site { uintptr_t rva; std::span<const uint8_t> bytes, mask; };
bool install(const char* feature, std::initializer_list<site> checks, auto&& do_install) {
  for (auto& s : checks)
    if (!verify_masked(base + s.rva, s.bytes, s.mask)) {
      log("[%s] contract mismatch at +0x%llX; feature disabled", feature, s.rva);
      return false;
    }
  do_install();
  return true;
}
```

## Present-driven frame loop

```cpp
void on_present_pre(const present_event& e) {
  std::scoped_lock lock(xr_mutex);
  if (!maintain_session(e.graphics) || !frame_open) return;
  if (!pair_published(current_pair)) { if (++pending > 4) end_frame_no_world(); return; }
  for (int eye = 0; eye < 2; ++eye) {
    acquire_wait(eye);                                 // xrAcquire/xrWaitSwapchainImage
    blit_state_restoring(ring[current_pair].eye[eye], swap_rtv[eye]);
    release(eye);
  }
  end_frame(projection_layer(sdk_views_of(current_pair)), menu_layers());
  retire(current_pair);
}
void on_present_post(const present_event& e, HRESULT hr) {
  std::scoped_lock lock(xr_mutex);
  if (FAILED(hr) || !session_running) return;
  xrWaitFrame(); xrBeginFrame();
  if (!state.shouldRender) { end_frame_empty(); return; }
  locate_views(); locate_head(); sample_actions();
  head_pose_bridge::publish(head_midpoint);
  controller_input::publish(frame);
  stereo::publish_view_family(++current_pair, eye_offsets, tangents);
  frame_open = true;
}
```

## Camera boundary (example for a Quake-like engine)

```cpp
void set_view_stub(void* a1) {
  // The engine has filled refdef.origin/axis from the player state at this point.
  if (head_pose_bridge::apply_camera(refdef->origin, refdef->axis, cmd_time, view_angles, policy))
    lod_origin = refdef->origin;          // keep LOD consistent for both eyes
  original_set_view(a1);
}
```

## First three days plan

1. Day 1: proxy plus logging, device and Present hooks, OpenXR colored frames (M1–M2).
2. Day 2: find the camera struct and the render-scene call; observe-only census (M4 start).
3. Day 3: derive eye views and render twice (try strategy B in a capture first;
   if it is not re-entrant, plan strategy A); copy to the swapchain; verify parallax.
