# 00 — Feasibility and strategy selection

Decide these before writing code. Record the answers in `docs/target.md` in the new
project and tag each one *confirmed* (observed), *decision* or *pending*. Most of
Overlord's cost was not "put two images in a headset"; it was making an engine that
assumes one view, one thread-of-truth and fixed per-frame budgets behave correctly
twice per frame. Section 7 lists the engine properties that drive that cost, so you
can estimate it before committing.

Contents: 1 target identification · 2 legal and ethics gate · 3 generic tool or
bespoke · 4 stereo strategy · 5 API binding · 6 runtime choice · 7 engine properties
that dominate cost · 8 day-one spikes · 9 go/no-go gate · 10 `docs/target.md` template.

---

## 1. Target identification

| Question | How to answer | Why it matters / Overlord |
|---|---|---|
| Exact executable build | SHA-256, file size, PE timestamp/`SizeOfImage`/checksum, a sentinel dword, PDB GUID if any | All addresses are build-specific. Overlord is **three-layer**: exact size on the file (`game_data::supported_binary_size`), SHA-256 + PE identity recorded into a snapshot (the support bundle prints it), and a sentinel DWORD on the *mapped* image (H2-only: `*(DWORD*)0x140123456 == 0xE465E151`, `src/client/main.cpp: verify_version`). |
| Bitness, ASLR, base | PE header (`ImageBase`, `DllCharacteristics & DYNAMIC_BASE`) | Fixed-address hooks need a fixed base. Overlord clears `DYNAMIC_BASE` in a cached copy and refuses to continue unless the mapped base equals `0x140000000` (before imports/TLS are touched). A proxy-DLL port can use RVAs instead. |
| Packing / DRM / integrity | Entropy, section names, Arxan/Denuvo/Steam stub signatures, self-checksumming | Integrity checks may crash on patched code. Overlord's `arxan.cpp` only *observes* the first 15 bytes of 14 ntdll `Dbg*` entry points once a second and logs; it neutralises nothing and refuses to restore live code. **H2's on-disk `.text` is encrypted**, so all addresses and byte masks were established from the running process and a runtime code dump (03 section 3). Budget for this if the target is protected. |
| Anti-cheat | EAC/BattlEye/Vanguard drivers or services | Kernel anti-cheat means **stop**. Online-only means **stop**. |
| Graphics API | Imports of `d3d11.dll`, `d3d12.dll`, `vulkan-1.dll`, `opengl32.dll`, `d3d9.dll` | Chooses runtime binding and how hard stereo is (section 5). |
| Engine | Strings, module names, asset formats (`.ff`, `.pak`, `.uasset`, `.pck`, `.bsp`) | Generic injector (UEVR, REFramework) or bespoke (Overlord style) (section 3). |
| Render threading | Frontend (game thread builds draw lists) / backend (render thread executes) split? Single-threaded D3D device? | Decides where to publish eye views, where to re-execute per eye, and **which thread may touch the D3D immediate context** (in Overlord, even copy-completion queries stay on the renderer thread). |
| Scripting | GSC/Lua/Papyrus/Blueprint VMs | Cutscenes, scripted cameras, mission logic and UI may live in scripts you can hook or read; script-polled input has timing traps (02 section 13). |
| Config / cvar system | Are names hashed? Saved where? | H2 stores hashed names in config; launcher and hooks must understand `0xHASH` and plain names (02 section 14). Many comfort toggles are existing cvars. |
| Units | Player eye height / known object sizes / gravity description | World scale in units per meter (CoD: 1 unit = 1 inch = 0.0254 m, 39.37 units/m; Unreal: 100 cm; Unity: 1). |
| Projection convention | Capture the projection matrix: reverse-Z? infinite far? row/column vectors? LH/RH? | Every per-eye matrix must match exactly. |
| Command/angle format | usercmd packing, delta angle, move axis quantisation | H2 packs 360 degrees into 2^24 and adds `ps->delta_angles`, unlike Quake's 16-bit; silent-bug territory (06, 07). |
| Fixed budgets | Arena/pool/cache sizes and their overflow warnings | Stereo doubles per-frame packing (section 7, 02 section 12). |
| Save games | What handles/values are serialised | Enlarging caches can break saves (02 section 12). |
| Thread-local storage | Does the exe use static TLS? Inlined TLS accesses? | Matters only for the host-exe loader (02 section 2.7). |

---

## 2. Legal and ethics gate

- Only single-player or offline games the user legally owns. Never help bypass DRM,
  anti-cheat or online integrity checks.
- Overlord refuses unsupported/pirated copies by exact size + hash + sentinel, ships
  **no game assets** (the packager never collects base game data; custom audio/models
  are compiled into the engine's package format from your own sources, read back and
  only the compiled artifact is distributed), and observes rather than defeats the
  protection stub. Do not ship PDBs (local paths) in archives.
- Provenance: inherited fonts, voice/audio files, submodules without a licence are
  explicit review items, not clearance; every bundled dependency keeps its licence file
  in the package (11).
- A VR fork of an existing mod must neutralise the upstream's network services
  (update feed, message of the day), because the upstream can replace VR binaries.
- Inherited multiplayer-era code (crypto, machine id, toast, info-string) is not part
  of the VR method; do not carry it to a new port.

---

## 3. Is a generic tool enough?

- **Unreal Engine 4/5** -> try UEVR first (stereo via `IStereoRendering` emulation).
  Go bespoke only if you need physical weapons/hands beyond what UEVR plugins give
  (you can still write a UEVR plugin).
- **RE Engine** -> REFramework VR.
- **Source / id Tech / CoD-family / custom engines** -> bespoke Overlord-style mod.
- **Unity (Mono)** -> BepInEx/MelonLoader + Unity XR plugin injection is often simpler
  than native hooking.

---

## 4. Choose a stereo strategy

Ranked best-quality first. Overlord uses **A**, in its record-local form.

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **A. Engine-native per-eye execution** | Derive two eye views from the engine's center view, publish them, and run the engine's world render pass (backend record) once per eye with eye-local camera data, isolated targets and history. | True stereo with full engine quality (lighting, particles, post FX). Synchronous, no judder. | Most RE work (culling union, temporal history, per-eye constant buffers, tessellation, depth-hack viewmodel, dynamic lights, dynamic arenas, capacity). |
| **B. Call the scene renderer twice** | At the frontend call `RenderScene(view)` once per eye. | Simplest when the renderer is re-entrant. | Many engines are **not** re-entrant: the first call consumes or finalises frame-global state. |
| **C. Alternate-eye rendering (AER)** | Render left on even frames, right on odd; reproject. | Easy and cheap. | Temporal disparity, motion sickness, halves per-eye rate. Fallback only. |
| **D. Instanced stereo via shader patching** | Patch vertex shaders to write `SV_RenderTargetArrayIndex`. | One pass, fast. | Rewrites every shader (DXBC patching) and all screen-space passes. Very hard. |
| **E. Depth-based synthetic stereo** | Reproject the mono frame using the depth buffer. | Trivial. | Not real VR: disocclusion artifacts, wrong specular. Debug only. |

**Rule:** start with B only if a capture proves the render function is re-entrant per
frame. Otherwise go straight to A. Never ship E. Use C only as a documented fallback.

### 4.1 What "A" really is in Overlord (two failed attempts first)

Overlord **tried B-shaped variants and they failed**; keep the lesson even if your
engine differs:

1. *Dual-record:* synthesising a second scene record before the end-of-frame boundary
   caused the engine's fatal "Tried to use (null)" drop in the one frame it ran (the
   next level entry succeeded once the one-shot stopped mutating).
2. *Second scene-owner call:* corrupts the surface builder's transient state (same-build
   A/B).

Both are recorded as `state=retired` manifests (`diagnostics/renderer_evidence.cpp`).
The shipped design is **record-local stereo view-slot derivation** (06 and 05 for
details):

- **Frontend (game thread):** let the engine fill the center view slot; derive two eye
  slots from it by running the engine's own *pure* finalizer on each (called twice per
  record family, results validated); publish the pair immediately before the draw-
  surface generator, so the render thread can claim it before any consumer enqueues.
- **Backend (render thread):** the only per-eye transaction boundary trusted is the
  engine's per-record owner. Eye 0 runs the engine's own arena record with eye-local
  fields copied in; eye 1 runs a private clone of the whole record (an optional
  auxiliary scope/optic view runs between them). Anything that is not a valid world
  record runs natively: fail closed to flat rendering.
- Gate production on an evidence census (strict multi-part acceptance, 03 section 5.2)
  and distinct-eye readback; never submit half an eye pair.
- **Audit all per-view data**, not just the camera slot: dynamic-light descriptors,
  model placement caches, previous-VP/history, fog-ray constants, FX camera planes,
  depth-hack projection, effect index cursors (05).
- **Capacity audit** is part of the exit criteria (02 section 12).

---

## 5. API-specific binding notes

- **D3D11** (Overlord): OpenXR `XR_KHR_D3D11_enable`; swapchain images are
  `ID3D11Texture2D`. Copy/blit from the engine's eye targets on the **game's own
  device** and immediate context. Validate adapter LUID and feature level. Overlord
  only **queries** `ID3D11Multithread::GetMultithreadProtected()` and records it:
  forcing global serialisation changed every game call and still reproduced the
  kernel failure; GPU work belongs on the actual renderer owner. Preferred state
  preservation: draw into a deferred context and `ExecuteCommandList(list, TRUE)` to
  restore the native pipeline; tag own command lists so hooks ignore them.
  OpenVR needs a DXGI 1.1+ factory before device creation (substitute
  `CreateDXGIFactory` with `CreateDXGIFactory1`, same riid; 02 section 11).
- **D3D12**: `XR_KHR_D3D12_enable` with the game's command queue. Signal/wait fences
  around the copy; hook `ExecuteCommandLists` and Present.
- **Vulkan**: `XR_KHR_vulkan_enable2`. Instance/device creation must satisfy runtime
  requirements, which is hard after the game created them; intercept `vkCreateInstance`/
  `vkCreateDevice` to add extensions.
- **D3D9/OpenGL**: a D3D11 interop device (`WGL_NV_DX_interop2` or shared D3D9Ex
  surface) and submit through D3D11; expect extra copies.

---

## 6. Runtime choice

Default to **OpenXR** with suggested bindings for Oculus Touch, Valve Index and HTC
Vive; keep **OpenVR** only as a manually selected backup. Overlord puts both behind a
`runtime_backend` facade fixed for the process lifetime (no automatic switching after
failure, which would hide errors). The choice is resolved in the host *before* the game
loads (it decides which hooks install); selection happens at init (use `vr_reinit` after
connecting a headset). Environment hygiene (implicit API layers, runtime override,
SteamVR detection) is in 04.

---

## 7. Engine properties that dominate cost (probe these early)

| Property | If true, expect | Overlord evidence |
|---|---|---|
| Fixed per-frame arenas/lists/caches sized for one view | Overflow warnings under stereo; ABI-level patches with offline census | 256 -> 512 KiB surface arena via 242 verified instruction patches; static transparent lists x4; lighting cache 4096 -> 12288 static slots; asset pools x2 |
| Stateful scene owner (consumes frame state) | Replay strategies fail; need pure-function derivation and per-record clone | Two retired experiments (4.1) |
| Temporal history (SSR, TAA/SMAA, previous VP) | Per-eye isolation of targets before first read; per-eye previous-VP constants | SSR target 91, AA histories 16..21, auxiliary slot |
| Per-eye data outside the scene record (lights, instance caches) | Opposite half-IPD errors per eye | Dynamic-light descriptors needed per-eye rebasing (lights are *not* view-independent) |
| Single-threaded D3D device and driver sensitivity | Strict thread affinity for queries and copies; no forced multithread protection | Present-side polling caused an NVIDIA fault |
| Encrypted/packed code | Runtime code dump before patching; hooks only after unpack | H2 `.text` encrypted; late trigger on `GetProcAddress("InitializeCriticalSectionEx")` |
| Static TLS in the exe | TLS donor DLL plus patches for inlined accesses (host-exe loader only) | `tlsdll` + 66 sites |
| Script VM drives missions/UI | VM call bridge, builtin interceptors, bytecode contracts, script-polled input windows | `gsc/*`, `notifies.cpp` (02 section 13) |
| Hashed cvar names | Embed hash function for launcher/config editing | `game/dvar_hash.hpp` |
| Native save games serialise derived handles | Range guards when enlarging caches | lighting handle guard |
| Time-scaled gameplay (bullet time) | Producer continuity: server ticks slow while tracking continues | breach/slow-motion handling (07) |
| Viewmodel depth hack / separate FOV | Per-eye depth-hack projection restore; FX flag leakage | 05 |
| Scripted camera/body ownership per mission | Camera policy composition, per-scene classification from native state | 06, 10 |

Rule of thumb: a flat game whose renderer is re-entrant, single-view-budgeted per call
and has no temporal history is a weekend of work (B). Overlord is the opposite case.

---

## 8. Day-one spikes (cheapest evidence first)

1. Prove identity and load: exact size/hash, `DYNAMIC_BASE`, TLS usage, imports, packer.
   Decide host-exe vs proxy (02 section 1).
2. Hook device creation and Present; log device generation, adapter LUID, swapchain
   desc, which thread presents.
3. Capture a frame in RenderDoc/PIX: projection convention, pass list, depth-hack draws,
   history textures, constant buffer holding view-projection.
4. grep the engine for cvar names (`cg_fov`, `r_*`), print the cvar list; find the
   camera struct; find the script VM; list thread contexts.
5. Start an observe-only census at the frame boundary (view slot, backend record):
   counts, threads, before/after hashes (03 section 5).
6. Inventory fixed budgets: search logs/strings for `exceeded`, `not drawing`, `too
   many`.
7. Write the first `docs/target.md` and the go/no-go decision.

---

## 9. Feasibility gate (go/no-go)

Proceed only if all are true:

- [ ] Single-player/offline mode exists; no kernel anti-cheat.
- [ ] One executable build can be pinned (Steam/Battle.net updates break addresses, so
      plan a version check and update procedure; **never fall back to another build's
      addresses**).
- [ ] You can locate (03): the swapchain, the final camera/view struct, the scene render
      entry, the per-tick input command builder, the viewmodel skeleton/transform.
- [ ] A frame capture (RenderDoc/PIX/Nsight) shows a projection matrix you can reproduce
      bit-for-bit for the mono case.
- [ ] You have a plan for the code-access problem (encrypted/packed `.text`: runtime
      dump) and the late-trigger point.
- [ ] You know whether the renderer is re-entrant (B) or you accept the cost of A, and
      which fixed budgets overflow in stereo.
- [ ] The headset operator and the engineer can agree capture windows (CPU/mock tests
      never establish headset acceptance).

---

## 10. `docs/target.md` template

```
Build:      size / SHA-256 / PE timestamp / sentinel / PDB GUID
Load:       host-exe | proxy | injector; late trigger = ...; ASLR = ...; TLS = ...
Protection: packer/DRM = ...; observed, not defeated; code access = runtime dump (date, scope)
API:        D3D11 (device created by game at ...); present thread = ...
Engine:     name; render split (frontend/backend); scene owner is stateful? (confirmed/decision)
Conventions: units, projection (reverse-Z?), handedness, angle packing, entity sentinels
Boundaries: camera / view slot / backend owner / command builder / skeleton / fire / UI / VM (address, contract, evidence label)
Budgets:    arenas/pools/caches and overflow warnings
Threads:    contexts and the pipeline each hook runs on
Risks:      ...
Decision log: strategy A/B/C, runtime OpenXR, date, evidence
```
