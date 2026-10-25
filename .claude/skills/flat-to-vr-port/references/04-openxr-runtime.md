# 04 — D3D11 interception and the VR runtime layer (OpenXR primary, OpenVR backup)

Scope: how to get the game's D3D11 device and swapchain, how to run an OpenXR
(or OpenVR) session *inside the game's own Present*, and how finished eye textures
reach the compositor. Stereo rendering itself is `05`; head pose use is `06`;
controller actions are `07`; menu/HUD capture is `09`.

Overlord files (all under `src/client/component/` unless noted): `d3d11.cpp/.hpp`,
`d3d11/swap_chain_hook.cpp`, `d3d11/texture_diagnostics.cpp`, and in `vr/`:
`runtime_backend.cpp`, `openxr_runtime.cpp`, `openxr_dispatch.cpp`, `openxr_d3d11.cpp`,
`openxr_layer_policy.cpp`, `openxr_startup.hpp`, `openxr_menu.cpp`, `texture_blit.cpp`,
`native_render_session.cpp`, `present_transaction.hpp`, `runtime_failure_history.hpp`,
`openvr_runtime.cpp`, `steamvr_runtime.cpp`, `steamvr_input.cpp`, `menu_overlay.cpp`,
`native_conversion_command_list.hpp`, `vr_component.cpp`. Reference docs:
`docs/vr-runtime-rendering.md`, `docs/client-installation.md`.

## 0. The contracts in one screen

1. The mod owns **no frame thread**. Every XR/OpenVR frame call happens on the thread
   that calls the game's `IDXGISwapChain::Present` (the "Present owner"): *pre* work
   (submit/`xrEndFrame`) runs before the real Present, *post* work (`WaitGetPoses` /
   `xrWaitFrame`+`xrBeginFrame`+pose sampling) runs after it.
2. Everything that touches a D3D object is keyed by a **device generation**; a swapchain
   additionally by an **epoch**. Stale generation/epoch is rejected, never "fixed up".
3. A pose family (one predicted time → eye poses + FOVs) has **one identity** (pair id)
   shared by the pose sample, the engine's view family, the ring slot and the submit.
4. A texture handed to the compositor is **owned by it** until a proven retirement
   boundary (`xrEndFrame` returned / next successful `WaitGetPoses`). Unproven → quarantine,
   never reuse.
5. Cleanup never overwrites the **first** failure; teardown is retryable with the object
   graph retained; teardown ordering is explicit.
6. Resize/device-loss callbacks only **record intent**; the next real Present-pre does the work.
7. Never show the wrong image: no half pairs, no desktop-backbuffer substitute, no stale
   pair. Skip the world layer instead (menu layers may still be submitted).
8. Environment (runtime selection, API layers, elevation, SteamVR IPC) is validated
   **before** loading any runtime, and every override is scoped and restored.

---

## 1. D3D11 interception layer

Expose a small event bus so VR code never touches vtables. Overlord's real API
(`d3d11.hpp`):

```cpp
namespace d3d11 {
  struct device_snapshot { ComPtr<ID3D11Device> device; ComPtr<ID3D11DeviceContext> context;
                           D3D_FEATURE_LEVEL feature_level; uint64_t generation;   // 0 == invalid
                           device_creation_status creation; };                     // flags, thread, LUID match...
  struct present_event { IDXGISwapChain* swap_chain; device_snapshot graphics;
                         uint64_t frame_index; time_point timestamp; UINT sync_interval, flags; };
  struct resize_event  { IDXGISwapChain* swap_chain; device_snapshot graphics;
                         UINT buffer_count, width, height; DXGI_FORMAT format; UINT flags; uint64_t resize_index; };
  token subscribe_device_created_early(fn);  // on the D3D11CreateDevice caller, BEFORE the game gets the device
  token subscribe_device_created(fn);        // at promotion (first real Present on that device)
  token subscribe_device_destroying(fn);
  token subscribe_present_pre(fn);           // before the real Present
  token subscribe_present_post(fn /* (event, HRESULT) */);
  token subscribe_resize_before(fn);         // before the real ResizeBuffers
  device_snapshot get_device_snapshot();  graphics_status get_graphics_status();
  unique_lock<recursive_mutex> acquire_gpu_queue_interop(gpu_queue_client = unknown);
  bool is_inside_present_gpu_scope();     bool enable_graphics_hooks();   void shutdown_graphics_hooks();
}
```

### 1.1 Get the device by import substitution, keep the DXGI 1.1 lineage

- Replace `D3D11CreateDevice` and `CreateDXGIFactory` through the import resolver
  (`component::load_import`, `d3d11.cpp` end of file). The game's own call hands you the
  device *before* it can issue a command. No vtable scanning needed for creation.
- `CreateDXGIFactory` → **`CreateDXGIFactory1(riid, pp)` with the same `riid`**
  (`create_dxgi_factory_stub`). Reason: OpenVR's D3D11 `Submit` requires the app's adapter
  to descend from a DXGI 1.1 factory *before device creation*; creating a Factory1 later
  cannot repair an existing device. Cheap insurance even if you only ship OpenXR.
- After the real `D3D11CreateDevice`, **observe only**: `device→IDXGIDevice→GetAdapter→GetParent(IDXGIFactory1)`
  succeeds, adapter LUID equals the requested adapter's (`factory_identity_match`), record
  creation flags (`GetCreationFlags`; OpenVR path rejects `D3D11_CREATE_DEVICE_SINGLETHREADED`),
  creation thread, counters (`dxgi_factory1_create_count`, `d3d11_create_index`).
  The game keeps its own adapter, flags and feature levels.
- **Do not force `ID3D11Multithread::SetMultithreadProtected(TRUE)`.** Overlord only
  *queries* `GetMultithreadProtected()` and records it. Forcing global immediate-context
  serialization changed every game/Present call and still reproduced the same kernel
  (dxgmms2 wait-packet) failure (`d3d11.cpp` ~1823). Put your GPU work on the real
  renderer/Present owner thread and serialise explicit short sections with the gate (§1.6).
- Debug-only (Overlord `#ifdef DEBUG`): OR in `D3D11_CREATE_DEVICE_DEBUG` when a dvar asks; redirect
  `ID3D11InfoQueue::AddMessage` by overwriting its vtable slot to the mod console (1000 ms
  rate limit per message id, drop `GETPRIVATEDATA_MOREDATA` noise caused by your own
  `GetPrivateData` probes, keep 12 distinct warning-or-worse samples, stack capture only
  for severity ≤ ERROR, **no D3D getters / symbol loading / disk I/O inside that callback**);
  tag every shader with `SetPrivateData(guid_shader_bytecode, …)` +
  `WKPDID_D3DDebugObjectName` after the engine's shader-create wrappers so draw-time hooks can
  map a bound shader back to bytecode/name.

### 1.2 Candidate → active device promotion, generations, early event

A D3D11 game may create several devices (probes, adapter enumeration). Do not treat the
last `D3D11CreateDevice` result as "the" device.

```
D3D11CreateDevice stub:   candidate.generation = ++next_device_generation
                          device->SetPrivateData(candidate_generation_guid, generation)   // survives COM re-wrapping
                          candidate_device = candidate;   install texture diagnostics on it
                          fire device_created_early        // caller thread, before the game sees the device
                          (if hooks were requested) swap_chain_hook::install(candidate)
swap-chain Present (§1.4): after a *successful, non-TEST* original Present on a swapchain that passed the filters
                          promote_candidate_device(device):
                              inspect_candidate_device → recover generation via GetPrivateData;
                              reject if generation <= current.generation (stale)
                              fire device_destroying(previous) → re-init GUI on new device
                              current_device = candidate; reset ALL per-generation counters
                              fire device_created
```

- `device_created_early` fires on the creating thread: use it for things that must precede any
  game draw (private data, debug hooks, vtable observers). `device_created` fires only on the
  *confirmed presenting* device: build XR session/swapchains and install context-vtable hooks
  there. (Overlord installs most hooks in `vr_component.cpp::on_device_created`; runtime
  **init itself is still deferred to Present-pre.** Note: the repo currently has no in-tree
  subscriber of the early event — the texture-diagnostics install is done directly in the
  stub — so treat it as an available extension point.)
- Hooks are *requested*, then installed when a device exists (`enable_graphics_hooks()` records
  `graphics_hooks_requested`, returns false "waiting for a D3D11 device"); the create stub
  installs the swapchain hook later. VR can therefore be toggled after game start without
  ordering assumptions.
- Present/resize recorders ignore results whose `generation` ≠ current.

### 1.3 Listener dispatch contract

Listeners live in a token→entry map under `graphics_mutex`. Dispatch **copies matching
callbacks into a vector under the lock, releases it, then invokes each** via `invoke_listener`,
which sets a `thread_local inside_graphics_callback`, catches every exception into
`graphics_status.last_error`, and afterwards runs any deferred shutdown. This avoids
(a) listener unsubscribing during dispatch, (b) exceptions escaping into the game's Present,
(c) holding a lock while calling VR code.

### 1.4 The swap-chain hook (`d3d11/swap_chain_hook.cpp`) — much more than "hook slots 8/13"

`Present` (vtable slot **8**) and `ResizeBuffers` (slot **13**) are shared by *every* swapchain
in the process (game, overlays, launcher WebViews, the runtime's own windows).

| Concern | What Overlord does |
|---|---|
| Discovery | From the game's own device: `IDXGIDevice→GetAdapter→GetParent(IDXGIFactory)`; make a 1×1 `WS_POPUP` `STATIC` window + dummy windowed swapchain (`R8G8B8A8_UNORM`, 1 buffer, DISCARD); read `vtable[8]`,`[13]`; validate both pointers executable (`hook_validation::validate_executable_target`); destroy the dummy. Using the game's device/factory means the discovered implementation is the real one. |
| Is this swapchain ours? | `desc.OutputWindow == game HWND`, `BufferUsage & RENDER_TARGET_OUTPUT`, `GetDevice()` has the same **COM identity** as the tracked device (`QueryInterface(IUnknown)` on both, compare pointers — `same_identity`). Otherwise `rejected` → pass through untouched. |
| Three-way classification | `rejected` / `candidate` / `active`. A candidate becomes active only **after the original Present returned success and `!(flags & DXGI_PRESENT_TEST)`**. `DXGI_PRESENT_TEST` on the active chain is forwarded with no callbacks. |
| Epochs | Each promoted swapchain gets `++next_swap_chain_epoch` stored with `SetPrivateData(GUID, uint64)`. An epoch ≤ the active epoch is a stale predecessor → rejected; newer → candidate. Combined with the device generation this stops use of resources from a dead device/swapchain. |
| Present stub order | bump frame index → `record_present_begin` → **`notify_present_pre`** → original Present (inside a `wait_timing` scope) → on failure record `GetDeviceRemovedReason()` → `record_present_result` → **`notify_present_post(event, hr)`**. |
| ResizeBuffers stub | acts only for the active chain (same pointer, generation, epoch). Calls **`notify_resize_before` *before* the original** so VR code can drop every backbuffer reference (`ResizeBuffers` fails with outstanding refs), then refreshes the swapchain description. |
| Concurrency | `recursive_mutex callback_mutex` held across the whole callback (serialises Present vs Resize across threads, allows re-entry); `shared_mutex hook_lifecycle_mutex` taken *shared* (with a thread_local depth counter) by callbacks and *exclusive* by install/reset/shutdown, so un-hooking waits for in-flight callbacks; `thread_local inside_present/inside_resize` guards (re-entrant calls just forward to the trampoline); `atomic active_callback_count`. |
| Deferred lifecycle | `install()` called from inside a callback stores `pending_install` and returns false; `shutdown()` from inside a callback sets `shutdown_pending`. Both execute in `run_pending_install()` from `finish_callback` **after all locks are released**. Never create/disable a detour from within the detour that is executing. |
| Alternate targets | If a later discovery yields different Present/Resize pointers than already hooked (another overlay hooked first), log "Ignoring alternate DXGI hook targets" and do **not** hook again; re-install after shutdown re-`enable()`s the existing detour. |

Frame index and `present_event` are produced here; `present_transaction` (§5) consumes them.

### 1.5 Deferred teardown of the whole graphics layer

`shutdown_graphics_hooks()` cannot run if called from inside a graphics callback, a swapchain
callback or a device transition (`device_transition_count`, thread_local flags,
`has_active_callbacks()`): it sets `graphics_shutdown_pending`; the last RAII
`scoped_device_transition` / listener scope exit calls `process_deferred_graphics_shutdown()`.
Re-entrant teardown from Present is the classic exit-time crash. Corollary in `vr_component.cpp`:
component shutdown *requests* `runtime.shutdown()` and only unsubscribes listeners once
`runtime.shutdown_complete()` is true, checked from `on_present_post` and `pre_destroy`
(`finalize_component_shutdown_if_ready`) — never unsubscribe while a Present transaction is mid-flight.

Engine-exit ordering (H2-specific example, general lesson): drain the render thread
(`R_SyncRenderThread`) and run component teardown **before** the engine's own shutdown
unloads driver libraries (NVAPI) that may still be attached to a device you retain
(`shutdown.cpp`).

### 1.6 Threads and the GPU queue gate

- `acquire_gpu_queue_interop(client)` is a `recursive_mutex` with try-lock-first, contention
  counters, total/max/last wait (µs) and a **waiter record** (thread id, client =
  present/resize_buffers/openvr, start tick) so a watchdog can say who is stuck. Use it only for
  *short, explicit* immediate-context work (a `Flush`, a blit execute) — **never hold it across a
  blocking runtime call** (`WaitGetPoses`, `xrWaitFrame`, `xrWaitSwapchainImage`).
- `is_inside_present_gpu_scope()` (thread_local depth) is a tripwire: entering a Present-pre/post
  runtime transaction while it is set is fatal (`openxr_runtime.cpp` `on_present`,
  `openvr_runtime.cpp`). The `record_present_gpu_scope_begin/end` plumbing exists but the current
  swap-chain hook does not open it, so today it always reads false (unverified whether
  another module relies on it).
- Query/capture polling belongs on the **renderer thread**, not Present: a Present-side
  `GetData` raced `CopyResource`/`End` on the game's single-threaded device and surfaced later
  as an NVIDIA driver fault. Overlord polls from the `R_EndFrame` hook
  (`scheduler.cpp` ~161). Decide per D3D immediate context which thread owns it and do
  *all* its queries/copies there.
- The Present owner thread can be distinct from the renderer thread and the scene-owner
  thread; Overlord tracks all three and exposes an `owner_contract` validity flag (§13).

### 1.7 Other D3D hooking notes

- COM vtables are shared across devices (runtime/WebView2 devices too). Hooking an
  `ID3D11Device` method by vtable patch (e.g. `CreateTexture2D` slot 5 for the flight recorder,
  `d3d11/texture_diagnostics.cpp`) needs a per-device identity table (unknown devices forward to the
  saved original), must not re-patch a slot another middleware already patched, and must
  save/restore `GetLastError`. Never read the descriptor with a raw deref from hook context
  (Overlord uses a guarded read).
- Third-party Present-hooking overlays (MSI Afterburner is a documented startup blocker in
  Overlord's README) are why the alternate-target guard exists: if someone else hooked first, do not
  stack a second detour on a different target; surface the condition in status instead.

---

## 2. Backend selection and runtime/environment hygiene

### 2.1 API backend vs runtime provider (two independent choices)

- **API backend** (OpenXR default / OpenVR backup) is chosen **once** per client
  (`runtime_backend.cpp::choose_backend`): `H2V_VR_BACKEND` = `openxr` | `openvr` | `steamvr`
  (case-insensitive alias); unknown non-empty value → backend marked invalid, every call
  no-ops with `runtime_unavailable` and the reason as `last_error`. Backends are a
  `std::variant<unique_ptr<openvr::…>, unique_ptr<openxr::…>>`, immutable after construction;
  no automatic switch after a failure. The launcher saves a `vr_runtimeBackend` preference
  and applies it before loading the game binary; for direct starts an explicit env var wins.
- **Runtime provider within OpenXR** (SteamVR, Virtual Desktop VDXR, Oculus, WMR…) is selected by
  the OpenXR loader from the registered active runtime or `XR_RUNTIME_JSON`. A manifest never
  changes the API backend (`XR_RUNTIME_JSON=steamxr…` still means OpenXR).
- OpenVR resolves its own client via `openvrpaths.vrpath` (`VR_GetRuntimePath`), **not** the
  OpenXR manifest — the manifest may name another vendor/architecture (§11).

### 2.2 OpenXR provider selection (`openxr_layer_policy.cpp`, `openxr_startup.hpp`)

All of this runs before the loader is loaded, on every (re)initialisation (no cached override):

1. **Explicit `XR_RUNTIME_JSON` always wins** (`runtime_override_source = existing_environment`).
2. **Elevated (high-integrity) processes**: the OpenXR loader ignores `XR_RUNTIME_JSON` (and HKCU
   layer registrations) when elevated. If an override is set *or needed*, fail with a blocking
   error naming "high integrity" instead of silently using another runtime.
3. **Steam Link auto-selection** (`choose_startup`): switch to SteamVR/OpenXR automatically only
   if no explicit runtime, SteamVR isn't already the active runtime, and OpenVR reports a
   *connected* HMD whose `Prop_ActualTrackingSystemName_String == "vrlink"`. The optional
   remote-client-id property is diagnostic only (its error must not veto). Rule: neither an
   installed runtime nor a running streamer proves which route the user wants; Virtual Desktop
   can feed both VDXR and SteamVR, so ambiguity keeps the configured runtime.
4. **Automatic override mechanics**: manifest ≤ 1 MiB, parse `runtime.library_path` (UTF-8, no
   NUL, relative resolved against manifest dir, file must exist), then `SetEnvironmentVariable`
   for **this process only**; restored with `gsl::finally` around init
   (`restore_runtime_preference` unsets only if the value is still the one the policy installed).
   If the previously selected manifest vanished → fail "selected OpenXR runtime became
   unavailable", never fall through to another provider. System registry is never modified.
5. **Active-runtime discovery without the loader** (diagnostics): `HKLM\SOFTWARE\Khronos\OpenXR\1\ActiveRuntime`
   (64-bit view) or the env override; SteamVR recognised by filename `steamxr/steamvr` or manifest
   key `VALVE_runtime_is_steamvr`; vendor DLLs are never loaded as OpenVR clients.
6. **OpenVR metadata query happens first and is closed first.** SteamVR controller reference
   components / HMD driver identity are read with a short `VR_Init(VRApplication_Background)` …
   `VR_Shutdown()` (`steamvr_controller_reference.cpp`), *before* any OpenXR loader/instance exists.
   No OpenVR SDK connection may overlap an OpenXR session or the Present-owned loop.
   Missing static reference → that hand is unavailable (fail closed; message says `vr_reinit`).
   (Details of the pose-normalisation data are in `07`.)

### 2.3 Incompatible implicit API layers

A layer built for Oculus-API engines can wrap and break a direct-OpenXR game.
`apply_native_implicit_layer_policy()`: enumerate `HKLM\SOFTWARE\Khronos\OpenXR\1\ApiLayers\Implicit`
(`KEY_WOW64_64KEY`) plus **HKCU only when the process is not high integrity** plus the default
Virtual Desktop Streamer install path; a registry value DWORD == 0 means *enabled*. Read each
manifest; if `api_layer.name == "XR_APILAYER_VIRTUALDESKTOP_oculus_compatibility"`, set that layer's
own `disable_environment` variable to `"1"` for this process (invalid/missing variable name is a
blocking error). Generic rule: parse manifests, use the layer's declared disable variable, never
delete registry keys; leave unrelated layers alone. (The disable variable is not restored;
status exposes manifest count, disabled list, warnings.)

### 2.4 Loader discovery

`openxr_loader.dll` is **app-local**, built from the pinned Khronos SDK (`release-1.1.62`
via `tools/build_openxr_loader.py`; embedded as a launcher resource and restorable by the
preflight "Fix"). Search: next to the mod module, then next to the EXE, with
`LoadLibraryExW(path, nullptr, LOAD_LIBRARY_SEARCH_DLL_LOAD_DIR)`. Then fill a dispatch table:
global functions through `xrGetInstanceProcAddr(NULL, …)`, ~45 instance functions through a macro
after `xrCreateInstance` (`openxr_dispatch.cpp`). Benefits: no hard import (the game still starts
without VR), the loader can be freed again for re-init, tests substitute a mock loader.

---

## 3. OpenXR initialisation, step by step

Overlord's real order (`openxr_runtime.cpp::initialize_locked`), with the failure state each
step maps to. Distinct states matter: they drive diagnostics and "retry vs give up".

| # | Step | Failure → state |
|---|---|---|
| 1 | validate graphics + tear down any prior session | `waiting_for_graphics` |
| 2 | OpenVR metadata query → runtime preference → layer policy | `runtime_unavailable` (blocking error text) |
| 3 | load loader; `xrEnumerateInstanceExtensionProperties`; **require `XR_KHR_D3D11_enable`**; enable `XR_KHR_composition_layer_cylinder` only if present | `loader_missing` / `runtime_unavailable` |
| 4 | `xrCreateInstance` (apiVersion 1.0.0; app/engine name = product) | `runtime_unavailable` |
| 5 | `xrGetSystem(HMD)`: `XR_ERROR_FORM_FACTOR_UNAVAILABLE` → message "no available headset; check connection and runtime, use `vr_reinit` after reconnecting" | `no_hmd` |
| 6 | require `graphicsProperties.maxLayerCount >= menu surfaces + 1` (Overlord: 7 menu surfaces + 1 projection = 8) | `runtime_unavailable` (`XR_ERROR_LAYER_LIMIT_EXCEEDED`) |
| 7 | view config PRIMARY_STEREO must be **exactly 2 views**; blend mode = first enumerated (expect OPAQUE) | `runtime_unavailable` |
| 8 | `xrGetD3D11GraphicsRequirementsKHR`; **adapter LUID of the game device must equal `requirements.adapterLuid`**; `device->GetFeatureLevel() >= minFeatureLevel` (error prints both LUIDs) | `graphics_mismatch` |
| 9 | `xrCreateSession` with `XrGraphicsBindingD3D11KHR{ game device }` | `runtime_unavailable` |
| 10 | LOCAL + VIEW reference spaces (identity pose) | |
| 11 | swapchain format; **common eye extent**; tell the engine the eye size; eye swapchains | |
| 12 | actions (`07`); `head_pose_bridge::request_recenter()` ("a new LOCAL space cannot reuse a reference from an older session") | |
| 13 | state `session_idle`; **wait for `XR_SESSION_STATE_READY`** | |

Notes:
- **LUID rule** is the #1 hybrid-GPU failure. Export `NvOptimusEnablement=1` and
  `AmdPowerXpressRequestHighPerformance=1` from the host so laptops pick the discrete GPU, and print the
  game adapter LUID in diagnostics. `openxr_d3d11.cpp::create_private_device` (private device on the
  runtime's LUID, `D3D11CreateDevice` resolved via `GetProcAddress` on `d3d11.dll` so you do not
  re-enter your own hook) exists but is **unused** in the production path — Overlord always
  binds the game's device.
- **Swapchain format**: preference order `R16G16B16A16_FLOAT`, `R8G8B8A8_UNORM_SRGB`,
  `B8G8R8A8_UNORM_SRGB`, `R8G8B8A8_UNORM` (first one the runtime lists; none → fail
  `XR_ERROR_SWAPCHAIN_FORMAT_UNSUPPORTED`). Menu surfaces independently prefer 8-bit sRGB
  (`R8G8B8A8_UNORM_SRGB`, then `B8G8R8A8_UNORM_SRGB`) and fall back to the eye format. For an
  `_SRGB` swapchain write *linear* values and let the RTV encode.
- **Common eye extent**: take the max `recommendedImageRect{Width,Height}` over both views; require
  ≤ each view's `maxImageRect…`; then set both views to it. Hand that size to the engine's
  scene-resolution request so the game renders at exactly the eye size (no submit-time rescale).
  Eye swapchain limits: usage `COLOR_ATTACHMENT` only, 1 sample, 1 mip, array 1, width/height ≤ 8192,
  area ≤ 33,554,432, image count 1..64, one RTV per image created with the swapchain format.
- Never use `xrCreateSession`-time assumptions about `LOCAL`: it is re-created per session.

```cpp
// condensed core (checks omitted)
const char* exts[] = { XR_KHR_D3D11_ENABLE_EXTENSION_NAME, XR_KHR_COMPOSITION_LAYER_CYLINDER_EXTENSION_NAME };
XrInstanceCreateInfo ici{XR_TYPE_INSTANCE_CREATE_INFO};
ici.applicationInfo.apiVersion = XR_MAKE_VERSION(1,0,0);
ici.enabledExtensionCount = cylinder_supported ? 2 : 1;   ici.enabledExtensionNames = exts;
xrCreateInstance(&ici, &instance);
XrSystemGetInfo sgi{XR_TYPE_SYSTEM_GET_INFO, nullptr, XR_FORM_FACTOR_HEAD_MOUNTED_DISPLAY};
xrGetSystem(instance, &sgi, &system);
XrGraphicsRequirementsD3D11KHR req{XR_TYPE_GRAPHICS_REQUIREMENTS_D3D11_KHR};
pfnGetD3D11GraphicsRequirementsKHR(instance, system, &req);      // must be called before xrCreateSession
// compare req.adapterLuid with IDXGIAdapter::GetDesc().AdapterLuid of the GAME device; feature level >= req.minFeatureLevel
XrGraphicsBindingD3D11KHR b{XR_TYPE_GRAPHICS_BINDING_D3D11_KHR, nullptr, game_device};
XrSessionCreateInfo sci{XR_TYPE_SESSION_CREATE_INFO, &b, 0, system};  xrCreateSession(instance, &sci, &session);
// reference spaces LOCAL + VIEW (identity pose); swapchains COLOR_ATTACHMENT, sampleCount 1, enumerate images -> RTVs
```

### 3.1 Session state machine (`poll_events`, polled in `maintain_session` each Present-pre and Present-post)

| Event / state | Action |
|---|---|
| `READY` and not running | `xrBeginSession(PRIMARY_STEREO)`; `running` |
| `STOPPING` | finish the open frame with **no world layer**, invalidate input + pose, `xrEndSession` (running := false even on error), state back to `session_idle`; a later `READY` re-begins (begin_count 2, end_count 1) |
| `EXITING` / `LOSS_PENDING` | `reinitialize_pending = true`, invalidate pose/views/input; the next Present-pre tears down and re-inits (no game restart) |
| `FOCUSED` | gates input and recenter; `VISIBLE`-only = render but neutralise input |
| `InteractionProfileChanged` | refresh the per-hand profile (`07`); clears admission |
| `ReferenceSpaceChangePending(LOCAL)` | remember `changeTime`; at the first frame with `predictedDisplayTime >= changeTime` request recenter + invalidate input |
| `xrPollEvent` failure | `recoverable_error` |

A *recreated session* (new generation) must not inherit "running": wait for a fresh `READY`.

---

## 4. The frame loop lives inside the game's Present

```
Present-post(frame N)            [d3d11 post listener → openxr on_present_post]
  validate transaction (§5.2); poll events
  begin_prediction(pair = N):
      try_arm_native(): native content proof must be COMPLETE and match generation/context/owner thread/eye size,
                        ring (re)built suspended; otherwise return (tracking only)
      xrWaitFrame → predictedDisplayTime        (blocks; never inside the GPU-queue gate)
      xrBeginFrame
      !shouldRender → end frame with 0 layers, invalidate pose, return
      reference-space-change check (recenter at changeTime)
      xrLocateViews(LOCAL, predictedDisplayTime) → 2 eye poses + FOVs   (count must be 2)
      xrLocateSpace(VIEW in LOCAL, same time)   → head validity gate
      gates: view flags POSITION|ORIENTATION valid; head location likewise; quaternion norm within 0.01 of 1;
             pose_filter::valid (|pos| ≤ 1e5, orthonormal within 0.05, det ≥ 0.95)   else: zero-layer frame, invalidate
      engine-stereo: reject canted displays |dot(qL,qR)| < 0.99999 (≈0.512° between eyes; |dot| so q and −q are equal)
      render head = midpoint(eye positions), slerp(eye rotations, .5)       // ONE located sample
      publish_tracking_pose(head, pair, now, reference_policy)             // recenter allowed only if state==FOCUSED
                                                                            // and view+head carry the TRACKED bits
      inputs.sample(xrSyncActions + xrLocateSpace grip/aim at the same predicted time) → controller_input::publish
      menus_.observe_backdrop(...); decide ui_only (frontend or movie theater → no world pair)
      eye offsets = R_headᵀ·(eye_pos − head_pos);  projections = tan{left,right,down,up}
      native_render_session.admit_pair(N); engine_stereo_bridge::publish_view_family(N, offsets, projections)
… game simulates and renders frame N+1; the stereo renderer renders BOTH eyes of pair N …
Present-pre(frame N+1)           [openxr on_present → complete_native]
  not published yet: wait at most 4 Presents (pending_presents ≥ 4) then end the frame without a world layer
  deferred (benign CPU dependency learned) → omit world; failed → frame_failure
  acquire the published pair; per eye (sequentially):
      xrAcquireSwapchainImage → xrWaitSwapchainImage(10 ms) → blit (linear→swapchain) → Flush (in gate) → xrReleaseSwapchainImage
  menus_.prepare (quad/cylinder layers + pointer)
  xrEndFrame(predictedDisplayTime, blend, layers=[projection(views = ORIGINAL SDK eye poses+FOVs, subImage=eye swapchain), menus…])
  retire pair N (release; discard_unpublished if never published; quarantine if retirement fails)
real Present
```

Key properties:
- **Same prediction for render and submit.** The projection layer uses the *original* `xrLocateViews`
  poses/FOVs of that pair; never re-sample. (A mock EndFrame can assert this — §14.)
- **Present ownership** is verified, not assumed (§5.2). Runtime waits (`xrWaitFrame`,
  `xrWaitSwapchainImage`) never run inside your GPU-queue scope.
- Eye-render and head/offset data come from **one** `xrLocateViews` sample. A separate VIEW-space
  query only gates tracking/recenter eligibility; its orientation is *not* a cant test (predictions
  may update between queries at the same display time).
- Parallel-but-rotated views (shared head rotation with eye offsets rotated about it) and quaternion
  pairs differing only by sign/scale (w=.999 vs −1.001) must be **accepted**; truly canted eyes are
  fatal for shared-camera engines (error string carries raw quaternions, eye angle, limit, pair, display
  time, view flags, session generation — formatted only on rejection).
- Not ready yet (bootstrap/content proof pending, `ui_only` with no menus) → end the frame
  with no world layer. **Never show the desktop image.** Missing/invalid tracking bits → zero-layer
  frame, invalidate pose/views, keep the session (recovers on the next valid sample).
- `ui_only` (frontend menus or movie theater): no stereo pair is admitted; the frame carries menu
  layers only. Pause dim: when a fresh non-frontend menu publishes `scene_dim`, the eye blit multiplies
  linear colour by `(1 − dim)`.
- Initialisation, resize and teardown also happen only at Present-pre/post of the real Present
  (`maintain_session`), so there is no competing runtime worker thread.

### 4.1 Frame-failure policy (`frame_failure`)

Record the *initiating* failure (operation + XrResult) **first**, then settle any acquired/waited
swapchain images (wait + release), end the frame with no layers, invalidate input and pose, set
`recoverable_error` without re-recording history, then `teardown_preserving_error()` (restores state,
last error, XrResult, init stage afterwards). A rejected view family, failed `xrLocateViews`, failed
admission → fail + zero-layer end. A failed `xrEndFrame` → `reinitialize_pending`.

---

## 5. Present transactions and ownership

### 5.1 Pairing
`present_transaction::key{active, frame, generation, thread_id}`: *pre* opens it; *post* validates
`matched | missing_pre | frame_mismatch | generation_mismatch | thread_mismatch | present_failed`.
Any non-match ⇒ invalidate input/pose/views and `fatal_for_vr` (the game keeps running flat).
`owner_changed()` detects a *different thread* driving Present on the same device generation
(allowed only while reinitialising).

### 5.2 Rules (both backends)
- Pre entered while another transaction is active, or inside the Present GPU scope → fatal.
- VR is never initialised outside the real Present-pre (CPU/WARP fixtures may use a separate
  synthetic entry; native mode refuses a virtual Present).
- Auto-initialisation is attempted **once per valid device generation** (`initialization_generation_`
  survives session teardown; missing graphics doesn't consume an attempt; repeated failure on the
  same generation needs `vr_reinit`/enable). A device-destroy callback may be missing: a
  generation change seen at Present still triggers exactly one attempt.
- Disabled or terminally shut-down VR never restarts because a device changed; late callbacks
  (resize/new device) never revive it.

---

## 6. Error-code subtleties (OpenXR)

| Situation | Correct handling |
|---|---|
| `XR_SESSION_LOSS_PENDING` returned as a *positive success code* from `xrWaitSwapchainImage` | Success (`XR_FAILED(r)` is false). Both eyes still wait + release. The state-change *event* LOSS_PENDING, by contrast, schedules re-init. Always test `XR_FAILED(r)`, not `r != XR_SUCCESS`. |
| `XR_TIMEOUT_EXPIRED` from `xrWaitSwapchainImage` | **Not** a success for ownership purposes (`call_result::operator bool`): the image stays acquired-but-unwaited and must remain legal to wait + release later (teardown/`frame_failure` `settle_eye` does exactly that). State → `recoverable_error`. |
| `XR_ERROR_FORM_FACTOR_UNAVAILABLE` from `xrGetSystem` | `no_hmd`; user message includes runtime name + `vr_reinit`. |
| `xrEndSession` | Legal only after STOPPING. It sets running := false **even when it returns an error**; `XR_ERROR_SESSION_NOT_RUNNING` is tolerated; never call it again from shutdown; never call it outside STOPPING (teardown just `xrDestroySession`s — legal from any state once no thread uses the handle). |
| First failure vs cleanup failure | Keep the first XrResult/operation (`last_xr_result`, `failures.first`); swapchain cleanup must not overwrite an init failure; shutdown must not erase failure history. |
| `shouldRender == FALSE` | `xrEndFrame` with zero layers, `submitted_frames` unchanged, state stays running; diagnostics distinguish `empty` from `world` and `menu_only`. |
| Null cleanup proc after `XR_SUCCESS` from `xrGetInstanceProcAddr` | Map to `XR_ERROR_FUNCTION_UNSUPPORTED` (teardown failure, retryable). |
| `XrResult` names | Keep a macro table for every result so logs say `XR_ERROR_CALL_ORDER_INVALID`, not `-37`. |
| Failure ring (`runtime_failure_history`) | First failure kept forever + ring of 8; consecutive identical (session, code, operation, detail) coalesce into `repetitions`; strings truncated 256/128/512. |

---

## 7. Teardown, retry, resize, device loss

### 7.1 Teardown order (`teardown_locked`)
1. invalidate inputs/haptics, request-menu off, pose invalid; finish any open XR frame (omit world)
2. if nothing created → done; else `++cleanup_count`, invalidate engine views/compositor/capture
3. **wait for GPU idle** (`D3D11_QUERY_EVENT`, `Flush`, poll `GetData(DONOTFLUSH)` with `Sleep(0)`, 500 ms
   deadline; returns true if the device was removed) — commands referencing swapchain images must retire
4. destroy menu surfaces (finish acquire→wait→release first), destroy action spaces/set
5. settle acquired eye images (wait + release)
6. `xrEndSession` **only if state == STOPPING** (see §6)
7. destroy eye swapchains → spaces (VIEW, LOCAL) → session → instance (fetch `xrDestroyInstance` lazily if
   the table slot was empty) → reset dispatch → `FreeLibrary(loader)`
8. release native ring (`native_render_session::invalidate(generation)`), blit objects, graphics snapshot

**Retryable**: any step that fails returns `recoverable_error`, `applied_enabled = false`,
`loader_loaded` stays true (the object graph is retained); a later re-init finishes teardown first.
Handles are nulled only after their destroy succeeded. Counters `created == destroyed` must hold after
any failure path (assert in tests).

### 7.2 Resize, device loss, reinit
- `on_resize_before` only stores the generation in an **atomic**. At the next Present-pre
  `maintain_session` sets `reinitialize_pending`; retire submitted targets, reset temporal history, recreate
  session and ring. No GPU/runtime work and no runtime lock in the resize callback; consecutive requests
  coalesce; a failure doesn't retry every Present; no recenter is requested.
- `on_device_destroying` (device transition under lock): tear down immediately when the generation matches.
- Reinit command flow (`vr_reinit`, `vr_component.cpp`): disable stereo/head bridges, invalidate
  views/pose, reset temporal history, push world scale/eye swap, set desired enabled,
  `request_reinitialize`, re-enable bridges; the real init runs at the next Present-pre. Recenter also
  resets temporal history + menu anchor.
- Disable → full teardown (loader unloaded); enable → rebuild. Session/device recreation bumps
  `session_generation`/`device_generation` exactly once per attempt.

### 7.3 Thread-safe status
`status_` (guarded by `mutex_`) is copied to `status_snapshot_` under a separate `status_mutex_`
(`publish_status_locked`); `get_status()` reads only the snapshot so diagnostics never wait behind the
frame lock. Every lock site is tagged for contention measurement (`wait_timing::site::…`).

---

## 8. Eye texture hand-off: ring, pair lifecycle, conversion

The game renders each eye on its own thread; the compositor must never see a half-updated or reused
image. `native_render_session.cpp` is **shared by both backends**.

### 8.1 Ring and pair states
- `pairs_` = **2 pairs × 2 eyes** of ordinary same-device textures: `D3D11_USAGE_DEFAULT`, bind
  `RENDER_TARGET | SHADER_RESOURCE`, mips 1, array 1, 1 sample, no CPU access, `MiscFlags == 0`
  (never shared handles). `pair_id` is the predicted-pose-family id (OpenXR: Present frame index;
  OpenVR: Present frame index of the `WaitGetPoses`), so pose, view family and slot share one identity.
- States: `free → rendering → published → (release → free) | quarantined`.
- **Admission gate** (`accepting_pairs_` + `expected_pair_id_`): exactly one stereo producer per pose
  family is admitted (`admit_pair(id)`); a wrong-pair copy is rejected *without* closing the gate and
  records evidence; the gate **closes automatically** when both eyes complete. `suspend_acquisition()`
  freezes it; `ensure_copy_ring` refuses to rebuild unless acquisition is suspended **and** every slot is
  free. A valid-but-older pose family is declined (`accepts_pair`) so the engine's original path runs
  normally without occupying a slot.
- Per-eye lifecycle: `acquire_target`/`copy_eye` → engine renders → `complete_rendered_eye` validates the
  texture is exactly the ring's; the mask logic forbids concurrent eyes; both eyes done ⇒ `published`.
- **Three ways to end an unfinished pair — keep them distinct:**
  | Verb | Meaning |
  |---|---|
  | `defer_unpublished_pair` | benign: renderer learned a CPU-side dependency; allow a fresh pose family (OpenXR: end frame without world) |
  | `discard_unpublished_pair` | renderer failure / never reached the compositor; storage returns to the ring |
  | `quarantine_pair` | retirement unproven (published pair whose check was revoked, or discard could not be proven safe): never leased again in this device session |
- Any failure inside `copy_eye` quarantines the whole pair and stores **value-only first-failure
  evidence** (stage enum, HRESULT, pair ids, thread, pointers, expected sizes) that later errors and
  invalidate do not erase.
- **Retirement boundary**: OpenXR: after `xrEndFrame` (`retire_native_prediction`: release if published,
  else discard, else quarantine). OpenVR: only at the next *successful* `WaitGetPoses`.
- Dimensions come from the runtime's recommended size; out of limits ⇒ fail, **no clamping**.
- Validate every source/destination: same device by **COM identity** (`QueryInterface(IUnknown)` on both,
  not raw pointer), exact size/format, mips/array/samples/usage/bind/CPU/misc as above, source ≠
  destination alias. Check `GetDeviceRemovedReason()` after each eye.
- The desktop mirror previews the **latest published** pair on the same owner thread — never a pair that
  is rendering, incomplete or quarantined; projection/camera metadata is frozen with the image at
  conversion time.

### 8.2 Format and colour contract
| Stage | Format / encoding |
|---|---|
| Engine final output (H2-only example) | `R11G11B10_FLOAT`, bind SRV\|RTV\|UAV, display-encoded (already gamma) |
| Ring texture | **linear**; format-parameterised: `R16G16B16A16_FLOAT` (production), `R10G10B10A2_UNORM`, `R8G8B8A8_UNORM`, `R11G11B10_FLOAT` allowed; `CheckFormatSupport` must report `TEXTURE2D | SHADER_SAMPLE | RENDER_TARGET` |
| OpenXR swapchain | R16F first, else 8-bit sRGB (RTV encodes) |
| OpenVR submit | `ColorSpace_Linear`, the ring texture itself |

Conversion is **mandatory even when formats match** (an equal-format bypass produced a grey washout):
fullscreen-triangle VS (`SV_VertexID`, positions `(-1,-1),(-1,3),(3,-1)`, no vertex buffer, null input
layout, `Draw(3)`) + PS doing `Load()` and `x ≤ .04045 ? x/12.92 : ((x+.055)/1.055)^2.4`. Tests decode the
destination pixel and assert it equals sRGB→linear of the source within ~0.007.

### 8.3 State-preserving transfer: deferred context + `ExecuteCommandList(list, TRUE)`
Draw on the game's immediate context **without save/restore code**:
1. record the draw on a per-renderer **deferred context** (`CreateDeferredContext(0)`), `FinishCommandList(FALSE)`;
2. `immediate->ExecuteCommandList(list, RestoreContextState = TRUE)` restores the *entire* native pipeline
   state (including shader class instances);
3. check the immediate context type, same-device ownership of every SRV/RTV, source ≠ destination;
4. dynamic constants via `UpdateSubresource`/`Map(WRITE_DISCARD)` on the deferred context.

Caching (`native_render_session`): one command list per (pair slot, eye, source-target index), rebuilt
only when the source identity changes; source SRV cached by texture identity (a *different* texture in the
same cache slot is a hard fail `source_identity`, quarantining the ring). `texture_blit.cpp` records per
call (16-entry round-robin SRV cache keyed by texture+encoding), executing under the GPU-queue gate.

**Tag your own lists.** `SetPrivateData(GUID, 8-byte magic)` on every mod-made command list and deferred
context (`native_conversion_command_list::mark / mark_recording_context`) so a global
`ExecuteCommandList`/Draw census hook can distinguish "state-preserving mod work" from arbitrary game
lists. Never mark the game's immediate context. An untagged list is treated as opaque (resets history).
Fallback when you cannot use a deferred context: an RAII `scoped_d3d_state` snapshotting RT/DSV, blend,
depth, raster, 16 viewports, 16 scissors, IA layout/topology/VB/IB, VS/PS + class instances, PS
SRV0/sampler0/CB0 (`scene_compositor.cpp`) — and restore **all** of it.

Other draw hygiene: compile HLSL once per process (function-local static, cache *failure* too), recreate
per-device objects when the `ID3D11Device*` changes; blit modes in `texture_blit` are `linear`
(`rgb·(1−dim)`), `encoded_premultiplied` (UI: un-premultiply → sRGB-decode → re-premultiply in linear)
and `encoded_opaque` (movie, alpha forced 1); for encoded sources the SRV format is reinterpreted
`*_SRGB → *_UNORM` so hardware does not decode.

### 8.4 Per-eye composition callbacks
HUD/optic/narrative layers are composited per eye **inside `copy_eye`, after display conversion and
before publishing** (`eye_composition.hpp`, ordered registry of 14 layers): on the game's serialized
immediate-context owner; no waits, no runtime IPC, no native-session re-entry, no retained destination
targets, restore all D3D state; metadata is the immutable pair-matched copy (never read later backend
globals; model origins come from the *current* eye record, not history). Details in `05`/`09`.

---

## 9. Menu / movie compositor layers (OpenXR)

Compose the UI as compositor layers, never into the eye image (unless a diegetic panel needs scene depth).
`openxr_menu.cpp`:

- **One XR swapchain per surface** (size = captured image size, recreated on size/format change; menu
  surfaces prefer 8-bit sRGB). Slots: `0..3` menus, `4` background/theater, `5` backdrop, `6` cursor
  (`menu_surface.hpp`, `surface_count = 7`).
- **Layer order is explicit back-to-front** (SDK order, independent of slot order): background, then menus
  with the backdrop slot inserted just before the *active (top)* menu, then cursor.
- **Geometry (starting values)**: menu plane **0.9 m wide at 0.5 m**; each older menu pushed back
  **+0.25 m** (up to 4); popup dark backdrop **0.05 m** behind the active menu; "theater" (frontend/movie)
  cylinder with radius = depth (3 m base + 4×0.25 reserved = 4 m for the background slot), width =
  `depth·π/2` (90° arc). Depth animates toward target with `approach_depth` (2 m/s, `dt` clamped 0.1 s,
  starts from the current depth → no drift/queued animations); the child moves in, parents recede.
- **Anchor** = head position + horizontal (yaw-only) heading, re-captured when menu session id, input
  `reference_generation` or device generation changes (and on recenter).
- **Layer kind**: `XrCompositionLayerCylinderKHR` if the extension was enabled (centre = anchor + forward·(distance − radius),
  radius = distance, centralAngle = width/radius, aspectRatio = width/height), else `XrCompositionLayerQuad`.
  Pose rotation from basis `(right, up, −forward)`; eye visibility BOTH.
- **Flags/colour**: captured UI is premultiplied and gamma-encoded ⇒ decode to linear in the blit
  (`encoded_premultiplied`) and set `XR_COMPOSITION_LAYER_BLEND_TEXTURE_SOURCE_ALPHA_BIT`. Add
  `XR_COMPOSITION_LAYER_UNPREMULTIPLIED_ALPHA_BIT` **only** if your texture is *not* premultiplied
  (Overlord never sets it). Opaque movie background: flags 0 + `encoded_opaque`.
- **Freshness rejects** (each counted into a diagnostic struct with a state string): menu state valid only if
  enabled and age ≤ **250 ms**; image owner revision == current revision; per-layer image owner id ==
  menu id (`owner_mismatch`); device generation (`generation_mismatch`); cursor ≤ 250 ms (`stale_cursor`).
  Losing images → state `menu_images_missing`, pointer not ready, **frame still ends**.
- **Pointer**: ray = runtime aim pose forward (−Z column), intersect the *active* surface geometry →
  (u,v) → `canvas.source_uv` (letterbox correction: native UI fitted into 1920×1080 with transparent
  padding; a hit in padding is a miss and transfers nothing) → publish
  `{session, revision, stamp, ready, hand, hit, u, v}`. `ready` only if the active surface is at its target
  depth, interactive and input-focused. `pointer_owner` switches hand only on the *other* hand's fresh
  trigger edge (never on a ray miss or tracking loss). Neutral-arming/stick rules are in `09`.
- **Movie theater**: copy the game's swapchain back buffer (`chain->GetBuffer(0)`) into a **3-slot pool**
  (a slot is free when `shared_ptr::use_count()==1`) after validating the descriptor (≤ 8192, 1 mip, 1
  array, 1 sample), never retain the backbuffer itself; show it on the big cylinder, opaque.
- A menu-only frame contains menu layers and **zero projection layers**; every swapchain acquire→wait→release is
  finished before its destroy.
- **OpenVR equivalent** (`menu_overlay.cpp`): one `IVROverlay` per slot, key
  `overlord.native.menu.<pid>.<slot>`, flags `IsPremultiplied`, `NoBackside`, `IgnoreTextureAlpha` for
  movies, texture submitted as `ColorSpace_Gamma` (UI is gamma-encoded), `SetOverlayWidthInMeters`,
  curvature = `width / (2π·radius)`, sort order theater 0 / menus `20+2i` / backdrop `19+2·active` /
  cursor 100, `SetOverlayTransformAbsolute(TrackingUniverseStanding)` with columns (right, up, −forward,
  centre). Push only state differences. The overlay keeps an immutable texture lease until the SDK
  acknowledges replacement/clear; a rejected/failed lease is **quarantined** until the compositor acknowledges
  or the runtime detaches. Ray hit: `ComputeOverlayIntersection` (distance in (0,20) m, UV in [0,1], v measured
  from the bottom). Dark backdrop width scales with its radius/depth; frontend backdrop is 1.25× wider.

---

## 10. Haptics (runtime side)
Pulses are consumed after the frame is published and acknowledged with a delivery code; a haptic failure never
disables rendering. OpenXR `xrApplyHapticFeedback` only while focused; one pending pulse per hand ≤ 0.1 s,
1–320 Hz, 100 ms expiry (`controller_haptics`); profile change or input invalidation clears pending pulses.
Action/binding details: `07`.

---

## 11. OpenVR backup backend (`openvr_runtime.cpp`, `steamvr_runtime.cpp`)

A complete, separate backend selected with `H2V_VR_BACKEND=openvr|steamvr`. It reuses the ring, view
family and Present-transaction machinery, so most contracts are identical; the differences are below.

### 11.1 Frame protocol (all on the Present owner thread)
```
Present-POST(frame N):
   if a pair was submitted:  WaitGetPoses → release_pair(submitted)  (retirement boundary)
   else (tracking-only / unarmed): WaitGetPoses with tracking-only contract
   HMD pose from the renderPoses array (bDeviceIsConnected && bPoseIsValid) → publish_tracking_pose
   input sample; GetEyeToHeadTransform + GetProjectionRaw per eye → publish_view_family(N, eye offsets, tangents)
   admit_pair(N); phase := collecting
Present-PRE(frame N+1):
   advance_prepared_frame → prepare_scene returns pending | deferred | ready | fatal
   ready → render_eye ×2 prechecks, then Submit(Eye_Left), Submit(Eye_Right) immediately; retain pair until next WaitGetPoses
```
Phases: `idle_unarmed → need_pose → collecting → submitted_waiting_present`, and `fatal_holding_pair`.
`Texture_t{ ring texture, TextureType_DirectX, ColorSpace_Linear }`, bounds (0,0,1,1), `Submit_Default`;
**no shared handle, cross-device import, CPU readback, or Flush** — the ring texture is a plain same-device
texture produced by the proven immediate-context-ordered copy. Pose/GPU-transport arming: the GPU transport arms
only after the owner pass proves both eyes completed, distinct and non-zero on the current context/generation/owner
thread; until then SteamVR is connected with `gpu_transport_unarmed` and head tracking stays live.
Pose invalidity maps to distinct reasons (`hmd_disconnected`, `hmd_pose_invalid`, `compositor_focus_lost`).
UI-only (frontend/movie): discard any prepared pair, suspend native acquisition, keep sampling tracking/input and the overlay presenter.

### 11.2 Production format and the format probe
Candidates `R16G16B16A16_FLOAT → R10G10B10A2_UNORM → R8G8B8A8_UNORM`. **Only candidate 0 (R16F) may become
production**: UNORM acceptance is recorded as diagnostic evidence only, because quantising an HDR scene is not an
eligible fallback (the one-shot frame is retired and the transport returns to tracking-only). If both eyes
return `TextureUsesUnsupportedFormat`, retire at the next `WaitGetPoses` and advance; if every candidate fails
the transport stays unarmed. `CheckFormatSupport` result, ring-ready, submit results and `accepted_eye_mask`
per candidate go in status. After a failed submit call `ClearLastSubmittedFrame()`.

### 11.3 Focus loss and error mapping
- `VRCompositorError_DoNotHaveFocus` from `WaitGetPoses` **or** `Submit` (even before the first Submit) is
  *recoverable*: keep every submitted/prepared resource, never detach or free compositor-visible memory,
  invalidate pose/input (`compositor_focus_lost`, code retained), clear FadeToColor/FadeGrid, retry on a later
  matched Present. A pair already handed to `Submit` stays owned by the compositor (not retired, not resubmitted
  while `WaitGetPoses` keeps failing) and is retired **exactly once** after the next successful `WaitGetPoses`;
  resume with the same ring and format candidate. Focus loss never rejects a format.
- A hard error in either eye (`InvalidTexture`, …) is not masked by the other eye's focus error → fatal after
  safe retirement; non-focus `WaitGetPoses` errors (`IsNotSceneApplication`) are fatal (`teardown_preserving_error`,
  §11.6).
- `VREvent_Quit` → `AcknowledgeQuit_Exiting()` + `reinitialize_pending`.

### 11.4 Projection pitfall
`IVRSystem::GetProjectionRaw(eye, &left, &right, &top, &bottom)`: the vertical names are **historically inverted**
— `top` is already the negative tangent of the bottom edge (your *down*) and `bottom` the positive tangent of the
top edge (your *up*). Pass through, do not negate or swap (`projection_tangents_from_raw`; a test pins
`(-1.376, 0.839, -1.428, 0.966)`). Eye offsets come from `GetEyeToHeadTransform`; dimensions from
`GetRecommendedRenderTargetSize` (validated, **never clamped**).

### 11.5 Compositor effects
`SetTrackingSpace(TrackingUniverseStanding)` at init. Movie "theater" and loading transitions:
`FadeGrid(0, true)` + `FadeToColor(0,0,0,0, alpha=1, false)` (black backdrop with application overlays above it);
the **pause dim** is `FadeToColor` with `alpha = scene_dim` (~140/255 in H2), independent of UI pixels and
persisting through a temporary empty-menu transition (OpenXR does the same by scaling the eye blit). Send alpha
only on change. Leave theater only after a fresh native pair was actually submitted. `GetFrameTiming` is sampled
only every 64 retired pairs (it is compositor IPC). `WaitGetPoses` duration histogram buckets 2/11/50/100 ms.

### 11.6 Preflight before `VR_Init` (fail early, with actionable text)
In order (`configure_direct_submission`, `initialize`):
1. engine-stereo hook installed **and** target identity matched *before* opening SteamVR (else the compositor sits
   in "waiting" until a level renders);
2. reject a device created with `D3D11_CREATE_DEVICE_SINGLETHREADED`;
3. the proven scene-owner thread == the Present thread (the OpenVR queue owner);
4. `validate_compositor_adapter`: device descends from a DXGI 1.1 factory (`adapter→GetParent(IDXGIFactory1)`) and
   `IVRSystem::GetDXGIOutputInfo` adapter LUID == the game device LUID.
Environment hardening (`steamvr_runtime.cpp`):
- `path_registry_scope`: copy `%LOCALAPPDATA%\openvr\openvrpaths.vrpath`, replace its `log` array with a process-local
  temp dir, set **`VR_PATHREG_OVERRIDE`** for the process (restored/deleted in the destructor); an explicit caller
  value is respected. OpenVR then never needs a writable SteamVR log dir.
- `diagnose_ipc_environment`: find `vrserver.exe` (Toolhelp) and compare **session id, elevation and user SID** with this process
  before the blocking `VR_Init`; produce text like "different elevation levels — run both at the same level". It
  deliberately does **not** open the single-instance `SteamVR_Namespace` pipe (it is often busy; probing it
  creates false failures and perturbs startup). "vrserver not running" is its own message.
- Runtime = `VR_GetRuntimePath` (OpenVR path registry), never the OpenXR manifest.
- `VR_Init(VRApplication_Scene)`; then `register_application` **before the first `PollNextEvent`**:
  `AddApplicationManifest(<exe dir>/<product>.vrmanifest)` + `IdentifyApplication(pid, application_key)`; manifest
  paths resolved against the EXE directory, never the CWD; registration failure is diagnostic only and must not
  disable rendering or input. Debug/standard builds use separate app keys. The packaged `.vrmanifest` needs
  `binary_path_windows` == the packaged EXE, relative forward-slash `image_path` (920×430) /
  `image_path_capsule` (600×900) / `action_manifest_path`, and every `default_bindings[].binding_url` present.

Fatal path = `teardown_preserving_error()`: **`VR_Shutdown()` before releasing native targets** (the ring owns
submitted/quarantined textures until OpenVR has fully detached), keep the fatal evidence; the game keeps running flat.
Resize: `on_resize_before` only stores the generation; the next Present-pre detaches OpenVR **before** retiring
submitted targets, resets temporal history and recreates the session (discarding a pose family whose targets were rebuilt).

### 11.7 OpenVR input (summary; binding details in `07`)
- Manifest at **`<exe dir>/vr_input/actions.json`** (absolute path required; the repo source dir is `data/vr_input/`),
  action set `/actions/gameplay`, `usage: leftright`, **no sub-action paths** (left/right are separate actions), one
  binding JSON per controller type (`oculus_touch`, `knuckles`, `vive_controller`), mandatory `localization` block.
- Action types: `boolean` for *all* buttons including trigger and grip (analog→digital thresholds live in the
  **binding**: Knuckles grip `force`, activate .08 / deactivate .02; trigger `value`, .01/.005; `haptic_amplitude:"0"` on
  every button source to suppress default click haptics), `vector2` move/turn, `pose` grip (`/pose/raw`) and aim
  (`/pose/tip`), `vibration` haptics. Trigger touch uses a separate `touch` source *without* `force_input`.
- Loader: `SetActionManifestPath` → `GetActionSetHandle` → `GetActionHandle` each. Core handles (move, turn, sprint,
  jump, trigger, squeeze, primary, secondary, grip, aim) are **required** (failure aborts all VR input init; the
  manifest's `requirement` field only affects SteamVR's binding UI); menu_toggle, menu_recenter, trigger_touch and
  haptics are **optional** (failure zeroes the handle). `UpdateActionState` only when focused; poses via
  `GetPoseActionDataForNextFrame(TrackingUniverseStanding)`; unfocused or failed update publishes neutral samplers and
  clears haptics. `Prop_ControllerType_String == "knuckles"` on both role devices → glyph hint.
- **Pose normalisation**: SteamVR returns the pose of whatever the binding names, so the adapter reads the action's
  binding (`GetActionBindingInfo`: exactly one binding, `/user/hand/left|right`, `/pose/<name>`), the device's render
  model, and the *static* render-model components `openxr_grip` / `openxr_aim` (`GetComponentState`; non-static rejected),
  composing `source_from_standard`; cached per (hand, grip/aim) by input-origin handle, generation bump on any
  device/binding event, retry ≤ 1/s; missing metadata ⇒ `pose_reference_unavailable`, never a guess.
- Bindings differ per profile: Touch left X = `menu_recenter`, `left_primary` unbound; Knuckles has no `menu_toggle`;
  Vive lacks recenter/primary/secondary/trigger-touch — re-home or accept feature loss and say so in the
  binding description.

---

## 12. Legacy / non-engine-stereo strategies (do not copy as the main path)

Kept in the repo for fixtures and as a fallback pattern if you cannot render per eye in the engine
(`scene_compositor.cpp`, `scene_mode::backbuffer`; docs call the cross-device experiments *historical*):
- Mono backbuffer: `GetBuffer(GetCurrentBackBufferIndex())` → `CopyResource` (or `ResolveSubresource` for MSAA) into a 2×2 ring; reject typeless backbuffers.
- Projection remap for a symmetric game frustum: `scale_u=(tr−tl)/(2·half_x)`, `offset_u=(tl+half_x)/(2·half_x)`,
  `scale_v=(tu−td)/(2·half_y)`, `offset_v=(half_y−tu)/(2·half_y)`; clamp UV to half a texel.
- Cross-device copy sync: `IDXGIKeyedMutex::AcquireSync(1,500)` → copy → `D3D11_QUERY_EVENT`+`Flush` → poll 500 ms →
  `ReleaseSync(0)`; **on timeout do not release** (releasing while the copy was in flight let the producer write during
  `CopyResource` and was observed as an NVIDIA access violation). Copy directly into the active slot — a two-step copy
  doubled full-resolution traffic exactly where TDRs were seen.
- Sidecar data (per-pair HUD parameters) keyed by `pair_id % 64` in a bounded cache; a miss never substitutes "the newest".

---

## 13. Status you should expose (`vr_status`)

Copy these into any port; they are what made field failures diagnosable:
- backend + selection reason; runtime name/system name; runtime manifest/library/**selection source**
  (`existing_environment | system_active_runtime | automatic_steam_link_vrlink`), override active/set-by-policy,
  implicit-layer policy (manifest count, disabled list, warnings); last init stage; `loader_loaded/instance/session/session_running`.
- failure ring (total, retained, first + recent: tick, session, frame, repetitions, code, operation, stage, detail, runtime, system);
  last error as *operation + XrResult name*.
- session generation, device generation, READY/STOPPING/begin/end/cleanup counts, `initialization_attempt_count`.
- frame submission counters: completed / world / menu_only / empty, last layer counts, age; menu-layer preparation
  (candidates/missing/invalid/generation_mismatch/stale_cursor/owner_mismatch).
- adapter match (both LUIDs), recommended eye size, swapchain/ring format, graphics transport (`h2_device_openxr`,
  `h2_device_direct`, `gpu_transport_unarmed`), native renderer ready/error.
- pairs admitted/published/retired/quarantined/deferred/exhausted/rejected, conversion attempts/completions/failures,
  first-copy-failure snapshot, command-list builds/executions, source-view creations/identity matches.
- Present transaction: pre/post counts, last completed frame, last post HRESULT, violation count, **thread split**
  (renderer vs scene owner vs present owner) + `owner_contract` valid; GPU-queue gate stats (acquires, contention, max wait, waiter).
- OpenVR extras: `WaitGetPoses` count/last/max/avg + histogram, submit timing, submit→retirement latency,
  compositor frame timing, application-registration error, **format-probe table** (per candidate: CheckFormat, support bits,
  ring ready, create error, left/right submit result, accepted eye mask, retired).
- tracking: sample count/failures, recenter pending/count, pose validity flags, controller profile per hand, controller reference ids.
- Honest evidence labels: "CPU call returned ≠ GPU retired"; status getters use `try_lock`/snapshots and never wait behind GPU work.

---

## 14. Testing the runtime layer (without a headset)

Details in `11`; the contracts worth encoding because they caught real bugs:
- A **fake `openxr_loader.dll`** (exports `xrGetInstanceProcAddr` + `Mock*` control functions) with a strict call-order state
  machine returning real error codes (double-acquire, release-before-wait, destroy-while-acquired, destroy-session-with-live-children,
  `EndFrame` display time ≠ `WaitFrame` time, `xrEndSession` outside STOPPING…), real D3D11 swapchain textures, fault
  injection per call, and per-call counters. At `xrEndFrame` it reads each eye's first pixel back and checks left/right colour
  identity, that eye poses/FOVs equal the located ones exactly, and an alpha "write epoch" that must change every frame
  (catches stale or duplicated textures). After the test `FreeLibrary` and assert the module is gone.
- Lifecycle assertions: distinct state per init failure (`runtime_unavailable/no_hmd/graphics_mismatch`) with loader released;
  first failure preserved; `XR_TIMEOUT_EXPIRED` leaves the image releasable; `LOSS_PENDING` result treated as success;
  invalid tracking → zero-layer frames, session untouched; teardown with an injected failure at each destroy step then
  retry; created == destroyed; device recreation → generation 2 and rendering resumes; mismatched Present pre/post →
  `fatal_for_vr` and no further `xrWaitFrame`; canted views rejected with evidence.
- OpenVR: `#include` the production `.cpp` with `VR_Shutdown` macro-redirected and fakes derived from **strict stubs** whose
  every virtual throws; assert focus-loss pair retention (`submit_calls` constant, retired once), hard error not masked,
  resize never detaches inside the callback, `GetProjectionRaw` pin, theater/dim via `FadeGrid/FadeToColor`.
- Ring on WARP: admission, exhaustion (third pair refused until `release_pair`), conversion pixel check, sentinel-state
  restore after `ExecuteCommandList(…, TRUE)`, command-list/SRV cache counters, quarantine on source swap.
- Runtime selection as a pure function with a truth table; elevated CI runners behave differently (high integrity).

---

## 15. Pitfall checklist (all learned the hard way)

- Forced `SetMultithreadProtected(TRUE)`; polling D3D queries from Present on a single-threaded device; holding the GPU gate across a blocking runtime call.
- Treating the last created device as the game's; promoting on a `DXGI_PRESENT_TEST` or failed Present; not checking HWND/device identity (mirror windows, launchers).
- Un-hooking or creating a detour from inside the detour; unsubscribing listeners during an in-flight Present.
- Doing resize/device-loss work inside the `ResizeBuffers` callback (or after the original call).
- LUID mismatch on hybrid GPUs; creating a DXGI 1.0 factory for OpenVR; `SINGLETHREADED` devices.
- `XR_RUNTIME_JSON` under elevation; incompatible implicit layers; holding OpenVR and OpenXR connections at once.
- Re-sampling poses between render and submit; acquiring both eyes before waiting; sharing a pair slot across pose families.
- Reusing a texture after a failed/unknown retirement; clearing the first error during cleanup; calling `xrEndSession` outside STOPPING.
- Calling both `Submit` eyes partially and then reusing the pair; clamping recommended OpenVR dimensions; negating `GetProjectionRaw` tangents.
- Showing the desktop image or the other eye while a pair is missing.
