# 04 — D3D11 interception and the OpenXR runtime

Overlord files: `component/d3d11.cpp`, `d3d11/swap_chain_hook.cpp`,
`vr/openxr_runtime.cpp`, `openxr_dispatch.cpp`, `openxr_d3d11.cpp`,
`openxr_input.cpp`, `openxr_menu.cpp`, `texture_blit.cpp`,
`native_render_session.cpp`, `runtime_backend.cpp`, `present_transaction.hpp`.

## 1. D3D11 layer

Expose a small event API so VR code never touches vtables:
```cpp
namespace d3d11 {
  struct device_snapshot { ComPtr<ID3D11Device> device; ComPtr<ID3D11DeviceContext> context;
                           uint64_t generation; /* bumps on every device (re)creation */ };
  struct present_event { device_snapshot graphics; IDXGISwapChain* swap_chain; uint64_t frame_index; };
  listener_token subscribe_device_created(fn); subscribe_device_destroying(fn);
  subscribe_present_pre(fn);  subscribe_present_post(fn /* (event, HRESULT) */);
  subscribe_resize_before(fn);
}
```
- Replace `D3D11CreateDevice` through the import resolver. Record the adapter, and
  enable `ID3D11Multithread::SetMultithreadProtected(TRUE)` (Overlord records whether
  this succeeded).
- Swapchain hook: get the Present (slot 8) and ResizeBuffers (slot 13) pointers
  from a dummy swapchain, then detour them. Track "inside callback" state so
  shutdown can wait.
- A **device generation** counter guards every resource you create, so
  nothing created on an old device is ever used.
- A **GPU queue scope** (`d3d11::acquire_gpu_queue_interop()`) serialises your
  immediate-context work against the game's and the runtime's.

## 2. Loading OpenXR

Load `openxr_loader.dll` (shipped beside the EXE, built from the pinned SDK) with
`LoadLibrary`, then fill a dispatch table with `xrGetInstanceProcAddr`
(`openxr_dispatch.cpp`). Benefits: no hard import (the game still starts without
VR), and mocks can be injected in tests.

Runtime selection: respect `XR_RUNTIME_JSON` if set. Otherwise use the system
active runtime. Only set an override for the duration of initialization, and
restore it afterwards. Report `runtime_manifest`, `runtime_library` and `runtime_selection`.

## 3. Initialization sequence

```cpp
// Instance
const char* exts[] = { XR_KHR_D3D11_ENABLE_EXTENSION_NAME /*, XR_KHR_COMPOSITION_LAYER_CYLINDER if present */ };
XrInstanceCreateInfo ici{XR_TYPE_INSTANCE_CREATE_INFO};
strcpy(ici.applicationInfo.applicationName, "MyGameVR"); ici.applicationInfo.apiVersion = XR_API_VERSION_1_0;
ici.enabledExtensionCount = n; ici.enabledExtensionNames = exts;
xrCreateInstance(&ici, &instance);

XrSystemGetInfo sgi{XR_TYPE_SYSTEM_GET_INFO}; sgi.formFactor = XR_FORM_FACTOR_HEAD_MOUNTED_DISPLAY;
xrGetSystem(instance, &sgi, &system);              // XR_ERROR_FORM_FACTOR_UNAVAILABLE → "HMD not connected"

// Graphics requirements: MUST match the game's adapter
XrGraphicsRequirementsD3D11KHR req{XR_TYPE_GRAPHICS_REQUIREMENTS_D3D11_KHR};
pfnGetD3D11GraphicsRequirementsKHR(instance, system, &req);
// compare req.adapterLuid with IDXGIAdapter::GetDesc().AdapterLuid of the GAME device;
// mismatch → fail with "game and VR runtime use different GPUs" (graphics_mismatch)
// compare device->GetFeatureLevel() >= req.minFeatureLevel

XrGraphicsBindingD3D11KHR binding{XR_TYPE_GRAPHICS_BINDING_D3D11_KHR}; binding.device = gameDevice;
XrSessionCreateInfo sci{XR_TYPE_SESSION_CREATE_INFO}; sci.next = &binding; sci.systemId = system;
xrCreateSession(instance, &sci, &session);

// Spaces: LOCAL for world, VIEW for head
XrReferenceSpaceCreateInfo rs{XR_TYPE_REFERENCE_SPACE_CREATE_INFO}; rs.poseInReferenceSpace.orientation.w = 1;
rs.referenceSpaceType = XR_REFERENCE_SPACE_TYPE_LOCAL; xrCreateReferenceSpace(session, &rs, &local);
rs.referenceSpaceType = XR_REFERENCE_SPACE_TYPE_VIEW;  xrCreateReferenceSpace(session, &rs, &view);

// Views / swapchains (PRIMARY_STEREO)
xrEnumerateViewConfigurationViews(... XR_VIEW_CONFIGURATION_TYPE_PRIMARY_STEREO ...); // recommended w/h per eye
xrEnumerateSwapchainFormats(...); // prefer DXGI_FORMAT_R8G8B8A8_UNORM_SRGB or R16G16B16A16_FLOAT
XrSwapchainCreateInfo sc{XR_TYPE_SWAPCHAIN_CREATE_INFO};
sc.usageFlags = XR_SWAPCHAIN_USAGE_COLOR_ATTACHMENT_BIT | XR_SWAPCHAIN_USAGE_SAMPLED_BIT;
sc.format = chosen; sc.sampleCount = 1; sc.width = w; sc.height = h; sc.faceCount = 1; sc.arraySize = 1; sc.mipCount = 1;
xrCreateSwapchain(session, &sc, &eye[i].handle);  // then xrEnumerateSwapchainImages → XrSwapchainImageD3D11KHR → RTVs
xrEnumerateEnvironmentBlendModes(...); // choose OPAQUE
// actions: see 07 (xrCreateActionSet, xrSuggestInteractionProfileBindings, xrAttachSessionActionSets)
```
Session state machine via `xrPollEvent`: `READY` → `xrBeginSession(PRIMARY_STEREO)`.
`STOPPING` → `xrEndSession`. `EXITING`/`LOSS_PENDING` → tear down and mark reinit
pending. `FOCUSED` gates input. `VISIBLE`-only means render but neutralise input.
`XrEventDataReferenceSpaceChangePending` → request recenter at `changeTime`.

## 4. The frame loop belongs inside the game's Present

Overlord's contract (read `openxr_runtime.cpp`: `on_present` → `complete_native`
→ `on_present_post` → `begin_prediction`):

```
Present-post(frame N)  (game just presented)
  └ begin_prediction(pair=N):
      xrWaitFrame → frameState.predictedDisplayTime
      xrBeginFrame
      if !shouldRender: end frame with 0 layers, invalidate poses, return
      xrLocateViews(LOCAL, predictedDisplayTime) → eye poses + FOVs
      xrLocateSpace(VIEW in LOCAL)              → head validity gate
      head_render = midpoint(eye positions), slerp(eye rotations, .5)
      reject canted displays: |dot(qL,qR)| < 0.99999 → unsupported (shared-camera engines)
      publish head pose (head_pose_bridge::publish_tracking_pose)
      sample actions (xrSyncActions, xrLocateSpace grip/aim at the same time)  → controller_input::publish
      eye offsets = R_head^T · (eye_pos - head_pos)  (meters, head-local)
      projections = {tan(angleLeft), tan(angleRight), tan(angleDown), tan(angleUp)}
      admit native pair N; publish view family (pair N, offsets, projections) to the stereo renderer
… game simulates and renders frame N+1, stereo renderer renders BOTH eyes of pair N with those views …
Present-pre(frame N+1)
  └ complete_native():
      if pair not yet published: wait at most ~4 Presents, then end the frame with no world layer
      acquire published eye textures of pair N
      per eye: xrAcquireSwapchainImage → xrWaitSwapchainImage(10ms) → blit → Flush → xrReleaseSwapchainImage
      prepare menu layers (quad/cylinder)
      xrEndFrame(displayTime = predicted, layers = [projection(views=SDK eye poses+FOVs), menus…])
      retire pair N (release ring slot)
```
Key properties:
- **Same prediction for render and submit**: the projection layer uses the *original
  SDK eye poses and FOVs* from `xrLocateViews` for that pair. Never re-sample.
- **Present ownership**: verify that pre/post occur on the same thread, frame index
  and device generation (`present_transaction::validate`). A mismatch is fatal
  for VR (the game keeps running flat).
- Runtime waits (`xrWaitFrame`, `xrWaitSwapchainImage`) never run inside your
  GPU-queue scope.
- Not ready yet (bootstrap/content proof pending) → end the frame with no world
  layer. Never show the desktop image.

## 5. Eye texture transfer

- Overlord copies each eye's final engine output into a ring of
  `R16G16B16A16_FLOAT` linear textures (`native_render_session`), then a
  state-restoring fullscreen blit (`texture_blit.cpp`) writes the swapchain image.
  If the swapchain is `_SRGB`, write linear values and let the hardware encode them.
- Same device, non-shared textures. Validate size, format, mips=1, array=1,
  samples=1, DEFAULT usage, RTV|SRV binding, no CPU access. A mismatch rejects the pair.
- **Save and restore all pipeline state** around any draw you issue on the
  game's immediate context (IA/VS/PS/OM/RS/viewports/scissors/samplers/CBs).
- A pair is retired only after `xrEndFrame` or, for OpenVR, the next successful
  `WaitGetPoses`. If retirement fails, quarantine the slot. Never reuse it.

## 6. Resize and device loss

`ResizeBuffers` only queues the device generation. At the next real Present-pre,
retire submitted targets, reset temporal history and recreate the VR session/ring.
On device recreation, try exactly one auto-init per generation.

## 7. Menu/compositor layers

`XrCompositionLayerQuad` or `XrCompositionLayerCylinderKHR` (if the extension
is present) in LOCAL space, positioned from the head pose when the menu opens.
Overlord's captured UI is premultiplied and gamma-encoded, so it is decoded to
linear before composition. Set `XR_COMPOSITION_LAYER_BLEND_TEXTURE_SOURCE_ALPHA_BIT`
for translucent UI. Add `XR_COMPOSITION_LAYER_UNPREMULTIPLIED_ALPHA_BIT` **only**
if your texture is *not* premultiplied. In frontend/menu-only
or movie mode, submit UI layers without a world projection layer.

## 8. OpenVR backup (if needed)

`VR_Init(VRApplication_Scene)` → `WaitGetPoses` at the Present boundary →
`Submit(Eye_Left/Right, Texture_t{DirectX, ColorSpace})`. SteamVR Input needs an
absolute action-manifest path resolved from the installed EXE location
(`data/vr_input/actions.json` plus per-controller binding JSONs).
`DoNotHaveFocus` is recoverable: retain the pair until the next successful
`WaitGetPoses`. Register a `.vrmanifest` so SteamVR lists the app.

## 9. Status you should expose (`vr_status`)

backend, runtime name, system name, session state, init stage, last error
(operation + XrResult name), adapter match, swapchain format/size, frames
submitted/world/menu-only/empty, pairs admitted/published/retired/quarantined,
tracking sample count, controller profile per hand, and pose validity flags.
