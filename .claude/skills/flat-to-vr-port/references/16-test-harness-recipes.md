# 16 — Test-harness recipes for a VR mod (Overlord style)

Policy (what each test level proves, CI, labels) is in `11-diagnostics-testing-release.md`
section 6. This file is the *how*: concrete recipes with code sketches, all taken from
Overlord's `tests/` (about 60 standalone executables registered in `premake/vr_tests.lua`).
None launches the game or needs a headset. Nearly all are Windows + MSVC (Python adapters find
the compiler with `vswhere` + `vcvars64.bat`); only the pure-policy header suites are portable.

Build order for a new port (cheapest, highest-yield first):

1. Header-only policy suites + injected `check` (section 1) — start with the input snapshot,
   head/camera policy and any inventory/state adapter.
2. Fake `openxr_loader.dll` + synthetic scene mode (section 5) before the first headset session;
   this is also the milestone-3 deliverable: a left=magenta/right=cyan runtime mode that isolates
   runtime faults from game faults.
3. asmjit bridge tests (section 4) for *every* mid-function hook before it is installed.
4. WARP tests for every blit/blend/state-restore path (section 7).
5. Production-source slicing (section 2) for adapters that cannot be linked without the engine.
6. Cross-cutting invariants (section 15) as a checklist when writing each new suite.

Common conventions across the repo: tests build with warnings as errors; a failing `check` is
counted (not thrown) so one run lists every violated contract; probes print `PASS: ...` /
`FAIL: reason` and return non-zero; PASS lines state what they are *not* ("CPU/WARP only; no
H2/HMD acceptance"); every consumer takes an explicit `now` so tests can drive 10-200 ms cadences.

---

## 1. Header-only suites with an injected `check()`

One header per concern, one aggregator `.cpp` per executable that owns `check` and the exit code.
No test framework.

```cpp
// closed_bolt_tests.hpp  (suite)
namespace closed_bolt_tests {
template<class Check> void run(Check check) {
    // The message IS the contract; a failure line reads as the violated rule.
    check(capacity(rules, 15, 15, boundary::tactical, kind::refill) == 16,
          "tactical refill keeps the chambered round (15+1)");
    check(!plan(rules, s, replay_of(request)), "revision replay is rejected");
}}
// aggregator main (hand_pose_tests.cpp style)
int main() {
    int failures{};
    const auto check = [&](bool ok, const char* name) {
        if (!ok) { std::cerr << "FAIL: " << name << '\n'; ++failures; } };
    forearm_twist_tests::run(check);
    arm_orientation_tests::run(check);
    horizontal_heading_tests::run(check);
    ...
    return failures ? 1 : 0;           // `inline int run()` variants return a failure count
}
```

- `template<class Fixture, class Check> run(Check)` runs the *same scenario list* against many
  profile instances (reload suites over every weapon profile). The fixture exposes `step()`
  (advance a fake clock 10 ms and bump `input.sequence`), `trigger(bool)`, `button(bool)`,
  `interrupt()`, `adopt(state)`, a `writable` flag and counters (`commits/attempts/spent`).
  `step(false)` replays a *duplicate* frame (same sequence) to prove idempotence.
- **Controllers consume scalar contact geometry** (`slide_distance`, `magazine_top_in_well`,
  `hand_in_gun`, `insertion_alignment`) produced by a separate sampler from tracking + rig, so a
  mechanical controller is unit-testable with no mesh, GPU or tracking.
- Print `combinations checked: N` from every sweep so a silently shrunk matrix is visible.
- Tests can pull a suite into a larger `main` of another slice by `#include` (aggregators include
  30-50 suites; the mock smoke test includes 3).

### 1.1 Native authority as a compare-and-swap mock (the testable form of "native owns state")

```cpp
int native_loaded{15}, native_reserve{90}, commits{}; bool writable{true};
auto commit = [&](const auto& tx) {                 // fixed_drum_tests.hpp / lever_reload_tests.hpp
    if (!writable || tx.before != native{native_loaded, native_reserve}) return false;
    native_loaded = tx.after.loaded; native_reserve = tx.after.reserve; ++commits; return true; };
// plan() is pure:  tx = plan(rules, state, request{op, weapon, instance_generation, revision, actor});
// the adapter commits to native first, then state = tx.next only if accepted.
```

Assert with it: a failed write keeps escrow/owner and is **not queued, deferred or retried from the
held input** (needs a fresh approach); interrupt refunds exactly once and a failed refund keeps
escrow for a later retry; the accepted native shot's debit equals the physical model's debit;
`total_rounds(state) + spent == initial` after every operation; stale instance generation / weapon
token / replayed revision / overflowed revision (`UINT64_MAX`) all reject; external script debits
are reconciled, not faulted or refunded.

---

## 2. Compile the production source slice against boundary fakes

Goal: test the *actual* adapter, not a re-implementation, so drift between test and production is
impossible. Used by `tests/background_update_tests.py`, `config_validation_tests.py`,
`lua_restart_tests.py`, `vr/aim_assist_adapter_tests.py`, `native_ballistics_adapter_tests.py`,
`native_shield_adapter_tests.py`, `hei_feedback_adapter_tests.py`, `launcher_reserve_adapter_tests.py`.

```python
source = (root / "src/client/component/updater.cpp").read_text(encoding="utf-8-sig")

def function(name, indent="\t\t", contents=source):      # brace-matching extractor
    start = contents.index(indent + name)
    opening = contents.index("\n" + indent + "{", start) + len(indent) + 1
    depth, end = 1, opening + 1
    while depth:
        depth += (contents[end] == "{") - (contents[end] == "}")
        end += 1
    return contents[start:end]

harness = PREAMBLE_FAKES + function("void start_update_check()") + ... + MAIN
# PREAMBLE_FAKES re-declares the engine types and replaces ONLY the game/OS endpoints:
#   namespace game { Dvar_FindVar, G_TraceCapsule }, utils::io, scheduler, ui_scripting, console
# and records/asserts at the boundary (exact native address + argument values, event ORDER).
(out / "adapter.cpp").write_text(harness)
driver = f'@echo off\ncall "{vcvars}" >nul\nif errorlevel 1 exit /b 1\n' \
         f'cl /nologo /std:c++20 /EHsc /W4 /WX adapter.cpp /Fe:adapter.exe\n'
subprocess.run(["cmd.exe", "/d", "/c", str(driver)], cwd=out, check=True)
subprocess.run([out / "adapter.exe"], check=True)
# vswhere = %ProgramFiles(x86)%/Microsoft Visual Studio/Installer/vswhere.exe -> installationPath
```

Rules that made these useful:

- Slice by a marker (`namespace X`, `function(signature)`) so production text is compiled
  **unchanged**; the fakes live only in the preamble. Build Debug *and* optimized
  (`/Od /Zi` and `/O2 /DNDEBUG`) when optimizer bugs matter (see `11` §7.1).
- Fakes assert as well as stub: `utils::hook::invoke<R>(address, args...)` checks the exact native
  address constant and argument values and logs `debit / settle / notify / bullet` events; the test
  then asserts the **order** (ammo debit commits *before* the native bullet and the script notify),
  that a failed settle does not refund an uncertain commit, same-tick shots of two weapons/instances
  are distinct, a reentrant call from notify/settle is rejected (`invalid_context`) without double
  fire, script callbacks that replace the player entity / roll back time / bump the epoch yield
  `context_changed` with **no** bullet, and a non-server-thread call is rejected.
- **Fail open to native:** loop over every precondition (tracking, focus, staleness 150 ms,
  ownership revision, memory readable, thread authority, wrong means-of-death...) and require each
  individually to fall through to native behaviour (shield adapter: 13 rejections).
- Fixtures cover pause/menu/disable/unload/time-discontinuity ("consume without delayed replay"),
  non-finite and absurd (1e8) positions, resource independence (visual missing but audio plays),
  native rejection never retried, only changed counts committed, unchanged values not rewritten.
- Nothing touches the user's profile, game or network (HTTP tests use a loopback server with a
  `/stall` endpoint to test timeouts, size limits, cancel and shutdown-while-stalled).
- Variant: `#include` the production `.cpp` into the test TU, redirect an SDK symbol with a macro
  and give the test a `friend struct runtime_test_access` (see 6.2). Variant: exercise a
  Lua adapter against fake host tables in an embedded Lua state (14.1).

---

## 3. Synthetic native layouts

Fake the *bytes*, call the production decoder/adapter, and compare bytes afterwards.

### 3.1 Byte arrays at literal offsets

```cpp
std::array<std::byte, record_size> rec{};                 // exact extent of the engine struct
auto put = [&](size_t at, auto v) { std::memcpy(rec.data() + at, &v, sizeof v); };
put(0x100, origin_xyz); put(0x148, near_plane);
const auto before = rec;
const bool ok = derive_eye_slot(rec, eye);                // code under test mutates the bytes
// only documented fields changed, all other bytes preserved, short/truncated spans reject first:
check(only_changed_at(before, rec, {0x100, 0x148}), "only the documented fields change");
check(!adapter({rec.data(), rec.size() - 1}), "truncated span rejected before any read");
```

- **Restate offsets independently of production constants** and `static_assert` equality
  (`ssr_history_probe.cpp`) so a shared wrong constant cannot self-prove; reconstruct captured
  fields at their literal offsets so a decoder off-by-one cannot pass (`tube_reload_tests.cpp`).
- Always assert: the observer never writes (byte vectors equal before/after), truncated sizes,
  unreadable pointers, ambiguous/duplicate identities (reject both reads and writes, leave bytes
  untouched), negative/huge counts, padding bits not part of identity, all destinations preflighted
  before allocating any (no partial commit), aliased keys rejected.
- Real pages for pointer-validation code: `VirtualAlloc` buffers with canary words around them;
  assert zero writes to canaries and rejection of null, `uintptr` overflow, reserved-uncommitted,
  `PAGE_GUARD`, `PAGE_NOACCESS` and range-straddling pointers; **guard pages must be rejected
  before any read consumes the guard**. Native-memory reader: rejects null/low (<0x10000)/zero
  length/>1 MiB/overflow, contains a cross-page fault (`VirtualProtect(NOACCESS)`), shows no stale
  protection cache after restoring, and does no `VirtualQuery` per field.

### 3.2 Fake PE image, TLS and cache (loader tests, `tests/loader/game_loader_tests.cpp`)

```cpp
std::string executable() {                                // zero-filled buffer of the pinned size
    std::string bytes(supported_binary_size, '\0');
    IMAGE_DOS_HEADER dos{}; dos.e_magic = IMAGE_DOS_SIGNATURE; dos.e_lfanew = 0x80;
    IMAGE_NT_HEADERS64 nt{}; nt.Signature = IMAGE_NT_SIGNATURE; /* Machine, Magic, ImageBase=base, flags */
    memcpy(bytes.data(), &dos, sizeof dos); memcpy(bytes.data() + 0x80, &nt, sizeof nt);
    bytes.back() = 'X';  return bytes; }
// positive: ASLR bit cleared, EVERYTHING else byte-identical (undo the patch => equals original)
// negative: e_lfanew in {-1, 0, 0x7fffffff, size-1}, bad DOS/NT signature, wrong ImageBase, truncated
template<class F> void rejects(F&& op, const char* why) { /* expect std::runtime_error */ }
```

Also: cache reuse when identical, same-size and different-size stale refreshed, **an in-use cache is
leased** (cannot be opened for write or deleted while mapped; a failed refresh does not damage it),
competing writer `.lock` rejected, empty rejected, wrong-size original refused; relocated images
(unexpected base) refused. TLS bootstrap: `VirtualAlloc` fixtures + `VirtualProtect` page states
(NOACCESS/READONLY/EXECUTE_READ), a read-only `MapViewOfFile` view to force a genuine
`VirtualProtect` failure *after* a first permission change succeeded — assert **no partial write**
and protections restored; zero-fill region verified (`'g'`*N, 16 zeros, untouched `'x'`); directory
size/bounds/ordering/overflow/capacity validated.

### 3.3 Bytecode / instruction inspectors: truncation and bit-flip sweeps

Any hook placed inside interpreted script bytecode (or a matched instruction sequence) needs an
exact-layout inspector that returns `std::nullopt` on *any* deviation, and tests that prove it:

```cpp
const auto found = inspect(code, weapon, prop);           // hand-assembled 220-byte fake stream
check(found && found->replacements == expected_offsets, "exact replacement offsets");
for (size_t n = 0; n < code.size(); ++n)                  // every truncation rejected
    check(!inspect({code.data(), n}, weapon, prop), "incomplete script rejected before hooks");
auto mutated = code; mutated[i] ^= 0xFF;                  // each masked byte flipped
check(!inspect(mutated, ...), "changed layout rejected");
// plus: one extra statement (ambiguous) rejected; duplicate block rejected; different operand
// rejected; branch target outside the function rejected; invalid string ids rejected;
// locate by FULL masked signature, admit exactly one match, require the function extent.
```

(`airport_opening_tests.hpp`, `enemy_combat_tests.hpp`, `signal_flare_tests.hpp`, `vehicle_tests.cpp`.)
A mini interpreter model of the VM dispatch boundary can demonstrate why an observer on a redirected
opcode is skipped, and a stack simulation can prove an "entry clear" eats the caller's object.

### 3.4 Captured-witness data headers (regression for real layouts, no redistribution)

`tests/vr/*_data.hpp` contain only bone names, parents, bind transforms and a SHA-256 of the capture
with dated provenance comments ("no mesh or animation asset"). Fixtures assemble the base hands model +
receiver + optional attachments in every order and every glove skin, run the real resolver, and either
require the exact profile or require rejection. Add synthetic *negative* variants to every captured
fixture (renamed bone, chained hinge, wrong parent, NaN, zero basis, bone count 257, parent index 256,
10,000-character names). Build rigs from short name/parent lists:

```cpp
struct fixture { rig r; bone_definition bones[256]; model_definition models[N]; /* hands(68) + receiver + attach */ };
for (glove : gloves) for (underbarrel : ub) for (silencer : sil) for (optic : optics /*13*/)
  for (reversed : {false, true}) { ... check(select_profile(models, rig, bones).id == expected); ++combos; }
std::cout << "combinations checked: " << combos << '\n';
```

Negative cases to include in any "which asset am I looking at" adapter: wrong parent (attachment on a
hand/origin), duplicate or mutually exclusive attachment, unknown model name, **name without topology and
topology without name both rejected**, wildcard/prefix/suffix/uppercase lookalikes, `capacity+1`, an
unregistered *copy* of a profile (identity, not value), cache keyed by DObj pointer + resource generation +
asset epoch so a recycled address cannot alias.

---

## 4. asmjit hostile-callee bridge tests (mid-function hooks)

A hook that detours the middle of a function must preserve every register, flag, SIMD lane and stack
slot except the one value it intentionally changes. Test it by executing generated machine code
(`camera_bob_tests.cpp`, `ads_sprint_tests.cpp`, `animation_index_bridge_tests.hpp`,
`use_range_bridge_tests.hpp`, `launcher_aim_bridge_tests.hpp`, `native_menu_hook_tests.cpp`).

```cpp
asmjit::JitRuntime runtime;
auto jit = [&](auto emit) { CodeHolder code; code.init(runtime.environment());
    Assembler a(&code); emit(a); void* fn{}; check(runtime.add(&fn, &code) == kErrorOk, "JIT"); return fn; };

// 1. "Hostile" policy callee: records rsp&15 on entry (must be 8 = Win64 after CALL), calls the real
//    C++ policy, then CLOBBERS rcx/rdx/r8-r11, xmm0-xmm5 and sets/clears flags.
auto* hostile = jit([&](auto& a) {
    a.mov(rax, &alignment); a.mov(r10, rsp); a.and_(r10, 15); a.mov(qword_ptr(rax), r10);
    a.sub(rsp, 0x28); a.mov(rax, policy); a.call(rax); a.add(rsp, 0x28);
    for (auto r : {rax,rcx,rdx,r8,r9,r10,r11}) a.xor_(r, r);
    for (int i = 1; i < 6; ++i) a.pxor(xmm(i), xmm(i));
    a.clc(); a.ret(); });

// 2. Oracle: the ORIGINAL instruction sequence, e.g. { mulss xmm6,[rax+16]; ret } --
//    or embed the real captured native bytes with a.embed(code, size) (a leaf with no
//    external references can be replayed through every branch with and without the hook).
auto* original = jit([&](auto& a) { a.mulss(xmm(lane), dword_ptr(rax, 16)); a.ret(); });

// 3. Bridge under test, emitted by the PRODUCTION emitter.
auto* bridge = jit([&](auto& a) { emit_scale_bridge(a, lane, (uintptr_t)hostile); });

// 4. Harness: save non-volatile rbx/rsi/xmm6/xmm7, seed ALL gprs/xmm/flags with sentinels
//    (cmp rax,rax; stc), call the target, store rax..r11, pushfq, xmm0..7 into a result struct.
auto* harness = jit([&](auto& a) { ... a.call(qword_ptr(rsp, 0x20)); ... });
using probe = void(*)(void* target, const native_dvar*, void* out);
invoke(original, &native, expected.data());  invoke(bridge, &native, observed.data());
patch_only_the_intended_output(expected);                 // e.g. xmm6 lane = 3.f * factor
check(expected == observed, "bridge preserves GPRs, flags, every other SIMD lane, stack");
check(alignment == 8, "callee enters with standard Win64 alignment");
```

Details that caught real bugs:

- Compare the **whole** register/flag/XMM struct; mask `AF` (`flags &= ~0x10`, "TEST leaves AF
  undefined"). Include a six-argument function with two stack args to test stack-argument forwarding.
  Windows non-volatile `xmm6/xmm7` need explicit save in the harness.
- Entry stubs use `lea rsp,[rsp-0x28]` / `lea rsp,[rsp+0x28]` (not `sub/add`) so flags survive.
- **Installed-path test:** allocate a synthetic site near the real image base
  (`utils::memory::allocate_near(0x140000000, ...)` or `VirtualAlloc` at the real call-site offset),
  write the displaced prefix, `hook::call(site+5, relay)`, run it through the same harness and compare
  with the direct bridge. Create a **far** target (`VirtualAlloc` at `0x20000000000`/`0x50000000000`)
  to force the deployment-only failure: assert the plain rel32 `hook::call`/`jump` **throws before
  writing** (bytes still `0xCC`, neighbours untouched), the preserving relay (`FF 25` indirect,
  `PAGE_EXECUTE_READ` after publication, clobbers no register, `follow_branch(call)==relay`) works,
  and *reproduce the old regression*: the generic `mov rax,imm64; jmp rax` far relay destroys the live
  RAX (a dvar pointer) before the bridge can save it — assert `broken[0] == bridge address` for the bad
  relay and equality for the preserving one. Failed preparation (null bridge) leaves native bytes
  untouched; patch window size and NOP padding asserted.
- Same-technique variants: a "hostile resolver" wrapper that zeroes volatiles after the C++ call and a
  read-substitution hook that changes only RAX; a frame-read thunk that writes only the r32 destination
  (upper bits preserved); an instruction-relocation test (keep the SIB byte and replace all four disp32
  bytes of `movss xmm,[rcx*4+disp32]`; the validator rejects an already-corrupt addressing mode);
  a loop over every 16-bit handle (65,536) through a patched guard, and an assertion reproducing a
  recorded crash-dump arithmetic (`0x1848 - 0x3000 - 1 = 0xffffe847`).
- Tail-dispatch/trampoline emitters: assemble synthetic callers (set RBX/RSI to sentinels, call
  through RAX) and assert the return path/value and the native argument registers survive and only
  witnessed call sites provide the "player context" register.

---

## 5. Fake `openxr_loader.dll` mock runtime

The production code loads the loader with `LoadLibrary` + `xrGetInstanceProcAddr` (a dispatch table
resolved by name), so a DLL named `openxr_loader.dll` beside the test exe stands in for the whole
runtime. Overlord: `tests/vr/mock_loader.cpp` (~1150 lines), `mock_control.hpp`, `mock_smoke.cpp`,
`openxr_*_tests.hpp`; built by premake as a DLL project named `openxr_loader`
(`vr-mock-openxr-loader`).

### 5.1 Shape

```cpp
// exports: the real entry point + test-only control exports
extern "C" __declspec(dllexport) XrResult XRAPI_CALL xrGetInstanceProcAddr(XrInstance, const char* name, PFN_xrVoidFunction*);
extern "C" __declspec(dllexport) void WINAPI h2vMockReset();
extern "C" __declspec(dllexport) void WINAPI h2vMockSetScenario(scenario);
extern "C" __declspec(dllexport) void WINAPI h2vMockFailOnce(failure_point, int32_t xr_result);
extern "C" __declspec(dllexport) void WINAPI h2vMockQueueSessionState(int32_t);   // IDLE/READY/STOPPING...
extern "C" __declspec(dllexport) void WINAPI h2vMockSetViewFlags(uint64_t);       // tracking validity bits
extern "C" __declspec(dllexport) void WINAPI h2vMockSetHandFlags(unsigned hand, uint64_t);
extern "C" __declspec(dllexport) void WINAPI h2vMockSetActionValue(const char* action, float x, float y);
extern "C" __declspec(dllexport) void WINAPI h2vMockSetInteractionProfile(unsigned hand, const char*);
extern "C" __declspec(dllexport) void WINAPI h2vMockSetEyeExtent / SetGraphicsRequirements(LUID, level)
extern "C" __declspec(dllexport) void WINAPI h2vMockSetShouldRender / SetHeadHeight / SetRuntimeName
extern "C" __declspec(dllexport) void WINAPI h2vMockSetCylinderSupported / SetSyntheticChecks
extern "C" __declspec(dllexport) void WINAPI h2vMockGetStatistics(statistics*);
```

The test loads the control exports with `GetProcAddress` (a small RAII `mock_loader_control` class that
also resolves the DLL path next to `GetModuleFileNameW(nullptr)`); each test calls `reset()`,
`set_graphics_requirements(adapter LUID, feature level)` (so the mock refuses a mismatched game device)
and `set_scenario(...)`. At the end: `FreeLibrary` and **assert
`GetModuleHandleW(L"openxr_loader.dll") == nullptr`** to prove production does not leak a module
reference.

### 5.2 Strict call-order state machine (return real error codes)

| Violation | Mock returns |
|---|---|
| `xrAcquireSwapchainImage` twice; `xrReleaseSwapchainImage` before wait | `XR_ERROR_CALL_ORDER_INVALID` |
| `xrDestroySwapchain` while acquired (counted `destroy_while_acquired_rejected`); destroy session with live swapchains/spaces; destroy instance with live session | `XR_ERROR_CALL_ORDER_INVALID` |
| `xrBeginFrame` without `xrWaitFrame`; `xrWaitFrame` twice without begin | `XR_ERROR_CALL_ORDER_INVALID` |
| `xrEndFrame` `displayTime` != the `xrWaitFrame` predicted time | `XR_ERROR_CALL_ORDER_INVALID` |
| `xrLocateViews` `displayTime` != the predicted time (or capacity < 2) | `XR_ERROR_VALIDATION_FAILURE` |
| `xrEndSession` outside STOPPING | `XR_ERROR_SESSION_NOT_STOPPING` — but the mock still sets `running=false` (real runtimes do) |
| `xrSyncActions` unless FOCUSED | `XR_SESSION_NOT_FOCUSED` |
| haptics unless FOCUSED, duration > 100,000,001 ns, amplitude outside (0,1] | rejected |
| `xrCreateInstance` extension list != the exact list the app should enable (cylinder only if advertised) | validation failure |

### 5.3 Real textures and eye-pixel readback in `xrEndFrame` (proves "never show the wrong image")

The mock creates 2 real D3D11 textures per swapchain (render-target bind only) on the device given in
`XrGraphicsBindingD3D11KHR`, so the production blit/copy runs for real. In `xrEndFrame` it validates:

```cpp
// per projection layer: viewCount==2, subImage.imageArrayIndex==0, imageRect extent == eye extent,
// pose + fov EXACTLY equal to what xrLocateViews returned (same prediction for render and submit)
if (view.pose.orientation != located[eye].pose.orientation || view.pose.position != ... ||
    view.fov.angleLeft != -.7f /*...*/ || !read_first_pixel(session, view.subImage.swapchain, pixels[eye]))
    return XR_ERROR_VALIDATION_FAILURE;
// synthetic scene: left pixel red-ish, right pixel cyan-ish, left != right, alpha != 0, equal across eyes,
// and the alpha byte (a "write epoch") differs from the previous frame => rejects stale-texture
// resubmission, the same image in both eyes, and swapped eye order
if (!(p[0].r>200 && p[0].g<160 && p[0].b>200 && p[1].r<160 && p[1].g>200 && p[1].b>200) || p[0]==p[1] || ...)
    return XR_ERROR_VALIDATION_FAILURE;
```

`read_first_pixel` is a 1x1 staging copy. Quad/cylinder layers are validated for rect-inside-texture and
array index 0; the last menu pixel is stored in statistics so a test can assert premultiplied-UI decode.

### 5.4 Scenarios, fault injection, statistics

- `scenario`: `happy, canted_views, parallel_views` (rotated shared head, eye offsets rotated about it),
  `scaled_view_quaternions` (w=.999 / -1.001: equivalent rotations that must be ACCEPTED),
  `view_count_mismatch, invalid_eye_pose (w=0), runtime_unavailable, no_hmd
  (XR_ERROR_FORM_FACTOR_UNAVAILABLE), graphics_mismatch (adapter LUID+1), swapchain_image_failure,
  wait_timeout (first xrWaitSwapchainImage returns XR_TIMEOUT_EXPIRED), destroy_instance_proc_null`
  (`xrGetInstanceProcAddr` returns `XR_SUCCESS` with a NULL function).
- `FailOnce(point, xr_result)` queue consumed at acquire/wait/release image, destroy swapchain/space/
  session/instance, end_session, get_current_interaction_profile, locate_views.
- Statistics for every create/destroy and begin/end pair (`instances_created/destroyed`,
  `sessions_*`, `spaces_*`, `swapchains_*`, `images_acquired/waited/released`, `frames_waited/begun/
  ended`, `zero_layer_frames`, `projection_frames`, `invalid_session_end_rejected`,
  `texture_write_epochs_validated`, `instances_with_runtime_override`) so tests assert
  **created == destroyed after every failure path**.

### 5.5 Test skeleton and the lifecycle contracts worth asserting

```cpp
void expect_wait_timeout(mock_loader_control& loader, const d3d11::device_snapshot& g) {
    configure(loader, g, scenario::wait_timeout);
    openxr::runtime_backend runtime;  runtime.set_desired_enabled(true);
    runtime.set_scene_mode(scene_mode::synthetic);
    require(runtime.initialize(g), "init");
    loader.queue_session_state(XR_SESSION_STATE_READY);
    runtime.on_present(g, 1);
    const auto s = runtime.get_status();
    require(s.state == runtime_state::recoverable_error, "wait timeout is recoverable");
    require(s.last_xr_result == XR_TIMEOUT_EXPIRED, "cleanup did not overwrite the first XrResult");
    const auto st = loader.statistics();
    require(st.images_acquired == 1 && st.images_waited == 1 && st.images_released == 1,
            "timed-out image stays legal to wait/release later");
    runtime.shutdown();
}
```

Assert (each a real pitfall): failure scenarios map to **distinct states** with `applied_enabled=false`
and the loader handed back (`loader_loaded=false`); the **first** failing XrResult/operation survives
later cleanup failures and shutdown; `XR_SESSION_LOSS_PENDING` as a *positive success code* from
`xrWaitSwapchainImage` is success (use `XR_FAILED`, not `!= XR_SUCCESS`); `xrEndSession` flips running
off even on error and is skipped outside STOPPING; a recreated session generation does not inherit
"running" and waits for a fresh READY; STOPPING->READY re-begins; view flags 0 / position-only /
orientation-only => frames closed with zero layers, no images acquired, graph untouched; `shouldRender=FALSE`
=> zero layers, state stays running (`frame_submission.empty` vs `world` vs `menu_only` counted);
**teardown retry** (inject a failure at each destroy step: state `recoverable_error`, loader and object
graph retained, stereo gate deactivated immediately, re-init finishes the teardown, second `shutdown()`
succeeds, counters balance); null cleanup proc after `XR_SUCCESS` => `XR_ERROR_FUNCTION_UNSUPPORTED`;
device recreation (new LUID) => session rebuilt, `device_generation==2`; strict scene source: with
`engine_stereo` but native renderer unproven, *no* images acquired and a zero-layer frame (never a
substitute image); left-only native publication does not acquire/resample/submit, pair slot retires
after the transfer, mismatched Present pre/post frame indices => `fatal_for_vr` and no further
`xrWaitFrame`; canted eyes (11 deg) => `fatal_for_vr`, error string carries the trigger evidence (raw
quaternions, view flags, pair id, display time, measured vs max eye angle ~0.5 deg); failed view
publication => head unavailable, controller frame loses focus + grip validity, `continuity_generation`
bumps, SDK frame closed exactly once; reference-height establishment only from the first FOCUSED +
fully TRACKED pose.

### 5.6 Environment-policy tests that ride on the mock

Runtime selection is decided once per client from an env var (unknown value fails closed); a leftover
`XR_RUNTIME_JSON` pointing elsewhere must not change the default; the override is set only for the
duration of instance creation (mock counts `instances_with_runtime_override`), the environment is
restored afterwards, an explicit user value is never overridden, selection is re-queried at every
re-init, a vanished selected manifest is an error (never a silent fallback), **elevated processes fail
closed with a "high integrity" message** (branch on `process_integrity_level()`, CI runners are
elevated), known-incompatible implicit API layers are disabled via their manifest's own
`disable_environment` variable only. The Steam Link/VD choice is a pure function with a tested truth
table; metadata queries must not overlap a live OpenXR instance (assert `instances_created ==
instances_destroyed` inside the query callback).

### 5.7 Companion probes

`real_loader_probe.cpp` (opt-in, real loader; asserts only "reached a safe state past the loader
stage"), `steamvr_hardware_probe.cpp` (opt-in; RAII env override selecting the backend, synthetic scene,
polls status every 5 ms for 8 s, requires `submitted_frames >= 90` and left==right released counts, and
its PASS text asks a human to verify magenta-left/cyan-right). Keep both out of regression batches.

---

## 6. Strict-stub OpenVR (or any SDK) doubles

### 6.1 Every virtual throws

```cpp
class IVRCompositor_stub : public IVRCompositor {        // generated from the vendored SDK header
public:
    EVRCompositorError WaitGetPoses(TrackedDevicePose_t*, uint32_t, TrackedDevicePose_t*, uint32_t) override
        { throw std::logic_error("unexpected IVRCompositor::WaitGetPoses"); }
    EVRCompositorError Submit(EVREye, const Texture_t*, const VRTextureBounds_t*, EVRSubmitFlags) override
        { throw std::logic_error("unexpected IVRCompositor::Submit"); }
    /* ... every other virtual ... */
};
class compositor final : public IVRCompositor_stub {     // a test overrides ONLY what it expects
    EVRCompositorError WaitGetPoses(...) override { ++wait_calls; return next_error; }
    EVRCompositorError Submit(EVREye e, const Texture_t* t, ...) override { record(e, t); return submit_result[e]; }
};
```

Accidental SDK use fails loudly instead of contacting SteamVR, and the stub documents the exact SDK
surface the backend uses. Overlay double (`openvr_overlay_test_interface.hpp`) additionally asserts API
misuse: transforms use `TrackingUniverseStanding`, `SetOverlayWidthInMeters` finite and > 0, intersection
only on visible overlays, handle leaks (`api.surfaces.empty()` after shutdown).

### 6.2 Inject through the production TU

```cpp
// openvr_lifecycle_tests.cpp
namespace vr { unsigned test_runtime_detach_count{}; void test_runtime_detach() { ++test_runtime_detach_count; } }
#define VR_Shutdown test_runtime_detach                 // fake keeps interfaces through Present-post
#include "component/vr/openvr_runtime.cpp"              // production implementation compiled once here
#undef VR_Shutdown
struct runtime_test_access {                            // `friend` of runtime_backend: attaches fakes,
    static void attach(runtime_backend&, graphics, IVRSystem*, IVRCompositor*, bool armed, bool probing);
    static bool submit(runtime_backend&, uint64_t pair);  // sets frame_phase/status without VR_Init
};
```

No shipping test API is added and the real SDK shutdown is never called (it would disturb a live headset).
The real `.cpp` is `removefiles`'d from the test target.

Contracts to assert (OpenVR-specific): `DoNotHaveFocus` from `WaitGetPoses` is recoverable (no teardown,
input gate reason retained, compositor fades cleared and restored); a pair already handed to `Submit` and
then interrupted by focus loss stays owned by the compositor (not retired, not resubmitted while
`WaitGetPoses` keeps failing; retired once after the next success); a hard error in either eye is not
masked by the other eye's focus error; non-focus pose errors are fatal; no scene => keep calling
`WaitGetPoses`, show a dark theater (`FadeGrid` + `FadeToColor`), admit no stale gameplay pairs, leave
theater only after a fresh native pair is submitted; resize only *requests* (rebuild at the next Present;
foreign-device resize and a disabled runtime do not retry init; a resize burst causes no init retry loop;
a generation change without a destroy callback triggers exactly one init attempt); Present-transaction
key `{frame index, device generation, owner thread}` with outcomes `matched / missing_pre /
frame_mismatch / generation_mismatch / thread_mismatch / present_failed`; shutdown with a Present pending
is deferred until Present-post, and late callbacks can never revive a terminally shut-down runtime;
`GetProjectionRaw` top/bottom inversion pinned (`projection_tangents_from_raw(-1.376,.839,-1.428,.966)`).

### 6.3 Link-time stubs for engine boundaries

Replace diagnostics, graphics status, owner-pass reports, scene-resolution requests and native-menu
boundaries with test `.cpp`s (`diagnostics_stub.cpp`, `native_menu_stubs.cpp`, `gpu_census_stubs.cpp`).
A stub must not claim what it cannot do (the scene-resolution stub answers "requires the real engine"
unless the test installs an override). A recursive-mutex "GPU queue interop gate" probe (attempt and
acquisition counters + condvar) lets a two-thread test prove Present-thread and runtime-thread
immediate-context work is serialized (`active_owners` never > 1).

---

## 7. WARP D3D11: readback, math, and state restore

```cpp
D3D11CreateDevice(nullptr, D3D_DRIVER_TYPE_WARP, nullptr, 0, nullptr, 0, D3D11_SDK_VERSION, &dev, &lvl, &ctx);
// (test_support.hpp create_graphics: HARDWARE first, WARP fallback, FL 11_1..10_0; a second device on
//  the same adapter for device-replacement tests; check feature support e.g. MSAA quality levels)
auto readback = [&] {                                    // STAGING copy; honour RowPitch
    desc.Usage = D3D11_USAGE_STAGING; desc.BindFlags = 0; desc.CPUAccessFlags = D3D11_CPU_ACCESS_READ;
    dev->CreateTexture2D(&desc, nullptr, &staging);  ctx->CopyResource(staging.Get(), output.Get());
    D3D11_MAPPED_SUBRESOURCE m{}; ctx->Map(staging.Get(), 0, D3D11_MAP_READ, 0, &m);
    for (y...) memcpy(result + y*w, (char*)m.pData + y*m.RowPitch, w*4);  ctx->Unmap(staging.Get(), 0); };
```

Pattern: tiny textures (8x8..64x64) with known pixels, run the **real** renderer class, read back, assert
exact or banded values.

### 7.1 State restore after every pass

```cpp
const D3D11_VIEWPORT sentinel{3,4,7,9,.2f,.7f};  ctx->RSSetViewports(1, &sentinel);
ctx->IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_LINELIST);
ID3D11ShaderResourceView* saved = ink_view.Get();  ctx->PSSetShaderResources(5, 1, &saved);   // unrelated slot
panel.draw(ctx, ...);
ctx->RSGetViewports(&n, &restored);   check(!memcmp(&sentinel, &restored, sizeof sentinel), "viewport restored");
ctx->IAGetPrimitiveTopology(&topo);   check(topo == LINELIST, "topology restored");
ctx->PSGetShaderResources(5, 1, &srv);check(srv.Get() == saved, "unrelated SRV restored");
```

Repeat after every pass; include RTV, scissor, VS/PS/RS state (assert "came back null" when the native
state was empty). **Preferred mechanism to test:** draw in a *deferred* context and
`ExecuteCommandList(list, TRUE)` to restore the native pipeline; tag your own command lists so hooks ignore
them (a conversion test binds a sentinel state — RTV, viewport {7,11,113,127,.125,.875}, scissor,
LINESTRIP, SRVs on PS and CS — and asserts every item after each copy).

### 7.2 What to assert under WARP

- Colour math: sRGB decode exactly once (1x1 render of linear (.25,.5,.75) through the mirror pixel shader
  reads back (137,188,225) +-2; half-alpha black in linear decode reads green 53..56); format-converting
  copies R11G11B10 -> {R16G16B16A16F, R10G10B10A2, R8G8B8A8} equal sRGB->linear of the source within 0.007
  (equal formats must not bypass decoding: "gray washout" regression).
- HDR preserved (float 4.0/2.0/3.0 survives a blur), reverse-Z depth-aware effects (nearer foreground
  stays sharp), no cross-eye contamination (same renderer, different scene per eye), clip-W < 0 corners
  handled, premultiplied-alpha blend modes (`REV_SUBTRACT` border), transparent padding leaves the
  destination untouched, partially-behind-the-eye quads keep their visible edge.
- Blend algebra: to flatten native HUD blends into an alpha-coverage layer, brute-force `a,b,d` in 0..255
  step 15 and require the layered form equals native within 1e-12; reject unproven modes (coloured
  reverse-subtract, additive dest, partial write mask, alpha-to-coverage without MSAA); call
  `CreateBlendState` on every derived desc so D3D validates it.
- Device replacement: a second WARP device; device objects rebuilt from shared bytecode, constants and
  shader objects per-device, retired devices never contribute to the current device.

### 7.3 Hook-installer probes on a real WARP context

Install the **production** vtable hooks on the WARP context and drive real D3D calls
(`execution_probe.cpp`, `draw_indexed_probe.cpp`, `constant_buffer_probe.cpp`, `output_merger_probe.cpp`,
`resource_ops_probe.cpp`). Read vtable slots for immediate and deferred contexts, assert the installed hook
count and that targets are distinct (and which slots are shared); drive `Draw/DrawIndexed/Instanced/
Indirect/Dispatch/ExecuteCommandList` and assert counters, per-API census, event identity (context,
caller, thread, bound RTV/DSV) and normalized args. Compile real HLSL with `D3DCompile` and attach
bytecode via `SetPrivateData(guid_shader_bytecode)` so the probe can reflect which slots a shader uses.
Assert all six `*SetConstantBuffers` stages are distinct hooks that pass through; an observer receives
a trampoline (not the hooked vtable) and sees exactly one native draw even when it re-enters; a private
copy of the vtable simulates an API-mode switch; retired contexts stay hooked but never contribute to the
current device; per-channel observer registration (detaching one does not displace another); a replay
guard detects unexpected OM binds; terminal reports are `memcmp`-equal after device replacement; and the
diagnostic copy itself never appears in the observer stream. Production never creates a deferred context;
only an isolated probe may.

### 7.4 GPU timing, readback and census probes

Async queries: polls with `D3D11_ASYNC_GETDATA_DONOTFLUSH` and a bounded retry count (the implementation
never calls Flush; the harness may); unstable/out-of-order/incomplete pair => terminal `unavailable` with a
specific error; unrelated retirement ignored; completed generation cannot re-arm. Readback comparators
compare only the exact indexed range (vertex 0 outside the range does not contaminate), keep a CPU
reference snapshot, and discard a stale reference on re-arm.

---

## 8. Fake-vtable device (hook observers with no GPU)

```cpp
HRESULT STDMETHODCALLTYPE fake_create(ID3D11Device*, const D3D11_TEXTURE2D_DESC* d,
        const D3D11_SUBRESOURCE_DATA* init, ID3D11Texture2D** out) {
    ++calls;  if (d != expected_descriptor || init != expected_initial) arguments_match = false;
    if (blocking) { SetEvent(entered); WaitForSingleObject(leave, 5000); }   // in-flight call
    if (SUCCEEDED(result) && out) *out = (ID3D11Texture2D*)0x123400;
    SetLastError(0x4321);  return result; }
std::array<void*, 6> fake_table{};   struct fake_device { void** table{fake_table.data()}; } fake;
fake_table[5] = (void*)fake_create;                      // slot 5 = CreateTexture2D
ID3D11Device* dev = (ID3D11Device*)&fake;                // first member is the vtable pointer
td::install(dev, /*generation*/7);   td::install(dev, 7);   // reinstall keeps the original method
```

Assert: the original's result, `LastError`, output pointer and arguments are unchanged; **an
unregistered device sharing the same method passes through untouched** and does not enter the game's
records; a failure output pointer is never dereferenced; the first failure (descriptor, caller, generation)
survives a 180-call ring wrap; an in-flight blocked call is held in an "active" slot independent of the
recent ring; the evidence lands in a real minidump supplement stream (`MiniDumpReadDumpStream`); a
4-writer x 300-call concurrency test checks field coherence and monotonic sequence numbers. WARP is added
to check the same observer against a real device and a simulated `DXGI_ERROR_DEVICE_REMOVED`.

---

## 9. Concurrency: seqlock stress, bounded queues, deterministic races

### 9.1 Seqlock/publication stress with torn-payload detection

```cpp
// content snapshots: payload A or B with caller A or B; a torn read mixes them
const auto hash_a = hash_bytes(values_a), hash_b = hash_bytes(values_b);
std::thread writer([&] { while (!start) SwitchToThread();
    for (size_t i = 0; i < 20000; ++i) {                 // worker publishes COMPLETE snapshots
        auto& v = (i & 1) ? values_b : values_a;
        observe_update_content(ctx, buffer, v.data(), sizeof v, true, (i & 1) ? caller_b : caller_a);
        (i & 1) ? bind_site_b(ctx, buf_b) : bind_site_a(ctx, buf_a);   // two noinline sites => distinct callers
    } });
start = true;
for (size_t i = 0; i < 4096; ++i) observe_vs_draw(ctx, i + 1, 0xC003, shader, buffer);   // owner records
writer.join();
// every recorded (hash, caller) pair must be one of the two legal combinations; slot metadata
// (caller, thread) must match the call site; the report is complete with ordinal_mismatches == 0.
```

Make the writer the *only* thread touching the immediate context during overlap. Use two
`__declspec(noinline)` bind functions to obtain distinct caller addresses. A 16-byte payload hashed with a
mix (FNV + mix) makes torn payloads detectable.

### 9.2 Bounded MPSC queue / recorder (`region_capture_tests.cpp`)

4 producers x 10,000 pushes into N=256 with a slow reader; invariant `received + dropped == 40000`,
rows never torn (`a == tick ^ thread`), producers never block; with the sink atomic null every call is a
no-op and the record is untouched; RAII phase scope emits begin/end once even with explicit `finish()`.
Binary-format tests feed the analyzer hand-packed files (see 13.3). Registries that outlive a zone
reload need a `reset_after_drain()` test (1000 reload epochs must not exhaust the ledger).

### 9.3 Deterministic race tests via admission-pause hooks

Plan the seam up front: `test_arm_admission_pause(stage)` (stages `after_gate_open`, `after_state_check`,
`after_gate_claim`), `wait_for_admission_pause()` (5 s deadline then release+fail),
`test_release_admission_pause()`. A natural D3D call is held mid-admission on another thread while the
owner closes the gate. Prove: a call admitted in the gate-opening window belongs to the new scope (not a
collision); a call already registered as a writer when the gate closes forces `closing` with
`active_writers==1` and the terminal state publishes only after the last writer; that case is an explicit
hard `failed/admission_collision`, never a silently dropped event in a "complete" census; two racing
`begin_*` hard-fail and are not silently merged; cross-thread `reset()` while a scope is active is
refused; reset quiesces concurrent writers (32 cycles with two hammering threads). Also: threaded
identity-registry test (one reader + batch publishers; the reader sees only complete immutable entries).
`H2VR_*_TESTING` defines swap hot-path hooks for stubs.

---

## 10. Seeded fuzzing of conservation properties

Any inventory/ammo/ownership adapter should survive random operation sequences with an invariant checked
**after every step**. Seed a PRNG (deterministic failures):

```cpp
std::uint32_t random = 12345;  int spent = 0;  const auto total = total_rounds(s);
for (int n = 0; n < 12000; ++n) {                        // break_action_tests.hpp
    random = random * 1664525u + 1013904223u;            // LCG
    const auto op = operation((random >> 16) % 9);
    const auto tx = plan(rules, s, {op, weapon, gen, s.revision, hand::right, other_hand, (random >> 8) & 1, float(random & 1)});
    if (tx) { s = tx.next; spent += tx.rounds_spent; }   // reject => state unchanged
    check(valid(rules, s) && total_rounds(s) + spent == total && !(s.live & s.spent),
          "random operation sequences preserve masks and ammunition");
}
```

Other uses in the repo: 10,000 random ops (manual bolt) with `valid()` + commit == native exactly; 50,000
`std::mt19937(44)` ops per rear hand with `total + spent == 30` (cylinder); 4,000-step LCG carry-state
sequences asserting slot/hand uniqueness each step; 128 shuffles of candidate offers to prove
order-independence of the arbiter (a refused commit never falls through to another candidate); randomized
frames in a Lua adapter (1000 frames leave no pending transitions).

---

## 11. Exhaustive, cadence and equivariance sweeps

- **Bounded exhaustive sequences:** all `11^5 = 161,051` five-operation sequences per policy (closed-bolt,
  AK, open-bolt) from a native-admitted state; at each accepted step assert `tx.before == native_ammo(s)`,
  `total(s) == total(tx.next) + tx.rounds_spent`, next state valid, an immediate replay of the same request
  rejected, unsupported features never synthesized, open-bolt never chambers.
- **Cadence sweeps:** a gesture detector or controller must give the same mechanical outcome (exactly one
  extraction, one feed, conserved ammo) at every consumer tick. Loop `hz in {45, 60, 72, 90, 120, 144}`
  (and 15/20/30 for slow sources), phase offsets {0, .35, .75}, and slow-consumer intervals 200/250 ms;
  melee at 10/20/50/100 ms; independent fire at 25/50 ms; a falling rail rendered at 45/90/144 Hz against a
  20 Hz source publication; smooth-turn integration over 30/72/90/120/144 Hz must total exactly the same
  angle per second; exponential finger smoothing `1 - exp(-ln2 * t / half_life)` verified to 3e-6 at
  60/72/90/120/144 fps. A 1 s gap must create no catch-up travel; presentation follows the render-time pose
  without advancing simulation.
- **Scale/space invariance:** world scales {1, 20, 25, 39.37, 40, 80, 100}, gun frame identity and an
  arbitrary rotation+translation, left and right hands (hand-swapped mirrored skins); rotate/translate the
  whole skeleton + targets and require the IK result to transform identically.
- **Transactional rejection:** call a solver with a bad input (cyclic parents, out-of-range index, zero
  quaternion, NaN, 10,000-unit target, short output span, malformed basis) and `memcmp` that outputs and
  flags are bit-identical afterwards.
- **Combinatorial matrices** with printed counts (section 3.4); **conservation across orderings**
  (cock-then-insert vs insert-then-cock, shot-before-fetch, staged-before-eject yield the same totals).
- **Prediction-replay admission:** `history.allow(weapon, instance, command, native_before, ...)` — replay
  of an already-spent command refused, a genuinely new command accepted, evicted entries (ring ~150) and
  wrong instance/count refused (the direct test of "never mutate on read").

---

## 12. Evidence-field mutation tests for classifiers

Scripted-scene/phase classifiers take a struct of native evidence and return a policy; test them by
mutating **one evidence field at a time** and requiring `phase::none`:

```cpp
for (unsigned mutation = 0; mutation < 5; ++mutation) {  // scripted_sequence_tests.hpp
    auto e = valid_evidence;
    if (mutation == 0) e.boarding = false;               // each field the classification depends on
    if (mutation == 1) e.parent = 0;     if (mutation == 2) e.body = 0;
    if (mutation == 3) e.parent = 42;    if (mutation == 4) e.rig = "player_rig";   // unrelated rig
    check(airport::classify(e).stage == phase::none, "death/unlink/missing/stale/unrelated cannot inherit");
}
```

Rules enforced: entity references compare **id AND generation** (`same_entity(0,0)` false; 35371 vs 35372
false); a scene can be entered from a checkpoint mid-way, so classification needs no replayed local start
event; gaze alone cannot unlock an early action (needs the native admission flag); a sentinel flag alone
cannot capture later animations; ownership releases on native unlink/death; unknown/prefixed checkpoint
names (`crawl_extra`) -> none; presentation structs (camera profile, `suspend_weapons`, `allow_movement`,
`hide_body_arms`) are pure functions of the phase. Add property sweeps for geometric consumers: authored
pitch across the vertical pole (sweep 5..120 deg both signs) must not create yaw/roll; run every camera
policy with a normal and an alternate Euler-branch history across normal / invalidate / recenter stages and
require the same matrix within 0.003 (no 180-degree reversal).

---

## 13. Python tests for tooling and data contracts

### 13.1 Deployment is transactional (`tests/vr/test_dual_deployment.py`)

```python
spec = importlib.util.spec_from_file_location('deploy_client_pair', ROOT / 'tools/deploy_client_pair.py')
module = importlib.util.module_from_spec(spec); spec.loader.exec_module(module)

class DeploymentTests(unittest.TestCase):
    def setUp(self):                       # temp build/ game/ backup/ trees; old+new bytes per file;
        ...                                # a config.cfg 'preserve profile' file in game/
    def test_locked_debug_rolls_back_regular_pair(self):
        replace = module.os.replace; calls = 0
        def fail_third(src, dst):
            nonlocal calls; calls += 1
            if calls == 3: raise PermissionError('debug executable is running')
            return replace(src, dst)
        with patch.object(module.os, 'replace', side_effect=fail_third):
            with self.assertRaises(PermissionError): module.deploy(self.build, self.game, self.backup)
        for name in self.old: self.assertEqual((self.game / name).read_bytes(), self.old[name])   # byte-exact restore
        self.assertFalse(list(self.game.rglob('*.staging-*')))                                   # no leftovers
        self.assertFalse(json.loads(next(self.backup.glob('*/deployment.json')).read_text())['deployed'])
```

Also: missing or mismatched inputs abort **before any replacement** (`assertFalse(self.backup.exists())`),
mismatched paired configs (`ValueError`), `../outside.json` binding path rejected, unrelated game files
preserved, a failure at a specific destination (binding file) rolls back the binaries and manifest too, a
brand-new input directory is installed, second deploy gets a different backup directory.

### 13.2 Binding/manifest contracts (`test_knuckles_binding.py`, `test_trigger_touch_binding.py`; CI-run)

Load the action manifest and each binding JSON; assert: every binding output matches the declared action
type (`position->vector2`, `click/touch->boolean`, pose->pose, haptics->vibration); each action has exactly
**one** source; the thumbstick uses joystick mode; trigger click `mode=button`, `force_input=value`,
`0 < deactivate < activate <= 0.01`; trigger *touch* is a separate boolean bound to `touch` with no
`force_input`; grip `force_input=force` with `0 < deactivate < activate < 1`; pause/recenter/secondary
actions reachable on each controller profile; `default_bindings[].binding_url` exists inside the package and
cannot escape it; `haptic_amplitude` handling. Run before the build in CI.

### 13.3 Offline analyzers with hand-packed binaries (`test_region_capture_analysis.py`)

Build capture files in the test (`struct.pack` header `H2VREG01` + 3 u64, 11-u64 rows), include a truncated
tail, dropped events and a lost phase end, and assert the analyzer makes the loss explicit
(`timing_intervals_excluded_for_loss`, a lost end never pairs with a later begin, `trailing_bytes`
reported, `clean_stop` != loss-free). Same for the crash-evidence reader: it must parse a dump produced by
the native test and reject truncated structures.

### 13.4 Config/launcher contracts, fake readers, localization

- Config: defaults for every key; missing file / `{}` / `[]` / `null` / `123` return defaults; invalid
  values (empty, 600 chars, 1 MiB, embedded NUL, non-ASCII, negative, float, array, object) are normalized
  on read **without rewriting the file**; `set()` persists the sanitized value without damaging unknown
  extension keys. Compile production namespace in Debug and optimized modes (section 2).
- Launcher bridge/profile merge: last recognized assignment wins including hashed names
  (`seta 0x2BA94576 "baton"`) and case-insensitive duplicates; BOM, CRLF, comments, unrelated binds,
  compound lines, 100 kB lines and a missing final newline preserved; saving twice is a no-op; validation
  rejects unknown/missing fields, wrong JSON types, NaN/inf, out of range, injection in enums
  (`snap"; quit`); the bridge is UTF-8 validated, size/nesting limited, method allow-listed, origin exact,
  child-process arguments round-trip through `CommandLineToArgvW`; preflight repair idempotent with a fixed
  allow-list. React unit tests (`node:test`) cover `bridge.ts` and `model.ts` (session UUID discards stale
  replies, 64 pending / 30 s timeouts, strict numeric validation, revision-guarded save/revert).
- External-tool fakes: a `Reader` whose `read()` asserts, so a wrong EXE hash must abort **before any
  address read**; fixed capture deadline, no automatic second window, invalid readiness neither extends the
  deadline nor ends the session, a read error preserves the partial window plus an end marker.
- Localization completeness: every official locale covers and formats every key; partial catalogs fall back
  per sentence; arguments are never interpreted as templates; malformed/over-long templates yield empty
  (never a partial sentence); UTF-8 never split; shell/preflight key sets identical across locales.
- Low priority but cheap: locale parity is not translation quality.

---

## 14. Other harness pieces

### 14.1 Embedded Lua and script layers
Run the shipped Lua adapter in a real Lua state with fake host tables (`LUI.MenuBuilder`, `Engine`,
`Game`; `weapon_hud_lui_tests.cpp`, `killfeed_audio_tests.lua`): early and late registration both work;
unrelated defs untouched; duplicate registration still throws; invalid values (-1, 0, 1.5, 1025, inf,
`"15"`) leave a widget untouched; weak-table (`__mode='v'`) GC checks prove retired trees are not strongly
held and live element count stays bounded over 150 swaps; a failed constructor is not retried every tick and
releases its guard; profile-backed values are written only on the frontend `onmenuopen` callback and never
override a later user choice; a damage callback returns nil so feedback cannot replace native damage; dvar
strings like `"off"`, `"unknown"`, `"mw2019;quit"` disable a feature. Compile the shipped GSC with the
engine's compiler/assembler and require a non-empty script+stack. Lua context restart (`lua_restart_tests.py`):
16 restart cycles, exactly one live context, every callback retired before `lua_close`, native hooks survive.

### 14.2 Aggregated policies and probes already named
`aggregate_publication_tests.cpp` and the artifact audit are described in `11` §7.1; wait-timing,
support-bundle and texture-diagnostics tests are in `11` §1.4/§2/§4; `crash_evidence_tests.cpp` generates
real exception and watchdog dumps and unwinds from captured memory only (>= 4 frames, no module file),
proves the focus thread (not the writer) is dumped and resumed, skips invalid/inaccessible ranges, and
verifies the `IsProcessSnapshotCallback` handling does not misidentify a live process as PSS.
Callback registration: a move-only RAII slot owner (duplicate registration throws, moved-from handle cannot
clear the slot, reset cannot clear a replacement with the same function, destructor unregisters).

### 14.3 Launcher self-test
`-launcher-smoke <report.json> -launcher-probe-size WxH -launcher-probe-page ...` opens a hidden WebView2
with an isolated profile, never loads the game, runs layout/animation/geometry inspections in-page and writes
`renderer.ready` to a report; the Python wrapper copies the EXE into a temp dir with a synthetic config,
applies a 35 s timeout, checks `version == version.json`, the embedded `settingNames == settings-schema.json`
names (catches a stale bundle) and that every preset field is displayed, and cleans only dirs under its
output. A timeout writes `{"ok":false,"error":"Renderer startup timed out"}` so CI never hangs.

---

## 15. Cross-cutting invariants every suite should encode

Treat this as a checklist when writing each new gesture/adapter/render test.

| # | Invariant | Typical test |
|---|---|---|
| 1 | **Second-eye / replay idempotence:** any time-integrating consumer is a pure function of (input sequence, generation); the engine renders twice and replays prediction | call twice with the same sequence; state unchanged (`step(false)`) |
| 2 | **Held-at-entry / resume needs a neutral sample + fresh press** (initial held, pause/menu, focus loss, recenter, stale, future timestamp, sequence rollback, reconnect, weapon switch, checkpoint, mode switch) | `sample_buttons(held)` then resume => blocked until release |
| 3 | **Edges are counters** (`presses/releases`), not state: an unread tap between two samples still produces exactly one press; coalesced taps no catch-up; release+reclose cannot replay | slow consumer at 200 ms still sees the edge |
| 4 | **Tracking loss is not a release:** a real release during pose loss is deferred, delivered once with zero velocity impulse; a new press cancels an older deferred release; recenter/focus loss cancel it | deferred bit tests |
| 5 | **Identity = definition + instance generation (+ revision), never name**; ids compare id AND generation; recycled objects/addresses cannot match stale sessions | recycled generation rejected |
| 6 | **Queued render data is keyed by pair/publication/camera/record address**; eviction => *not found*, never "latest"; two guns sharing a record do not share a pose | eviction test returns not-found |
| 7 | **Duplicate frame / duplicate publication is a no-op;** a failed native write consumes the edge and is not retried from the same press | `writable=false` => no ammo change, no retry |
| 8 | **Fail closed / fail open to native:** every precondition tested individually; any failure returns native behaviour unchanged | precondition loop |
| 9 | **Transactional publish:** invalid input leaves outputs byte-identical; all-or-nothing batches; capacity failure without eviction | `memcmp` after rejection |
| 10 | **Generation counters fence staleness:** `sequence`, `reference_generation` (recenter), `continuity_generation` (tracking discontinuity), binding/action generation, instance, device, session, pair/publication, skeleton epoch, checkpoint epoch | each bump discards pending intent |
| 11 | **Per-consumer staleness thresholds, not one number:** 150 ms input/muzzle/ADS (151 rejects), haptics 100 ms, calibration settle 200 ms, smooth-turn dt cap 50 ms and gap >150 ms inactive, body-estimator gap reset 200 ms, damage overlay 250 ms, magazine insertion / cover-push cooldown 300 ms, slap sample gap 160 ms | boundary test at limit and limit+1 ms |
| 12 | **Bounded everything with visible overflow counters;** names/strings/arrays length-capped (10,000-char names, 257 bones, 1 MiB manifests) | oversize inputs rejected before allocation |
| 13 | **Frame-rate and scale independence** | cadence and world-scale sweeps (section 11) |
| 14 | **Presentation never mutates authority:** a visual full stroke cannot change ammo; stowed rendering is read-only (revision/rounds unchanged over 100 renders) | render N times then compare state |
| 15 | **Non-finite and extreme inputs** (NaN, inf, negative, zero basis, reflected axes) rejected without poisoning history | NaN sample then valid sample |
| 16 | **Per-hand independence:** missing sensor on one hand does not cancel or poison the other; run `{left,right} x {state variants} x {write ok, write fails}` | cross-product loop |
| 17 | **Rendering invariants under WARP:** restore caller state, no cross-eye contamination, HDR not clamped, reverse-Z occlusion, sRGB decode once, partial-clip geometry still draws | section 7 |
| 18 | **Hook-bridge invariants:** registers/flags/SIMD/stack alignment/args preserved; native bytes untouched on failed preparation | section 4 |
| 19 | **Evidence is first-failure-preserving and labelled;** unknown != equal; terminal evidence immutable | `11` §5 |
| 20 | **Tests prove their own negative:** reproduce the pre-fix failure; include synthetic negative variants beside captured fixtures; positive controls guard against vacuous passes | e.g. read-mask bit 2 positive control |

Also keep test names/messages as full sentences, print what the pass does *not* establish, record dated
provenance for captured fixtures, and never let CPU/mock/WARP results stand in for headset acceptance.
