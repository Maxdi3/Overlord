# 12 — Overlord reference map (where everything lives)

Repository: `github.com/Aeka0/Overlord` (fork `Maxdi3/Overlord`). About 218k
lines of C++ in `src/client`, of which about 160k are in `component/vr` (about 82k in
`vr/gameplay`). Target: *Call of Duty: Modern Warfare 2 Campaign Remastered*
(`h2_sp64_bnet_ship.exe`, IW engine, D3D11), version "Beta 7".

**The addresses below are for that one H2 build only. They are examples of the
kind of boundary to look for. Never reuse them on another game.**

## Startup and infrastructure

| Path | Role |
|---|---|
| `src/client/main.cpp` | Launcher vs game mode, binary loading, import substitution, version check, entry |
| `src/client/loader/loader.cpp` | PE mapping, imports, exception table, TLS |
| `src/client/loader/binary_cache.cpp` | Exact-size read, ASLR flag strip, atomic cache with lock |
| `src/client/loader/component_loader.*`, `component_interface.hpp` | Component lifecycle |
| `src/client/std_include.cpp/.hpp` | `/base:0x160000000`, payload section, `BASE_ADDRESS 0x140000000` |
| `src/common/utils/hook.*`, `hook_validation.*`, `signature.*`, `native_memory.hpp` | Hooking, byte verification, pattern scan, guarded reads |
| `src/client/component/scheduler.cpp` | Frame pipelines (`main`, `renderer`, `async`, …) |
| `src/client/component/d3d11.cpp`, `d3d11/swap_chain_hook.cpp` | Device creation, Present/Resize hooks, generations, GPU queue scope |
| `src/client/component/camera.cpp` | Final camera boundary (HMD composition) |
| `src/client/component/exception.cpp` | Crash dumps |
| `src/client/launcher/*`, `src/launcher-ui/*` | WebView2 launcher process and React UI |
| `src/client/game/*` | Inherited H2-Mod structs/symbols (`game::refdef`, `usercmd_s`, `playerState_s`, `WeaponDef`) |
| `src/client/component/vr/h2/` | Typed VR-specific native entry points |

## VR infrastructure (`src/client/component/vr/`)

| Path | Role |
|---|---|
| `vr_component.cpp` | VR dvars, commands, Present subscriptions, reinit, watchdog |
| `runtime_backend.cpp`, `runtime.cpp`, `vr_runtime.hpp` | Backend facade (OpenXR/OpenVR) |
| `openxr_runtime.cpp` | Session, frame loop (`on_present`/`complete_native`/`on_present_post`/`begin_prediction`) |
| `openxr_dispatch.cpp`, `openxr_d3d11.cpp`, `openxr_startup.hpp` | Loader dispatch, D3D11 binding, runtime selection |
| `openxr_input.cpp`, `steamvr_input.cpp`, `controller_input.*` | Actions → canonical frame |
| `openxr_menu.cpp`, `menu_overlay.cpp`, `native_menu.cpp`, `native_hud_capture.cpp` | UI layers, capture, pointer |
| `openvr_runtime.cpp`, `steamvr_*` | OpenVR backup |
| `head_pose_bridge.*`, `camera_rig.hpp`, `scripted_camera.hpp`, `roomscale_origin.hpp` | Tracking → camera / view angles |
| `engine_stereo_renderer.cpp` | All render hooks; owner pass orchestration |
| `engine_stereo_bridge.cpp`, `engine_stereo_view.cpp`, `engine_stereo_binding.cpp` | View family, eye slots, claims |
| `engine_scene_completion.cpp`, `engine_stereo_owner_pass.cpp`, `engine_stereo_eye_resources.cpp` | Per-eye transactions and isolation |
| `engine_stereo_tessellation.cpp`, `native_post_aa.cpp`, `native_flare.cpp` | Special-case render fixes |
| `native_render_session.cpp`, `texture_blit.cpp` | Eye ring, transfer to the swapchain |
| `desktop_mirror.cpp`, `recording_frame.cpp` | Spectator view |
| `pose_filter.hpp`, `stabilization.cpp`, `spatial_math.hpp` | Math and filtering |
| `diagnostics*`, `region_capture.cpp`, `engine_*_probe.cpp`, `engine_stereo_gpu_census.cpp` | Evidence and probes |

## Gameplay (`src/client/component/vr/gameplay/`)

| Path | Role |
|---|---|
| `controller_component.cpp` | Command hooks: create/normal input/finish move/update view angles |
| `controller_locomotion.hpp`, `controller_buttons.hpp`, `controller_stance.hpp`, `controller_firing.hpp`, `controller_ads.hpp` | Input policies |
| `hands/component.cpp`, `hands/pose_solver.hpp`, `hands/service.cpp` | Skeleton hook, IK, hand presentation |
| `weapon_render_pose_component.cpp`, `viewmodel_visibility.cpp` | Bind solved poses to render records; visibility |
| `weapon_interaction.cpp`, `aim_assist.cpp`, `weapon_recoil.hpp`, `independent_fire_runtime.*` | Muzzle firing, assist, recoil, dual wield |
| `weapon_registry.cpp`, `weapons/<name>/`, `families/` | Weapon recipes |
| `physical_reload_runtime.cpp`, `cylinder_runtime`, `tube_runtime`, `break_action_runtime` | Mechanical feeds |
| `interaction_coordinator.cpp`, `interaction_schedule.hpp`, `hand_interaction/runtime.cpp` | Arbitration |
| `weapon_carry_runtime.cpp`, `carry_interaction.hpp`, `native_carry*.cpp` | Holsters/inventory |
| `world_interaction.cpp`, `native_use.cpp` | Use/pickup |
| `campaign/`, `vehicles/`, `mounted_turret.cpp`, `ladder_runtime.cpp`, `fixed_sniper.cpp` | Scenes |

## H2 hook boundaries (for orientation)

| Boundary | Address |
|---|---|
| Version sentinel | `*(DWORD*)0x140123456 == 0xE465E151` |
| Engine shader creation wrappers | `0x140758A80` (PS), `0x140758B30` (VS), `0x140758960` (DS), `0x1407589F0` (HS), `0x140758830` (CS) |
| R_RenderScene (call site / fn) | `0x1403CA23E` / `0x14077B820` |
| View slot alloc / init / finalize | `0x14076D5D0` / `0x14077E340` / `0x14077F550` |
| Generate draw surfaces | `0x140778E60` (call `0x14077BC83`) |
| FX camera build | `0x140463310` (call `0x14045B98E`) |
| Backend record / target prepare (owner) / view bind / dispatch | `0x1407A6BE0` / `0x1407A7DA0` / `0x14079F6F0` / `0x14079EAE0` |
| Depth-hack projection | `0x140786D30` |
| Camera set-viewpos (call / fn) | `0x1403ACB2D` / `0x1403B07C0` |
| LOD origin global | `0x14EEE0890` |
| CL_CreateCmd / NormalInput / FinishMove | `0x1403D20F0` / `0x1403D13C0` / `0x1403D0300` |
| PM_UpdateViewAngles | `0x14068D7B0` |
| Client view angles / yaw | `0x141E8A65C` / `0x141E8A660` |
| Key-binding execute / notify-on-command | `0x1403CF1E0` / `0x1403D3A90` |
| Skeleton calc (DObj) | `0x140657D80` |
| Skin / submit / viewmodels | `0x14071F190` / `0x140774F10` / `0x14077F690` |
| Weapon fire / shot params / bullet trace | `0x140518A40` / `0x140518880` / `0x1404CBFE0` |

View slot layout (H2): origin `+0x100`, view-projection `+0x80`, inverse `+0xC0`,
reverse-Z near scale `+0x148`. Scene record size and eye-local field ranges are in
`engine_stereo_view.hpp`.
