---
kind: game
title: 'The Wolf Among Us: Garry''s Mod sandbox (Telltale Lua 5.1, ttarch2, PresentEx behind the Steam overlay, per-bone ragdolls)'
game: The Wolf Among Us
games_also: ["Garry's Mod"]
game_version: 'Steam app 250320, TheWolfAmongUs.exe 9,297,080 B, PE timestamp 1384476156 (2013-11-15), x86'
platform: windows
engine: native
route: native-hook
tools:
- Visual Studio 2022 + CMake (-A Win32)
- MinHook 1.3.3 (sources built directly; that tag has no CMakeLists)
- Dear ImGui 1.91.9b
- zlib 1.3.1
- Ghidra (analysis only) and a hardware write watch inside the mod (hwbp console command)
anti_cheat: 'none (single-player). The SteamStub .bind section is left untouched: the mod is a dinput8.dll proxy and hooks only after the game made its D3D9 device.'
status: working
agents:
- Claude Code (Opus 5.5)
humans: ["Daniel560928"]
date: '2026-10-21'
links:
- examples/twau-gmod-sandbox
- https://github.com/TsudaKageyu/minhook
- https://github.com/ocornut/imgui
tags: [telltale, telltale-tool, lua-5.1, float-lua, opcode-permutation, ttarch2, d3d9ex, presentex, steam-overlay,
  minhook, seh-guard, vectored-exception-handler, skeleton, bones, ragdoll, verlet, physgun, garrys-mod, vpk,
  freecam, sandbox, walk-boxes]
---
# The Wolf Among Us: Garry's Mod sandbox (Telltale Lua 5.1, ttarch2, PresentEx behind the Steam overlay, per-bone ragdolls)

> An x86 ASI, loaded by a `dinput8.dll` proxy, turns *The Wolf Among Us* (Telltale Tool, 2013) into a Garry's
> Mod-style sandbox:
> - **F1** freezes the story scene in place and **F3** resumes it.
> - In between: first person on the scene's walk boxes, noclip, HL2 guns, physgun, toolgun, a Q spawn menu,
>   Face & Finger Poser, and a map menu over all five episodes.
> - Corpses are per-bone verlet ragdolls written straight into the game's skeleton.
> - Everything from Garry's Mod and TWAU is read from the player's own installs at runtime.
>
> All of it ran in the real game over 31+ logged runs, checked with screenshots and the mod's log. Source and
> the full journal: [`examples/twau-gmod-sandbox`](../../../examples/twau-gmod-sandbox/) (`MODLOG.md`).

## Setup
- **Game:** Steam build of *The Wolf Among Us* (app 250320). The exe is 32-bit with **ASLR on**
  (DllCharacteristics 0x8140), so every address below is an RVA from the 0x400000 image base. It imports
  `d3d9`, `d3dx9_41`, `DINPUT8`, `fmodex` and `steam_api`.
- **DRM:** the exe has a SteamStub `.bind` section, but `.text` is plain on disk, so static RVAs are valid. The
  stub is never touched or removed.
- **`um scan`:** "Unknown native engine", no anti-cheat, no loaders.
- **Garry's Mod install (for its assets):** `garrysmod_dir.vpk`, `hl2_misc_dir.vpk` and `hl2_sound_misc_dir.vpk`,
  plus `HALFLIFE2.ttf`.
- **Build:** VS 2022 + CMake `-A Win32`. ImGui, MinHook and zlib are fetched at configure time.
- **Install:** `dinput8.dll` + `TWAU_GMod.asi` + `gmod\` go in the game folder. Back up saves with `um backup`
  before the first modded launch.

## Route and why
- **Native hooks plus the game's own Lua.** The ASI hooks `lua_pcall` to get the game's `lua_State` on the game
  thread, loads `gmod_sandbox.lua` with the game's own `luaL_loadbuffer`, and drives the engine only through
  Telltale's Lua API (`AgentSetWorldPos`, `ControllerSetTimeScale`, `CameraPush`, ...). The C++ side needs
  almost no engine struct offsets.
- **Native code for the rest.** Rendering (ImGui HUD, Source viewmodels) hooks D3D9. The ragdoll is the one
  place that touches engine memory (the skeleton, below).
- **Why not stock Lua or a loose-file mod:**
  - The game's Lua uses **float numbers**, so linking stock Lua against its `lua_State` would corrupt it.
  - The loose-file route (`_resourcedescriptions_`, resource sets in Lua) exists, but the native bridge doesn't
    need the script encryption key.
- **Physics is our own.** The exe has no physics engine (no PhysX, Havok or Bullet): the `Physics*` Lua
  functions only move agents. Ragdolls and rigid bodies are our own verlet simulation.

## How the game works (what we had to learn)

### Lua 5.1.4, statically linked, 32-bit float numbers
- **Spotting it:** `math.abs` stores with `fstp dword`, so `lua_Number` is a float.
- **Finding the C API:** the stdlib `luaL_Reg` tables are in `.rdata` (base, string, math). Disassembling
  `print`, `pcall`, `type`, `rawget`, `tonumber`, `assert`, `loadstring` and `luaopen_base` gives the whole C
  API. Key VAs:

  | function | VA |
  |---|---|
  | `lua_gettop` / `lua_settop` | 0x8415B0 / 0x8415D0 |
  | `lua_tonumber` (float in st0) | 0x841990 |
  | `lua_pushnumber` (float arg) | 0x841C30 |
  | `lua_getfield` / `lua_setfield` | 0x841ED0 / 0x8420F0 |
  | `lua_call` / `lua_pcall` (prologue `55 8B EC 8B 4D 14`, hookable) | 0x842370 / 0x8423D0 |
  | `lua_load` | 0x8424F0 |
  | `luaL_loadbuffer` | 0x8430B0 |
  | `luaL_register` | 0x8439D0 |

  `MODLOG.md` has the full list (about 30 functions) and how each one was identified. `LUA_GLOBALSINDEX` is
  -10002.
- **Engine functions:** Telltale registers them in code, so there are no `luaL_Reg` tables for them, but every
  name is in `.rdata`.
- **Several VMs:** the game creates **new Lua VMs**, three at startup and one per load. We anchor our table in
  the registry. A tick in a VM without it reloads our script, and a tick in another globals table puts it back.
- **Scripts and opcodes:** the game scripts are encrypted `.lenc`. Hooking `lua_load` (RVA 0x4424F0) sees them
  decrypted, as Lua 5.1 bytecode with float constants. Keep such dumps out of any repo. Telltale **permutes the
  opcodes**. Aligning the game's `string.dump(loadstring(s))` with stock 5.1's for the same source gives
  (0 operand mismatches over 85 instructions):
  MOVE 1, LOADK 2, LOADBOOL 3, LOADNIL 4, GETUPVAL 5, GETGLOBAL 6, GETTABLE 7, SETGLOBAL 9, SETUPVAL 10,
  SETTABLE 11, NEWTABLE 12, SELF 14, ADD 16, SUB 17, MUL 18, DIV 19, MOD 20, POW 21, UNM 22, NOT 23, LEN 24,
  CONCAT 25, JMP 27, EQ 28, LT 29, LE 30, TEST 31, TESTSET 32, CALL 33, TAILCALL 34, RETURN 36, FORLOOP 37,
  FORPREP 38, TFORLOOP 39, SETLIST 40, CLOSE 42, CLOSURE 43, VARARG 44.

### Telltale Lua conventions (verified in game)
- **Space:** Y is up, an agent's forward is **+Z**, and units are metres (Bigby's feet at y = 0.012). The math
  is right-handed, and positive pitch looks down.
- **Screen-right is `-cross(up, forward)`.** Don't calibrate it with `CameraGetScreenPosFromWorldPos`: that call
  reads last frame's camera.
- **Rotations:**
  - Quaternions are Lua tables `{x,y,z,w}`: `AgentGetWorldRotQuat` returns one and `AgentSetWorldRotFromQuat`
    takes one.
  - The global `Quaternion()` is not a constructor (it returns a number).
  - `AgentSetWorldRot(agent, Vector(pitch, yaw, roll))` takes degrees.
- **Scenes:**
  - `SceneGetActiveSceneNames()` lists each scene once per layer (8x). `(true)` adds `logic.scene` and the `ui_*`
    scenes.
  - **Pause the way the game does:** `SceneSetTimeScale(SceneFind(name), 0)`. `logic.scene` keeps running, and
    `ControllerSetScene(c, "logic.scene")` puts our NPC clips on its clock, so spawns animate in a frozen story.
- **Our camera:** `AgentCreate(name, prop, pos, nil, scene, false, false)`, then `CameraPush(agent)`.
- **Walk boxes:** `WalkBoxesPosOnWalkBoxes(pos, 0, PathAgentGetWalkBoxes(agent))` gives the nearest walkable
  point. The agent named after the scene carries the `Walk Boxes` property, and every path agent shares it.
- **No level raycast:** no engine call reaches level geometry, and `AgentAtScreenPos` only picks selectable
  agents. The walk boxes stand in for walls: more than 0.3 m off them counts as a wall. Bullets, rockets,
  blasts, the toolgun and NPC steps all use that test.
- **Lights:**
  - A fresh `module_light.prop` agent lights nothing, because TWAU lights only reach their light groups.
  - `AgentDuplicate` of the scene's own `light_CHAR_*` works, so the flashlight is that copy.
  - `"Light Type"` is an enum set by name, and the spot cone is a fraction.
- **Episodes:** each episode is a resource set (`Fables101` ... `Fables105`). Enable the target set and its
  sub-sets (`SubProject_GetResourceSets`) and raise its priority. **Never disable the running episode's set**:
  that crashes the game.

### `.ttarch2` archives (all 48 in `Pack\` parsed)
- **Outer wrapper**, one of two:
  - `NCTT`: magic, a u64 size, then the raw stream.
  - `ZCTT`: magic, a u32 chunk size (0x10000), a u32 chunk count N, then N+1 absolute u64 offsets. Each chunk is
    **raw deflate** (zlib wbits -15).
  - There is no `ECTT` (encrypted) wrapper in this game.
- **Inner `3ATT` stream:**
  - Header: version (2), name-table size, file count.
  - Then 28-byte entries: {u64 crc64, u64 offset, u32 size, u32 unknown, u16 name block, u16 name offset}.
  - A name sits at `block * 0x10000 + offset` in the name table.
  - Data offsets are relative to the end of the name table.
- **Contents:** 156,789 entries (94,941 unique names), among them 64,315 `.anm`, 22,825 `.chore`, 6,703
  `.d3dmesh`, 750 `.skl` and 284 `.scene`.
- **Skeleton families by prefix:** `sk54_` men, `sk55_` women, `sk15_` short characters, `sk18_colin` (the pig),
  `sk35_wolfBigby`, `sk37_crowCrooked` and `sk39_bufkin`. A clip from a sibling on the same family drives any
  of them.
- **Faces:**
  - `<face>_phoneme_default_<emotion>A.anm` is the rest face for an emotion.
  - `<face>_face_*` clips are the channels.
  - The face prefix is the skeleton name minus `skNN_`.

### D3D9Ex, and the Steam overlay in front
- **The device:** the game presents with **`PresentEx`** (vtable slot 121, `call [vtable+0x1E4]` at exe+0x1952FE)
  on an `IDirect3DDevice9Ex` from `Direct3DCreate9Ex`.
- **The overlay:** `gameoverlayrenderer.dll`, loaded right after our proxy, patches `Direct3DCreate9`,
  `Direct3DCreate9Ex` and `PresentEx`. Its trampolines jump back into d3d9 past the entry point, so a hook placed
  **before** the overlay's is never reached.
- **Working order:**
  1. On the game thread, about 40 ticks after start, make a throwaway **NULLREF** device and use it to hook
     d3d9's `EndScene`.
  2. The game's next `EndScene` hands over its real device.
  3. Hook that device's `Present`/`PresentEx`/`Reset`/`ResetEx`.

  Hooking last puts us in front of the overlay.
- **No managed pool:** D3D9Ex has no `D3DPOOL_MANAGED`. Use the default pool for static buffers, and fill
  textures in system memory, then `UpdateTexture`.

### Skeleton and bones (for the ragdoll)
- **Finding instances:** RTTI names `SkeletonInstance` (vtable RVA 0x5f95a4). Scanning private RW memory for
  that vtable finds every instance, and a spawned Faith adds one with 110 bones.
- **`SkeletonInstance`:**
  - +0x0c: the agent's node
  - +0x14: skinning matrices (4x4, 64 B each)
  - +0x1c: bone count
  - +0x24: bone nodes
  - +0x4c: the frame it was last updated
- **Bone node** (vtable RVA 0x5fb3e8, stride 0xd8):
  - +0x10: name hash
  - +0x18: flags (bit 0: the global transform is valid)
  - +0x1c / +0x2c: local quaternion / position
  - +0x3c / +0x4c: global quaternion / position
  - +0x64 / +0x68 / +0x70: parent / first child / next sibling
  - global = parent global × local (Hamilton product)
- **Writers,** found with a hardware write watch on every thread:
  - RVA 0x1e9520 (thiscall, 4 stack args) applies the animation mixer's result to each bone's local transform.
  - RVA 0x1eb9f0 (approximate) computes globals lazily.
  - RVA 0x3cdb50 builds the skinning matrix.
  - RVA 0x5d050 invalidates a node and its children.
- **Driving it:** write the local rotation, invalidate the node, and hook 0x1e9520 so ours is re-applied after
  the mixer. Otherwise the idle pose wins.

### The ragdoll (own simulation)
- **Structure:**
  - One verlet particle per bone, stepped at a fixed 120 Hz.
  - Short links (under 10 cm) merge into the parent's rigid piece: the pelvis takes the hips, the chest the
    collarbones, the head the face, the palm the knuckles.
  - Cone limits in the parent's frame: 30° for the spine and neck, 100° elsewhere.
  - Knees and elbows are one-way hinges, built on the farthest frame member, because shins and forearms have
    twist helpers.
- **Collision and twist:**
  - Self-collision uses capsules on bones longer than 8 cm. Pairs that already touched at death are skipped
    (Faith: 24 capsules, 206 pairs).
  - A chain bone is carried by its parent's rotation, then swung onto its child, so limbs don't keep the twist
    they died with.
- **Masses:** bones under 6 cm weigh 50 g, trunk bones `5 + 20·length` kg, limb bones `12·length` kg, and the
  pelvis at least 10 kg. Light twist helpers between heavy bones get 1 kg.
- **Hits and floor:** a hit spreads momentum (as if into 40 kg) over bones within 35 cm, by mass. Floor
  friction is Coulomb, applied once per step. A 12 m/s velocity cap per particle keeps a hard physgun swing
  from throwing a body across the map.
- **Physgun:** grips the bone nearest the aim ray and holds its orientation. The rest of the body keeps
  simulating and hangs off it.

## Build steps
See `examples/twau-gmod-sandbox/README.md`:
1. Configure with CMake `-A Win32` in VS 2022 and build `dinput8.dll` + `TWAU_GMod.asi`.
2. Run `twau_assetscan` against both installs. It should mount 7 VPKs and 48 archives and pass every check.
3. `um backup` the saves.
4. Copy `dinput8.dll`, `TWAU_GMod.asi` and `gmod\` into the game folder.
5. Launch through Steam. Uninstall by deleting those three.

## Verification
- **Offline:**
  - `twau_assetscan` on the real installs: 71 characters, 57 scenes over 5 episodes, all 8 viewmodels load with
    textures, bind-pose self-check error ≤ 2.7e-5.
  - `um publish check` passes, and so do the repo tests.
- **In game, runs 1–31** (MODLOG has each one):
  - F1 froze 583 agents.
  - Calibration: forward +Z, dot 1.000.
  - The player stops at walk-box walls.
  - Spawned NPCs animate in a frozen story.
  - Walls hold: a rocket at an NPC behind a wall explodes at the wall, and a toolgun aimed through a wall does
    nothing.
  - The flashlight raises characters' brightness: screen mean 47 → 66 → 47.
  - The Face Poser changes only the face region (mean pixel difference 5–11 there, 0 elsewhere).
  - **Ragdoll v2:** a rocket hit on Faith logs `110 bones, 397 constraints, 46 cones, 4 hinges, 24 capsules /
    206 pairs (32 ms)`. She ends on her side, nothing passes through the torso, and the knee bends the right
    way.
  - **Maps:** all **41 Episode 2–5 scenes** opened from the console with no crash, and the crash guard fired
    once.
  - F3 returns to the story with no errors in the log.
- **Not verified, or known gaps:**
  - Only the Steam build was tested.
  - `adv_barTripTrapExterior104` stays black even in story view.
  - `adv_openArmsLobby` and `adv_introEnding` are stubs with no level.
  - Some characters spawned into Episode 5 scenes render as black silhouettes.
  - The flashlight lights characters but not walls or floors.
  - The Big Bad Wolf (`sk35`) has no walk clip and glides.

## Gotchas
1. **Hooking `Direct3DCreate9` from `DllMain` saw nothing, and a temporary HAL device crashed the heap
   (`c0000374`).** **Cause:** the game uses `Direct3DCreate9Ex` + `PresentEx`, and the Steam overlay patches
   them after us. Its trampolines skip earlier hooks, and a device made off the game thread races the game's
   own `CreateDevice`. **Fix:** do D3D work only on the game thread: a NULLREF device hooks `EndScene`, and
   that gives you the real device to hook last (see above).
2. **Viewmodels drew `0 parts` with no error.** **Cause:** D3D9Ex has no `D3DPOOL_MANAGED`. **Fix:** use the
   default pool, plus system-memory textures and `UpdateTexture`.
3. **Linking stock Lua against the game's state would corrupt it.** **Cause:** `lua_Number` is float.
   **Fix:** call only the game's own Lua C API, by RVA.
4. **Our script vanished after a load.** **Cause:** the game makes a new Lua VM per load. **Fix:** anchor in
   the registry, and reload the script on a tick in a VM that lacks it.
5. **Most Episode 2–5 scenes crashed at exe+0xefd98 (`[ecx+0x24]` on a null scene).**
   - **Cause:** the game's own scene script calls the native `SceneGetCamera` (RVA 0x2cafb0) with a non-string
     argument before the scene has loaded.
   - **How it was found:** a vectored exception handler that logs registers, the ebp chain as RVAs, and the
     strings on the stack for the first three faults in the exe.
   - **Fix:** wrap that native (MinHook + SEH). A fault returns nil and logs a Lua traceback, and the script
     carries on.
   - Our own call with no argument crashes the same way, so guard arguments before calling natives. `pcall`
     can't catch a native fault.
6. **Switching episodes crashed.** **Cause:** disabling the running episode's resource set. **Fix:** only
   enable the target set and its sub-sets, and raise its priority.
7. **Mouse right turned left, and A/D were swapped.** **Cause:** screen-right was calibrated with
   `CameraGetScreenPosFromWorldPos`, which reads last frame's camera. **Fix:** use the constant
   `-cross(up, forward)`.
8. **Rockets went through walls.** **Cause:** one step out of the muzzle a rocket is already off the walk
   boxes, which the wall test reads as "noclip, no walls". **Fix:** remember that it was fired from the walk
   boxes, and explode on the near side once it would leave them.
9. **Logical helpers (`obj_lookAtBigby`, `obj_counterStation`) got "killed" and caught shots.** **Cause:** they
   are agents without geometry. **Fix:** only agents with a `D3D Mesh List` property are hittable.
10. **The ragdoll grabbed the wrong skeleton** (a 116-bone neighbour). **Cause:** matching by "biggest skeleton
    nearby". **Fix:** match by the root bone's parent, which is the agent's node. The heap scan takes about
    1 s, so run it on a thread at F1 and after each spawn.
11. **The corpse shot 8 m down the corridor.** **Cause:** a one-sided joint-limit push, physics stepped at the
    frame rate (~350 fps), and bone 0 mistaken for a root at the feet (it is the pelvis). **Fix:** symmetric
    corrections, a fixed 120 Hz step, and the pelvis as root.
12. **Bodies were stiff mannequins.** **Cause:**
    - A held corpse wasn't simulated at all.
    - Floor friction ran every solver iteration.
    - Every particle weighed the same: 15 finger bones outweighed the chest.

    **Fix:** the masses, per-step Coulomb friction and physgun grip described above.
13. **The physgun lifted the wrong Faith.** **Cause:** the Woodsman hallway already has the story's own Faith.
    **Fix:** test with a character the scene doesn't have (`spawn snow`). The log now names what the physgun
    holds.
14. **17 characters T-posed when told to stay.** **Cause:** no idle clip; Faith only ships a
    `...StandSmoke_weightShiftA.anm`. **Fix:** also accept `<name>Stand*_weightShift*`, borrow a sibling's idle
    on the same skeleton, and retry a clip that failed to start.
15. **Archive naming traps:**
    - The same scene file differs per episode (`adv_barTripTrap.scene` in 101–103), so a location is (scene,
      episode).
    - Names can contain spaces (`sk55_snowStandA _turn90LeftToWalk.anm`).
    - Story idles are `sk54_idle_<who>StandA.anm`.
16. **The build broke on a variable named `small`.** **Cause:** `small` is a macro from `rpcndr.h` once
    `dinput.h`/`d3d9.h` are included. **Fix:** rename it.
17. **Test driving:**
    - Mouse sensitivity is 0.12°/mickey: lifting needs `rel 0 -400`, not `-40`.
    - Input is polled with `GetAsyncKeyState`, so a tap shorter than a frame is lost.
    - ImGui sees a moved cursor one frame late, so a click right after a move lands on the old tile.
    - Click at the client-area centre, and send the mouse-down in the same call as the focus.
    - A real wheel tick on the game window once selected the camera weapon, so ask before focusing, and never
      while the user types.

## Assets
None generated. GMod viewmodels (MDL/VVD/VTX/VTF), sounds and the HUD font are decoded at runtime from the
user's own Garry's Mod install, and TWAU character, scene and clip lists from its own archives. Nothing from
either game is shipped.

## Cost and time
About two days (2026-10-18 to 2026-10-19), 31+ logged in-game runs.

## Open questions
- Light groups: how to add spawned characters to a scene's lights (black silhouettes in Episode 5), and how to
  light walls and floors from a new light.
- `adv_barTripTrapExterior104`: its lights come from the story, so it is black when opened directly.
- A real level raycast. The walk-box wall test treats low counters and beds as full walls.
- Locomotion clips for the Big Bad Wolf.
- Builds other than Steam (GOG, or the Telltale launcher).
