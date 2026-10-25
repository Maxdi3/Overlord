# 12 — Overlord reference map (where everything lives)

Repository: `github.com/Aeka0/Overlord` (fork `Maxdi3/Overlord`). Target: *Call of Duty: Modern
Warfare 2 Campaign Remastered* (`h2_sp64_bnet_ship.exe` / `MW2CR.exe`, IW engine, D3D11), version
"Beta 7" (`version.json`). Roughly 218k lines of C++ in `src/client`, about 160k in `component/vr`
(about 82k of those in `vr/gameplay`). The VR code sits on top of the inherited **H2-Mod** project
(momo5502/alicealys): everything under `src/client/component/*.cpp` and `src/client/game/` that is
not VR-specific is inherited infrastructure, marked **(inherited)** below. A new-game port should
read the VR rows and the *pattern* of the infrastructure rows; it should not copy the inherited
features.

> **Addresses are examples for one H2 build only.** They show the *kind* of boundary to look for. Never reuse them on another game or build.
> **"Launcher" is two unrelated things in this repo.**
> 1. **Desktop launcher** = the settings/preflight/launch process: `src/client/launcher/*`, `src/launcher-ui/*`, `src/client/resources/launcher/*`, `tests/vr/launcher_settings_tests.cpp`, `docs/vr-launcher-settings.md`, `docs/launcher-localization.md`.
> 2. **Rocket launcher** (RPG-7/AT4/Stinger/Javelin) = gameplay adapters `vr/gameplay/launcher_{feed,profile,support,targeting,presenter,runtime,native,aim_bridge}.*`, `native_launcher_fire.*`, `tests/vr/launcher_tests.hpp`, `launcher_aim_bridge_tests.hpp`, `docs/vr-launchers.md`, `docs/vr-javelin-screen.md`, `weapons/{rpg,at4,stinger,javelin}/`.
> 3. Mixed case: `launcher_cheats.*` / `launcher_cheats_policy.hpp` / `launcher_reserve_adapter_tests.py` implement the **desktop launcher's** cheat options (health/notarget/ammo) in the game process, despite the prefix.
>
> **`INJECT_HOST_AS_LIB` is defined** (`src/client/std_include.hpp`). The host executable is linked at `/base:0x160000000` and loads the cached, ASLR-flag-stripped game binary with `LoadLibrary` at its preferred `0x140000000`. The `.payload` section / `loader.load(self, data)` path (BINARY_PAYLOAD_SIZE 0x12000000, section merging) is **compiled out**; `game_module.cpp` instead hooks `GetModuleHandle*`/`GetModuleFileName*` so the game sees the host as its module. Code that tests "is this pointer a game function" (`gsc/script_extension.cpp`) therefore works by address range (`<= BASE_ADDRESS + BINARY_PAYLOAD_SIZE`).

Naming conventions inside `vr/` (use them to navigate):

| Suffix/prefix | Meaning |
|---|---|
| `*_policy.hpp`, `*_feed.hpp`, `*_gesture.hpp`, `*_geometry.hpp` | pure decision code on copied inputs, no engine types, CPU-testable |
| `*_runtime.cpp` | server-pipeline simulation that owns a domain's state and commits through native adapters |
| `*_presenter.cpp`, `*_presentation.*` | render/pose side; derives from committed state, never creates gameplay state |
| `native_*.{hpp,cpp}` | thin adapters to engine functions/hooks (the only files with H2 addresses besides `h2/`) |
| `*_profile.hpp`, `*_poses.hpp`, `weapons/<name>/` | authored, reviewed data (units, SHA-256 of source models in comments) |
| `engine_stereo_*` | native stereo renderer; `*_probe.*` are observe-only, fingerprint-gated, DEBUG/opt-in |
| `vr_*_status` console commands | one per subsystem; write `minidumps/overlord-*.txt` |

---

## 1. Repository layout

| Path | Contents |
|---|---|
| `src/client/` | the host executable and everything loaded into the game process (component system, VR) |
| `src/common/` | exception/minidump evidence (`exception/`) and generic host utilities (`utils/`) shared by game and launcher |
| `src/launcher-ui/` | React/Vite/TypeScript launcher UI hosted by WebView2 (desktop launcher) |
| `src/tlsdll/` | 0x2000-byte `thread_local` payload DLL exporting `get_tls_data()` (TLS donor for the hosted game image) |
| `data/` | shipped data: `cdata/` (loose scripts/UI Lua/audio overlays), `ui_scripts/vr_gameplay` (HUD Lua), `vr_input/` (SteamVR manifests), `zone/` (compiled `.ff`), `zone_source/` + `zonetool/` (asset sources) |
| `assets/` | icon, SteamVR cover art/manifest resources |
| `tools/` | offline Python/C++ tools (build, package, deploy, asset, evidence) |
| `tests/` | standalone test executables and Python/Lua harnesses (no framework) |
| `docs/` | ~135 design/contract/audit docs (`docs/README.md` indexes them) |
| `deps/` | submodules: minhook, asmjit, udis86, openxr (OpenXR-SDK), openvr, imgui, sol2, lua, gsc-tool (`xsk::gsc`), shader-tool, casclib, curl, zlib, GSL, json, rapidjson, stb, WinToast, discord-rpc, libtomcrypt/math |
| `premake5.lua`, `premake/` | build (`vr_tests.lua` declares ~60 test/probe executables; `msbuild_dependencies.lua` makes dependOnly projects real references); Release = `/O1`, `/GL-`, `/LTCG:OFF` (WPO off, see 13) |
| `licenses/`, `THIRD_PARTY_NOTICES.md` | bundled-dependency licence texts |

---

## 2. Host, loader, process

| Path | Role |
|---|---|
| `src/client/main.cpp` | launcher-vs-game mode (`-singleplayer`), import resolver (ExitProcess, GetProcAddress...), binary load, version sentinel check, entry. **Late hook trigger:** components' `post_unpack` runs on the game's first `GetProcAddress(..., "InitializeCriticalSectionEx")`; an exception there is fatal (message box + exit) |
| `src/client/loader/loader.cpp` | PE mapping/`load_library`, base verification before imports/TLS are patched, imports, exception table, TLS |
| `loader/binary_cache.*` | exact-size read of the game exe, ASLR-flag strip, atomic cache under `%APPDATA%/<mod>/bin/`, lease handle (FILE_SHARE_READ) held until mapped; stale in-use file = explicit error |
| `loader/component_loader.*`, `component_interface.hpp` | lifecycle: `post_start`, `post_load`, `post_unpack`, `pre_destroy` (idempotent, state-machine, thread-safe), `load_import`, `is_supported`; static `REGISTER_COMPONENT` |
| `loader/tls.*`, `tls_bootstrap.*`, `src/tlsdll` | TLS donor DLL; patches the TLS index and the current TEB block; validates directory bounds and rolls back permission changes |
| `loader/target_identity.*`, `image_contract.hpp` | hashes (original/loaded SHA-256, PE timestamp, image size), `compatibility_probe_passed`; gates stereo/head-pose bridges |
| `loader/seh.*` | SEH helper |
| `src/client/std_include.cpp/.hpp` | `/base:0x160000000` (lib mode), `BASE_ADDRESS`, `BINARY_PAYLOAD_SIZE`, GPU-preference exports, RNG stubs, PCH |
| `src/client/build_config.hpp`, `product.hpp`, `resource.*` | build-name constants, product/menu rebranding, resource ids (TLS DLL, dvar list, launcher assets, LUI scripts, embedded OpenXR loader) |
| `src/client/campaign_unlock_request.hpp` | one-shot launcher->game flag |
| `src/client/utils/mapents.*` | BSP map-entity text parser |
| `src/common/utils/hook.*`, `hook_validation.*` | MinHook `detour` RAII, asmjit `assembler` helpers, `call/jump/nop/set/inject`, far-jump table (`create_far_jump`), `create_preserving_near_jump`, vtable entry/IAT patch; `verify_masked_bytes`, `validate_executable_target` |
| `common/utils/signature.*` | IDA-style pattern scanner (offline/RE aid; runtime installs use pinned addresses) |
| `common/utils/native_memory.hpp`, `memory.*` | SEH-guarded bounded `read_bytes`, `is_bad_read_ptr`, `allocate_near`, pooled allocator |
| `common/utils/{io,flags,string,thread,nt,properties,startup_callbacks,latest_task_worker,concurrency,compression,binary_resource,image,http}` | atomic file writes, `-flag value` parser, helpers, thread naming/suspend, PE/module wrapper, `%LOCALAPPDATA%/h2-mod`, registration-ordered startup-only callback list, latest-wins worker, `concurrency::container`, zlib/ZIP, embedded resource extraction, stb loader, bounded libcurl |
| `common/utils/{cryptography,smbios,toast,info_string}` | inherited, unrelated to VR |
| `src/common/exception/{minidump,dump_evidence}.*` | bounded custom minidump user stream (ranges, `.pdata/.xdata`, code around PCs), supplement-provider registry, try-only gate; `IsProcessSnapshotCallback` must return S_OK |

Known defects in the inherited utils (do not copy): `nt::library::get_iat_entry` loops forever on an ordinal thunk (`continue` without advancing, `nt.cpp` ~189); `nt::library::get_proc` is missing its `return` for an invalid module; `game_module.cpp` registers one detour object for both `GetModuleHandleExA` and `...ExW`.

---

## 3. `src/client/component/` (top level)

### 3.1 VR-relevant engine integration
| Path | Role |
|---|---|
| `d3d11.cpp/.hpp` | device creation hook, listener/event API, candidate vs promoted device and **device generation**, GPU-queue gate, debug-layer redirect, shader tagging (`SetPrivateData(guid_shader_bytecode)`), query of multithread-protection (only queries it; forcing it did not help), D3D message table |
| `d3d11/swap_chain_hook.*` | production Present/ResizeBuffers hook: dummy swapchain from the game's own factory, HWND/BufferUsage/device-COM-identity filter, epoch tagging, candidate->active only after a non-TEST Present, deferred install/shutdown, shared_mutex lifecycle + recursive callback mutex |
| `d3d11/texture_diagnostics.*` | lock-free flight recorder (256 events, 32 active) via `CreateTexture2D` vtable patch with a device-identity table; minidump user stream `OVTX` |
| `camera.cpp/.hpp` | **final camera boundary** (`set_viewpos_now` call-site hook where HMD composition happens), LOD-origin fixup, paused free camera, Lua camera override |
| `scheduler.cpp/.hpp`, `scheduler_context.hpp` | five pipelines `async, renderer, lui, server, main` bound to native frame functions; `is_executing(pipeline)` assertions; the `R_EndFrame` hook runs VR frame maintenance and the passive frame-ownership hooks (`frame_state_transition`, `frontend_handoff`, `r_end_frame`) |
| `shutdown.cpp` | hooks `Com_Quit_f` wrapper and `Sys_Quit`: `R_SyncRenderThread()` then `component_loader::pre_destroy()` before native `Com_Shutdown` unloads NVAPI |
| `exception.cpp` | crash handler with pre-created emergency dump thread; RIP-redirect recovery to `Com_Error` |
| `arxan.cpp` | **observe/log only**: samples the first 15 bytes of 14 `ntdll!Dbg*` functions once per second; does not defeat protection. H2 `.text` is encrypted on disk |
| `memory.*` | replaces `PMem_Init` to give the engine 2x pools (script memory +16 MB) so stereo and mod assets have headroom |
| `patches.cpp` | keep `cg_fov`/`cg_fovScale` from being rewritten by script, unlock safeArea dvars, fps-cap modes (`com_waitEndFrameMode`), timer resolution, vid_restart fixes |
| `tls.cpp` | patches 66 inlined TLS access sites so a library-loaded EXE finds its TLS slots |
| `game_data.*`, `game_module.*` | supported binary name/size pin, profile folder; module-handle spoofing (active because lib mode) |
| `input.*`, `binding.cpp` | native input chain (console -> gui -> engine) and the VR UI input-ownership API; extended key-binding table |
| `native_ui_pointer.hpp` | three-state VR/native cursor ownership (native baseline, VR flag, restore after click) |
| `dvars.*`, `command.*`, `config.*` | dvar register/set hooks keyed by name hash, console command registry, typed JSON config |
| `fastfiles.*`, `database.*`, `filesystem.*`, `imagefiles.*`, `images.*`, `fonts.*`, `language.*` | custom zone loading and zone-list splicing, asset-pool growth (xmodel pool x2), disk overlay of the engine filesystem, per-fastfile image paks, PNG overrides, extra fonts, extra languages (Czech/Turkish) |
| `game_text.*`, `game_text_catalog.hpp`, `game_text_native.hpp`, `game_text/{catalog_types,template_contract}.hpp`, `game_text/locales/*.hpp` | compile-time VR prompt catalogs (74 keys, 16 locale values), see 17 |
| `ui_scripting.*`, `ui_script_modules.hpp`, `lui.cpp` | LUI/Hks extension API: `Engine.Localize` wrapper, `vr_weapon_hud` bridge, sandbox, overlay-resolving module discovery |
| `scripting.*`, `notifies.*`, `gsc/*` | script-VM lifecycle, opcode redirect/hook table, damage filter chain, builtin table extension, runtime compile, diagnostics, see 17 |
| `movement.cpp` | declarative PMove profile (`setpmoveprofile`) hooking `PM_WalkMove`/`PM_AirMove`, replay-safe edges |
| `gameplay.cpp` | Pmove patches/dvars (elevators, fall damage, ladder push) |
| `sound_output_diagnostics.cpp/.hpp` | opt-in WASAPI snapshot of the game's own audio device, read on the native thread right after init |
| `thread_names.cpp` | names the engine's thread contexts |
| `scene_models.*`, `scene_model_{identity,identity_bridge,culling,record,lighting,lighting_bridge,lighting_policy}.*`, `scene_pose_match.hpp`, `scene_submission_pool.hpp`, `scene_face_partition.hpp` | submit mod-built models through the engine's own scene path; identity registry; cull padding; lighting-cache expansion |
| `scene_rigid_part.*`, `scene_skeletal_model.*`, `scene_surface_indices.hpp` | private immutable rigid XModels over a loaded asset; retained native DObj for continuously posed props |
| `scene_surface_{layout,storage,storage_contract}.*`, `scene_static_surface_{layout,storage}.*` | stereo capacity patches: scene-surface arena 256 -> 512 KiB (242-instruction contract), static transparent lists |
| `branding.cpp`, `loadscreen.*`, `overlay.cpp`, `logger.cpp`, `console*.{cpp,hpp}`, `game_console.*` | version text, load screen, FPS/graph overlay, log redirect, console window/history/bounded output queue, in-game console |
| `gui/*` | ImGui dev overlay: script console, entity list, asset viewers/live editors, debug draw, input-capture ownership (`input_capture.hpp`) |
| `updater.*`, `motd.*` | **disabled for VR builds** (transport unconfigured, `H2VR_ENABLE_LEGACY_MOTD` undefined, no-auto-update marker) so the upstream feed cannot replace VR binaries |

### 3.2 Inherited H2-Mod features (not part of the VR method, skip when porting)
`achievements.*` (local `achievements.bin`), `battle_net.cpp`, `discord.cpp`, `map_patches.cpp` (IW6 light-grid compat), `mapents.cpp`, `pathnodes.cpp`, `menus.cpp`, `mods.*`, `mod_stats.*`, `renderer.cpp` (`r_fullbright`), `sound.cpp`, `motd.*`, `updater.*`. One reusable idiom in them: `_ReturnAddress()` caller discrimination to change behaviour per call site of a shared function (`map_patches.cpp`).

---

## 4. `src/client/game/` (typed engine layer, inherited and extended)

| Path | Role |
|---|---|
| `game.hpp`, `symbols.hpp` | `game::symbol<T>` typed absolute-address wrapper (`WEAK = selectany`), ~250 H2 function/variable addresses. **Unverified inherited table with duplicates** (`Quit`/`Com_Quit_f`, `hks_obj_tolstring`/`hksi_lua_pushlstring`, `AddBaseDrawTextCmd`/`R_AddCmdDrawTextWithCursor`): re-verify every entry a feature depends on |
| `structs.hpp`, `assets.hpp` | H2 structs/enums with `static_assert(sizeof/offsetof)` (the asserts are authoritative, inline offset comments are stale); `assets.hpp` = techset/material/xmodel/weapon/FX/map/vehicle/scriptable structs (7104 lines) |
| `dvars.*`, `dvar_hash.hpp` | hashed-name dvar registry; constexpr hash (`0xB3CB2E29 * (h ^ lower(c))`, seed xor `0x319712C3`) shared with the launcher so it can rewrite the game's saved config offline |
| `xmodel_index_contract.hpp` | XModel pool constants + operand-relocation helper (SIB vs disp32 pitfall) |
| `scripting/*` | refcounted C++ wrappers over VM values, call/notify/field get-set, `safe_execution` (game `_setjmp` + SEH), `stack_isolation` (512 slots); `scripting/lua/*` sol2 sandboxed mod Lua |
| `ui_scripting/*` | typed wrappers over the Havok Script VM used by LUI |

---

## 5. VR infrastructure: `src/client/component/vr/`

### 5.1 Runtime backends, frame loop, device
| Path | Role |
|---|---|
| `vr_component.cpp`, `vr_runtime.hpp` | VR dvars, commands, Present subscriptions, reinit (once per device generation), watchdog thread, shutdown |
| `runtime.cpp`, `runtime_backend.*`, `runtime_failure_history.hpp` | facade over `variant<OpenVR, OpenXR>`; backend selection (`H2V_VR_BACKEND`); deduplicating failure ring |
| `openxr_runtime.*` | session state machine, **Present-owned** frame loop (`begin_prediction` / `complete_native` / `finish_prediction`), teardown order |
| `openxr_d3d11.*`, `openxr_dispatch.*`, `openxr_startup.hpp`, `openxr_layer_policy.*` | LUID/private device/swapchains, `xrGetInstanceProcAddr` dispatch table, runtime selection (explicit `XR_RUNTIME_JSON` wins; auto SteamVR only for a `vrlink` HMD driver), disabling incompatible implicit API layers, integrity-level check |
| `openxr_input.*`, `openxr_menu.*` | action set, per-profile bindings, haptics, profile-refresh retry; menu/frontend/movie compositor layers (one swapchain per surface, quad/cylinder) |
| `openvr_runtime.*`, `steamvr_input.*`, `steamvr_input_diagnostics.*`, `steamvr_pose_adapter.hpp`, `steamvr_runtime.*`, `steamvr_controller_reference.*` | OpenVR backup: Present-owner transaction, same-device direct submit, IVRInput sampling with manifest `data/vr_input`, pose normalisation, environment/IPC preflight, app registration, controller type detection |
| `present_transaction.hpp`, `callback_registration.hpp` | pre/post Present pairing validator, RAII atomic function-pointer slot |
| `scene_compositor.*`, `frame_capture.*`, `texture_blit.*`, `scene_snapshot_cache.hpp` | legacy mono/capture compositor (diagnostic path), 4-slot capture ring + "native direct" atomic pair publication, fullscreen-triangle blit |

### 5.2 Input, tracking, camera
| Path | Role |
|---|---|
| `controller_input.*`, `analog_input.hpp`, `digital_button_gate.hpp`, `input_history.hpp`, `input_observation.hpp` | canonical immutable per-prediction frame (poses, press/release counters, generations), analog->digital latch with hysteresis, armed-after-neutral gates, bounded input diagnostics |
| `controller_calibration.*`, `controller_pose_pipeline.hpp`, `controller_pose_reference.hpp`, `controller_profile_retry.hpp`, `legacy_controller_pose.hpp`, `touch_controller_reference.hpp` | wrist/hand calibration with 200 ms settle, `legacy` vs `standard` pose pipeline (frozen at start), reference-frame normalisation, bounded profile retry (4 x 250 ms) |
| `controller_haptics.*` | one pending pulse per hand, 100 ms expiry |
| `head_pose_bridge.*`, `camera_rig.hpp`, `continuous_view_angles.hpp`, `game_view.hpp`, `roomscale_origin.hpp`, `scripted_camera.hpp`, `scripted_position.hpp`, `scripted_view.hpp` | tracking -> engine coordinates, recenter, camera composition policy, branch-continuous Euler + fused yaw, command-time contribution history (64 entries), room-scale origin compensation, camera policies for scripted/vehicle scenes |
| `body_pose.hpp`, `pose_filter.hpp`, `stabilization.*`, `spatial_math.hpp` | inferred body estimator, head/hand/desktop filters, quaternion/vec helpers (`rigid_delta`) |
| `camera_bob.cpp`, `camera_bob_bridge.hpp` | 7 call sites + 2 scalar-multiply sites through a register-preserving asmjit bridge; `vr_cameraBob` default ON |
| `remote_look.hpp`, `remote_view.hpp`, `remote_hud.cpp`, `remote_hud_policy.hpp` | UAV/remote-missile look and HUD re-projection |
| `settings.hpp` | single constexpr table of every VR setting (names, defaults, ranges) |
| `debug_options.hpp` | debug probes frozen at startup (`call_once`) |

### 5.3 Native stereo renderer (all the per-eye machinery)
| Path | Role |
|---|---|
| `engine_stereo_renderer.*` | every H2 render hook, transactional call-site installer, per-eye orchestration, public status/rejection reasons |
| `engine_stereo_bridge.*`, `engine_stereo_view.*`, `engine_stereo_binding.*` | published view family (IPD, world scale, tangents), pure eye-slot/culling/FX-plane math on byte layouts, lock-free 64-slot claim table binding frontend views to backend records |
| `engine_scene_completion.*`, `engine_scene_job_capture.*`, `engine_scene_resolution.*`, `engine_scene_extent.hpp` | join the engine's own job flags before snapshotting; force scene extent = runtime eye extent through the engine's resize path (requires supersample 1) |
| `engine_stereo_owner_pass.*` | per-pair owner transaction (admission, record clone, per-eye begin/end view, dynamic upload, display transform, ring copy, auxiliary scope view, bootstrap readback proof, quarantine) |
| `engine_stereo_execution.*` | execution census gate: hooks 9 context draw/dispatch/ExecuteCommandList slots, 17-state admission, **production gate** |
| `engine_stereo_eye_resources.*`, `engine_stereo_resource_ops.*`, `engine_stereo_output_merger.*`, `engine_stereo_pointer_cache.hpp` | per-eye history isolation: context binder hooks rewrite RTV/DSV/SRV/UAV to per-eye copies (SSR target 91, AA histories 16..21), resource ops, OM binding tracker |
| `engine_stereo_dynamic_arena.*`, `engine_stereo_dynamic_upload.hpp`, `engine_stereo_light_views.hpp`, `engine_stereo_tessellation*` | dynamic index-origin capture/replay, one upload advance per pair, per-eye dynamic-light descriptor rebasing, shared tessellation constant buffer |
| `engine_stereo_backend_target.*`, `engine_stereo_backend_view.*`, `engine_backend_probe.*`, `engine_view_probe.*`, `engine_stereo_probe.*` | observe-only backend/frontend lifecycle probes, fingerprint-gated |
| `engine_stereo_draw_indexed.*`, `engine_stereo_constant_buffer_probe.*`, `engine_stereo_material_buffer_probe.*`, `engine_stereo_particle_buffer_probe.*`, `engine_stereo_scene_batch_probe.*`, `engine_stereo_ssr_*`, `engine_stereo_effect_timeline.*`, `engine_stereo_gpu_census.*`, `engine_stereo_gpu_timing.*`, `engine_stereo_dxbc_declarations.hpp` | DEBUG/opt-in evidence tools; `gpu_census` is a **report-only one-shot** (not a production gate) |
| `engine_command_stream.*` | bounded walker + FNV hash of the recorded render command stream |
| `native_render_session.*`, `native_render_contract.hpp`, `native_stereo_source.*`, `eye_composition.hpp` | eye-target ring (2 pairs x 2 eyes), admission, display->linear conversion, ordered per-eye composition layers (14), target-registry layout |
| `native_post_aa*.hpp/.cpp`, `native_fullscreen_blur.*`, `fullscreen_blur_*`, `native_display_*.hpp`, `native_thermal.hpp`, `thermal_scene_policy.hpp`, `native_flare*.{hpp,cpp}`, `native_flare_projection.hpp` | per-eye native AA with VR-owned history, per-eye blur reinjection, display ping-pong contract, thermal/SSR contracts, lens flares converted to world-space quads (matrix-cache version stamps) |
| `auxiliary_scene.*`, `optic_*` (in `gameplay/`), `screen_scope_layout.hpp` | third bounded scene view for scopes/screens, scope placement |
| `opaque_mesh_renderer.*`, `world_beam_renderer.*` | depth-tested tint redraw, laser beam/spot quads |

### 5.4 UI / HUD / presentation
| Path | Role |
|---|---|
| `native_menu*.{hpp,cpp}`, `menu_surface.hpp`, `menu_overlay.*`, `menu_backdrop*`, `menu_acceptance.hpp`, `menu_input_trace.hpp`, `ui_canvas.hpp`, `movie_presentation.hpp` | LUI menu observation, command-range tagging, curved/flat panel layout, pointer ownership, OpenVR overlay presenter, held-trigger acceptance, 1920x1080 canvas fit |
| `native_hud_capture.*`, `native_hud_quad.hpp`, `native_hud_blend.hpp`, `native_hud_visibility_bridge.hpp`, `native_waypoints.*`, `native_ui_dispatch_bridge.hpp`, `native_script_image_bridge.hpp` | 2D draw-stream capture to textures, quad decoder, premultiplied blend rewrite, `cg_drawHUD` read override, marker/hint tagging, register-preserving observer relays |
| `narrative_ui*`, `damage_screen*`, `directional_ui*`, `presentation_options.*`, `hud_prompts.*`, `hud_controller.hpp` | subtitles/fades, damage overlay (250 ms currency), hit/grenade indicators, hide-HUD gate, native prompt key registry, controller-family glyph choice |
| `spatial_panel*`, `spatial_lines*`, `spatial_line_label.hpp`, `overlay_text_texture.*`, `native_caption_font*` | projected-quad shaders, debug line overlay with built-in font, GDI caption rasteriser, private GDI face from the game's TTF |
| `desktop_mirror*`, `desktop_stabilization.hpp`, `recording_frame*` | spectator view and streamer guide frame |

### 5.5 Diagnostics (`diagnostics.*`, `diagnostics/`, `region_capture*`)
`diagnostics.cpp/.hpp` (256-entry lock-free trace ring, three watchdog snapshots, stall reports, atomic live trace), `diagnostics/*_status.cpp` (one formatter per subject: session, environment, runtime, frontend, execution, scene, history, effect, owner, census, resource), `support_bundle.*`/`support_archive.*` (`vr_diagnose` ZIP, 8 MiB per file / 24 MiB total), `renderer_evidence.*`, `runtime_code_snapshot.*`, `wait_timing.hpp`, `report_paths.hpp` (see `vr/diagnostics/README.md`); `region_capture*` bounded MPSC perf recorder, binary `H2VREG01`; `vr/h2/{entrypoints,binding}.hpp` typed H2 SP entry points (`server_entity_dobj`, `client_entity_dobj`, `clip_capacity`, `weapon_type`, `weapon_selection_request`...).

---

## 6. VR gameplay: `src/client/component/vr/gameplay/`

Rules: `gameplay/README.md` (gameplay -> snapshots/adapters, never reverse; command mutation before history; arbitration in `hand_interaction`; recipes registered explicitly).

### 6.1 Command injection and locomotion
`controller_component.cpp` (hooks CreateCmd, NormalInput, FinishMove, UpdateViewAngles; `vr_input_status`), `controller_{locomotion,buttons,stance,firing,ads,palm}.hpp`, `grip_edges.hpp`, `ads_alignment.hpp`, `ads_comfort.hpp`, `ads_sprint.{cpp,hpp}` + `ads_sprint_bridge.hpp` (sprint-while-ADS predicate patches), `aim_assist*`, `trigger_discipline*`, `npc_collision*`, `native_action_slots.*`, `nightvision*`.

### 6.2 Hands, arbitration, render binding
| Path | Role |
|---|---|
| `hands/component.cpp` | the hook of the engine skeleton-calc: dispatch by object kind, epoch latching of accept/reject, binding cache, transactional IK |
| `hands/{native_rig,native_schema,rig_builder,rig_cache,skeleton}.hpp` | structural rig admission (roles by unique bone names), binding-identity cache with asset epoch |
| `hands/{pose_solver,pose_math,pose_mirror,pose_library,pose_schema,position_offset,contact}.hpp` | 2-bone arm IK with oriented bend plane, mirroring, authored poses, grip-local wrist offset, hand contact points |
| `hands/attachment_pose.*` | 128-entry ring of committed hand solves for render-time attachments (`for_record` / `for_current`) |
| `hands/service.*`, `status.hpp` | presentation dispatcher and status |
| `hand_interaction/{core,runtime,frame,pose_plan,access,constraints,mechanical_contacts,object_identity}.*` | the arbiter: sessions, candidates, decision/witness model, input history, shared slider |
| `interaction_coordinator.cpp`, `interaction_schedule.hpp`, `interaction_volume.hpp`, `interaction_debug.*`, `carry_interaction.hpp` | fixed batch order and domain dispatch; named phases without engine types |
| `weapon_render_pose_{component.cpp,cache.hpp}`, `weapon_render_pose.hpp`, `weapon_render_owner.hpp`, `weapon_model_anchor.hpp` | bind a solved pose to the exact native render record (object+matrices+epoch+stereo pair identity) |
| `viewmodel_visibility.*`, `rigid_part_visibility.hpp`, `skinned_part_visibility.hpp`, `immutable_surface_cache.hpp` | hide native viewmodel parts via surface mask / immutable metadata clones / skin index-buffer partition |
| `empty_hands_native.*`, `empty_hand_pose.hpp`, `forearm_twist.hpp`, `forearm_mount.hpp`, `arm_{clearance,geometry}.hpp`, `grip_presenter.hpp`, `part_*.hpp`, `palm_contact.hpp`, `knife_*`, `bar_hand_pose.hpp` | hands-only DObj, forearm twist finalisation, arm/part grasp machinery |
| `scripted_arms_*`, `scripted_body.*`, `scripted_camera_reference.*`, `scripted_control.hpp`, `native_scripted_control.hpp` | scripted full-body arm IK, hidden authored arms, authored camera tag read, weapon-permission bit |

### 6.3 Weapons: mechanics, carry, ammo
| Group | Files |
|---|---|
| Fire path | `weapon_interaction.*` (muzzle fire adapter, `fire_stub`/`parameters_stub`/`bullets_stub`), `shot_geometry.hpp`, `weapon_fire_delivery.hpp`, `weapon_recoil*.hpp`, `native_ballistics.*`, `independent_fire_*`, `native_bullet_trace.*`, `native_shield.*`, `shield_*.hpp`, `native_launcher_fire.*`, `muzzle_clearance.hpp`, `native_muzzle_contract.hpp`, `native_shot_history.hpp`, `hei_round*`, `enemy_combat*` |
| Reload boundary | `reload_boundary_component.cpp` (single owner of native fire/consume/reload-start/`PM_BeginWeaponReload` detours), `manual_feed_boundary.hpp`, `native_closed_bolt_*` |
| Mechanical feed families (pure planner + runtime + presenter) | `physical_reload_*` (detachable magazine, profile/rig/gesture/runtime/presenter), `detachable_magazine.hpp`, `closed_bolt.hpp`, `open_bolt.hpp`, `manual_bolt.hpp`, `tube_*`, `cylinder_*`, `break_action_*`, `belt_*`, `lever_*`, `underbarrel_*`, `charging_handle_*`, `handle_catch.hpp`, `rotating_bolt_gesture.hpp`, `receiver_bolt_release.hpp`, `magazine_*`, `pistol_magazine_capture.hpp`, `quick_reload*`, `chamber_cartridge.hpp`, `chambering_guide*`, `spent_case_*`, `ejection_scatter.hpp`, `falling_*`, `drop_presentation.*`, `hinge_travel.hpp`, `hinged_attachment.hpp`, `heartbeat_*` |
| Ammo authority | `native_ammunition.*` (compare-and-commit), `native_ammunition_storage.hpp`, `ammunition_transfer.hpp`, `weapon_clip_ledger.hpp`, `weapon_clip_projection.hpp`, `native_ammo_grant.hpp`, `native_reload_layout.hpp`, `reload_item*` (free-floating magazines/loaders), `weapon_inventory_snapshot.*` (save/restore) |
| Carry/identity | `weapon_carry*.{hpp,cpp}`, `weapon_holding.hpp`, `weapon_identity.hpp`, `weapon_instance_cache.hpp`, `weapon_holsters.hpp`, `holster_presentation.hpp`, `support_*.hpp`, `native_carry*.{hpp,cpp}`, `native_weapon_*.hpp`, `world_pickup_policy.hpp` |
| Catalog | `weapon_registry.cpp/.hpp`, `weapon_registration.hpp`, `weapon_profile*.hpp`, `weapon_profiles.hpp`, `weapon_attachments.hpp`, `weapon_reload_profiles.hpp`, `weapon_mechanics_profiles.*`, `weapon_native_traits.hpp`, `viewmodel_policy.hpp`, `weapon_pose_library.hpp`, `weapon_actions.hpp` (equip-animation substitution), `native_equip_presentation.cpp`, `native_animation_*` |
| Feedback/HUD | `weapon_feedback.*`, `weapon_sound_*.hpp`, `native_weapon_sound*.{hpp,cpp}`, `native_weapon_fx.*`, `native_followed_fx.*`, `native_fx_*` (checkpoint-safe FX names), `weapon_hud*.{hpp,cpp}` |
| Melee/throwables | `melee_*`, `native_melee*.*`, `fist_feedback.hpp`, `special_melee.hpp`, `grenade_*`, `native_grenade*.*`, `native_claymore.*` |

### 6.4 Equipment, world, scenes
| Group | Files |
|---|---|
| Equipment | `special_equipment_*`, `abdominal_interaction.hpp`, `body_equipment.hpp`, `body_{reach,supply}_*.hpp`, `chest_equipment.*`, `equipment_runtime.*`, `signal_flare_*`, `designator_*`, `notebook_*`, `sentry_*`, `javelin_*`, `launcher_*` (rocket), `fixed_sniper*`, `optic_*`, `mounted_turret*` |
| World | `world_interaction*`, `native_use.*`, `scripted_use_proxy.*`, `pickup_visibility.hpp`, `ladder_*`, `free_climb.hpp`, `climb_*.hpp`, `player_life.hpp`, `native_player_life.hpp` |
| Script glue | `script_function_extent.hpp`, `trainer.*`, `official_cheats*`, `exo_punch*`, `launcher_cheats*` (desktop-launcher cheats) |
| `campaign/` | `scripted_sequences.*` (coordinator, per-scene `view`), `scripted_sequence_policy.hpp`, `sequences/*` (airport, breach, cliffhanger, dcemp, ending, estate, favela, gulag, oilrig + angle bridge, rappel, roadkill, sliding, trainer, vehicle; `camera_policies.hpp`, `airport_opening.hpp`), `cliffhanger/*` (physical climb GSC bridge + carrier, pickaxes, C4 prop, snowmobile boarding), `ending/*` (crawl, blade pull, knife throw) |
| `vehicles/` | `native.cpp` (GSC bytecode hooks + builtin interception), `script_contract.hpp` (inspectors), `runtime.*` (state machine, steering), `steering.hpp`, `presentation.cpp`, `hud.*`, `policy.hpp`, `fire_intent.hpp`, `effect.hpp`, `model_pose.hpp` |

### 6.5 Weapon recipe data
`weapons/<name>/` one folder per receiver family (about 47: aa12 acr ak47 at4 aug cheytac de50 designator dragunov fal famas fn2000 g18 javelin l86 m1014 m14ebr m16 m1911 m240 m4 m79 m82 m9 m93r magnum44 mg4 miniuzi model1887 mp5 p90 pp2000 ranger riot_shield rpd rpg scar spas12 special_knives stinger striker tavor tmp ump usp vector wa2000 winchester1200), each with `profile.hpp` (assembly + admission `select()`), `poses.hpp` (idle wrists + finger rotations + equip-rest parts), `reload_{profile,interaction,poses}.hpp`, `mechanics.hpp`, `magazine_{fill,collision,contacts,grasps}.hpp`, `feedback.hpp`, README. Shared: `weapons/families/ar.hpp` (M4/M16/ACR closed-bolt rules), `weapons/attachments/{rifle,precision,heartbeat_poses,underbarrel_poses}.hpp`, `weapons/hand_poses/{edge_handle,left_handle,right_handle,side_pinky_up,magazine,shell_grasp}.hpp`, `weapons/belt_cover_contacts.hpp`, `weapons/README.md` (data contract: which file owns what, units, verification).

---

## 7. Data, scripts, resources

| Path | Role |
|---|---|
| `data/vr_input/{actions,oculus_touch,knuckles,vive_controller}.json` | SteamVR manifest (single `/actions/gameplay` set, boolean grip/trigger with binding-side thresholds, pose `raw`/`tip`, haptics, mandatory localization block) |
| `data/ui_scripts/vr_gameplay/__init__.lua` | HUD adapter (registerDef wrapper, private widget copies) |
| `data/cdata/scripts/*.gsc`, `killfeed/__init__.lua`, `ending/*`, `gulag/*` | loose GSC (killfeed ducking; inherited `replacefunc` patches) and mod Lua |
| `data/cdata/ui_scripts/*`, `src/client/resources/ui_scripts/*` | inherited menu features (achievements, MOTD, settings, wordle) and small VR-adjacent helpers (`campaign_unlock.lua`, `subtitles.lua`, `common.lua`; `updater.lua` dormant) |
| `data/zone_source/*.csv`, `build.txt`, `data/zonetool/*`, `data/zone/*.ff` | asset manifests, JSON dumps/binary inputs, compiled zones (see 17) |
| `src/client/resources/launcher/*` | `settings-schema.json` (71 settings), `help-content.json` (7 recipes, 75 articles), locale/preflight/shell strings, credits |
| `src/client/resources/dvar_list.json`, `json.lua` | dvar name/description table (10802 lines), vendored rxi json for the LUI VM |

---

## 8. Desktop launcher (separate process)

`src/client/launcher/`: `launcher.*` (orchestration), `webview_window.*`/`window.*` (WebView2 host), `bridge_protocol.hpp` (request/response over `window.chrome.webview`, 4 KiB payload bound), `vr_settings*.{hpp,cpp}` (profile merge: re-reads only the shared registry, atomic writer, 4 MiB read bound, readable-name or `0x<hash>` dvar spellings), `preflight*.{hpp,cpp}` + `risk_settings_config.hpp` (risk-settings gate), `game_language*.{hpp,cpp}` (official language-pack presence via CASC reads), `launch_process.*`, `localization.*`, `services.*`. `src/launcher-ui/src/`: `App.tsx`, `bridge.ts`, `model.ts` (schema-driven draft/validation), `useSettings.ts` (optimistic edit/save queue with revision-guarded revert), `Help*.ts(x)` (searchable, spoiler-gated), `i18n.ts`/`shellTranslations.ts` (9 locales), `layoutProbe.ts` (in-WebView renderer self-test), `components/*` (frameless-window controls, schema-driven `controls.tsx`, dialogs, motion). The game process never loads WebView2.

## 9. Tools (offline)

`package_client.py` (release overlay with qualification gates), `deploy_client_pair.py` (atomic dual-config deploy with hashes/backups/rollback), `build_sound_zone.py`, `build_melee_zone.py`, `prepare_melee_assets.py`, `prepare_melee_audio.py`, `generate_melee_poses.py`, `asset_dump.py`, `casc_images.py`/`casc_slice.cpp`, `generate_scene_surface_contract.py` (Capstone whole-image census -> compile-time all-or-none patch table), `read_crash_evidence.py` (parses the minidump user streams), `capture_equipment_visibility.py` (read-only external memory witness, EXE-hash-gated profile), `analyze_region_capture.py`, `build_launcher.py`, `verify_launcher.py`, `build_openxr_loader.*`, `premake_version.lua`.

## 10. Tests (`tests/`)

No framework: each executable prints PASS/FAIL. Header-only suites use `template<class Check> run(check)` fixtures aggregated by a `main`. Production `.cpp` is sometimes **sliced by brace matching and compiled against fakes** (`*_adapter_tests.py`, `background_update_tests.py`, `config_validation_tests.py`). Categories: loader/crash evidence (`tests/loader/`), D3D11/WARP probes (`execution_probe`, `eye_resource_isolation_probe`, `gpu_census_probe`, `resource_ops_probe`, `ssr_*_probe`), mock OpenXR runtime (`mock_loader.cpp` = fake `openxr_loader.dll` with control exports, `mock_smoke.cpp`), no-loader smoke, OpenVR strict-stub lifecycle, input/pose/camera/heading, hand rig/IK/interaction arbitration, weapon profile admission with captured bone fixtures (`*_data.hpp`, SHA-256 in comments), reload/ammunition conservation, campaign policy, launcher settings, text catalogs, Lua HUD against a mock host, asmjit bridge ABI tests (`camera_bob_tests`, `ads_sprint_tests`, `animation_index_bridge_tests`, `native_menu_hook_tests`, `scene_surface_storage_tests`, `model_identity_tests`), `aggregate_publication_tests` + `client_feature_parity_tests.py` (WPO/section audit). Python `test_*binding*.py`/`test_dual_deployment.py`/`test_region_capture_analysis.py` validate manifests, deploy rollback and the capture analyzer.

## 11. Docs

`docs/README.md` indexes everything and defines the labels *proposed / implemented / offline-verified / headset-accepted*. Parent contract: `vr-gameplay-interaction-architecture.md`. Subsystem contracts: `vr-runtime-rendering.md`, `vr-camera-architecture.md`, `vr-native-menus.md`, `game-text-i18n.md`, `scene-surface-storage.md`, `scene-static-surfaces.md`, `scene-model-lighting.md`, `vr-dynamic-light-stereo.md`, `vr-diagnostics-workflow.md`, `vr-launcher-settings.md`, `release-validation.md`, `releasing.md`, `development.md`, `source-provenance.md`. `docs/development.md` links `native-bindings.md`, which is not in the tree (stale link). Per-weapon/per-mission docs (`vr-m1014.md`, `vr-cliffhanger*.md`, ...) show the profile/adapter pattern. Several docs cite local-only evidence that is not in the repo; do not rely on those references.

---

## 12. H2 hook boundaries (examples of the kind of boundary to find)

Image base `0x140000000` (no ASLR; the exe is pinned by size and the sentinel `*(DWORD*)0x140123456 == 0xE465E151`). Struct offsets in the right column are examples of what the adapters read.

| Boundary | H2 address | Installed in | Notes |
|---|---|---|---|
| Version sentinel | `0x140123456` = `0xE465E151` | `main.cpp` | pin the exact build |
| `GetProcAddress("InitializeCriticalSectionEx")` | (import) | `main.cpp` | late `post_unpack` trigger |
| Shader create wrappers PS/VS/DS/HS/CS | `0x140758A80 / 0x140758B30 / 0x140758960 / 0x1407589F0 / 0x140758830` | `d3d11.cpp` | tag shaders; device from `D3D11CreateDevice` |
| `R_EndFrame` (renderer pipeline, VR frame maintenance) | `0x14076D7B0` | `scheduler.cpp` | queries polled here; main frame `0x140417FA0`, `G_RunFrame` `0x1404CB030`, Hks frame `0x140327880` |
| Reserve scene record / `R_RenderScene` | call `0x1403CA20E`/`0x1403CA23E`, fn `0x140779E00`/`0x14077B820` | `engine_stereo_renderer.cpp` | record size `0x8090`, world record type 4 |
| View slot alloc / init / finalize | `0x14076D5D0 / 0x14077E340 / 0x14077F550` | same | slot `0x170`: VP `+0x80`, inverse `+0xC0`, origin `+0x100`, axis `+0x10C`, tan-half `+0x140/+0x144`, reverse-Z near `+0x148` |
| Generate draw surfaces | fn `0x140778E60` (call `0x14077BC83`) | `scene_models.cpp`, renderer | mod models submitted just before it |
| FX camera build | fn `0x140463310` (call `0x14045B98E`) | renderer | FX camera planes at `+0x20` (a header comment says `+0x10`) |
| Backend record / owner (target prepare) / view bind / dispatch | `0x1407A6BE0 / 0x1407A7DA0 / 0x14079F6F0 / 0x14079EAE0` | renderer, `native_hud_capture.cpp` | owner proof range `0x1407A7DA0..0x1407A8306` |
| Backend view copy / depth-hack projection | `0x14078B5D0` / `0x140786D30` | renderer | |
| Target registry | base `0x150FD1700`, stride `0x40`, cap 304 | `native_render_contract.hpp` | `GfxImage*` `+0`, RTV `+8`, DSV `+0x10`; id 4 main scene |
| Camera set-viewpos (call / fn) | `0x1403ACB2D` / `0x1403B07C0` | `camera.cpp` | HMD composition |
| LOD origin global | `0x14EEE0890` | `camera.cpp` | |
| `CL_CreateCmd` / NormalInput / FinishMove | `0x1403D20F0 / 0x1403D13C0 / 0x1403D0300` | `controller_component.cpp` | edit before command history |
| `PM_UpdateViewAngles` | `0x14068D7B0` | same | `delta_angles` at ps `+0x98`, `viewangles` at `+0x108`; command angles `360/2^24` |
| Client view angles / yaw | `0x141E8A65C / 0x141E8A660` | | |
| Key-binding execute / notify-on-command | `0x1403CF1E0` / `0x1403D3A90` | `binding.cpp`, `world_interaction.cpp` | whitelist ids (37 `+gostand`, 73 `+activate`, 83 `+sprint`) |
| Skeleton calc (DObj) | `0x140657D80` | `hands/component.cpp` | DObj: matrices `+0xA8`, epoch `+0xB0`, hidden bones `+0xB8` |
| Skin / submit / viewmodels | `0x14071F190 / 0x140774F10 / 0x14077F690` | `weapon_render_pose_component.cpp` | call sites `0x14071F167 / 0x14071FF74 / 0x140779439` |
| Hidden-bone fill / rigid packet / skin packet | `0x140658AB0 / 0x14071E680 / 0x14071E020` | `viewmodel_visibility.cpp` | |
| Weapon fire / shot params / bullet trace | `0x140518A40 / 0x140518880 / 0x1404CBFE0` | `weapon_interaction.cpp` | `G_FireWeapon` call `0x1404AD1ED`; trace mask `0x280e831` |
| Ammo: fire-PM / consume / reload / `PM_BeginWeaponReload` | `0x140699240 / 0x14069E430 / 0x1406946D0 / 0x140693D00` | `reload_boundary_component.cpp` | earliest reload-begin hook |
| Scene surface arena (frontend) | arena offset `0x62b700`, first frontend `0x14ef1eb80`, stride `0x1039300` | `scene_surface_storage_contract.hpp` | 242 patched instructions |
| Static transparent budget table | `0x140990B78` | `scene_static_surface_layout.hpp` | |
| Model lighting init / reuse guard | `0x14075E230` / `0x14075D032` | `scene_model_lighting.cpp` | |
| XModel index getter | `0x140413D90` | `scene_models.cpp` | hook after its pool-base LEA |
| `Com_Quit_f` wrapper / `Sys_Quit` | `0x1405A52A0 / 0x14064EF10` | `shutdown.cpp` | |
| GSC opcode dispatch | `0x1405C90A5` (continue `0x1405C90B3`) | `notifies.cpp` | `r14` = bytecode pointer; program buffer ptr `0x14B5E0B78` |
| GSC builtin table patches / VM call | `0x1405BC7BC, 0x1405BCB6C, 0x1405BC7C2, 0x1405BCB78, 0x1405CA678`, call `0x1405CA683` | `gsc/script_extension.cpp` | `FUNC_TABLE_SIZE 0x1000` |
| `G_ShutdownGame` / `ClientSpawn` / `SV_CheckLoadLevel` | `0x1404CBAD0 / 0x1404B0710 / 0x1406B2940` | `scripting.cpp` | level-start callbacks |
| Hks start / shutdown / require | `0x140328BE0 / 0x1403203B0 / 0x1402B4DA0` | `ui_scripting.cpp` | `hks::lua_state` `0x1419D83E8` |
| Damage | `0x1404BD2E0` | `notifies.cpp` | filter chain |
| `cg_drawHUD` reads | 8 sites `0x140367A2A ...0x14038AD7E` | `presentation_options.cpp` | `48 8B 05 <rel32> 80 78 10 00` |
| 2D draw dispatch | `0x14079EAE0` (quad `0x14079CA50`, text `0x14079DDC0`) | `native_hud_capture.cpp` | |
| Camera bob | call sites `0x1403ADF80/9D, 0x1406AACC9/E9, 0x1406AAE15/31/63`, MULSS `0x1406AB363, 0x1406AB49A` | `camera_bob.cpp` | 7 + 2 |
| Globals | `cgs 0x141B96B20`, `refdef 0x141BC2500`, `g_entities 0x1452DDDA0` (stride 760), `g_clients 0x1455DA980`, `keyCatchers 0x14203F3C0`, `scr_VmPub 0x14BA9EE40`, `scr_VarGlob 0x14B617C00` | `game/symbols.hpp` | entity limits 3998/3999 sentinels |

Player-state offsets used by many adapters (H2): `commandTime +0x4C`, `pm_flags +0x54`, `e_flags +0x58` (turret/mount ownership `0x3000`; busy gate `0x103000`), `weapon +0x3BC` (low 9 bits), weapon flags `+0x3C0` (bit `0x80` = weapons disabled), owned weapons `+0x2F8` (15 x u32), health `+0x1EC`, action slots `+0x1FA0/+0x1FB0`. `usercmd_s` size 64, buttons ATTACK 1 / SPRINT 2 / MELEE 4 / RELOAD 0x10 / USE_RELOAD 0x20 / PRONE 0x100 / DUCK 0x200 / JUMP 0x400 / ADS 0x800.

When two reports or comments disagree on an offset, trust the `static_assert`/call-chain evidence (e.g. WeaponDef `gunModel` comment says 24, the assert says `0x28`; fuse fields `aiFuseTime`/`fuseTime` are reversed against the call chain).
