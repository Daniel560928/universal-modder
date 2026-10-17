---
kind: game
title: 'The Polar Express (2004): RenderWare D3D9 freecam, level switcher, hero teleport'
game: The Polar Express
games_also: []
game_version: 'PC retail, PolarExpress.exe 1,945,600 B, TimeDateStamp 0x416625f3, SizeOfImage 0x21b160'
platform: windows
engine: native
route: native-hook
tools:
- Ultimate ASI Loader 9.7.4 (x86, as dinput8.dll)
- MinHook 1.3.4
- Dear ImGui 1.91.9b-docking
- Ghidra (headless decompile of the whole exe)
- Visual Studio 2019 Build Tools + CMake (Win32)
anti_cheat: none (single-player only)
status: working
agents:
- Claude Code (claude-opus-5-5)
humans: []
date: '2026-10-17'
links:
- https://github.com/ThirteenAG/Ultimate-ASI-Loader
- https://github.com/ocornut/imgui
- https://github.com/TsudaKageyu/minhook
tags:
- freecam
- level-select
- teleport
- noclip
- imgui-overlay
- renderware
- direct3d9
- vertex-shader-skinning
- uac-virtualstore
---
# The Polar Express (2004): RenderWare D3D9 freecam, level switcher, hero teleport

> CinematicExplorer is one x86 ASI for THQ / Blue Tongue's *The Polar Express* (PC, 2004):
>
> - a Dear ImGui overlay with a dockable top navbar, in the style of CinematicUnityExplorer;
> - a 6DOF freecam with FOV, roll, freeze and time scale;
> - a level switcher over every chapter level and bonus game;
> - live hero / camera XYZ with bookmarks;
> - hero teleport and noclip.
>
> It was verified in the running game through a file-driven test bridge inside the mod, which takes
> backbuffer screenshots, and through the mod's own log.

## Setup
- **Install and build.** Retail install in `C:\Program Files (x86)\THQ\The Polar Express`, Windows 11.
  - The exe is 32-bit and imports `d3d9`, `DINPUT8`, `fmod`, `binkw32` and `MSVCR71`.
  - It is **not** TOSHI and **not** D3D8, whatever older posts say. The strings name RenderWare 3.x
    (Criterion), with game code by Blue Tongue.
- **Loader.** Ultimate ASI Loader x86 as `dinput8.dll` loads `*.asi` from the game root.
- **Video mode.** The game runs exclusive fullscreen at 640×480 by default. The overlay must work at that
  size.
- **Dependencies.** The plugin is built with CMake `-A Win32`. MinHook and ImGui come in through
  FetchContent.

## Route and why
- **Chosen route: a native hooks plugin.** It hooks:
  - D3D9 `Present` / `Reset` / `SetTransform` / `GetTransform` / `SetRenderState` /
    `SetVertexShaderConstantF`;
  - DirectInput `GetDeviceState` / `GetDeviceData` and the Win32 key-state functions;
  - one game function, `CoreTick`.
- **Why not data/asset edits:** nothing in the data can give a freecam.
- **Why not a camera-struct hack:** the render-API layer gives a freecam that doesn't care which game
  camera is active. The engine's own functions (StartLevel, its time scale, the hero's RwFrame) were then
  found with a full Ghidra decompile, kept outside the repo, plus runtime memory scans from inside the
  plugin.

## How the game works (what we had to learn)
- **Core singleton.** `*(void**)0x5d8100`.
  - `+0x228` is a float time scale; `CoreTick` passes `dt * scale` to the state stack. Writing 0 freezes
    the world while rendering continues.
  - `+0x238` is the current level object.
  - `+0x30` is the flow object: `+8` is the mode (1 boot, 2 movie, 3 level/front end, 4 level done,
    5 quit) and `+0xC` is the level code `module << 16 | sub`. The front end is mode 3, code 0.
  - `+0x34` is the state stack `{owner, items*, top, cap}`. Normal play has top = 1. A loader or pause
    menu adds a third item.
- **CoreTick.** At 0x4d83c0, `__thiscall(core, float dt)`. It is a safe place to run game calls and to
  apply per-tick writes.
- **StartLevel.** At 0x4f57a0, `__thiscall(flow, module, sub)`. It tears down the current level and builds
  the new one. Calling it from inside CoreTick, in mode 1/3 with stack top ≤ 1, loads any level straight
  from the front end or from inside a level. The level table (module, sub) comes from the chapter-select
  table at 0x5a2088, the bonus-menu calls and the names in `Data/Locale/English.dat`:
  - **Ch1:** (1,0) Scrooge's Plan, (1,1) The Lonely Boy, (1,2) The Lost Tickets.
  - **Ch2:** (2,0) Rooftop Rescue, (2,1) Chasing The Train.
  - **Ch3:** (3,0) Repair the Engine, (3,2) Stoke the Flame.
  - **Ch4** (module 1 again): (1,3) Toys! Toys!, (1,4) Scrooge's Last Stand, (1,5) Hot Chocolate.
  - **Ch5:** (5,0) Runaway Car, (6,0) North Pole Central.
  - **Ch6:** (7,0) Sorting Room, (7,1) Inside the Sack, (7,2) Final Delivery.
  - **Bonus games:** (1,6), (1,7), (1,8), (2,2) Ski Run, (3,3).
- **Cameras per frame (D3D9).** About seven `SetTransform(VIEW)` calls per frame, all from one RW call
  site (0x573FD6):
  1. A looping exterior camera: the scenery outside the train windows, near plane 1.0.
  2. The train-local camera: the car, hero and NPCs, near plane 0.1. This is the one the player looks
     through.
  3. An ortho shadow projector.
  4. The train-local camera again.
  5. 2D HUD cameras at (0,0,150) and (0,0,800).

  The world is left-handed and the view's third column is forward. A freecam that replaces every VIEW, or
  the last one, is wrong.
- **Skinning has two render paths, chosen by GPU caps.**
  - The fixed-function path uses WORLDMATRIX vertex blending, so characters follow a replaced VIEW
    automatically.
  - The vertex-shader path (0x52F167) uploads `(World·View·Proj)ᵀ` to c0–c3 and the bone palette from
    0x52F517. In this path characters ignore the VIEW matrix and need the constants corrected (Gotcha 3).
  - Two machines in this project took different paths.
- **The hero.**
  - Class `"boy"` (vtable 0x5a5098, ctor 0x41a980). The level object stores him at **level+0xA8**.
  - `boy+0x10` is his root `RwFrame`. Its modelling-matrix position (`frame+0x40/44/48`) is
    authoritative: the game's own spawn code writes exactly those three floats, and his physics body
    (`boy+0x80` gravity, `+0xC4` a position copy) re-reads the frame every tick.
  - After writing the frame, call `RwFrameUpdateObjects` (0x56bb80, cdecl, one argument; it marks the
    root dirty) so the hierarchy re-syncs.
- **Coordinates.** Hero and train-local camera share one space: at the start of Scrooge's Plan the hero
  stands at (0, 0.02, 40.5) and the camera is at (0, 2.2, 37.5) looking +z.
- **Input.**
  - Movement uses the arrow keys (not WASD) through DirectInput 8.
  - The main loop only ticks when its millisecond clock advances, and only while the window is focused.

## Build steps
1. Configure: `cmake -S . -B build -G "Visual Studio 16 2019" -A Win32`.
2. Build: `cmake --build build --config Release`. The output is `CinematicExplorer.asi`.
3. Copy the `.asi` and `.ini` into the game root next to Ultimate ASI Loader's `dinput8.dll`. Program
   Files needs an elevated copy (see Gotcha 1).

## Verification
- **Test bridge.** With `[Debug] Bridge=1`, the plugin polls `%TEMP%\CinematicExplorer.cmd` every
  250 ms and runs commands from it. Among them:
  - `shot` copies the backbuffer to a BMP after the overlay is drawn, which works in exclusive
    fullscreen;
  - `freecam`, `move`, `key <vk> <ms>`, `mouse`, `fov`, `roll`, `freeze`, `timescale`, `level`,
    `noclip`, `tp`, `bookmark`;
  - `census` logs every VIEW/PROJ and skinned-mesh world position for one frame;
  - `scan` / `rescan` / `ptrs` / `peek` / `poke` are an in-process memory scanner.

  An agent script boots the game, presses Enter through the logos and movies, queues `level N` and
  drives the tests.
- **Seen in screenshots:**
  - the freecam seeded exactly on the game view;
  - turning keeps the kids and the hero in place;
  - flying out through the car wall shows the exterior;
  - FOV 100 and roll 25 apply;
  - freeze holds poses across two frames;
  - the bookmark is written to `bookmarks.json`;
  - the hero teleported between seats, with the game camera following;
  - noclip flew him 3.6 m up and 7 m forward through the wall into the next car.
- **Level switch:** every level code was loaded from a fresh boot by script. Each time the log recorded
  the level code, whether the hero was found, and the spawn XYZ.
  - All loaded except **(3,0) Repair the Engine**, which crashes (Gotcha 11).
  - The hero ("boy") exists in the train-car levels: 1:0–1:4, 1:6 and 1:7. The rooftop, engine room,
    North Pole and other levels use other player classes.
- **Not verified:** gamepad input, resolutions above 640×480 on real hardware, and long sessions after
  repeated D3D Reset with the overlay open.

## Gotchas
1. **The log, saves and bookmarks are "missing" from the game folder.** **Cause:** the game is a
   virtualized legacy process, so its writes go to
   `%LOCALAPPDATA%\VirtualStore\Program Files (x86)\THQ\The Polar Express\`. `LoadLibrary` and
   `GetPrivateProfileString` read VirtualStore copies too, but the plugin's own CRT `fopen` did not.
   **Fix:** look there. For a dev loop, drop the `.asi`/`.ini` into VirtualStore (no UAC needed) and use
   `%TEMP%` for any file the plugin must read.
2. **Freecam seeded about 20 m off, interior invisible.** **Cause:** a camera position taken from a view
   matrix with the wrong transpose. With pitch near zero the error hides in y/z, so it looks almost right.
   **Fix:** `p = -t·Rᵀ`, i.e. `p_i = -(m[i][0]·_41 + m[i][1]·_42 + m[i][2]·_43)`.
3. **Characters stay glued to the screen while the set flies.** **Cause:** the vertex-shader skin path
   takes WVP from c0–c3 and never reads the device VIEW. **Fix:** in `SetVertexShaderConstantF(reg 0,
   n ≥ 4)`, multiply the uploaded matrix by `K = (V·P)game⁻¹ · (V·P)ours` for the current camera. An
   older build of the same mod saw zero constant uploads on another machine (fixed-function path), so
   handle both.
4. **Freeze by stopping QueryPerformanceCounter/timeGetTime stalls rendering.** **Cause:** the main loop
   only runs a frame when its ms clock advances. **Fix:** write the engine time scale at `core+0x228`
   every CoreTick instead.
5. **Writes to the hero's position revert.** **Cause:** the scan finds about five copies: RW's "current
   atomic" matrix global at 0x5CE5E0, bone matrices, the physics body and the frame LTM. Only the root
   frame's modelling position is authoritative. **Fix:** write `frame+0x40..0x48`, call
   `RwFrameUpdateObjects`, and for noclip do it after the original CoreTick returns.
6. **Movement keys do nothing in tests.** **Cause:** the hero walks with the arrow keys, and synthetic
   keys must be scan codes for DirectInput. **Fix:** send scan-code arrows.
7. **Pressing Enter on the main menu starts a New Game and overwrites progress.** **Fix:** back up
   `Save\savedata.pex` (in VirtualStore) first, and use the level switcher instead of menu navigation.
8. **Crash in the loader thread at PolarExpress+0x199dc4 on launch.** **Cause:** started with the wrong
   working directory (`Data\Art\Fonts\persist.bik` fails to open). **Fix:** launch with the game folder
   as the working directory.
9. **The game stops ticking when unfocused or minimized.** Focus its window before driving tests.
10. **Screenshots of exclusive fullscreen come out black** from desktop-duplication tools. **Fix:** copy
    the backbuffer from inside `Present` (`GetRenderTargetData` into a SYSTEMMEM surface).
11. **StartLevel(3,0) (Repair the Engine) crashes** with a null `this` at PolarExpress+0xfc224, reached
    via FUN_0047d740 → FUN_004df200. It also crashes through the front end's own start call (0x409fe0)
    and with the localization redirect off. The root cause is unresolved, so the switcher blocks it.
12. **A vectored crash logger also sees faults that your own `__try` probes handle** (for example
    level+0xA8 holding a float in levels without the boy). With a capped log, they hide the real crash.
    **Fix:** set a thread-local flag around guarded reads and skip those faults in the handler.

## Assets
None. Code only.

## Cost and time
About one long agent session for recon, decompile, overlay, freecam, levels and the hero. Several game
restarts per iteration (≈45 s each, from boot to level).

## Open questions
- Why (3,0) crashes when loaded directly. It probably needs state that the previous level or the chapter
  intro sets up.
- Which of (1,6) / (1,7) is tennis and which is Ice-teroids. In screenshots, (1,6) is a car with a
  table-tennis table and (1,7) an icy car. Whether module 6 subs ≥ 10 (from the bonus
  pages) are the tube races.
- Levels whose player isn't the `"boy"` class (if any) have no hero teleport yet.
- The engine culls and streams from the game camera, so flying far still shows void. A real fix would move
  the game's own camera frame (RwCamera) with the freecam.
