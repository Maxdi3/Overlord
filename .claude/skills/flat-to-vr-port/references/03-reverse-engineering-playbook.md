# 03 — Reverse-engineering playbook: what to find and how

Tools: IDA/Ghidra/Binary Ninja (static), x64dbg (dynamic), RenderDoc/PIX/Nsight
(GPU capture), Cheat Engine/ReClass.NET (live structs), Frida (passive
observation; Overlord used *bounded, passive* Frida sessions that changed no
arguments/returns), Process Monitor, and existing community symbol maps (for CoD:
H2-Mod/IW6x/S1x `game/symbols.hpp`). Look for leaked PDBs, console builds with
symbols, or open-source sibling engines (Quake 3 → id Tech → IW engine).

Write every finding as: **address/RVA, signature, meaning, evidence, build**.
Mark it *confirmed* only after a runtime observation.

## Target list (in the order you need them)

### 1. Graphics device and swapchain
- Import `D3D11CreateDevice(AndSwapChain)` / `CreateDXGIFactory*`. Hook via the
  import resolver or IAT.
- Present/ResizeBuffers: create a dummy swapchain, read vtable slots **8**
  (`Present`) and **13** (`ResizeBuffers`) (`d3d11/swap_chain_hook.cpp`). Hook
  `IDXGISwapChain1::Present1` too if the game uses it.
- Shader creation (`ID3D11Device::Create*Shader`) can be hooked to tag bytecode
  (Overlord's `d3d11.cpp` hooks the engine's wrappers at `0x140758A80` etc.).

### 2. Final camera / view definition
Engine names: `refdef`, `viewParms`, `CViewSetup`, `FSceneView`, `cg.refdef`.
- **Find it**: search for the FOV config var (`cg_fov`), then follow readers to
  the struct that holds `fov_x, fov_y, origin[3], axis[3][3]`.
- Or in RenderDoc: find the view-projection constant buffer and search memory for
  the camera origin floats.
- Then find **the last function before rendering that writes it**. In H2:
  `CG_SetViewPos`-like at call `0x1403ACB2D` → `0x1403B07C0`. That function is
  your **camera composition boundary** (06).

### 3. Scene render entry and view slots
- The frontend function that turns a camera into a renderable view (`R_RenderScene`,
  H2 `0x14077B820`), its view-slot allocator/initializer/finalizer
  (`0x14076D5D0`, `0x14077E340`, `0x14077F550`), and the draw-surface generator
  (`0x140778E60`) that culls and builds lists from the view slot.
- **Determine the layout of the view slot**: origin (`+0x100`), axis, the
  view-projection matrix (`+0x80`), its inverse (`+0xC0`), tan-half-FOV scalars,
  and the near/reverse-Z scale (`+0x148`). Do it by diffing slots across frames
  with known camera motion.
- Find the **finalizer**: the pure function that derives matrices from
  origin/axis/projection. Call *it* rather than writing derived matrices
  yourself, so jitter and other engine conventions stay intact.

### 4. Render backend (if threaded)
- The render thread's per-record execution: H2 `backend_record 0x1407A6BE0`,
  target prepare/world owner `0x1407A7DA0`, view bind `0x14079F6F0`, command
  dispatch `0x14079EAE0`, PostFX destination `0x1407B09A0`.
- Per-view constant uploads: projection cache, eye position, previous
  view-projection (temporal AA, SSR), depth-hack (viewmodel) projection
  builder `0x140786D30`.
- Resource registry: render targets by index, ping-pong buffers, history
  textures. You must isolate these per eye (05).

### 5. Input command pipeline
Quake-lineage engines: `CL_CreateCmd` → `CL_KeyMove/CL_MouseMove/CL_NormalInput` →
`CL_FinishMove` → store in the command ring (`cmds[cmdNumber & MASK]`).
H2: create `0x1403D20F0`, normal input `0x1403D13C0` (call `0x1403D2223`),
FinishMove `0x1403D0300` (call `0x1403D22A2`), client view angles
`0x141E8A65C`, yaw `0x141E8A660`. Also the key-binding executor
(`0x1403CF1E0`) and the notify-on-command route (`0x1403D3A90`) that scripts listen to.
- Find the command struct by watching writes when pressing W/mouse buttons.
- Find where the angle clamp happens (pitch limit) and the player-state update that
  copies command angles (`PM_UpdateViewAngles`, H2 `0x14068D7B0`).

### 6. Player state and entities
`playerState_s` (origin, velocity, viewHeight, viewangles, delta_angles,
pm_flags, weapon fields), `gentity_s`, `centity_s`. Read the predicted
player state from `CG_GetPredictedPlayerState(0)`.

### 7. Viewmodel and skeletons
- The animated object (DObj/skeletal mesh) for first-person arms and gun.
- The **skeleton calculation function** that writes bone matrices after the
  animation blend (H2 `DObjCalcSkel`-like `0x140657D80`). Hook *after* the
  original to overwrite bones (08).
- Skinning/submit boundaries (`0x14071F190`, `0x140774F10`) if you need to
  bind solved poses to specific render records.
- Bone/tag names: `tag_weapon`, `tag_flash` (muzzle), `j_wrist_le/ri`,
  `j_elbow_*`, `j_shoulder_*`, finger joints.
- The "depth hack" / viewmodel FOV projection, which you must correct for stereo (05).

### 8. Weapon fire path
- Server/client fire function (H2 `0x140518A40`), the shot-parameter builder
  (`0x140518880`, called at `0x140518B91`) that computes the start and direction,
  the bullet trace (`0x1404CBFE0`) and spread computation (`0x140518C2E`).
- Ammo fields (clip/stock per weapon index), the reload start/finish
  functions, and the weapon-switch functions.

### 9. UI
- 2D draw command stream (quads/text), the HUD dispatcher, the menu system
  (LUI/Scaleform/CoherentUI), the cursor/mouse input to menus, and video playback.

### 10. Scripts and scenes
- Script VM entry, notify system, `disableweapons`/`enableweapons`, player
  link-to-entity (rides, cutscenes), vehicle attach, and camera tags.

## Techniques that worked in Overlord
- **Census before change**: first install *observe-only* hooks that record
  call counts, threads, argument identities and before/after hashes into
  bounded rings. Only turn the hook into a modifier after the census proves the
  boundary (`engine_view_probe`, `engine_backend_probe`, `gpu_census`).
- **Diff struct memory** before/after an original call to find what it writes
  (`backend_target_prepare_probe_stub` captures record bytes before/after and
  reports changed byte ranges).
- **Read-only witnesses**: compare an expected value (e.g. a bone matrix you
  solved) before and after native work to prove nothing else overwrote it.
- **Asset capture**: dump models, bones and animations with a local script
  (`tools/asset_dump.py`, CASC readers) to measure grip/magazine geometry. Keep
  extracted assets out of the repo.
