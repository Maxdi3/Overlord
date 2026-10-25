# 02 — Loading the game and hooking infrastructure

Phase 1 deliverable: your code runs inside the game process, the exact target
build is proven, hooks install **all-or-nothing**, and you know which engine
thread every piece of your code runs on. Nothing in this file is VR-specific;
everything in later references depends on it.

**Overlord policy in one line:** pin one exact executable, scan for addresses
*offline*, and at runtime only use pinned addresses guarded by masked-byte
verification. Never scan-and-patch at runtime. Never fall back to another
build's addresses. If any check fails, the feature stays off and the game
behaves as the flat original.

Contents: 1 loading strategies · 2 Overlord's loader step by step · 3 component
system · 4 hook primitives and the four relay tiers · 5 verification and guarded
reads · 6 fail-closed transactional install · 7 the no-runtime-scan policy ·
8 struct and entry-point discipline · 9 scheduler / execution contexts ·
10 reentrancy and deferred teardown · 11 D3D11/DXGI interception at the hook
layer · 12 engine capacity expansion · 13 script-VM hook points · 14 other
engine-service patches · 15 bugs not to copy · 16 build configuration ·
17 Overlord file pointers.

---

## 1. Loading strategies

| Option | Mechanism | Use when | Cost |
|---|---|---|---|
| **A. Host executable** (Overlord, H2-Mod/IW6x/S1x lineage) | Your exe maps the original game exe as a DLL at its preferred base, rewrites imports/TLS itself, then calls its entry point. | You want a deterministic base, a launcher, no injector, and control of every import. | TLS/imports/SEH/DRM-stub quirks you must handle yourself (section 2). |
| **B. Proxy DLL** (`dxgi.dll`, `d3d11.dll`, `version.dll`, `winmm.dll` next to the exe) | Forward all exports to the system DLL; init from `DllMain` (defer heavy work). `addr = GetModuleHandle(nullptr) + rva`. | Unprotected game with ASLR or a stable base, simplest start. | No import substitution; you need another "late trigger" (below) if the exe unpacks itself. |
| **C. External injector** | `CreateProcess(SUSPENDED)` -> remote `LoadLibrary` -> resume. | Proxy slot unavailable. | Same RVA approach as B; anti-cheat will object. |
| **D. Payload-in-image** (Overlord's compiled-out fallback) | Host links at the game's base with a reserved BSS payload section, `memmove`s the game's sections into itself, rebuilds imports and the exception table. | The game cannot be `LoadLibrary`'d at its preferred base. | You must register unwind data yourself (section 2.9). |

Whatever you choose, three questions must have written answers (put them in
`docs/target.md`, see 00): *how do I prove this is the exact build*, *when is
game code ready to patch* (unpacked/decrypted), and *which thread runs what*.

**Late-trigger rule.** Hooks on game code may only be installed after any
packer/DRM stub has finished. Overlord (host loader) uses the game's first
`GetProcAddress("InitializeCriticalSectionEx")` call as the signal. A proxy-DLL
port needs its own equivalent: the first call into one of your forwarded
exports, the first `Present`, or a hooked well-known init import. Pick one,
fire it exactly once (a `static bool handled` guard), and record the choice as
a *decision*. If the on-disk code is encrypted (H2's `.text` is), your masks
are compared against *decrypted runtime memory*; see 03 section 3.

---

## 2. Overlord's host loader, step by step (`src/client/main.cpp`, `src/client/loader/*`)

```
main()
 |- hide console, per-monitor-v2 DPI
 |- no "-singleplayer" flag? -> LAUNCHER mode (separate process role, never touches game/hooks/XR)
 '- game mode:
     limit_parallel_dll_loading()              IFEO MaxLoaderThreads=1 (2.4)
     component_loader::post_start()            host up, game not loaded
     game::environment::set_mode()
     launcher_vr_settings::initialize_startup_options()   freeze init-only settings (2.8)
     load_binary()                             2.1-2.3, 2.5-2.7
     verify_version()                          sentinel DWORD on the MAPPED image (2.6)
     component_loader::post_load()             clean() drops unsupported components first
     entry_point()                             game runs; first GetProcAddress("InitializeCriticalSectionEx")
                                               -> component_loader::post_unpack() (hooks installed here)
```

A `gsl::finally` runs `pre_destroy()` if startup fails early
(`premature_shutdown` flag), and another runs it after the entry point returns.
The game's `ExitProcess` import is substituted by `exit_hook`
(`pre_destroy()` then `exit(code)`); the game's own quit path calls
`pre_destroy` earlier at `Com_Quit_f`, `ExitProcess` is only the fallback.
Exceptions in `post_start`/`post_load` show a message box and return 1;
exceptions in `post_unpack` are caught inside the substituted `GetProcAddress`
(message box, `exit_hook(1)`) because that code is called from game code.

### 2.1 Read, identify, patch ASLR in memory

1. `binary_cache::read_original(path)`: hard-fails on exact size mismatch
   (`game_data::supported_binary_size`) and reads exactly that size.
2. `target_identity::capture_original`: SHA-256, file size, PE
   timestamp/`SizeOfImage`/checksum into a mutex-protected snapshot that the
   support bundle prints (it proves which exe ran).
3. `binary_cache::patch_aslr(data)`: validates DOS signature, `e_lfanew` bounds,
   NT signature, machine AMD64, PE32+ magic, **`ImageBase == required base`**, then
   clears `IMAGE_DLLCHARACTERISTICS_DYNAMIC_BASE` in the in-memory copy. Everything
   else stays byte-identical.

### 2.2 Cache with a lease (closes the check-to-map race)

`binary_cache::prepare(%APPDATA%/<mod>/bin/<game>.exe, data)`:

- A `<cache>.lock` file opened `GENERIC_WRITE`, share mode 0, serialises writers
  (and protects the atomic writer's sibling `.tmp`) *without* waiting on another
  running game.
- If the cache already matches byte-for-byte (size first, then 64 KiB chunked
  `memcmp`, so a giant corrupt cache cannot blow memory) reuse it. Otherwise
  `write_file_atomic` (tmp + `MOVEFILE_WRITE_THROUGH` rename), reopen, re-verify.
- Returns a **`lease`** holding the file handle open with `FILE_SHARE_READ` only.
  That denies write/delete/replace from the identity check until `LoadLibrary`
  has mapped the file (the lease object lives for the whole of `load_binary`).
- Errors are user-actionable ("Close other game instances and try again").
- Reusable for any "patched copy of the game exe" approach.

### 2.3 Map, then gate on the base before touching anything

`loader::load_library(cache)`: `LoadLibrary` the cached exe, then compare
`actual_base == preferred_base == required base` (`game_image::supported_mapping`,
`loader/image_contract.hpp`). On mismatch `free()` and throw **before imports or
TLS are patched**: every native hook uses absolute addresses, so a relocated
image is unusable. The host itself is linked `/base:0x160000000`
(`std_include.cpp`) so the game's preferred base is free, and with
`/DYNAMICBASE:NO` so the host is never randomised into it
(`premake5.lua`: `/DYNAMICBASE:NO /SAFESEH:NO /LARGEADDRESSAWARE /LAST:.main`,
16 MiB stack from `std_include`). A proxy DLL does not need these flags.

### 2.4 IFEO `MaxLoaderThreads=1`

`limit_parallel_dll_loading()` writes
`HKLM\Software\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\<host exe>\MaxLoaderThreads = 1`
(`REG_DWORD`). Needs admin; silently ignored if it fails. Purpose: the NT loader's
parallel worker threads race with a hand-patched IAT/TLS/hook install of an exe
mapped as a DLL. Cheap hardening for any DLL-loaded exe.

### 2.5 Imports through a resolver

`loader::load_imports` walks the IAT itself. For each import it asks the
resolver (`(dll, function)`; ordinals are `"#N"`). The host resolver substitutes
`ExitProcess`, `SystemParametersInfoA`, `GetProcAddress`, then delegates to
`component_loader::load_import`, which asks every component (last non-null
answer wins). Components substitute `D3D11CreateDevice` and `CreateDXGIFactory`
(section 11). Unresolved imports throw with dll and function name; the
fallback is `LoadLibrary` + `GetProcAddress`. IAT slots are written with
`utils::hook::set`. Ordinal lookups (`lpProcName <= 0xFFFF`) pass through
`get_proc_address` untouched.

### 2.6 Version sentinel (second-line identity)

After the file-level size/SHA check, `verify_version()` reads a DWORD at a fixed
address of the *mapped* image and compares it to a magic (H2-only:
`*(DWORD*)0x140123456 == 0xE465E151`), then
`target_identity::mark_compatibility_probe_passed()`. Components that patch
code later refuse to run unless that flag is set
(`scene_surface_storage.cpp`). Pitfall: the read is a raw dereference; it is
safe only because the fixed-base mapping already succeeded. In new code use a
guarded read (section 5.3).

### 2.7 TLS for an exe loaded as a DLL (two cooperating pieces)

The OS loader sets up static TLS (`__declspec(thread)`) only for the process
image and implicit DLLs, so a `LoadLibrary`'d exe has none.

1. **Donor DLL** (`src/tlsdll/dllmain.cpp`): a tiny DLL declaring
   `thread_local char tls_data[0x2000]`, embedded as a resource
   (`TLS_DLL`) and extracted to `<mod>-tlsdll.dll`. `tls::allocate_helper()`
   loads it (only one allocation per process; second call throws).
2. **`tls::initialize_game_tls(game, helper, thread_slots)`**
   (`loader/tls_bootstrap.cpp`): reads both `IMAGE_TLS_DIRECTORY64`s; requires
   `game.total_size <= helper.total_size`; takes the helper's TLS index; finds
   the *current thread's* block via `TEB->ThreadLocalStoragePointer`
   (`__readgsqword(0x58)`) + `index*8`; then (a) writes the helper's index into
   the **game's** `AddressOfIndex`, (b) copies the game's initialised template
   and zero-fill into the current thread's block and into the helper's template
   area (threads created later inherit it), (c) copies the game's directory
   over the helper's. TLS callbacks are not run. `has_template` is false when
   both VA and Size are 0 and throws if only one is.
   Safety: every range is bounds-checked, `VirtualQuery`-committed, not
   GUARD/NOACCESS; overflow-safe arithmetic; an RAII `write_access` changes
   protection per region (a template can straddle pages), finishes *all*
   `VirtualProtect` changes before the first write, restores in reverse, and
   throws if a restore fails. Tested with synthetic PEs (`tests/loader`).
3. **Inlined TLS accesses** (`component/tls.cpp`, H2-only second half): the
   game's optimiser dropped the `index*8` term in 66 hand-listed inlined
   accesses (`mov rax, gs:[0x58]; mov rax,[rax+...]`). At `post_unpack` each
   9-byte site is NOP'd and `far_call`ed into an assembled stub
   `mov rax, gs:[0x58]; add rax, tls_index*8; ret` with `tls_index` read from the
   game's TLS directory. Lesson: after hosting a game, grep its code for TLS
   accesses that assume index 0.

### 2.8 Settings that decide which hooks install

Anything that selects hooks/backends must be resolved in the host **before**
`load_binary` (`initialize_startup_options`; env var for the XR backend with
"launcher start: config wins; direct start with existing env: keep it").
Launcher edits the game's own config file (see 09 and 11 for launcher contracts).

### 2.9 Payload-in-image fallback and SEH (`loader::load`, `loader/seh.cpp`)

Compiled out here (`std_include.hpp` defines `INJECT_HOST_AS_LIB`; the
`.payload` section path is not built). Recipe if you ever need it: host linked
at the game's base with a `BINARY_PAYLOAD_SIZE` BSS section (H2: `0x12000000`);
`memmove` each section; rebuild imports; `RtlAddFunctionTable` the game's
`.pdata`, then patch ntdll's `DYNAMIC_FUNCTION_TABLE` entry and hook
`RtlpxLookupFunctionTable` (found by disassembling `RtlLookupFunctionTable` for
its single internal CALL) so C++/SEH unwinding works through manually placed
code; finally overwrite the host's NT headers with the game's. Hand-mapped
modules are also invisible in a normal minidump module list: add a custom
minidump stream with module bases, `.pdata` ranges, unwind info and code around
candidate return addresses (11 section on crash evidence).

### 2.10 Module identity when the host is a library (`component/game_module.cpp`)

With `INJECT_HOST_AS_LIB`, `GetModuleHandle{A,W,ExA,ExW}(NULL)` is hooked to
return the *game* image and `GetModuleFileName*` for NULL/game to return the
host path, so game code that resolves its own image or path (resource lookup,
self-checks) keeps working. See bugs (section 15).

### 2.11 Protection stubs: observe, do not defeat

`component/arxan.cpp` only snapshots the first 15 bytes of 14 ntdll `Dbg*`
entry points at `post_load`, re-compares once a second on the async scheduler,
logs, and refuses to restore live code (other threads may execute a partially
written instruction). It does not neutralise any check. H2's on-disk `.text`
is encrypted, so everything the project knows about code addresses came from
the running process (03 section 3). Do not defeat DRM to enable piracy;
only the owned, legitimately launched copy may run modded. Kernel anti-cheat or
online-only means stop (00).

### 2.12 Dev conveniences that pay for themselves

`startmap <map>` (engine spawn function after an exists-check) to skip menus;
god/noclip/notarget toggles; an in-process ImGui overlay (entity/asset/material
browsers, world-space debug draw, script console) installed lazily on the first
frame after assets are ready and drawn from a game frame call site. UI must
not mutate game state directly: queue work onto the owning pipeline
(section 9). The overlay borrows the engine's UI key-catcher bit, observes the
native catcher setters through byte-verified detours, and exposes
`captures_input()` so the VR pointer/injection stand down while it is open.

---

## 3. Component system (copy this pattern)

```cpp
class component_interface {
public:
  virtual ~component_interface() = default;
  virtual void post_start()  {}   // host up, game not loaded; may throw premature_shutdown_trigger
  virtual void post_load()   {}   // game mapped, entry not yet called
  virtual void post_unpack() {}   // game code unpacked: install hooks HERE (once)
  virtual void pre_destroy() {}   // idempotent shutdown
  virtual void* load_import(const std::string& lib, const std::string& fn) { return nullptr; }
  virtual bool is_supported() { return true; }   // per-feature / per-build gate
};
#define REGISTER_COMPONENT(T) namespace { static component_loader::installer<T> __component; }
```

Contract details (`loader/component_loader.cpp`):

- Each phase has a `static bool handled` guard, so it runs once.
- `post_load()` first calls `clean()`: components whose `is_supported()` is false
  get `pre_destroy()` and are erased. This is the per-feature/per-build gate.
- A component may throw `premature_shutdown_trigger` from `post_start`/`post_load`
  to exit cleanly (`main` returns 0, no error box). Other exceptions abort with a
  message box. In `post_unpack` an exception is fatal for that load.
- `pre_destroy` is a 3-state machine (`not_started/running/complete`) with mutex
  and condvar: a re-entrant call on the same thread returns immediately; a call
  from another thread **waits for completion**; per-component exceptions are
  caught and sent to `OutputDebugString`.
- The component vector is a leaked `new` (no static-destruction-order problems).
  Order is translation-unit static-init order, so never depend on it. Where
  order matters (two components touching one operand), make each hook robust to
  either order (section 4.8).
- `component_loader::get<T>()` uses `typeid` comparison.
- Fixed-capacity observer tables break at the (N+1)th registration: Overlord's
  zone-unload observer table of eight broke boot when a ninth presenter was
  added. `utils/startup_callbacks.hpp` is the fix: registration-order,
  de-duplicated, startup-only list of function pointers, read-only and
  allocation-free during dispatch, null rejected, late/concurrent registration
  unsupported. Use it instead of a general event bus.
- Other small primitives worth copying: `utils/latest_task_worker.hpp` (one
  running + one pending slot, latest wins, cooperative stop; for settings
  saves, preflight, scans) and `thread::set_name` (named threads make dumps and
  watchdog output readable).

---

## 4. Hook primitives and the four relay tiers (`src/common/utils/hook.hpp`)

| Primitive | Use |
|---|---|
| `detour d; d.create(addr, stub)`, `d.invoke<R>(args...)` | MinHook trampoline at a function entry. Use for functions with many callers or where you must call the original. |
| `hook::call(site, stub)` / `hook::jump(site, target)` | Rewrite one `E8`/`E9 rel32`. Intercepts only that caller. **Throws if stub is outside +-2 GB** (`is_relatively_far`), before writing. |
| `hook::far_call<Base>(site, dest)` / `far_jump` / `create_far_jump<Base>(dest)` | 12-byte `mov rax, imm64; jmp rax` thunk in a 4 KiB page placed near `Base`, de-duplicated per destination; the rel32 site points at the thunk. **Clobbers RAX.** |
| `create_preserving_near_jump(base, target)` | 6-byte `FF 25 00000000` + 8-byte pointer in a page near `base`, flipped to `PAGE_EXECUTE_READ`, icache flushed. **Changes no register or flag.** Returns null on allocation failure (callers must fail closed). |
| `assemble(lambda)` (asmjit) | Custom stubs/bridges. `pushad64/popad64` do **not** save flags or XMM and push `rsp` itself; `call_aligned` aligns to 16 and reserves 0x40 shadow space. Save flags/XMM explicitly. |
| `hook::nop/set<T>/copy/inject` | Patch bytes. All do `VirtualProtect(RWX)`, write, restore, `FlushInstructionCache`. `inject(ptr, data)` writes a bare rel32 operand. |
| `follow_branch`, `extract<T>` | Decode `E8/E9` to the real target (chase thunks). |
| `get_vtable_entry(obj, &Iface::Method)` | Portable COM slot index (calls the member-pointer thunk on a fake object whose vtable is an array of functions returning their index; x64 MSVC only; default window 100). |

### 4.1 Choosing a relay (decide per site: is any register live across it?)

1. **Entry detour** if many callers and registers are the ABI's.
2. **Direct rel32** if the stub is within +-2 GB of the site. JIT/asmjit memory
   usually is *not*: the first deployment of Overlord's equip hook failed at
   startup with a rel32 range error. Preflight `is_relatively_far` for **every**
   site before patching any.
3. **`create_far_jump` (RAX scratch)** is valid only when RAX is provably dead at
   the site (for example the bridge entry initialises RAX before calling the
   native selector).
4. **`create_preserving_near_jump`** for mid-function CALL sites with live RAX or
   flags. Camera-bob scalar sites needed it: the generic relay destroyed RAX
   (a dvar pointer) before the bridge could save it, reproduced as a regression
   test.

The patched CALL stays exactly 5 bytes; do not overwrite neighbouring bytes.
Allocate **relays before mutating any call site** ("ASLR must not turn a valid
adapter into a 32-bit relative-branch startup failure").

**`memory::allocate_near(address, size, protect)`** walks *downward* one page at
a time from `address` calling `VirtualAlloc` until one succeeds; it stops at
`lpMinimumApplicationAddress` and returns null. Consequences: relays always sit
*below* the game image (the host maps the game at its preferred base with free
space below), it can be slow in a fragmented space, and on another game you must
verify that region is free or search both directions.

### 4.2 Bridge ABI contract

An asmjit bridge that inserts a C++ policy call into a native function must
preserve: volatile GPRs (rax, rcx, rdx, r8-r11), flags, XMM0-5 (and upper lanes),
stack alignment (callee sees `rsp & 15 == 8` at entry; use `lea rsp,[rsp-0x28]`
not `sub`, so flags survive), shadow space and stack arguments. Only the
intended output (RAX or one scalar) may change. Some native leaves rely on
non-ABI contracts (a caller expecting RCX preserved): a plain C++ detour there
crashed at load; wrap them with a bridge. Document in a comment which registers
are free at each patch site. "Redirect call sites, not the function" when the
function has other meanings: only the *witnessed* call sites of a shared query
are redirected, the rest keep native semantics. A `thread_local` pointer can scope
a replaced getter to one outer call.

### 4.3 Test every bridge with a hostile callee

(`tests/vr/camera_bob_tests.cpp`, `animation_index_bridge_tests.hpp`,
`ads_sprint_tests.cpp`, `use_range_bridge_tests.hpp`)
Assemble in executable memory: a hostile callee (records `rsp & 15`, clobbers
rcx/rdx/r8-r11, xmm0-5, sets carry, returns), the original instruction sequence
as oracle, and a harness that seeds all GPRs/XMM/flags with sentinels and
captures them afterwards. Assert equality for the whole struct except the one
intended output (mask AF where the original leaves it undefined). Test the
**installed** path: place the fake native image at the real base region and the
bridge at `0x20000000000+` to force the >2 GB failure; assert plain `hook::call`
throws before writing, relay bytes are `FF 25` and the page is
`PAGE_EXECUTE_READ`. Embed the real captured native bytes as the oracle where
the function is a self-contained leaf. Run every 16-bit handle through a guard
bridge when it checks ranges.

### 4.4 Detour hygiene

- `create_disabled` + `enable`: publish originals/flags before the hook goes live
  so the first call cannot see a null trampoline. `get<T>()` returns the trampoline.
- `clear()` returns bool; the destructor ignores failure; move-assignment throws
  if the old hook cannot be removed. `enable` tolerates `MH_ERROR_ENABLED`.
- Never create/disable a detour from inside the detour that is executing
  (section 10).
- A rel32 rewrite is not atomic. It is a one-shot loader operation done at
  `post_unpack` before worker threads run; never reuse it as a runtime toggle.
  (`suspend_other_threads` exists but can deadlock on a held loader/heap lock;
  keep the window tiny and allocate nothing inside it.)
- If another overlay already hooked the same function, do not hook again
  (section 11, alternate-target guard).

### 4.5 Observing a mid-function result

To read an engine-internal value that is never returned from a function, replace a
displaceable instruction (for example a 6-byte `mov ecx,[rip+x]`) with
`call relay; nop`. The assembled thunk replays the displaced load, then calls C++
with arguments taken from live registers/stack slots, reserving `0x28` bytes so
the nested call is aligned. Use a preserving relay.

### 4.6 Per-call-site discrimination

A detour on a shared function can alter behaviour per caller using
`_ReturnAddress()`; a call-site rewrite plus a `thread_local` query context is
usually cleaner and cheaper. The weapon-fire and command hooks rewrite only the
sub-steps they own (`NormalInput`, `FinishMove`, `UpdateViewAngles` call sites
beside one detour on the command constructor).

### 4.7 Redirect inputs, not outputs

Where possible change the *input* to a native function (the animation-slot
selector so equip clips become the current weapon's idle before notetracks are
read) instead of overwriting final results later. Patch the *derived* flag at its
producer, not the input state (sprint-while-ADS removes one flag from four
predicates and the derived slow-walk bit at its producer; it never clears ADS
state or command buttons).

### 4.8 Two components, one native location

When two components touch one function, hook **after** the operand the other one
relocates and verify both the first instruction and the tail, so ordering does
not matter (the model-identity getter is hooked after its pool-base LEA because
the fastfiles component enlarges the pool). The shared contract lives in a pure,
unit-testable header (`game/xmodel_index_contract.hpp`).

### 4.9 Relocating an instruction operand: compute, verify, never guess

For `movss xmm0,[r8+rcx*4+disp32]` (`F3 41 0F 10 84 88 disp32`, length 10) the
disp32 starts at +6; +5 is the SIB byte. Patching +5 corrupts the address
calculation and a displacement byte. Derive the displacement offset from the
decoded instruction (Capstone `disp_offset`), verify the opcode prefix (load vs
store), write the four little-endian bytes, keep the helper pure (takes a byte
span) so it can be unit-tested. Compiler-folded and "inverse" operands
(`lea r10,[rbp+rax+(stack_base - arena)]`) need `stack_offset = original +
native_offset; new = stack_offset - displacement` (section 12).

### 4.10 COM vtable patching (when there is no game address)

`ID3D11Device::CreateTexture2D` (slot 5) is patched by overwriting the vtable
slot (`VirtualProtect` + `InterlockedExchangePointer`) in the flight recorder
(`d3d11/texture_diagnostics.cpp`). Because COM implementations (including XR
runtimes and WebView2) **share vtables across devices**, the stub looks up the
calling device in a 32-entry identity table and forwards unknown devices to the
saved original untouched; if middleware already patched the slot
(`current != stub` and device known) it leaves it alone to avoid a recursion
cycle; it saves/restores `GetLastError`, copies the descriptor with a safe read,
and records `_ReturnAddress()`. Context vtable hooks (draw, SetConstantBuffers
slots 7/16/22/62/66/71, Map/Unmap 14/15, DrawIndexed 12) pre-flight reject
aliased slot targets, keep retired trampolines alive for late calls (never free a
trampoline a draw may be using), and never create deferred contexts in
production.

---

## 5. Verification primitives and guarded reads (`utils/hook_validation.hpp`, `utils/native_memory.hpp`)

### 5.1 `verify_masked_bytes(address, {bytes, mask, size})`

- Walks the range region by region with `VirtualQuery`, so patterns spanning two
  pages work; requires `MEM_COMMIT`, readable, not `PAGE_GUARD`/`PAGE_NOACCESS`
  in each region; guards address overflow.
- On failure returns a status enum (`invalid_argument, address_overflow,
  query_failed, uncommitted_memory, inaccessible_memory, mismatch`) plus
  `mismatch_offset`, expected, actual, mask. Log "byte mismatch at +0x7".
- Mask is a per-byte AND, so nibble wildcards work; `0x00` = wildcard (relocated
  operands, rel32). Use full `0xFF` for prologues and for `E8` sites.
- `validate_executable_target(ptr)`: committed, not guard/noaccess, executable.
  Use it for any pointer you will call or detour.

### 5.2 What to verify for each site

1. The call/instruction bytes at the site (for `E8`, decode `rel32` and require
   `site+5+rel32 == expected callee`; `call_matches`).
2. The callee's prologue (9-40 bytes, full mask).
3. That the callee is executable, and that the stub is within rel32 range.
4. For pointer *slots* read from fixed globals (native config variables), verify
   the object's **identity** (hash/name/type) before trusting it.
5. For script bytecode hooks: exact-layout inspector (section 13).

```cpp
template <size_t N>
bool verify(uintptr_t address, const uint8_t (&bytes)[N]) {
  std::array<uint8_t, N> mask{}; mask.fill(0xFF);          // 0x00 = wildcard
  return bool(utils::hook_validation::verify_masked_bytes(
      reinterpret_cast<void*>(address), {bytes, mask.data(), N}));
}
```

### 5.3 Guarded native reads

`utils::native_memory::read_bytes(&out, ptr, size)` / `read_at(base, offset, out)`:
SEH-guarded bounded `memcpy`; rejects null, `< 0x10000`, size 0 or `> 1 MiB`,
address overflow; the `__except` filter catches only `EXCEPTION_ACCESS_VIOLATION`
and `EXCEPTION_IN_PAGE_ERROR`; on failure discard the destination (may be
partially written). The header states the key limit: it **does not prove
lifetime, thread ownership or a coherent snapshot**, so pair it with identity
checks (entity number + spawn generation, object epoch). Rules: keep `__try` in
a destructor-free leaf function; read whole structs in one call (no per-field
`VirtualQuery`); never cache permissions across native calls. Prefer these to the
older `memory::is_bad_read_ptr` (single `VirtualQuery`, first page only, racy).
When you read raw engine memory, validate finiteness and ranges before use.

---

## 6. Fail-closed, transactional install

### 6.1 Feature-level (the common case)

```cpp
bool install(const char* feature, std::initializer_list<site> checks, auto&& do_install) {
  for (auto& s : checks)
    if (!verify_masked(base + s.rva, s.bytes, s.mask)) {
      log("[%s] contract mismatch at +0x%llX; feature disabled", feature, s.rva);
      return false;                      // original game behaviour, nothing patched
    }
  return do_install();                   // allocate relays first, then patch
}
```

Every adapter's `post_unpack` runs **all** of its byte witnesses, `call_matches`
and `is_relatively_far`, then installs. On any miss: one log line, return, the
game keeps native behaviour. Optional sub-features have their own checks and
disable only themselves (`controller_component.cpp`: one command-hook set,
all-or-nothing; ADS, stance and remote-axis extras gate independently). Skipping
one site of a multi-site policy is how a feature half-works: Overlord's first
camera-bob policy covered two sites and missed ADS angular bob; the shipped
policy covers seven call sites plus two scalar multiplies under one contract
table.

A diagnostics component may instead record "contract rejected" and return, so it
degrades without failing the game; a core component throws and the loader treats
it as feature failure (message box). Decide per component.

### 6.2 ABI-level (all-or-none with rollback)

When patches change a data/ABI format (4-byte to 8-byte IDs, relocated arena),
mixed state is corruption, so there is no partial install and no runtime toggle
(`scene_surface_storage.cpp`):

1. Gate on `target_identity::get().compatibility_probe_passed`.
2. Compute the relocation displacement; reject if not positive, `> INT32_MAX`, or
   not a multiple of the unit.
3. Copy the compile-time patch table; **verify every site's `before` bytes first**
   and compute all replacements (dynamic operands relocated), also verify
   adjacent data such as a diagnostic string.
4. Write with `VirtualProtect(RWX)` + `memcpy` + restore + `FlushInstructionCache`;
   count `applied` *before* each write so the current site is also restored if
   the protection restore fails; **re-verify each written site**.
5. A `gsl::finally` rolls back every written site (reverse order, restoring
   `before`) unless `committed`.
6. Install order for dependent hooks: inner observers and consumers first, the
   record **producer last** (it must never publish into a half-installed
   consumer). Capture original protections before writing; rollback re-verifies
   and **throws if rollback is incomplete** ("refusing to start with partially
   patched code").

Scene hook recipe used throughout the stereo layer: compare 5 bytes at the site
(`0xFF` mask) -> decode rel32 and require the expected callee -> require callee
executable -> require stub in range -> capture protection -> write -> re-read
and verify.

---

## 7. The never-scan-and-patch policy

- Runtime pattern scanning across versions contradicts fail-closed. Overlord
  pins **one** exact exe (size, SHA-256, sentinel, PE identity) and hard-codes
  verified addresses.
- `utils::signature` ("48 8B ?? ?? E8", `?` per nibble pair, odd nibble count
  throws, trailing wildcards trimmed; SSE4.2 `_mm_cmpestrm` only for patterns
  `<= 16` bytes; parallel scan across `hardware_concurrency()/2` threads when
  the range exceeds `cores*10`; default range is the main module's
  `SizeOfImage`; `process()` returns all hits) is an **RE and diagnostic aid**.
  Use it offline, or to *locate and then pin* an address.
- Offline tooling generates compile-time patch tables from a clean, owned dump:
  `tools/generate_scene_surface_contract.py` (docstring: "Never scan-and-patch at
  game runtime: the generated, reviewed instruction contracts are compiled into
  the module and checked as one transaction"). Method in 03 section 4.
- Script bytecode is located by *masked signature + operand cross-check +
  uniqueness* inside a bounded function extent, never by a free scan (section 13).
- Runtime may do bounded **discovery of live objects** (find a resident weapon
  definition among 512 tokens, scan loaded materials) but not discovery of code.
  Such scans are budgeted, validate names/charsets, rank deterministically, and
  never fabricate a definition.

---

## 8. Struct layout and entry-point discipline

### 8.1 Layout

```cpp
static_assert(sizeof(game::usercmd_s) == 64);
static_assert(offsetof(game::usercmd_s, buttons) == 4);
static_assert(offsetof(game::playerState_s, viewangles) == 0x108);
```

- A `static_assert` on every offset you rely on; name unproven regions
  `__pad_*`/`char __pad2[59380]` and keep a comment with the **evidence**
  (disassembly address) next to each proven field
  (`PM_UpdateViewAngles (0x14068D7B0) adds delta_angles at +0x98 and stores at +0x108`).
- **Trust asserts over comments.** `game/assets.hpp` has inline offset comments
  that disagree with its asserts (`WeaponDef::gunModel` comment 24, assert 0x28;
  `iAmmoIndex` 1584 vs 1752); comments come from other builds. Reconstructed SDK
  labels can also be wrong: native fuse fields `+0x7D4/+0x7D8` are reversed vs the
  call chain; trust call-chain evidence.
- Second tier: **runtime identity** of dynamic objects (hash/type of config
  variables read from slots; `weaponClass` zero-based vs SDK enum shifted; entity
  generation).
- Restate offsets in tests independently of production constants with
  `static_assert` equality.
- Sentinels are engine-specific: H2 entity "world/none" = 3998/3999 (IW's 1022/2046
  are wrong here), 4000-slot entity table, usercmd angles packed 360/2^24.

### 8.2 Typed native entry-point layer (`vr/h2/binding.hpp`, `entrypoints.hpp`, README)

- `vr::h2::function<Result(Args...)>` stores only an address and calls through a
  **canonical fixed signature**; consumers cannot deduce a different ABI.
  `read_only_global<T>`: `get()` returns a const pointer, `read()` dereferences;
  "a readable pointer does not establish readiness or ownership".
- The layer declares ABI facts **only**: no scanning, admission, byte
  verification, hook install, thread-ownership proof or dynamic caching. Those stay
  at the owning hook adapter; call-site addresses, continuation points and feature
  predicates belong to the adapter, not to a generic list. No per-frame scans, no
  permissive fallback to another build.
- Keep it separate from the inherited mod's `game::` symbol table and reuse
  correctly typed `game::` symbols instead of declaring a second address.
- Opaque handles (`struct dobj;`): declaring a pointer type does not authorise
  reads or caching of object lifetime; inspect layout only at a guarded/locked
  boundary.
- Ask the engine for **live** values (`clip_capacity(ps, weapon, alt)`) instead of
  static data; a "weapon selection request" global is a request, not confirmation
  of the equipped weapon; dual-wield queries do not admit paired weapons as
  independent instances.
- **Inherited symbol tables are unverified.** `game/symbols.hpp` (~250 entries,
  `game::symbol<T>{addr}`, header-only via `WEAK = __declspec(selectany)`) has
  duplicate addresses with different names/signatures (`Quit`/`Com_Quit_f`,
  `hks_obj_tolstring`/`hksi_lua_pushlstring`, two draw-text symbols at one
  address). Each entry a VR feature depends on needs a masked prologue check
  and/or runtime sanity check.

---

## 9. Execution contexts: the scheduler (`component/scheduler.cpp`, `scheduler_context.hpp`)

Mod code must run on the right engine thread without a hook per feature, and must
be able to *assert* where it is. Overlord has five named pipelines bound to native
frame boundaries:

| Pipeline | Runs | H2 hook (example) |
|---|---|---|
| `async` | private "Async Scheduler" thread, 10 ms sleep, disconnected from the game | none |
| `renderer` | inside the `R_EndFrame` hook, **before** the original | `0x14076D7B0` |
| `lui` | before the Hks/LUI frame, only if the Lua state exists | `0x140327880` |
| `server` | after the original `G_RunFrame` | `0x1404CB030` |
| `main` | after the original main frame | `0x140417FA0` |

API: `scheduler::loop(fn, pipeline, interval)`, `once`, `schedule(fn -> cond_end/cond_continue)`,
`on_game_initialized` (polls `Sys_IsDatabaseReady2` from `main`, then posts).

Rules:

- `execute(type)` sets a `thread_local` RAII `execution_scope`;
  `scheduler::is_executing(pipeline)` is the cheap assertion. The scope is a
  **dispatcher scope, not a permanent OS-thread assignment** (H2 runs startup and
  in-level server frames on different threads); never assert on thread ids.
- Use it as a precondition guard enforced by code: model/asset registration and
  skeletal-model creation return false unless `is_executing(pipeline::main)`.
- New tasks go to a separate `new_callbacks_` container merged at the start of
  `execute`, so tasks can schedule tasks without iterator invalidation; a task
  returns `cond_end` to erase itself; per-task `interval` and `last_call`.
- Game-state mutation (give/take/kill/set position) runs on the `server`
  pipeline, preferably through script calls instead of memory pokes; `command::execute`
  is a no-op until a `scheduler::once(main)` sets `game_initialized`.
- **Where VR frame maintenance runs matters.** `poll_capture` and `prepare_frame`
  run on the *renderer thread* in the `R_EndFrame` hook: D3D11 immediate-context
  queries (`End`/`GetData` for copy completion) must stay on the thread that
  produced the native eye copy. Polling from Present raced `CopyResource/End` on
  the game's single-threaded device and surfaced as an NVIDIA driver fault. The
  same hook runs *after* the frame's scene records are built, so it cannot define
  the pose for those records; per-eye pose must be sampled and frozen at the
  per-view owner.
- Three passive frame-ownership hooks (frame-state transition call site
  `0x1403D8D0A`, `R_EndFrame`, frontend handoff `0x1403D8D14`) wrap the originals
  in `gsl::finally` scopes and feed the stereo renderer's state machine of engine
  phases: "none of these hooks may synthesize a scene". An inactive probe takes
  the minimal `execute(renderer); original();` path.
- Script-VM calls only on the owner thread; the stack-isolation guard protects
  stack state, not concurrency.
- Name engine threads (`Main, Backend, Worker0-7, Server, Cinematic, Window, Input,
  Database, Sound`) from the engine's thread table.

### 9.1 Thread-local reentrancy guards

Every stub that might be re-entered (skeleton calc, render owner, observers) needs
a `thread_local bool in_hook` guard and a `gsl::finally` restore; foreign or nested
owners must not inherit the outer lifecycle. See `component/vr/gameplay/hands/component.cpp:
calc_skeleton_stub`, `component/vr/engine_stereo_renderer.cpp` (`observation_active`,
`dynamic_upload_owner_scope`). Preparation jobs run on worker threads and can nest:
save and restore the previous value.

---

## 10. Reentrancy and deferred teardown

Re-entrant teardown from inside Present is the classic exit-time crash.

- Listener dispatch (`d3d11.cpp`): listeners live in a token map under a mutex;
  dispatch **copies** the matching callbacks into a vector under the lock, releases
  it, then invokes each through `invoke_listener` which sets
  `thread_local inside_graphics_callback`, catches all exceptions into
  `graphics_status.last_error`, and afterwards runs deferred shutdown if safe.
  This survives callbacks unsubscribing during dispatch and keeps the lock away
  from VR code.
- `shutdown_graphics_hooks` cannot tear down from inside a callback/device
  transition: it sets `graphics_shutdown_pending`; the last scope exit
  (`scoped_device_transition`, RAII) runs `process_deferred_graphics_shutdown`.
- Swap-chain hook (`d3d11/swap_chain_hook.cpp`): `std::recursive_mutex callback_mutex`
  held across the whole callback (serialises Present vs Resize, allows reentry);
  `std::shared_mutex hook_lifecycle_mutex` taken shared (thread-local depth counter
  avoids self-deadlock) by callbacks and exclusive by install/reset/shutdown, so
  unhooking waits for in-flight callbacks; `thread_local` reentrancy flags forward
  re-entrant calls to the trampoline. `install()` from inside a callback stores
  `pending_install`; `shutdown()` sets `shutdown_pending`; both run in
  `finish_callback` after locks are released.
- Shutdown **ordering** (`component/shutdown.cpp`): hook the quit wrapper and
  `Sys_Quit`; both call `R_SyncRenderThread()` first, then
  `component_loader::pre_destroy()`, before native `Com_Shutdown` which unloads
  NVAPI while the mod still holds a device with NVAPI interfaces attached.
  Bytes of both entry points are verified. Not run on cancellation/renderer
  restart. Lesson: find the engine's unload order and retire your COM/XR owners
  before it pulls driver libraries.
- Console/log output must never block producers: bounded queue with drop counter,
  independent first/last-error history, `CancelSynchronousIo` retry loop on exit
  (cancellation is edge-triggered).

---

## 11. D3D11/DXGI interception at the hook layer

(The XR-runtime side is in 04; this is the part that belongs to loading/hooking.)

- **Import substitution, not vtable scanning, for device creation.** The d3d11
  component returns stubs for `dxgi.dll!CreateDXGIFactory` and
  `d3d11.dll!D3D11CreateDevice`: the game's own call hands you the device before it
  can use it.
- **DXGI 1.1 factory lineage.** The stub forwards `CreateDXGIFactory` to
  `CreateDXGIFactory1(riid, pp)` with the same riid (OpenVR `Submit` needs a 1.1+
  factory before device creation); after device creation verify
  device->IDXGIDevice->GetAdapter->GetParent(IDXGIFactory1) and compare adapter LUIDs.
  Cheap insurance even for an OpenXR-only port.
- **Candidate vs promoted device.** Each created device is published as a candidate
  with a monotonically increasing generation, tagged with `SetPrivateData(guid,
  generation)`; stale candidates (generation <= current) are rejected. A candidate
  is promoted only when the swap-chain hook sees a real Present on it. Promotion:
  fire `device_destroying(previous)` -> swap `current_device`, reset
  per-generation counters -> fire `device_created`. Everything VR owns is keyed by
  generation. `device_created_early` fires on the creating thread before the
  device is returned to the game (use it for private data, shader tagging);
  `device_created` fires at promotion (build XR session/swapchains).
- **Hooks are requested, then installed when a device exists**
  (`enable_graphics_hooks()` records the request).
- **Swap-chain hook.** Discover targets from the game's own device: adapter ->
  factory, create a 1x1 `WS_POPUP` window and dummy windowed swapchain
  (R8G8B8A8_UNORM, BufferCount 1, DISCARD), read `vtable[8]` (Present) and
  `vtable[13]` (ResizeBuffers), validate both with `validate_executable_target`,
  destroy the dummy. Filtering inside the stub: a swapchain is the game's only if
  `OutputWindow == game HWND`, `BufferUsage & RENDER_TARGET_OUTPUT`, and
  `GetDevice()` has the same COM identity (`QueryInterface(IUnknown)` pointer
  compare). Three-way classification `rejected` (pass through) / `candidate` /
  `active`; a candidate is promoted only after the original Present returned
  success and `!(flags & DXGI_PRESENT_TEST)`; each promoted chain gets an epoch
  via `SetPrivateData`, `epoch <= active` is rejected; `DXGI_PRESENT_TEST` on the
  active chain passes straight through; `ResizeBuffers` notifies **before** the
  original so VR code can drop every backbuffer reference (it fails otherwise).
  Present stub order: bump frame index -> `notify_present_pre` -> original Present
  (timed) -> on failure record `GetDeviceRemovedReason` -> `notify_present_post`.
  **Alternate-target guard:** if a later discovery yields different Present/Resize
  pointers than already hooked (another overlay hooked first), do not hook again;
  re-install after shutdown re-`enable()`s the existing detour.
- **Do not force `ID3D11Multithread` protection.** Overlord only queries
  `GetMultithreadProtected()`; forcing global serialisation changed every game
  call and still reproduced the kernel failure (d3d11.cpp ~1823). Keep GPU work
  on the real renderer owner.
- **GPU-queue interop gate:** recursive mutex, try-lock-first, contention counters,
  waiter record (thread, client, start tick) so a watchdog sees who is stuck;
  blocking runtime calls must never hold it.
- **Debug layer redirect** (DEBUG only): vtable-patch `ID3D11InfoQueue::AddMessage`,
  1000 ms per-message-id rate limit, bounded sample table, no D3D getters, symbol
  loading or disk I/O inside the callback; `D3D11_CREATE_DEVICE_DEBUG` is ORed in
  only when requested.
- **Shader tagging:** detour the engine's shader-create wrappers and attach
  bytecode (`SetPrivateData(guid_shader_bytecode,...)`) and the material name
  (`WKPDID_D3DDebugObjectName`) so draw-time hooks can recognise passes without
  hashing DXBC and tools can disassemble later (DEBUG only in Overlord).

---

## 12. Engine capacity expansion (stereo needs ~2x per-frame packing)

Fixed per-frame engine arenas, caches and pools are tuned for one view. Stereo plus
VR-only models can overflow them. Symptoms: equipment vanishing when you turn, TV
textures changing, native warnings such as
`MAX_SCENE_SURFS_SIZE((1 << (16 + 2))) exceeded - not drawing surface`,
"Too many visible static model surfaces", "Too much model lighting used".
**Make "grep engine logs for 'exceeded / not drawing / too many' during stereo
tests" an acceptance step.** In H2 captures the surface cursor hit 313,592 and
301,376 bytes against 262,144 with 780/855 failed packings; every generator entry
started at cursor 0, so it was per-frame pressure, not a leak.

### 12.1 The recipe

1. **Characterise the ABI.** Overlord's `uint16 surfId` is a scaled *byte offset*
   (units of 4 bytes), not an ordinal, so raising only the capacity check wraps IDs
   into earlier surfaces. Change the unit (4 -> 8 bytes: mask `0x3fffc -> 0x7fff8`,
   shift `0x12 -> 0x11`, hidden-record stride 4 -> 8, reservation bound
   `0x40000 -> 0x80000`, address scale `*4 -> *8`, brush prefix `28 -> 32`, brush
   surface index advance `6 -> 3`, diagnostic string digit) to get 512 KiB per
   frontend.
2. **Census every operand offline** (Capstone over the whole captured `.text` +
   `.pdata`; 03 section 4). A literal-base-only search missed 24 `base+field`
   operands and caused two crashes (null brush-surface pointers via `+0x62b708`,
   `+0x62b728`). Preserve each field addend; handle compiler-folded forms and
   inverse operands; exclude same-valued RVAs/strings outside the audited code
   range (the generator asserts none exist outside it). Assert exact census counts
   so a different build or an incomplete audit fails generation (H2: 89 arena
   references with field addends `{0:65,4:7,6:2,8:2,16:1,24:2,40:6,48:2,92:2}`, 59
   mask sites).
3. **Generate a compile-time contract table** `{address, before[], after[], size,
   field, dynamic-kind, reason}` (H2: 242 patches: 91 relocated, 24 compound, 1
   folded; dynamic kinds 0 none / 1 `arena+field` (displacement + original addend) /
   2 folded immediate (`arena_offset/4` becomes `displacement/8`) / 3 modular inverse
   (`(0-arena_offset)&0x3ffff` becomes `(0-displacement)&(capacity-1)`)). Include `old == new` guard entries that verify bank-identity
   assumptions without modifying them. Rule in the header: never remove a subset
   "as a performance optimisation".
4. **Keep the native bank spacing.** Two frontends are consumed concurrently with
   in-flight backend workers, so the replacement bank keeps the native frontend
   stride (`banks[native_frontend_stride + capacity]`, one displacement selects the
   bank from either frontend pointer; must be positive, `<= INT32_MAX`, multiple of
   the unit). Do not compact or share a scratch buffer (aliases in-flight frames).
   A static zero-filled BSS bank makes the gap free.
5. **Apply all-or-none with rollback** (section 6.2), startup only.
6. **Fix the mod's own readers.** A native-patch census cannot catch C++ consumers:
   `plan_skin_packets` must use the shared `record_bytes` (hidden=8, skinned=56,
   rigid=(bones+2)*32); a stale hidden stride of 4 broke original-magazine hiding.
7. **Tests:** run the patched bytes in a harness (asmjit), loop all 65,536 IDs, last
   slot, hidden entries at begin/middle/end, 512 hidden records, and reproduce the
   recorded crash arithmetic as an assertion. Verification proves the *listed*
   sites matched, not that the list is complete: the coverage census is a separate
   test.

### 12.2 Neighbouring budgets patched the same way

- **Static transparent list budget** (`scene_static_surface_layout.hpp`): budget
  table of 14 camera-region rows x 2 `uint16` (output bytes, sort count) x surface
  types; lit-transparent region 3 raised 1024 -> 4096 bytes per type; backing bank
  x4 (`0xD2000 -> 0x348000`); 15 operand sites including two **inverse-displacement**
  `lea` operands that recover stack sorting pointers from the arena base (imm = 
  `stack_offset - displacement`; leaving them shifts sorting and end pointers by the
  relocation delta and crashes). When moving a buffer search for *compensated*
  address arithmetic, not only direct references.
- **Static-model lighting cache** (`scene_model_lighting*`): two initializer
  minimum-quota compares (`cmp edx,0x1000`) go through preserving relay bridges that
  choose 8192 when reserved <= 8192, else native 4096; `total_slots` is the next
  power of two of `reserved + minimum`, and returns 0 (keep native) when
  `reserved > 7*4096` so the fixed 16K static tables can never be overrun.
  **Savegame hazard:** `DynEntityClient::lightingHandle` is archived including the
  old partition offset, so the dynamic-reuse site gets a range guard (only handles
  in `(static, total]` reach the native index math; others take the allocate path,
  using only the register native code clobbers next). Always check what an enlarged
  cache serialises into saves.
- **Asset pools** (`fastfiles.cpp reallocate_asset_pools`): doubles XModel,
  XModelSurfs, Sound, LoadedSound, XAnim, LocalizeEntry, SoundCurve, Attachment
  pools; XModel growth re-points ~150 RIP-relative references to 5 int arrays, 3 bit
  arrays and one 6x array plus count immediates, via a relocation helper that
  verifies original instruction bytes and throws on mismatch. Enlarge the engine's
  memory pool init too (`memory.cpp` replaces `PMem_Init` with a larger reserve and
  patches the dependent constant).
- **Script builtin table** enlarged to 0x1000 (section 13).
- Pitfall: stereo also doubles *model submissions*; native identity-by-pool-index
  code needs a registry for mod-built descriptors (`scene_model_identity.hpp`: lock-free
  append-only batches, capacity 512, reset only at the native asset-unload barrier).

---

## 13. Script-VM hook points (when mission logic lives in a VM)

(Mission/cutscene use of these is in 10; this is the hook machinery.)

- **Restore diagnostics before building on the VM.** Shipped builds strip error
  text; `gsc/script_error.cpp` re-implements the `Scr_Get*` accessors and link
  error paths ("type X is not an int", "unknown script/function X::Y"), and
  `find_function(codepos)` maps bytecode pointers to function/file.
- **Builtin table** replaced by an own `void* func_table[0x1000]`
  (`FUNC_TABLE_SIZE`): patch the count constant, table size operand and four code
  references; custom ids start after `function_id_start = 0x320` (pre-increment);
  custom functions are re-registered whenever the game re-resolves its table.
  Custom functions (pointer outside the game image) run inside a try/catch: a C++
  exception never crosses game frames; the message is stored and
  `Scr_ErrorInternal()` raises a script error.
- **`intercept_builtin(name, cb)`** (max 8, startup only): matched by
  `func_table[id-1] == function`; `cb` returns true if it supplied the return value
  through an isolated call. Observe/redirect one operation only (`notifyoncommand`,
  `vectordot`, `bullettrace` inside one known function, `magicbullet`).
- **`notifies::set_gsc_hook(source, target, guard)`**: opcode-dispatch redirect with
  a 20 MB hit bitmap for O(1) "is there a hook here". Guards: configuration guard
  (`eligible()`) and call-site guard (`called_from(range)`: caller bytecode extent
  from the sorted function table; frame depth capped at 31). Hooks on the redirect
  target are not chained; the VM decodes the target opcode immediately without
  consulting hooks there, so observe the *next* instruction. Bind at level start,
  clear at shutdown, rebind after checkpoint load.
- **Bytecode contract:** a masked signature (relocated string ids and relative call
  operands masked), searched only inside the function extent (<= 2048 B), every
  `GetString` operand cross-checked against live interned strings, **exactly one
  match** else fail closed; static-assert named offsets; test with truncation and
  bit-flip sweeps (`signal_flare_script.hpp`, `airport_opening_tests.hpp`).
- **Pitfall:** bypassing a function's parameter bindings including
  `OP_checkclearparams` still left `OP_clearparams (0x4D)` releasing values until a
  `SCRIPT_CODEPOS` entry while a new call has `PRECODEPOS`; the release crossed the
  frame and freed caller references, crashing unrelated input/lighting/render code
  later. Rule: validate and execute the original prologue, redirect only the body.
  A null-pointer guard at the crash site would have hidden the real damage.
- **Runtime-compiled "virtual sources"** (`register_virtual_source(name, source,
  map)`, startup only): name must start with `scripts/vr_`, `[A-Za-z0-9_/]`, no `..`
  or `\`, <= 128 chars, not ending `.gsc`; source non-empty and <= 65536 bytes; map
  <= 64 chars; no duplicates; optional per-map gating; never replace a native asset
  name. Compiled bytecode lives in the engine's script memory arena; freed at
  scripting shutdown.
- **Calling the VM from mod code** (`game/scripting/*`): push args in **reverse**
  onto the VM stack, set out/in param counts, call, read the result, clear out
  params; method calls pass an entity ref, globals `{0xFFFF,0xFFFF}`. Wrap in a
  **stack-isolation** RAII object (private 512-slot array, restore four stack
  fields) so a call from a hook that fires mid-script does not trample the live
  frame. Native script errors `longjmp`; give mod calls their own error level
  using the **game's own `_setjmp`/`longjmp`** plus an SEH wrapper. Refcount every
  stored value (`AddRef`/`RemoveRef`). A detoured script function called from Lua
  must disable its own redirect for the duration.
- **Script-polled input:** some waiters poll command-derived state every 50 ms and
  the engine's edge latch clears each frame; a one-command pulse is missed. Hold
  the bit and keep short taps visible >= 100 ms. `notifyoncommand` listeners do not
  see usercmd bits; also enqueue the native reliable command notification (07).

---

## 14. Other engine-service patches (all behind verified contracts)

| Need | Technique | Overlord |
|---|---|---|
| Settings as engine cvars | Register through the engine's dvar system with the SAVED flag; embed a name/description table when the engine stores hashed names (H2: `0xB3CB2E29 * (h ^ lower(c))`, seed xor `0x319712C3`; a `"0x…"` string is the hash itself); put the hash in a `constexpr` header the launcher can include | `game/dvar_hash.hpp`, `component/dvars.cpp` |
| Force/veto engine settings | Hook register/set by name hash (`dvars::override::*`, `disable::*`) | comfort cvars |
| Keep the game from re-forcing FOV | Hook the script save-dvar function and ignore `cg_fov`/`cg_fovScale` | `patches.cpp` |
| Extra key bindings | Extend the bind table past the native whitelist by hooking write/lookup/execute functions | `binding.cpp` |
| Loose-file overlay on packed archives | Hook the VFS interface vtable slots (open/read/tell/size/close/exists) to serve loose `.ff/.pak/.flac/.bik` first; fake handles tracked in a map | `database.cpp` |
| Replace assets | `override/<name>` rawfile lookup in `DB_FindXAssetHeader`; PNG texture override through the image setup call; custom streamed image file index | `fastfiles.cpp`, `images.cpp`, `imagefiles.cpp` |
| Ship VR content | Compile your own sources into engine-native zones (`h2_vr_sounds`, `h2_vr_melee`) and load them through the engine's zone loader; register `fastfiles::on_pre_unload` callbacks to drop borrowed pointers *before* assets/strings are freed | `fastfiles.cpp` |
| Native UI input ownership | Call the engine's own key/mouse event functions directly (so VR input does not count as physical activity), "physical wins", release all VR-owned keys on focus loss | `input.cpp`, `native_ui_pointer.hpp` |
| Process isolation | Separate player-profile folder so the original save/config are safe | `game_data.cpp` |
| Platform services | Replace online services with local stubs (prologue `mov al,1; ret`, NOP a popup) | `battle_net.cpp`, `achievements.cpp` |
| Crash handler | Pre-created emergency dump thread, copy the exception record into static storage, hijack `SetUnhandledExceptionFilter` (also jump-patched), RIP-redirect recovery only on the main thread with limits | `exception.cpp` |
| Debug conveniences | Stall detector around asset lookups (> 100 ms warns) | `fastfiles.cpp` |

A VR fork of an existing mod must neutralise the upstream's network services
(updater server list empty, MOTD disabled, no-auto-update marker) because the
upstream feed can replace VR binaries.

---

## 15. Bugs not to copy (all in Overlord's inherited utilities)

| Where | Bug |
|---|---|
| `src/common/utils/nt.cpp:189` `nt::library::get_iat_entry` | `if (ordinal_number > 0xFFFF) continue;` has no pointer increment: infinite loop for any by-name import; also resolves by ordinal-casting a name RVA, so it only works for ordinal imports. Walk `OriginalFirstThunk`, skip `IMAGE_ORDINAL_FLAG` correctly, compare `IMAGE_IMPORT_BY_NAME::Name`. (Overlord resolves game imports through `load_import` instead.) |
| `nt.hpp:~58` `get_proc` | `if (!is_valid()) T{};` missing `return`. `invoke_this` calls `get<...>` with the wrong arity (does not compile if instantiated). `nt::library()` default-constructs the *running exe*, not an empty handle. |
| `component/game_module.cpp:87-88` | `handle_ex_w_hook` is created for `GetModuleHandleExA` and then re-created for `...ExW`; `handle_ex_a_hook` is never created, so the A path calls an uninitialised detour. |
| `component/fastfiles.cpp:790` | Comment says `DB_LOAD_SYNC`, code passes `DB_LOAD_ASYNC`. |
| `memory::is_bad_read_ptr/is_bad_code_ptr` | Single `VirtualQuery`, first page only, racy. Use `verify_masked_bytes`/`validate_executable_target`/`native_memory`. |
| `pushad64` | Pushes `rsp`, does not preserve flags or XMM. |
| `main.cpp verify_version` | Raw dereference instead of guarded read. |
| `utils/http` | Pins a DNS server and relies on libcurl defaults for TLS verification; do not copy. |
| `utils::io::read_file` | Casts size to `uint32_t` (breaks above 4 GiB; harmless for configs). |
| Inherited | cryptography, smbios, info_string, toast, image modules are multiplayer-era; do not carry them to a new port. |
| Doc-only | `game/assets.hpp` offset comments disagree with its asserts (trust asserts). `game/symbols.hpp` has duplicate addresses (section 8.2). |
| Process | A Debug-only install gate made optimised clients fall back to native rendering: **build mode is not a feature switch**; the parity audit (11) catches compiled-out components. |
| Process | A fixed-capacity observer table (8) broke boot at the ninth registration. |

---

## 16. Build configuration lessons (`premake5.lua`)

- **Release disables WPO for every project** (`/GL-`, `/LTCG:OFF`, keep `/O1`
  optimisation): MSVC v142 (19.29) WPO placed nonzero-initialised mutable aggregates
  (snapshot publication buffers of 15,416 B, 204,840 B, 30,024 B) in read-only
  `.rdata` or dropped the writes (level-load write AVs). A section pragma fixed
  placement but not lost-write semantics. Gate with a runtime publish/readback
  test (`vr-aggregate-publication-tests`) plus a PE/PDB section audit
  (`tests/vr/client_feature_parity_tests.py`). Debug passing proves nothing about
  Release; Release has fatal warnings.
- **RelWithDebInfo** is the performance/gameplay config (Speed, Release CRT, no
  runtime checks, `/JMC-`, `H2VR_PROFILE_BUILD`); Debug costs 35-44 ms per eye pair
  of JMC/RTC helper overhead. Use RelWithDebInfo for play and perf, Debug for
  troubleshooting.
- Static CRT, `characterset ASCII` (A/W API selection separate from `/utf-8`
  source), C++latest, `/bigobj`; a few TUs `NoPCH` so tests can compile them
  standalone. Link: `/DYNAMICBASE:NO /SAFESEH:NO /LARGEADDRESSAWARE /LAST:.main`.
- Pin dependencies as submodules; build the OpenXR loader from the pinned SDK
  (CMake utility project) and ship it app-local; per-dependency licence files are
  part of the package (11).

---

## 17. Overlord file pointers

| Mechanism | File |
|---|---|
| Loader flow, import substitution, late trigger | `src/client/main.cpp` |
| Cache, lease, ASLR patch | `src/client/loader/binary_cache.{hpp,cpp}` |
| Map, imports, TLS, exception table | `src/client/loader/loader.cpp`, `tls.cpp`, `tls_bootstrap.cpp`, `seh.cpp`, `image_contract.hpp`, `src/tlsdll/dllmain.cpp` |
| Target identity | `src/client/loader/target_identity.*`, `component/game_data.*` |
| Component system | `src/client/loader/component_interface.hpp`, `component_loader.*` |
| Hook toolbox | `src/common/utils/hook.{hpp,cpp}`, `hook_validation.*`, `memory.*`, `native_memory.hpp`, `signature.*`, `nt.*`, `startup_callbacks.hpp`, `latest_task_worker.hpp`, `io.cpp (write_file_atomic)` |
| Typed entry points | `src/client/component/vr/h2/{README.md,binding.hpp,entrypoints.hpp}` |
| Scheduler | `component/scheduler.{hpp,cpp}`, `scheduler_context.hpp` |
| D3D11/DXGI layer | `component/d3d11.cpp`, `component/d3d11/swap_chain_hook.cpp`, `d3d11/texture_diagnostics.*` |
| Shutdown ordering | `component/shutdown.cpp` |
| Offline census/contract generation | `tools/generate_scene_surface_contract.py` |
| ABI patch components | `component/scene_surface_storage.cpp` + `_contract.hpp`, `scene_static_surface_*`, `scene_model_lighting*`, `scene_model_identity*` |
| Script VM | `component/gsc/script_extension.cpp`, `script_loading.cpp`, `script_error.cpp`, `component/notifies.cpp`, `component/scripting.cpp`, `game/scripting/*` |
| Protection observer | `component/arxan.cpp` |
| TLS call-site patches | `component/tls.cpp` |
| Hook-bridge tests | `tests/vr/*bridge_tests*`, `camera_bob_tests.cpp`, `tests/loader/game_loader_tests.cpp` |
