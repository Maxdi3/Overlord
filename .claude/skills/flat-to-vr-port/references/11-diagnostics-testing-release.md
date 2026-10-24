# 11 — Diagnostics, testing, acceptance and release

Overlord files: `vr/diagnostics.cpp`, `vr/diagnostics/*` (status sections, support
bundle/archive, renderer evidence, wait timing), `component/exception.cpp`
(minidumps), `vr/region_capture.cpp`, `premake/vr_tests.lua`, `tests/vr/*`,
`.github/workflows/build.yml`, `tools/package_client.py`, `tools/deploy_client_pair.py`.
Docs: `vr-diagnostics-workflow.md`, `development.md`, `releasing.md`,
`release-validation.md`, `vr-region-capture.md`, `vr-interaction-diagnostics.md`.

## 1. Diagnostics you must build early

- **`vr_status`** prints all subsystem snapshots and writes them atomically to
  `diagnose/<mod>-status-latest.txt`. It includes build identity (git rev, dirty
  flag, PDB GUID) and target identity (original and cached game hashes).
- **Per-feature status commands** (`vr_fire_status`, `vr_input_status`, …) with
  counters for applied/rejected and a last-reason string.
- **One-button support bundle** (`vr_diagnose` / Ctrl+Shift+F8): about 5 s
  collection into one ZIP with two fresh statuses, console history, traces,
  filtered settings, preflight results and recent dumps. Size-bounded
  (8 MiB per file, 24 MiB total). Never uploaded automatically.
- **Stall watchdog thread**: every 250 ms it checks Present progress and
  scene-hook entry/exit. On a stall it writes status plus a focused minidump.
- **Crash handler**: a minidump plus a text sidecar with the exception context.
  Keep matching PDBs.
- **Bounded rings** for traces (fixed arrays, sequence numbers). File writes happen
  on a control thread, never on render or present.
- Status getters never take locks held across runtime pacing.

## 2. Test pyramid

| Level | What | Overlord examples |
|---|---|---|
| Pure CPU unit tests | Math, policies, state machines, ammo invariants, input edges, IK | `vr-weapon-mechanics-tests`, `vr-hand-pose-tests`, `vr-controller-input-tests`, `vr-campaign-policy-tests`, `vr-aim-assist-tests` |
| Synthetic native layouts | Fake structs and bytes to test adapters and verifiers | `vr-native-binding-tests`, `game-loader-tests` (synthetic PE/TLS) |
| WARP D3D11 tests | Blits, eye resource isolation, mirror crops, color conversion | `vr-desktop-mirror-tests`, `vr-d3d11-eye-resource-isolation-probe`, `vr-engine-ssr-history-probe` |
| Mock runtime | OpenXR doubles: lifecycle failures, exact display-time submission, partial pairs, focus, haptics | `vr-runtime-mock-smoke`, `vr-runtime-no-loader-smoke`, `vr-openvr-lifecycle-tests` |
| In-game observe-only probes | Census of hooks, threads and record identities | `engine_view_probe`, `engine_backend_probe`, `gpu_census` |
| **HMD acceptance** | A human in the headset with a checklist | Binocular alignment, parallax, transparency, particles, reflections, terrain edges, weapon/hand alignment, all controls, pause/menus/video, recenter, focus loss, resize/device recovery, exit |

Each test target is a standalone executable (premake `test_executable`). CI
builds Debug and Release (warnings as errors) on Windows.

**Labeling rule:** CPU/mock/WARP results never claim headset acceptance. Release
notes list what was HMD-validated, on which headset and runtime.

## 3. Debugging workflow

1. Identify the run: build commit, EXE/PDB hashes, map/checkpoint, headset,
   runtime, GPU driver, settings.
2. Save the evidence before restarting.
3. Pick the smallest observation: status → region capture (timing) → focused trace
   → dump.
4. Separate input, mechanical state, native result and visual presentation before
   tuning offsets.
5. Agree a capture window with the headset operator. Outside it, assume the
   headset is off.

## 4. Release

- Package only mod files: `overlord.exe`, `openxr_loader.dll`, `<mod>.vrmanifest`,
  `vr_input/`, mod fastfiles/scripts and licenses. **Never game assets.**
- Pin the supported game build and refuse others with a clear message.
- Document runtime selection (OpenXR default, OpenVR backup) and known issues
  (overlay tools like MSI Afterburner hooking Present can conflict).
- License: Overlord is GPLv3 (inherited from H2-Mod/IW6x/S1x). A fork or reuse of its
  code must stay GPLv3 and keep notices (`THIRD_PARTY_NOTICES.md`,
  `docs/source-provenance.md`).
