# 17 — Script VMs, UI scripts, localization and custom assets

Many flat games run their missions, menus and HUD on **embedded VMs** (Quake/IW GSC,
Lua/Havok Script, Source VScript, Blueprint-style bytecode). A VR port that only
injects input and re-renders the camera cannot make those scenes physical, cannot
re-word prompts, and cannot ship its own sounds or models. This file is the
contract for integrating with the game's VMs and content pipeline. Read it
together with `02-injection-and-hooking.md` (byte verification, relays),
`07-input-and-commands.md` (script waiters), `09-ui-hud-menus.md`,
`10-comfort-and-scripted-scenes.md` (scene adapters) and `11-diagnostics-testing-release.md`.

Everything H2-specific (addresses, opcode bytes, token ids, map names) is a
labelled example. The *shape* of each technique is what transfers.

## 0. Overview: four VMs and one content pipeline

| Layer | Overlord's instance | What the port does with it | Thread/owner |
|---|---|---|---|
| Mission script VM | H2 GSC (bytecode, server side) | call into it, add builtins, intercept builtins, redirect/skip bytecode, inject VR-only script sources, observe notifies | **server pipeline only** |
| Menu/HUD VM | H2 LUI on Havok Script (Lua 5.1 fork) | wrap definitions, localize hook, event injection, private widget copies, read-only C++ snapshot bridge | `lui`/main pipeline, under LUI critical section |
| Mod-script VM | sol2 Lua per script folder (H2-Mod inheritance) | sandboxed user scripting over the GSC bridge (killfeed, hooks) | server pipeline (`run_frame`) |
| Native C++ | the mod | the only place that touches XR, D3D, tracking | many (see `01-architecture.md`) |
| Content pipeline | ZoneTool CSV/JSON -> `.ff` fastfiles | custom sounds, materials, fonts, models, localized strings | offline tool + process-lifetime zones |

Core principles (all verified in code, see sections below):

1. **Restore diagnostics before building on the VM.** Shipped builds strip script error text; Overlord re-implements the `Scr_Get*` accessors and the link/runtime error paths so every VM mistake prints a precise message (`gsc/script_error.cpp`).
2. **Drive scripted logic through the triggers the flat game uses** (notify the waiter, raise the same flag from script, send the real command bit). Never set mission flags, mount vehicles or fake outcomes from C++.
3. **Observe, don't consume.** Interceptors return "not handled" unless they own the whole replacement; the native call keeps running.
4. **Every bytecode edit is a contract**: exact shape + uniqueness + bounded size, all-or-nothing, fail closed to native behaviour, unbound at shutdown, re-bound on every level start.
5. **VM access has an owner thread.** Script calls only from the server pipeline; command construction and render threads read published snapshots (rule 7 in `SKILL.md`).
6. **Plugin state lives in engine save state** (script fields/arrays on entities or level), not in process globals, so checkpoints/loads restore it.

---

## 1. Calling the script VM from native code (the bridge)

Overlord: `src/client/game/scripting/{execution,safe_execution,stack_isolation,script_value,entity,array,vector,variable_value,function,functions}.*`.

### 1.1 Value model and refcounts
- A VM value is `{union u; int type}`. Script types (H2 numbering): UNDEFINED 0, OBJECT/POINTER, STRING 2 (interned id), ISTRING 3 (localized), VECTOR 4, FLOAT 5, INTEGER 6, FUNCTION 9 (code position), ANIMATION 13, STRUCT 19, ENTITY 21, ARRAY 22, FREE 24. Entities/arrays/structs are ids into a global variable pool; strings are interned ids; vectors are VM-allocated; bool is an int.
- **Refcount hygiene is mandatory.** Every wrapper that stores a value calls `AddRefToValue` on assign/push and `RemoveRefToValue` on release; copy/move are explicit. `script_value(const char*)` calls `SL_GetString` (which adds a ref) and removes its own temporary ref with a scope guard after the wrapper's AddRef. A missed or doubled ref corrupts the VM later and far from the cause.
- The pool is finite: H2 `scr_VarGlob` = 56320 object slots, 65536 child buckets, 384000 child values; child storage sharded by `id & 3` with stride `0xA000`; integer array keys encoded `(index - 0x800000) & 0xFFFFFF`; array length lives in the object's `next` field. Count non-FREE entries as a leak gauge (`getvarusage`).

### 1.2 Calling convention (builtin function or method)
```cpp
// args pushed in REVERSE order onto the VM operand stack
for (auto it = args.rbegin(); it != args.rend(); ++it) {
    auto* slot = ++vm->top; ++vm->inparamcount;
    *slot = it->raw(); AddRefToValue(slot->type, slot->u);
}
vm->outparamcount = vm->inparamcount; vm->inparamcount = 0;   // the callee reads them as "out params"
call_guarded(builtin, entref);                                  // see 1.4
// the callee's Scr_Add* result arrives as a NEW in-param:
if (vm->inparamcount) { Scr_ClearOutParams(); vm->outparamcount = vm->inparamcount;
                        vm->inparamcount = 0; result = vm->top[1 - vm->outparamcount]; }
```
- Methods take the target through `scr_entref_t{entnum, classnum}`. **Global functions use `{0xFFFF, 0xFFFF}`** (entity id 0 maps to it); `is_method_call` is `*(int*)&entref != -1`.
- Function lookup: the GSC compiler context holds name->index maps (`func_map`, `meth_map`); index `< 0x1000` -> `function_table[index-1]`, else method table `[index - 0x8000]`. The lookup is **case-sensitive on the canonical lowercase name** (the code lowercases a copy and never uses it, a latent bug: pass lowercase). Token ids resolve via the compiler context, `_IDnnn` literals, or `SL_GetCanonicalString`.
- Running script *code* (not a builtin): `exec_ent_thread(entity, codepos, args)` = isolate stack -> push reversed args -> `AddRefToObject(id)` -> `AllocThread(id)` -> `VM_Execute(localId, pos, argc)` -> `RemoveRefToObject(result)`. `call_script_function(entity, "file", "func")` resolves `codepos` via `get_function_pos(file, func)` (table built while scripts load).
- Notify: isolate stack, push reversed args, `Scr_NotifyId(entityId, SL_GetString(event), argc)`. This is how native code wakes a script waiter (Overlord boards a snowmobile by sending `trigger` to the story's own use proxy, never by setting its flags).

### 1.3 Stack isolation
`stack_isolation` (RAII) swaps `top/maxstack/inparamcount/outparamcount` to a private **512-slot** array and on destruction calls `Scr_ClearOutParams` and restores the four fields. Needed whenever native code calls the VM **from inside a hook that fires mid-script** (e.g. from a builtin the script is currently executing); a bare push/pop tramples the live frame's operand stack.

### 1.4 Error containment (setjmp + SEH)
Native script errors do not return: they `longjmp` to `g_script_error[g_script_error_level]`.
```cpp
bool call_guarded(fn, entref) {
    ++*g_script_error_level;
    if (game::_setjmp(&g_script_error[*g_script_error_level])) { --*g_script_error_level; return false; }
    bool ok = seh_call(fn, entref);   // __try { fn(entref); return true; } __except(EXECUTE_HANDLER) { return false; }
    --*g_script_error_level; return ok;
}
```
- Use the **game's own `_setjmp`/`longjmp` symbols** so the `jmp_buf` layout matches the engine's longjmp. A mod-side C++ `setjmp` is a different format.
- Field get/set (`Scr_SetObjectField`, `GetEntityFieldValue`) need the same guard; on error return a NONE value.
- Wrap every mod->VM call in this; otherwise a script typo or a bad argument kills the process.
- Pitfalls: `script_value::is<bool>` accepts any integer; `array::operator[]` template can fall off the end without return; `find_function_index` ignores its lowercased copy (all in the inherited bridge, do not copy).

### 1.5 Entity fields vs struct fields
Native class fields (`level`, player, `.health`, `origin`) go through `fields_table[classnum][name] -> offset`; everything else is an object variable addressed by token id. Entity-ness check: for `classnum == 0` the pool slot must be `SCRIPT_ENTITY`. Scene identification in adapters uses script state (linked parent, `animname`, level flags, object variables), see 10.

### 1.6 Re-entrancy guard for hooks called from Lua
Calling a detoured GSC function from Lua sets `notifies::hook_enabled = false` for the duration (scope-guarded) so the hook does not recurse into itself.

---

## 2. Extending the VM: builtin table, custom builtins, interceptors

Overlord: `src/client/component/gsc/script_extension.{cpp,hpp}`.

### 2.1 Enlarging the builtin table
- Replace the VM's builtin function table with your own `void* func_table[0x1000]` (`FUNC_TABLE_SIZE`): patch the count constant, the table-size operand, and every code reference (RIP-relative operands via `set<uint32_t>(RVA(&func_table))`, an `inject`ed absolute operand, and the indirect call-site, which is NOP-ed and replaced by a call to `vm_call_builtin_function`). H2: five table patches + call patch + error-print patch (addresses in 12).
- Custom ids start at `function_id_start = 0x320` pre-incremented, so the **first custom id is 0x321**. `scr_get_function_stub` re-registers every custom function each time the engine re-resolves its table (each script load); do not register once.
- Names are added to the compiler context maps (`func_add(name, id)`), so compiled GSC can call them and runtime lookups by name work. If a name already exists the existing id is reused (that is how you override a stock builtin).
- Dispatcher `vm_call_builtin_function(fn)`: first interceptors (2.3), then **"custom" = function pointer outside the game image** (`ptr > BASE_ADDRESS + BINARY_PAYLOAD_SIZE` is custom; the host module is based above that range, 0x160000000, so the test works by address). Custom functions run in `execute_custom_function` inside try/catch: **C++ exceptions never cross engine frames**; the message is stored, `force_error_print` is set (print even without developer mode) and `Scr_ErrorInternal()` raises a normal script error. Always throw `std::runtime_error` from a builtin to report a bad argument.
- Method ids are >= 0x8000-based; error printing distinguishes function vs method by `function_id > FUNC_TABLE_SIZE`.

### 2.2 Custom builtins as the VR bridge
GSC (small VR-only scripts, section 4) calls native builtins such as `vrphysicalclimbbegin/state/milestone/fallside/entryready/end`, `vrendingpullwork`, `vrendingknifehint`, `vrcheatflaremonitor`; the builtin reads/writes native VR state and returns a value. Rules:
- Builtins are called on the server pipeline (the VM thread). They may read published snapshots and call VM helpers, but never block on the render/XR thread.
- Keep every builtin **total and cheap**: bounded work, validate every argument, return a defined value on any failure.
- Return values that scripts compare against fixed contracts (see the killfeed count check in 8.3).

### 2.3 `intercept_builtin(name, callback)`
```cpp
using builtin_interceptor = bool(*)(game::BuiltinFunction);   // true = "I supplied the return value"
void intercept_builtin(const std::string& name, builtin_interceptor cb);
```
- **Startup-only, bounded registry (max 8)**; registration throws on null callback, unknown name, full table or id out of range. Matching: `func_table[id - 1] == function` at dispatch time. Only builtin *functions* (not methods) can be intercepted.
- `false` = preserve the native call (observe); `true` = the interceptor already produced the return value, **using an isolated call** (`stack_isolation` + guarded call) and `Scr_ClearOutParams()`.
- Overlord's 5 uses: `notifyoncommand` (trainer hint observation, returns false), `vectordot` (ammo-cache gaze check, one call site), `bullettrace` (laser-designator trace replaced with muzzle origin/direction), `magicbullet` + `playfxontag` (vehicle gun).
- **Restrict every interceptor to the call site you mean.** Accept the call only if (a) `scr_function_stack->pos` lies inside the known native function's byte extent (`in_range(begin,end)`), (b) the frame's owner is the player (`player_scope()`), (c) `outparamcount` and argument types/values match (e.g. weapon name), and otherwise return false so the original runs untouched. A broad interceptor changes unrelated missions.
- To fire a physical shot from a scripted weapon loop: replace the position arg with the controller muzzle, require a cover trace head->muzzle, call the original isolated+guarded, then deduct ammo by compare-and-write on the script object field (refuse if it changed; skip if the game's sustain-ammo dvar is set), acknowledge the XR press once, emit haptics. On any refusal clear out-params and return true so the native loop sees no shot (vehicles/native.cpp).
- Never re-enter `VM_Execute` from a builtin that is already running inside the native shooting loop: compute the needed script object on the server coordinator beforehand and return it from the builtin.
- Notification/aliasing and damage filters are the other two bounded observer registries (section 5.4).

---

## 3. Hooking script *code*: redirects, guards, bytecode contracts

Overlord: `component/notifies.{cpp,hpp}`, `vr/gameplay/script_function_extent.hpp`, `campaign/cliffhanger/physical.cpp`, `campaign/sequences/airport_opening.hpp`, `vehicles/script_contract.hpp`, `signal_flare_script.hpp`.

### 3.1 The dispatch hook
The VM interpreter loop is patched (H2 `0x1405C90A5`, `r14` = bytecode pointer) with an assembled stub: `pushad; call execute_vm_hook(pos); if true -> r14 = target; popad`. A **byte-per-position table** (`bool gsc_hooks_buffer[0x1400000]`, 20 MiB) indexed by `pos - program_buffer` gives an O(1) "is there a hook here" before the map lookup, so unhooked opcodes cost one load.

```cpp
notifies::set_gsc_hook(const char* source, const char* target, bool(*guard)() = nullptr);
```
- It is a **redirect**: when the VM would execute `source`, execution continues at `target`.
- The VM **decodes the target opcode immediately and does not consult hooks there**; hooks on the target are not chained. To observe a place that is also a redirect target, hook the *next* instruction (flare: `waiter_observe = waiter_prepare + 1`, reached by both the VR skip and the native fallthrough). Coalesce adjacent skipped statements in the resume point.
- `guard()` is evaluated **at dispatch time**; false = run the original opcode. A `hook_enabled` one-shot flag suppresses re-hooking while a Lua hook calls the original.
- `replacefunc(what, with)` (a registered builtin) is just `set_gsc_hook(what, with)` with no guard. Replacing a whole function means copying its body (e.g. H2-Mod's `_nextmission` copy): fragile across builds, valid only with a pinned executable. Prefer redirecting the **minimum boundary** and calling the original helpers.
- Lifecycle: Lua hooks are cleared on Lua restart (`clear_callbacks`), GSC redirects only when scripts are freed (`on_shutdown(free_scripts && !post_shutdown)` clears the table). Bind in `on_level_start`, unbind in `on_shutdown`, never retain VM string references through shutdown, bump an epoch so consumers drop stale state.

### 3.2 Finding the code
- `get_function_pos(file, function)` -> bytecode pointer. Files and functions are keyed by token ids (`_id_BB6C`, `_id_B33F` in H2) or names; the registration tables expose aliases, so resolve by *address*.
- **Function extent:** next greater function address in the file's sorted table; reject if not found or `> 65536` bytes (`script_function_extent`). Everything below searches only inside that extent.
- `install_hooks()` catches every exception, records a `registration` reason string and leaves native behaviour; `hooks_ready` is only set after **all** sites resolved and bound. A missing function or extent is a soft failure of that feature, not of the game (contrast the loader's hard fail, see `01-architecture.md` section 2).

### 3.3 Two kinds of guards
| Guard | Question | Example |
|---|---|---|
| **configuration guard** | should VR own this script *at all* (VR on, hands on, trace signature verified, map name, linked parent `animname == "worldbody"`)? | cliffhanger `eligible()` |
| **call-site guard** | was this function called from the one native caller I mean? | `called_from(span)`: look at `scr_VmPub->function_frame[-1].fs.pos`, require it inside the caller's extent; reject frame depth above 31 |

Claim ownership from **configuration, not transient readiness**: if the guard also demanded "assets and HMD ready", a redirect that fell through would let the native loop start and become unhookable. The replacement script instead *waits inside the bridge* (`while (!vrphysicalclimbentryready()) wait 0.05;`) until VR is ready. (physical.cpp:117-119)

### 3.4 Bytecode contracts (fail closed)
Never patch by hand-picked offsets. A script patch is accepted only if **all** of:
1. **Signature + mask** over the function's extent. Mask relocated operands (interned string ids, relative call targets): flare block = 259 bytes, searched within an extent <= 2048 bytes.
2. **Operand cross-check:** every string operand at the fixed offsets is compared with the live-resolved interned id for the expected strings (`flare@9/21/205, fire@92/109, drop@97/188, hint@72`).
3. **Uniqueness:** exactly one match; zero or two = reject all.
4. **Bounded size** and named offsets tied to the signature by `static_assert`.
5. **Shape counts:** airport opening requires exactly 8 weapon statements with expected names/methods, entry bytes `32 89 79 2e`, exit `89 a0 01 2e`, last bytes `6a 34`; vehicle inspectors require exactly 1 reload, 1 refill, 1 consume site and N `waittillmatch` sites (zodiac 1+1, snowmobile 2+3) plus 1 or 2 shot tails; the dog-knockdown adapter verifies the caller bytecode and redirects its first field read to an existing native `return 1`.
6. **Whole-statement skips only.** To skip a statement redirect from its first opcode to the resume point after its arguments *and* its `DecTop`/cleanup; the redirect must be stack-neutral so no argument or PRECODEPOS marker is left on the VM stack.
Any failure: `++binding_failures`, status line `... contract rejected`, **no partial install** (bind all sites or none), native behaviour remains.

### 3.5 The OP_clearparams pitfall (cost real days)
H2 `OP_clearparams (0x4D)` removes values from the stack down to a `SCRIPT_CODEPOS (7)` marker. A freshly called function has a `SCRIPT_PRECODEPOS (8)` marker under its arguments that only the function's own **parameter prologue** converts. Redirecting at function entry into a replacement that starts with a clear/check-params opcode ran past PRECODEPOS and **released the caller's live references**, which crashed unrelated input/lighting/render code. Rules:
- Execute the original prologue, redirect only the **body** (`target_body_offset` verifies the prologue bytes `2c 00 2c 01 2c 02 32` before choosing body +7).
- A continuation that replaces a return value is `{OP_CallBuiltin1(0x1a), id_lo, id_hi, OP_Ret(0x19)}` placed *after* the original parameter prologue; never insert clear/check-params opcodes in it.
- A null-pointer guard at the crash site would have hidden the real corruption; find the frame-crossing.

### 3.6 Rewriting waited events in place
To make a script `waittill` private VR notifications: patch the string-id operands of the verified block (`utils::hook::set(block + 92, private_fire)`), map events per mode with the notify alias (5.4), keep flat mode's original controls. Restore edits on shutdown (`restore_events` while keeping control-flow guards when loading a save; full `restore` when scripts are freed).

### 3.7 Save compatibility of bytecode edits
After loading a flat save the old waittill and a forced item may both be live: a one-shot `recover()` on a 50 ms server loop ends exactly that mission thread via a notify, uses the script's own public methods (`allowfire`, `enableweaponswitch`, `switchtoweaponimmediate`, `takeweapon`, `setsaveddvar`) and restarts at the guarded native entry. The trainer scans `scr_VarGlob->objectVariableValue` for a `VAR_NOTIFY_THREAD` on the level entity to recover a restored `notifyoncommand` waiter.

---

## 4. Embedded VR-only script sources (`scripts/vr_*` virtual sources)

Ship a handful of GSC lines **inside the DLL** so the script cannot drift from the native builtins it calls.

```cpp
gsc::register_virtual_source("scripts/vr_cliffhanger_physical", script_source, "cliffhanger"); // post_unpack only
```
Validation (`script_loading.cpp`): name starts `scripts/vr_`, only `[A-Za-z0-9_/]`, no `..` or `\`, no `.gsc` suffix, <= 128 chars, source non-empty and <= 65536 bytes, optional map <= 64 chars `[A-Za-z0-9_]`, no duplicates; any violation throws. **Never replace a native asset name.** With a map tag the source loads only when the `mapname` dvar matches.
- Compiled by the in-process `xsk::gsc` compiler (dev mode with line/column devmap when `developer_script`, else prod), placed in the engine's persistent script arena (`PMem_AllocFromSource` `PMEM_SOURCE_SCRIPT`, `custom_script_mem_size`; `ERR_FATAL "Out of custom script memory"` if exceeded) and freed in the scripting shutdown callback.
- `main` handles run in `g_load_structs_stub` (before the level's struct load), `init` handles after the level load stub. A `db_is_x_asset_default` stub makes custom scripts look non-default; `db_get_raw_buffer_stub` serves uncompressed raw files.
- Loose `.gsc` in search paths (`scripts/`, `scripts/<map>/`) and rawfile assets named `scripts/*.gsc` use the same loader (that is the inherited mod path, e.g. killfeed ducking).
- Overlord's embedded sources: `cliffhanger_physical` (climb loops, carrier), `ending_interaction` (crawl, knife pull, hint text), `official_cheats` (inventory escrow wait loops), `exo_punch`. Each is a few dozen lines that call `vr*` builtins.

### 4.1 Authoring rules for the embedded GSC
- Redirect only the boundary you need ("only the two native climb loops are redirected; entry, first-wall exit, failure and final exit remain native"). Call the same native helpers for falls/cleanup so flags, notifies, saves behave identically.
- **Declare checkpoint fields in the compiled script** (`body.vr_climb_entry = undefined; body.vr_climb_handoff = 0;`, `declare_saved_fields()`), so canonical field ids are registered consistently before either save loading or C++ access.
- GSC arrays are copy-on-write: helpers must return the array.
- Poll builtins at `wait 0.05` (matches the engine's 50 ms script cadence); never busy-loop.
- Text: produce hint strings via a builtin and pass them into the native hint call (`vrendingknifehint`) so native layout, blink and cleanup still apply.
- Story flags (progress milestones) are **set by the replacement script from physical milestones**, not from C++ (rise > 4/70/100 units set bits that the script converts to `player_begins_to_climb`, `fourth_swing`...). Authority for story state stays with the script.
- Adapt notetrack handlers instead of reimplementing them: wrap the native hit callback to capture the real impact point/normal, then delegate.

---

## 5. Lifecycle, notifications, damage

### 5.1 Level lifecycle
`scripting::on_level_start(cb)` fires after native spawn (`ClientSpawn`) **and after a saved-level load** (`SV_CheckLoadLevel` with a non-null save), so consumers invalidate derived state "even if player pointers/time repeat". `scripting::on_shutdown(cb(free_scripts, post))` is called before and after native `G_ShutdownGame`. Typical use: bind hooks in `on_level_start`; clear them in `on_shutdown(free_scripts && !post)`.

### 5.2 Defensive stubs worth copying
`Scr_Delete` refuses entity 0 (the player); `get_spawn_point_stub` falls back to `g_entities[0]` when a map has no spawn; level-start callbacks reset scheduled notifies and restart the Lua engine (`engine::start()` always calls `stop()` first because spawn/load can restart without a shutdown).

### 5.3 Observing script state for input
Some GSC waiters use `notifyoncommand` (binding notify), some **poll command-derived state every 50 ms** (`meleebuttonpressed`, attack/ADS, sentry placement); the engine's edge latch clears each frame, so a one-command pulse is missed. Deliver per receiver type: binding notify, **held bit with >= 100 ms visibility**, or a real command bit that reaches the native weapon state. Poll scripts on the server pipeline and publish `{epoch, timestamp}`; command construction reads only the published copy ("no script VM access from command construction"). Details in `07-input-and-commands.md`.

### 5.4 Bounded observer registries (all startup-only)
| Registry | Semantics | Cap |
|---|---|---|
| `intercept_builtin` | see 2.3 | 8 |
| `scripting::on_notify_alias(fn)` | for each VM notify the original event is **always delivered**; `fn(owner, event, args)` may name **one additional** event (deduped). Lets VR emit private notifies without replacing native ones | 8 |
| `notifies::add_entity_damage_filter(fn)` | chain: Lua callbacks (may return replacement damage) -> C++ filters (retained across script/Lua reloads) -> native; `damage == 0` short-circuits | unbounded list, null rejected, dedup |
| `set_gsc_hook` guards | per hook | one per source position |
Template: damage filter = enemy melee scaling (`MOD_MELEE`, AI attacker, player target) while the engine stays authoritative.

### 5.5 Other declarative script-to-native APIs
- **PMove profile** (`setpmoveprofile(player, array)`, `movement.cpp`, `docs/pmove-scripting.md`): a validated, atomically swapped profile executed inside predicted+authoritative movement hooks with replay-safe press edges keyed by `serverTime` and a generation counter. This is how scripts add sprint/slide/dash without per-frame VM calls. See 07.
- `luinotify(name, data)` builtin queues a LUI event on the `lui` pipeline: the GSC -> menu path.

---

## 6. Scripted-scene movement through the VM (carrier / proxy pattern)

Short recipe; full scene policy lives in `10-comfort-and-scripted-scenes.md`.
1. The replacement script spawns a hidden non-solid `script_model` with model `tag_origin` (the **carrier**), `playerlinktodelta(carrier, "tag_origin", 1, 0,0,0,0, 1)`, `playersetgroundreferenceent(carrier)`, hides the authored body, then loops on a native state builtin at 50 ms.
2. C++ moves the carrier by writing the entity `origin` through the script field API after a **swept capsule** (`G_TraceCapsule`, hanging volume derived from the standing bounds, `slide(from, delta, 0.005*units)`). Verify by re-reading the origin (error if it differs by > 0.02 units: "native carrier movement was rejected"). Link-to-delta preserves the native eye offset: read it only after the link advanced (`wait 0.05`).
3. Handoff back to the authored animation: `body.origin = body_start + carrier.origin - carrier_start`, unlink, relink to the body's `tag_player`, show, delete carrier, play the native exit animation. Failure path uses the native fall helper.
4. Persist plugin state in one VM array on the carrier (`vr_climb_save`: body, start, finish, final, milestones, result, fall side, per-hand support records) on every attach/detach/milestone. Two restore paths: from entity fields after a load, and from the serialized GSC thread's arguments. Validate on restore: ids < 56320 and `SCRIPT_ENTITY`, entnum < 4000, `|start-finish| <= 1600`, world scale in [0.01, 10000], unit normals within 0.01, bits masked, results clamped; "missing historical contacts do not manufacture a release/fall". Detect rollback when the client command time goes backwards; compare entity keys (entity + generation) each tick to detect recycling; rebase tracking baselines after any restore.

---

## 7. Mod Lua (sol2) and the GSC<->Lua bridge

Overlord: `game/scripting/lua/*`, `component/ui_scripting.cpp` (UI Lua is separate, section 8).
- `SOL_ALL_SAFETIES_ON`, `SOL_PRINT_ERRORS 0`; one `sol::state` per script folder (`<search_path>/scripts/<dir>/__init__.lua`) plus a generic first context used by a console `lua` command. `engine::start()` always `stop()`s first; `stop()` clears native callbacks while states are alive (no dangling sol functions).
- **Sandbox:** nil `package.searchers[3,4]`, `package.loadlib`, `os.execute/exit/getenv/remove/rename/setlocale/tmpname`; replace `io` with path-restricted wrappers; `require` prefixed with the script folder; `include(file)` loads once.
- **Double-buffered task lists** (`new_callbacks_` merged at dispatch start; deletion two-phase: flag, then erase) so a callback that registers another callback never invalidates iteration; `endon(entity,event)` emulates GSC endon; timers run off the game clock so pause/slow-mo are respected.
- Conversions: Lua ints/bools/doubles <-> script ints/floats; strings starting `&` become localized ISTRINGs; Lua 1-based indexes decrement for VM arrays; `nil` assignment erases; every builtin is auto-bound as `entity:name(...)` / `game:name(...)`.
- **Lua closures as GSC callbacks:** `convert_function` registers the closure as a Lua hook under a synthetic code position (`notifies::get_hook_count()+1` cast to `char*`, `is_variable = true`) and passes that as a `SCRIPT_FUNCTION`; the dispatch hook recognizes the fake position and runs the closure, then jumps to a two-byte `{CHECK_CLEAR_PARAMS, END}` stub. Reusable trick to give an engine VM that only understands code positions a host callback.
- Cost: each context calls `collect_garbage()` once per frame and on destruction.
- **C++ <-> GSC <-> Lua IPC reference (killfeed):** Lua calls `game:scriptcall("scripts/killfeed_ducking", "trigger")` and validates the return equals `DUCK_GROUP_COUNT = 50`, permanently disabling ducking if the contract drifts. GSC owns time-bound state: `trigger()` sets `level.killfeed_duck_until = gettime() + 150`, arms `recover_duck()` **before** submitting ("a partial activation must also expire"), re-trigger only extends the deadline ("never compound activations"), `clearsubmix(name, 0.18)` on release, `init()` retires leftovers on a new VM.
- Kill detection without a crosshair: queue a pending hit on damage; resolve as a kill if the `death` notify arrives or the entity is no longer alive after the next frame (timeout 0); absence from the world alone is not proof of death.
- Hot-reload safety: scripts must tolerate being re-run on every VM start; GSC owns cleanup if the Lua context reloads.

---

## 8. LUI / Havok Script (the menu and HUD VM)

Overlord: `game/ui_scripting/*` (typed Hks wrappers), `component/ui_scripting.cpp`, `data/ui_scripts/vr_gameplay/__init__.lua`, `vr/native_menu.cpp`.

### 8.1 Driving the VM from C++
- `lua_State*` comes from a global. Push = write `state->m_apistack.top++`; **always save and restore `top`** around temporary pushes. Call = push function and args, `vm_call_internal(state, nargs, -1, 0)`, read results below `top`, restore `top`.
- Keep Lua objects alive from C++ with the registry (`hksi_luaL_ref(state, -10000)` / `luaL_unref`) in copyable RAII wrappers.
- Hks numbers are 32-bit floats (`TNUMBER`), so an int check is `(int)n == n`; 64-bit ints are a separate type. Do not read the float union before checking the type tag.
- **Event injection:** take `LUI_EnterCriticalSection()`, call `Engine.GetLuiRoot():processEvent({name=..., dispatchChildren=true, ...args})`, leave the section (scope guard). Use this to drive native menus with VR-pointer events instead of faking mouse input.
- Reset on VM shutdown: `hks_shutdown_stub` calls `vr::native_menu::reset_vm()`, clears registered C++ functions and globals, so VR state never holds stale VM objects across `vid_restart`/mod loads. UI scripts must tolerate re-running.
- Module resolution: each folder with `__init__.lua` is a module; `require("name")` resolves to `<folder>/name.lua` by hooking the asset lookup and load calls; scripts run via `loadstring` + `pcall`; overlay precedence low->high (base, then mod, then localized), the **last winner per name is chosen before any Lua executes** and a failed read of the winner is an error, never a fallback to a stale cached copy.
- Sandbox: stub `io`, `os`, `serialize`, `hks`, `debug`, `coroutine`; offer safe wrappers.

### 8.2 C++ -> Lua read-only snapshot bridge
`vr_weapon_hud` table: `active()`, `count()`, `source(i)` (returns a table `{valid, weapon, loaded, reserve, capacity, maximumReserve, clipType, lowThreshold, name, definition, feed}`) and `commit(i, visible)` (acknowledges that a source was actually rendered, so the capture knows what to composite). The server pipeline samples ammo into `source_snapshot` (SEH-guarded read of definition fields); LUI only reads copies. **Pattern:** expose VR state to the game's own UI with a read-only snapshot + acknowledgement callback; never draw the HUD yourself if the engine already can.
- Gate by dvar: the script returns immediately unless `vr_weaponHudNativeSource` is set and `vr_enable` is true, so it can ship in the normal data folder.

### 8.3 Wrapping definitions without shipping game code
- File header states the rule: "Original gameplay adapter; no extracted game scripts or decompiler dependency". Adapt by wrapping the registry, never by copying derived scripts.
- Wrap `LUI.MenuBuilder.registerDef` and, **if the definition is already registered, patch `definitions.WeaponInfoHudDef` in place** ("Some HKS sessions register SP HUD definitions only when loading a level"). Keep wrappers idempotent with flags (`_vrNativeAmmoAdapter`, `_vrAmmoPipAdapter`), `pcall` every build and latch a `failed` flag so a bad native build cannot loop.
- **Private widget copies:** build one copy of the game's own HUD widget per hand through its factory (`builder.buildItems(original(), {}, slot)`); run its init/refresh with the global `Game` table temporarily swapped for a proxy (`setmetatable({...}, {__index = nativeGame})`, `rawset` then restore under pcall) so each copy renders that hand's weapon from the C++ snapshot. Remove the copy's own refresh timer, hide the original under a mod-owned zero-alpha container (native `animateToState` rewrites alpha on the same element), and force the ammo widget "on" by wrapping `animateToState` ("off" -> "on").
- Temporary monkey-patch idiom: replace the shared function only during the native builder call, `pcall`, restore, then `error(menu, 0)` on failure ("a failed build must not leave the wrapper stacked on the shared function").
- Off-by-one native data (closed-bolt chamber reports clip+1): an `__index` fallback on the pip table returns the last pip only for index capacity+1 and only when the VR chamber dvars are on; other nil indices still surface real bugs.

### 8.4 Resource-pool and lifetime pitfalls (all found the hard way)
| Pitfall | Fix |
|---|---|
| The engine's UI **animation pool is finite**; the pause-menu constructor allocates from it before the next HUD refresh (and the refresh timer may stop while paused) | wrap the pause menu type factory (`registerType` and `m_types_build.sp_pause_menu`): release VR copies first, run the original under a `pauseConstruction` counter |
| `close()` only detaches; a running transition keeps the discarded tree alive | `stopTree`: remove `transition_complete*` handlers first, then `cancelAnimateToState()` on every descendant before the element leaves the tree |
| Previous level's HUD retained by owner tables | weak-key tables (`__mode="k"`) for owners |
| GC thrash | `collectgarbage` only on ownership transitions, never on stable refresh; drop references before collecting |
| Native pip removal during refresh | guard `removeElement` to stop the child's animations first |
Test strategy: run the **shipped** Lua file against a fake LUI host (`tests/vr/weapon_hud_lui_tests.cpp`); test module discovery separately (`tests/ui_script_modules_tests.cpp`).

### 8.5 Menu-extension idioms (inherited, useful for an in-game VR settings/recenter entry)
`LUI.MenuBuilder.registerType/registerPopupType`; override a native menu by assigning `LUI.MenuBuilder.m_types_build[name]`; `LUI.addmenubutton("main_campaign", {...})`; `LUI.onmenuopen("sp_pause_menu", fn)`; reorder pause buttons with `menu.list:removeElement/insertElement`; `Engine.InFrontend()` gates frontend-only scripts; launcher preferences (campaign unlock, subtitles) are applied once per process inside the menu VM with a static `applied` flag so a VM reload cannot overwrite a later in-game choice.

---

## 9. Localization and prompt text

Two independent layers; do not conflate them.

### 9.1 The game's own strings (data side)
- Zone localization files are flat `{"KEY": "text"}` JSON (`data/zonetool/localizedstrings/<language>.json`), compiled by `localize,<file>` rows into the language zone. Tokens inside native strings: `&&1 &&2` positional args, `^1..^7` colour codes, `[{+attack}]`/`[{+activate}]`/`[{+actionslot 2}]` bind placeholders resolved to the player's key/glyph at runtime, private-use glyph characters (`\u0001...`) for gamepad button icons, `[{FAKE_INTRO_TIME:...}]` macros. `@KEY` in Lua text fields means "localize this key". Identity-placeholder rows (`"KEY": "KEY"`) make a missing key show its name.
- **Escape arbitrary strings** (mod names, descriptions) before passing as `&&1` args (`string.el` helper) or `&&`/`@` corrupts the format.
- Other languages are *subsets* of English; whether a missing key falls back to English at runtime was not established (pending).
- A **script update cannot rely on new localized keys** unless the matching zone ships; use literals in scripts or ship the zone (`credits.gsc`).
- Adding a missing subtitle needs **both** the localized string and the sound-alias JSON that references the key, per language zone.

### 9.2 VR prompt replacement (code side): hook the lookup, not the table
Do not edit the game's string tables to re-word "Press [F] to use". Replace at the lookup:
- `Engine.Localize` is wrapped in the UI VM (only single-string calls): chain `fixed_menu_text` -> alias -> description rewriting -> `hud_prompts::replace(source::lui, key)`; fall through to the native function otherwise (`ui_scripting.cpp`). The world cursor-hint path wraps the original ownerdraw call so an incomplete prompt family falls back to the stock text (H2 `0x14038AF9C -> 0x1403869F0`).
- Registry keyed by **native source key identity** (config-string field, LUI key), never by matching displayed prose; scoped by producer + map + feature state (`vr_enable`, feature availability); unknown keys return immediately; **no cached translated text** (language change applies on the next draw); formatter caps 4096 bytes, 8 args, 16 runs; `<em>` markup becomes emphasis colour escapes; native hint lifetime/fade/timing stays native.
- **Exact-locale rule:** if the exact current locale lacks a translation the whole override declines (native text shown); no cross-language fallback and no mixed fragments.

### 9.3 Compile-time catalogs (`component/game_text/*`)
- `enum class key` (74 semantic keys) and `locale` (16 values: the 14 written locales of the 17 official packs, plus Czech and Turkish; `es-419` aliases `es`). `catalog = std::array<std::u8string_view, key_count>`.
- Locale files use **named entries** (`{key::x, u8"..."}`), order irrelevant; `make_catalog` is `constexpr` and throws on duplicate/invalid keys, so a mistake fails the build.
- **Argument schema is language-independent** (`parameters(key)` bitmask: `{button}`=1, `{item}`=2). `template_parameters()` is a constexpr parser that rejects unknown placeholders, stray `<`, `>`, `}`, nested or unbalanced `<em>` and patterns > 4096 bytes. A translation with incompatible parameters fails the build; word order may change.
- **Missing translation = omit the entry** (Czech/Turkish carry 4 keys). Never copy English into another locale; that marks it present and suppresses the native fallthrough.
- Markup contract: `<em>...</em>` marks literal controls/locations; a shared formatter turns it into emphasis runs; locale files never embed renderer colour codes; `{button}` takes emphasis from its typed argument. UTF-8 sources, `u8` literals, MSVC `/utf-8` on every configuration including PCHs and tests (independent of the Win32 A/W charset). Language follows the game's `loc_language` dvar. Catalogs are embedded: no runtime file loading.
- Per-controller-family keys (`button_menu_recenter` vs `knuckles_button_menu_recenter`) because Index layouts lack X/Y.
- Each catalog is validated in its own constexpr evaluation to stay inside the compiler step budget.
- Known quality caveats: `zh-TW` contains simplified characters in places, `it` has a gender error; "reviewed" is not guaranteed. The key list doubles as an inventory of the interaction adapters a campaign port ends up needing (pickup/use/resupply holds, rappel, C4, night vision, breach charge, notebook/UAV, fixed sniper, heartbeat sensor, vehicles, trainer prompts, cinematic skip).

### 9.4 Fonts for non-Latin text
Any VR text surface must have a CJK/Arabic/Cyrillic strategy or it draws boxes. Overlord reads the game's own TTF bytes (`fonts/bank.ttf`, asset DB), registers a **private GDI face** with `AddFontMemResourceEx` (4 MiB cap), holds it under a lease and drops it in the fastfile pre-unload callback (asset memory dies on unload); a GDI caption rasteriser (outline, tint, RTL, strict UTF-8) renders to an immutable D3D11 texture. Extra fonts ship as their own zones (`h2_mod_font_*`, `ara_h2_mod_font_default_bold`) loaded from a font-zone table. The LUI helper `language.isnonlatin/isasian/isarabic` changes text offsets/fonts for these locales.

---

## 10. Custom asset packaging (ZoneTool CSV/JSON -> `.ff`)

Overlord ships its own sounds, melee models/animations, UI materials, fonts and localized strings as **fastfile zones** that the engine loads like stock content.

### 10.1 Source layout
| Path | Role |
|---|---|
| `data/zone_source/<zone>.csv` | one `type,name` row per asset: `material,x`, `sound,alias`, `localize,english`, `stringtable,video/subtitles.csv`, `xmodel,`, `xanim,`, `ttf,fonts/bank.ttf`, `techset,,2d`; `//` comments allowed |
| `data/zone_source/build.txt` | list of 36 zones to build |
| `data/zonetool/<zone>/...` | JSON dumps (`materials/*.json`, `sounds/*.json`) plus binary inputs (`images/*.dds`, `loaded_sound/*.flac`) |
| `data/zone/*.ff` | the only **tracked compiled** outputs (`h2_vr_sounds.ff`, `h2_vr_melee.ff`) |
Zone naming: `h2_mod_*` shared zones (ui, common, pre_gfx, fonts, patches); per-language `xxx_h2_mod_common` with 3-letter codes (ara cze deu eng ens fra ita jpf jpp kor pol por rup rus sch sna spa tch tur); per-language voice patches (`eng_h2_mod_patch_af_caves`, `rus_h2_mod_patch_dc_whitehouse`). Localized voice assets must be built **per language zone**.
- 2D UI material JSON is tiny: `techniqueSet->name: "2d"`, `sortKey 60`, `cameraRegion 12`, one texture entry `{image, semantic 0, samplerState 226, typeHash}`. A mod can re-expose an existing game image under a new material name by shipping only the JSON.
- Sound alias JSON: `aliasName`, `subtitle` key, `soundfile{type 1 = loaded, type 2 = streamed}`, `flags`, `dspBus`, `volMod`, `priority`, `speakerMap` (`front_only` UI, `radio` VO).

### 10.2 Load order (`fastfiles.cpp`)
The loader hooks the engine's zone-list builders and **splices extra zones into the list the engine hands to `DB_LoadXAssets`**: `h2_mod_pre_gfx` before the graphics zones; `h2_mod_common`, then the original zones with `h2_mod_ui` inserted right after `code_post_gfx`; `h2_vr_sounds` and `h2_vr_melee` after common; mod zones by priority. `try_add_zone` adds a zone only if the `.ff` exists (a missing optional zone is skipped, not fatal), tags it `DB_ZONE_COMMON | DB_ZONE_CUSTOM`, and for localized zones first adds `<language-code>_<name>`. The `h2_vr_*` zones are not localized. Unload goes through `fastfiles::on_pre_unload` callbacks: every cache holding asset pointers (models, fonts, partitions, fx, identity registries) retires there.

### 10.3 Build recipe (offline, never inside the playable install)
```
tools/build_sound_zone.py --h2-root <game> --zone-tool <ZoneTool exe>      # -> data/zone/h2_vr_sounds.ff
tools/build_melee_zone.py                                                   # -> data/zone/h2_vr_melee.ff
```
1. **Throwaway junctioned game tree.** Create a staging dir under the workspace; copy only the executable, `.build.info`, `.product.db` and DLLs; for each other game directory create a **junction** (Windows `CreateJunction`) or symlink (other hosts) except an exclude list (`dump, minidumps, mods, players2, trainer, zonetool, zone, zone_source, h2-mod`); copy the zone CSV into `zone_source/` and the JSON/binary assets into `zonetool/<zone>/`. Cleanup removes **only the links** (`os.rmdir` on a junction never touches its target) after asserting the staging path is workspace-owned. The real installation is never mutated.
2. **Preflight asset checks before the tool runs:** alias JSON `aliasName` matches, `count == len(head)`; each FLAC resolves inside the asset root; FLAC header must be `fLaC` with **min/max block size both 1024** (`04 00 04 00` at bytes 8..12; H2 rejects other block layouts with "Stream must have a constant blocksize"); melee images must not reference source CASC streams.
3. Run ZoneTool as `-unattended -unbuffered-io -buildzone <zone>`, then `-verifyzone`, then `-dumpzone`. Some linker builds keep their UI loop alive after finishing: on timeout accept the run only if the output contains `Successfully compiled fastfile "<zone>"`. Fail on `[ ERROR ]` in the log.
4. **Read back** the dump: every alias exists with the same variant names/count, every recording is present and non-empty, model identity and root/bone counts match, animation identity matches. Compile success alone hides missing variants.
5. Copy the `.ff` to `data/zone/`.
- **Alias flag trap:** H2 uses bit `0x20` for *random recording start offset* (the inherited struct calls it `is3d`) and `0x01` for looping. One-shot punch/impact aliases (`vr_fist_hit`, baton sounds) must keep `flags & 0x21 == 0` or the attack is cut. Enforced in both preflight and read-back.
- **Policy:** compile once, ship the compiled `.ff`; raw ZoneTool exports, source FLACs and extracted assets stay out of Git (`.local/...`, git-ignored); the ordinary client build never compiles zones or needs ZoneTool; the packager admits only zone names present in `data/zone_source`, treats optional-language zones as optional, validates `binding_url`/application paths, and never copies original game files, saves or PDBs (`tools/package_client.py`). Packaged tree: `data/cdata` (`.lua .gsc .cfg .json .csv .flac`) -> `h2-mod/`, the VR HUD Lua -> `h2-mod/ui_scripts/vr_gameplay`, the two `.ff`, licenses.
- **Legal:** asset provenance must be reviewed (OFL fonts need their full licence text; voice FLACs; models ported from another title are owner-specific and legally grey, do not recommend for a new port). Never ship extracted game assets; asset-discovery tools stay outside the mod (`vr-weapon-asset-workflow.md`: candidate -> desktop_verified -> hmd_pending -> hmd_verified).

### 10.4 FX names in the save-game dictionary
World-space or followed FX that the mod rebuilds as private immutable descriptors must survive save/load. Overlord gives each a bounded presentation name added to the native **name/old-pointer dictionary** before the save worker streams it: `__overlord_fx/world/<source>`, `__overlord_fx/follow/<8-hex mask>/<source>`, `__overlord_fx/casing/<8-hex mask>/<source>`; source <= 255 bytes, no NUL, may not start with the prefix; capacity 520 variants (512 graph nodes + 8 followed) against 2048 native effect slots; ids are one-based and **append rather than renumber** (checkpoint compatibility). Restore reconstructs from current assets; older checkpoints with unmapped private pointers go through recovery only if the effect is empty/unbound/single-reference, otherwise reject via the native recoverable error. Lesson for any engine: **anything the engine serializes by pointer or name must exist in the dictionary the saver walks**, and a mod-created object that is not in the dictionary makes saves crash on checkpoint/Resume. Likewise **enlarged caches embed layout-derived values in saves** (lighting handle includes the static partition offset, so the reuse site gets a range guard); see 05.

---

## 11. Dev tooling that accelerates VM work

- **In-process ImGui overlay** (`component/gui/*`): script console (queues commands onto the server pipeline), entity browser, per-asset viewers/live editors (techset, material, xmodel, weapon, fx, rawfile, scriptfile, stringtable, shader, vertexdecl), world-space debug draw, `spawn_xmodel`. Input capture is ownership-based: borrow the engine's UI key-catcher bit on open, clear it on close only if added, byte-verified detours on the engine's catcher setters clear the "added" flag when the engine writes it itself ("observe native intent; never rewrite it"); `gui::captures_input()` makes the VR pointer/injection stand down. UI never mutates game state directly: closures run on the server pipeline.
- Asset viewers re-resolve each frame by asset *name* via the asset DB (never cache pointers across frames; zones unload); `zoneIndex == 0` means the default placeholder asset (missing).
- `developer_script` dvar -> compile with devmap -> `vm_error_stub` prints opcode name, builtin name and a callstack with file/line/column.
- Console: `getfunctionptr <name>`, `dumpMapEnts` (decodes GSC token ids), `vr_*_status` per adapter (counters + last decision string, written to `minidumps/overlord-*.txt`).
- Offline: `tools/generate_scene_surface_contract.py` (Capstone census) for native patches; `read_crash_evidence.py` for dumps.

---

## 12. Porting checklist for an engine with an embedded VM

1. Find the VM entry points: call-builtin dispatch, opcode loop, error path (longjmp/exception), value stack, string interning, object pool. Write down which thread runs them.
2. Restore readable errors. Add a call bridge with reverse-order arg push, stack isolation, guarded calls and refcount-safe wrappers. Unit-test the pure parts.
3. Enlarge the builtin table (or find a free-id range), add `vr*` builtins that throw on bad input. Add a bounded interceptor registry.
4. Add the opcode-dispatch redirect with a hit table and guards. Write bytecode inspectors with masks, operand cross-checks, uniqueness and shape counts; test them against captured bytecode of the pinned build.
5. Decide per scene: redirect the minimum boundary; embed a tiny virtual script; keep story flags script-owned; store plugin state in engine save state; declare saved fields.
6. Hook the VM lifecycle: after spawn **and** after saved-level load; before/after shutdown; free scripts; restore bytecode edits.
7. UI VM: find the lua_State, definition registry and localize function; wrap, never copy; gate by dvar; mind the animation pool and GC.
8. Prompts: key-based registry, exact-locale fallthrough, compile-time catalogs, controller-family keys, a font that can draw your locales.
9. Assets: CSV manifest + JSON dumps + ZoneTool, throwaway junctioned tree, read-back verification, FLAC/alias traps, compile-once-ship-`.ff`, save-dictionary names for mod-created objects.
10. Test: pure `classify(evidence)` for scenes, shipped Lua against a mock host, GSC compile smoke (`tests/vr/killfeed_audio_tests.cpp`), inspector tests with captured byte spans, fuzz the validators (oversized names, NaN, truncated streams).

## 13. Quick reference: Overlord files

| Topic | Files |
|---|---|
| VM bridge | `src/client/game/scripting/{execution,safe_execution,stack_isolation,script_value,entity,array,vector,variable_value,function,functions,animation}.*` |
| Builtins/interceptors | `component/gsc/script_extension.*` |
| Runtime compile, virtual sources, main/init | `component/gsc/script_loading.*` |
| Diagnostics | `component/gsc/script_error.*` |
| Opcode redirect, damage chain | `component/notifies.*` |
| Level lifecycle, notify alias | `component/scripting.*` |
| Function extent, signatures | `vr/gameplay/script_function_extent.hpp`, `signal_flare_script.hpp`, `campaign/sequences/airport_opening.hpp`, `vehicles/script_contract.hpp` |
| Embedded scripts | `campaign/cliffhanger/physical_script.hpp`, `campaign/ending/script.hpp`, `exo_punch_script.hpp`, `official_cheats_script.hpp` |
| Mod Lua | `game/scripting/lua/*`, `data/cdata/scripts/killfeed/__init__.lua`, `data/cdata/scripts/killfeed_ducking.gsc` |
| LUI | `game/ui_scripting/*`, `component/ui_scripting.cpp`, `data/ui_scripts/vr_gameplay/__init__.lua` |
| Localization | `component/game_text/*`, `component/game_text*.hpp`, `vr/hud_prompts.*`, `docs/game-text-i18n.md` |
| Asset pipeline | `data/zone_source/*`, `data/zonetool/*`, `tools/build_sound_zone.py`, `tools/build_melee_zone.py`, `tools/package_client.py`, `component/fastfiles.cpp` |
| FX save names | `vr/gameplay/native_fx_checkpoint*.{hpp,cpp}` |
