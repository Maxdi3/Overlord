# 02 — Loading the game and hooking infrastructure

## Option 1: Host-executable loader (Overlord / H2-Mod / IW6x / S1x lineage)

The mod *is* the executable. It maps the original game into itself:

1. Read the original game EXE and verify its exact size or hash
   (`binary_cache::read_original`).
2. Clear `IMAGE_DLLCHARACTERISTICS_DYNAMIC_BASE` in an in-memory copy
   (`binary_cache::patch_aslr`) and write it atomically to a cache path
   (`%LOCALAPPDATA%/<mod>/bin/<game>.exe`). Lock it while in use, and replace it
   if it's stale.
3. Link the host with `/base:0x160000000 /DYNAMICBASE:NO` so the game's
   preferred base `0x140000000` is free (`std_include.cpp` pragmas).
4. `LoadLibrary` the cached EXE (`loader::load_library`). Verify the
   mapped base equals the required base, or abort before patching anything
   (`image_contract.hpp`).
5. Walk the import table yourself (`loader::load_imports`). Each import goes
   through a resolver that lets components substitute functions
   (`component_loader::load_import`). Overlord substitutes `D3D11CreateDevice`,
   `CreateDXGIFactory`, `ExitProcess`, `GetProcAddress`, and
   `SystemParametersInfoA`.
6. Initialise the game's TLS (`loader::load_tls`, `tls_bootstrap.cpp`), since
   a DLL-loaded EXE does not get static TLS automatically.
7. Use a late trigger to install hooks after an unpacker/DRM stub has run. Overlord
   uses the game's first `GetProcAddress("InitializeCriticalSectionEx")` call
   to fire `component_loader::post_unpack()`.
8. Verify the version sentinel, run `post_load`, and call the game entry point.

Pros: total control, deterministic base, no injector. Cons: more setup, and
DRM stubs may detect the unusual load.

## Option 2: Proxy DLL (most common for other games)

Drop a `dxgi.dll` / `d3d11.dll` / `version.dll` / `winmm.dll` next to the EXE.
It forwards all exports to the system DLL and runs init in `DllMain` (defer
heavy work to a thread or to the first forwarded call). Compute
`base = GetModuleHandle(nullptr)` and use RVAs: `addr = base + rva`.

## Option 3: External injector

`CreateProcess(SUSPENDED)` → `LoadLibrary` via remote thread/APC → resume. Same
RVA approach as option 2.

## Component system (copy this pattern)

```cpp
class component_interface {
public:
  virtual ~component_interface() = default;
  virtual void post_start() {}      // host started, game not loaded
  virtual void post_load() {}       // game mapped, entry not yet called
  virtual void post_unpack() {}     // game code unpacked: install hooks here
  virtual void pre_destroy() {}     // idempotent shutdown
  virtual void* load_import(const std::string& lib, const std::string& fn) { return nullptr; }
  virtual bool is_supported() { return true; }
};
#define REGISTER_COMPONENT(T) namespace { static component_loader::installer<T> __component; }
```
Each feature is one file with one component. Ordering is registration order.
Exceptions in `post_start`/`post_load` abort startup with a message box.
`pre_destroy` swallows exceptions per component.

## Hook primitives (`src/common/utils/hook.hpp`)

| Primitive | Use |
|---|---|
| `detour d; d.create(addr, stub); d.invoke<R>(args...)` | Function-entry detour (MinHook-like trampoline). Use for functions with many callers. |
| `hook::call(call_site, stub)` | Rewrite one `E8 rel32` call instruction. Use when only *one* caller should be intercepted. Overlord uses this a lot, e.g. `camera.cpp: utils::hook::call(0x1403ACB2D, set_viewpos_now_stub)`. |
| `hook::jump(site, target)` | Redirect a function or block. |
| `hook::nop(site, len)` / `hook::set<T>(addr, value)` / `hook::copy` | Patch bytes or data. |
| `get_vtable_entry(obj, &Iface::Method)` | Find a COM vtable slot portably. |
| `create_preserving_near_jump(base, target)` | A RIP-indirect relay that changes no registers or flags, for mid-function CALL sites with live RAX. Overlord uses it for render call sites far from the hook code. |
| `assemble(lambda)` | asmjit for custom mid-function stubs (save/restore volatile registers). |
| `signature("48 8B ?? ?? E8", module).process()` | Pattern scanning when addresses must survive minor updates. |

Call-site rewriting needs the stub within ±2 GB of the site, or a near relay
(the H2-Mod host is mapped close to the game for this reason).

## Byte verification before every hook (mandatory)

```cpp
template <size_t N>
bool verify(uintptr_t address, const uint8_t (&bytes)[N]) {
  std::array<uint8_t, N> mask{}; mask.fill(0xFF);  // 0x00 = wildcard (relocations, rel32)
  return bool(utils::hook_validation::verify_masked_bytes(
      reinterpret_cast<void*>(address), {bytes, mask.data(), N}));
}
// install:
if (!verify(0x140518B91, parameter_call) || !verify(parameter_address, parameter_entry)) {
  console::warn("[VR fire] contract mismatch; feature disabled\n");
  return;                                  // fail closed: original game behaviour
}
utils::hook::call(0x140518B91, parameters_stub);
```
Also `validate_executable_target(addr)` checks that the page is committed and
executable. Verify the call instruction at the site *and* the callee's prologue.
Overlord's `weapon_interaction.cpp` verifies 6+ sites before installing 2 hooks.

## Guarded native memory reads

Use `utils::native_memory::read_bytes(&out, ptr, size)` / `read_at(base, offset, out)`
for any pointer chase in hooks that might see freed or garbage memory.
Readability is **not** proof of lifetime or ownership, so pair it with generation
or identity checks (entity number + spawn generation, object epoch fields).

## Struct layout discipline

```cpp
static_assert(sizeof(game::usercmd_s) == 64);
static_assert(offsetof(game::usercmd_s, buttons) == 4);
static_assert(offsetof(game::playerState_s, viewangles) == 0x108);
```
Put a `static_assert` on every offset you rely on, and document how each was
confirmed (disassembly reference, capture).

## Typed native entry points

Keep one header that declares each native function once, with a fixed signature
and address (`vr/h2/entrypoints.hpp`, `binding.hpp`). Consumers call through it.
Call-site addresses belong to the hook adapter that owns them, not to the
shared table.

## Scheduler

H2-Mod's `scheduler` hooks the game's frame functions and offers pipelines
(`main`, `renderer`, `server`, `async`, ...): `scheduler::loop(fn, pipeline, interval)`,
`once`, `on_game_initialized`. Use it to run code on the right engine thread
without new hooks.

## Thread-local reentrancy guards

Every stub that might be re-entered (skeleton calc, render owner) needs a
`thread_local bool in_hook` guard and a `gsl::finally` restore. See
`hands/component.cpp: calc_skeleton_stub`.
