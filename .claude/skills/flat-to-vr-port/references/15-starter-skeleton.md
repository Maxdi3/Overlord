# 15 — Starter skeleton for a new VR port

A minimal layout and the **non-negotiable plumbing** to start a new game port in the
Overlord style. It is meant to be grown, not shipped as is. Fill in the `TODO(RE)`
items from the reverse-engineering playbook (03). Two entry variants are shown:
**proxy DLL** (simplest) and **host executable** (Overlord's, for a deterministic
base, launcher and import control; details in 02).

The plumbing rules the skeleton enforces:

1. One exact build is pinned; anything else disables VR and runs the flat game.
2. Addresses are **pinned constants**, found offline (or by a one-time runtime code
   dump), and verified with masked bytes at install. **No runtime scanning.**
3. Hooks install **all-or-nothing** per feature, once, at a known "game code is ready"
   point, with relays allocated before any site is patched.
4. Native ABI facts live in one typed table; call-site addresses live in the adapter
   that owns them.
5. Every piece of code knows which execution context it runs on.
6. Observation (census) precedes modification; evidence is bounded and non-blocking.

```
mygame-vr/
  premake5.lua            # or CMake: x64, C++20/latest, static CRT, /bigobj
                          # Release: /GL- /LTCG:OFF (see 02 s16); RelWithDebInfo for play/perf
                          # host-exe variant only: /base:<free base> /DYNAMICBASE:NO /SAFESEH:NO /LARGEADDRESSAWARE
  deps/                   # minhook, OpenXR-SDK (built as app-local loader), gsl, asmjit; capstone (tools only)
  src/
    dllmain.cpp           # PROXY: exports (dxgi.dll/version.dll) + bootstrap     | HOST: main.cpp loader (02 s2)
    loader/               # HOST variant: binary_cache (lease), loader (imports/TLS), tls_bootstrap, target_identity
    loader/component_loader.{hpp,cpp}   # post_start/post_load/post_unpack/pre_destroy/load_import/is_supported
    common/hook.{hpp,cpp}               # detour (RAII), call/jump (throw if >2GB), far_call, preserving near jump, assemble
    common/hook_validation.{hpp,cpp}    # verify_masked_bytes, validate_executable_target
    common/native_memory.hpp            # SEH-guarded bounded reads (destructor-free leaf)
    common/io.hpp                       # write_file_atomic
    common/startup_callbacks.hpp        # startup-only, allocation-free dispatch list
    game/
      target.hpp          # build pin: size/hash/sentinel; image base; struct static_asserts
      entrypoints.hpp     # typed native signatures: function<R(Args...)>(addr), read_only_global<T>(addr)
      structs.hpp         # only proven fields, __pad_ regions with evidence comments, static_assert(offsetof)
    component/
      scheduler.cpp       # pipelines bound to engine frame functions (02 s9)
      console.cpp         # bounded non-blocking log + command registry (vr_status, vr_reinit, vr_recenter, vr_diagnose)
      d3d11.cpp           # import substitution, candidate/promoted device, Present/Resize detours, events
      d3d11/swap_chain_hook.cpp
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
        diagnostics/            # status sections, support bundle, evidence (11)
  tools/                  # OFFLINE only: census/contract generator (Capstone), crash-evidence reader, packaging
  tests/                  # CPU tests for math/policies, bridge tests with asmjit, mock OpenXR loader (11)
  data/vr_input/          # OpenVR manifests if supporting the backup
  docs/target.md          # 00 section 10
```

---

## 1. Pin the build, fail closed

```cpp
// game/target.hpp
namespace target {
  inline constexpr std::uint64_t image_base = 0x140000000;   // TODO(RE)
  inline constexpr std::uint64_t file_size  = 0;             // TODO(RE) exact size
  inline constexpr std::uint32_t sentinel_rva = 0, sentinel_value = 0;   // TODO(RE)
  bool verify_build();   // size + SHA-256 of file, PE identity, then GUARDED read of the sentinel on the mapped image
  bool ready();          // set only after verify_build() passed; adapters check this before patching
}
```

`verify_build` failing means: log one line, return, **do not install anything** (the
game runs flat). Record identity (hash, PE timestamp, `SizeOfImage`) in a snapshot the
status report prints. Use a guarded read for the sentinel (Overlord's raw dereference
is a known shortcut, 02 section 2.6).

---

## 2. Bootstrap

### 2a. Proxy DLL

```cpp
BOOL APIENTRY DllMain(HMODULE m, DWORD reason, LPVOID) {
  if (reason == DLL_PROCESS_ATTACH) {
    DisableThreadLibraryCalls(m);
    load_system_proxy_target();                 // forward exports to System32 dxgi.dll
    if (!target::verify_build()) { log("unsupported build; VR disabled"); return TRUE; } // fail closed
    MH_Initialize();
    component_loader::post_start();             // host-level setup, no game code patched yet
    // Install hooks that must precede device creation (D3D11CreateDevice via your own export/IAT) now.
    // Game-code hooks wait for the LATE TRIGGER: the game has finished unpacking/decrypting.
    //   proxy options (pick one, record it as a decision): first call into one of your forwarded exports,
    //   first Present, or a hooked well-known init import. Fire component_loader::post_unpack() exactly once.
  }
  return TRUE;
}
```

Keep `DllMain` free of heavy work (loader lock): defer to the trigger or a worker.
If the game's code is encrypted, take the **runtime code snapshot** at the trigger,
before your own patches (03 section 3.2).

### 2b. Host executable (Overlord's flow, summarised from 02 section 2)

```
main(): launcher-vs-game mode -> limit_parallel_dll_loading (IFEO MaxLoaderThreads=1) ->
  post_start -> resolve init-only settings (they decide which hooks install) ->
  load_binary: read_original (exact size) -> capture_original (SHA-256) -> patch_aslr (in memory) ->
               binary_cache::prepare (lock file, 64 KiB compare, atomic write, FILE_SHARE_READ lease) ->
               LoadLibrary -> require mapped base == image_base -> load_imports (resolver) -> load_tls ->
  verify_version (sentinel) -> post_load (drops unsupported components) -> entry point
  game's first GetProcAddress("InitializeCriticalSectionEx") -> post_unpack (install hooks, once)
```

---

## 3. Components

```cpp
class component_interface {
public:
  virtual ~component_interface() = default;
  virtual void post_start()  {}
  virtual void post_load()   {}
  virtual void post_unpack() {}                 // install hooks here
  virtual void pre_destroy() {}                 // idempotent, reentrancy-safe
  virtual void* load_import(const std::string& lib, const std::string& fn) { return nullptr; }
  virtual bool is_supported() { return true; }  // gate per feature/build
};
```

One feature = one file = one component. Registration order is not a contract.
`pre_destroy` is idempotent, waits if called from another thread, and swallows
exceptions per component. Retire D3D/XR owners **before** the engine unloads driver
libraries (`R_SyncRenderThread()` then `pre_destroy()` at the engine's quit entry
points).

---

## 4. Fail-closed hook install (feature level)

```cpp
struct site { uintptr_t addr; std::span<const uint8_t> bytes, mask; };   // addr is an absolute VA (or base+RVA)

bool install(const char* feature, std::initializer_list<site> checks, auto&& do_install) {
  if (!target::ready()) return false;
  for (const auto& s : checks) {
    const auto r = hook_validation::verify_masked_bytes((void*)s.addr, {s.bytes.data(), s.mask.data(), s.bytes.size()});
    if (!r) { log("[%s] contract mismatch at 0x%llX (%s); feature disabled", feature, s.addr, to_string(r.status)); return false; }
  }
  return do_install();        // allocate relays FIRST; patch only after every allocation succeeded
}

void post_unpack() override {
  // verify: the 5 bytes at the call site (E8 rel32, full mask), the decoded target == expected callee,
  // the callee prologue, executable target, stub within rel32 range (or a relay)
  install("camera", {{kCamCall, e8, ff5}, {kCamCallee, prologue, ff9}},
          [&] { hook::call(kCamCall, &set_view_stub); return true; });
}
```

Rules: verify **all** sites of a feature before patching **any**; a failed optional
sub-feature disables only itself; one site of a multi-site policy skipped = half-working
feature. For ABI/format-changing patches (arena resize) use the transactional pattern
in 02 section 6.2 (verify all, write all, re-verify each, rollback in `finally`), and
generate the table **offline** (03 section 4).

Choose the relay by register liveness (02 section 4.1): direct `rel32` if in range,
far relay only if RAX is dead, `create_preserving_near_jump` for mid-function sites,
`detour` for entry points. **Test every mid-function bridge with a hostile callee**
(02 section 4.3).

---

## 5. Typed native entry points

```cpp
// game/entrypoints.hpp: ABI facts only. No scanning, no admission, no hook install, no caching.
template <class Sig> class function;
template <class R, class... A> class function<R(A...)> {
  std::uintptr_t addr_;
public:
  explicit constexpr function(std::uintptr_t a) : addr_(a) {}
  R operator()(A... a) const { return reinterpret_cast<R(*)(A...)>(addr_)(a...); }
};
template <class T> class read_only_global {                 // a readable pointer != readiness or ownership
  std::uintptr_t addr_;
public:
  explicit constexpr read_only_global(std::uintptr_t a) : addr_(a) {}
  const T* get() const { return reinterpret_cast<const T*>(addr_); }
};

inline constexpr function<int(const void* player_state, std::uint32_t weapon, bool alt)> clip_capacity{0 /*TODO(RE)*/};
```

Callers use it; the adapter that owns a feature still verifies the prologue and the
thread/lifetime before calling. Prefer *live* queries (capacity, selection) over
static definition data. Put a `static_assert(offsetof(...))` on every struct field you
read and cite the evidence; trust asserts over comments.

---

## 6. Guarded reads and struct discipline

```cpp
static_assert(sizeof(game::usercmd_s) == 64);                       // TODO(RE)
static_assert(offsetof(game::playerState_s, viewangles) == 0x108);  // TODO(RE): cite disassembly address

playerState_s ps{};
if (!native_memory::read_at(ps_base, 0, ps)) return;   // SEH-guarded, bounded; readable != alive,
                                                        // pair with generation/identity (entity number + spawn generation)
```

---

## 7. Execution contexts (scheduler)

```cpp
enum class pipeline { async, renderer, lui /*if the engine has a UI VM*/, server, main };
scheduler::loop(fn, pipeline::server, 10ms);     // game-state writes go here, via script calls where possible
scheduler::once(fn, pipeline::main);
assert(scheduler::is_executing(pipeline::main)); // dispatcher scope, NOT a thread id
```

Bind each pipeline to an engine frame function by detour (renderer: just *before* the
original end-of-frame; server/main: after the original). Tasks added during execution
go to a separate list merged at the start of the next execute. Keep D3D immediate-
context calls (including copy-completion queries) on the renderer pipeline. Name the
engine's threads.

---

## 8. Present-driven frame loop (sketch)

```cpp
void on_present_pre(const present_event& e) {
  std::scoped_lock lock(xr_mutex);
  if (!maintain_session(e.graphics) || !frame_open) return;
  if (!pair_published(current_pair)) { if (++pending > 4) end_frame_no_world(); return; }   // never half a pair
  for (int eye = 0; eye < 2; ++eye) {
    acquire_wait(eye);                                  // xrAcquire/xrWaitSwapchainImage
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
  locate_views(); locate_head(); sample_actions();      // sample ONCE into an immutable snapshot
  head_pose_bridge::publish(head_midpoint);
  controller_input::publish(frame);
  stereo::publish_view_family(++current_pair, eye_offsets, tangents);
  frame_open = true;
}
```

Notes: `on_present_pre/post` come from the swap-chain hook, which only fires for the
game's real chain after a successful, non-`DXGI_PRESENT_TEST` Present; `ResizeBuffers`
is notified *before* the original so you can release backbuffer references. Copy-
completion polling (`GetData`) runs on the renderer pipeline, not here (02 section 9).
The XR calls themselves are in 04.

---

## 9. Camera boundary (example for a Quake-like engine)

```cpp
void set_view_stub(void* a1) {
  // The engine has filled refdef.origin/axis from the player state at this point.
  camera_request req = arbitrate_owner(/* scripted, sniper, mounted, remote, ladder, optic */);   // one place
  if (head_pose_bridge::apply_camera(refdef->origin, refdef->axis, cmd_time, view_angles, req))
    lod_origin_global = refdef->origin;     // keep LOD/cull/streaming consistent for both eyes
  original_set_view(a1);                    // mutate FIRST, then call the original
}
```

---

## 10. Observe-only census first

Before any modifier: one-shot immutable captures (bytes before/after the boundary call),
per-hook counters with thread ids, caller `_ReturnAddress()`, a bounded non-blocking
ring (try-lock, drop and count), and an offline analyser that excludes intervals across
loss windows. Turn the hook into a modifier only after the census proves the boundary;
leave `state=retired` notes for dead ends (03 sections 1, 5, 8).

---

## 11. Tests and tooling from day one

- CPU tests for math/policies; each hook bridge gets a hostile-callee asmjit test; a
  synthetic-PE test for the loader (host variant); a mock `openxr_loader.dll` for the
  runtime (11).
- A **feature-parity audit** of the built exe/PDB (entry points exist as emitted
  functions; mutable publication buffers live in writable sections) catches compiled-out
  components and WPO miscompiles.
- Offline `tools/` for the census/contract generator and the crash-evidence reader.

---

## 12. First three days plan

1. **Day 1:** identity pin, bootstrap, logging, component loader, device and Present
   hooks (import substitution or vtable), OpenXR coloured frames (M1-M2). Record the
   late trigger and thread map.
2. **Day 2:** capture a frame; find the camera struct, scene-render call, view slot;
   observe-only census (M4 start). Decide strategy A vs B from evidence on re-entrancy
   (03 section 8).
3. **Day 3:** derive eye views and run the engine per eye (B only if proven re-entrant,
   otherwise plan A with record-local derivation); copy to the swapchain; verify
   parallax; run a capacity audit (engine logs for `exceeded / not drawing`).

Release checklist items to start early: per-dependency licence files, never ship game
assets, never ship PDBs, version/provenance from one source file (`version.json`;
Git metadata is diagnostic only).
