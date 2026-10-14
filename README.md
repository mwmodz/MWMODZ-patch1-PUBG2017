# PUBG_Match v6: how it works + every setting

PUBG_Match is a server-side DLL of match fixes:

- blue zone timing;
- the blue wall before the first circle;
- the plane: route, smoothness, F to jump, eject at the end;
- trusting player movement (no rubber-banding at rocks);
- loading map sections the server skipped.

It never touches the bots. Every feature can be switched on or off in `PUBG_Match.ini`.

**Files**
- `PUBG_Match.dll`
- `PUBG_Match.ini`, next to the DLL.
- Log: `PUBG_Match_LOG.txt`, next to the DLL. Every line is written straight to disk, so after a
  crash the last line shows where it stopped. Lines are also printed as `[Match]` in the console.

**Injection:** by `0 - Start PUBG Server.bat`, after PUBG_Status and before PUBG_Loot. It only
starts on the server: it waits up to 10 min for a `TslGameMode` world, and if it's on a client it
logs that and does nothing.

---

## How it runs

- **Hook.** It chains its own MinHook hook onto `UObject::ProcessEvent`, like the other mods.
  Everything happens **on the game thread** inside that hook. It finds the game thread itself:
  first it guesses the process's oldest thread, and if that guess is wrong, it re-detects it from
  the hook after 2 s.
- **Two kinds of work:**
  - Per-call checks: movement trust and the F-to-jump detection. These are cheap; function names
    are looked up once and cached.
  - A "think" at most every 0.1 s: zone, plane, map sections, jump assist.
- **Settings reload.** `PUBG_Match.ini` is re-read **every time a new world loads**, i.e. every
  match. Your server restarts after every match anyway. Values are read as numbers: on/off
  settings are `1`/`0` (anything non-zero = on). A missing key keeps the code default.
- **Distances:** in **cm** (100 = 1 m). Erangel is 0…816000 on X and Y; the centre is 408000.

---

## 1. Blue zone `[Zone]`

**The problem:** the server closes each shrink in **half** the time the game announces to players.
Players see the circle creep slowly, then snap shut. Example: announced 300 s, done in 150 s.

**The fix:** H4TIUX's fix, the same idea as his UE4SS Lua.
- When a shrink starts, the duration players are shown is multiplied by `ReleaseScale` (0.5).
  The game state is then pushed out to players immediately, so clients animate the circle at the
  server's real speed.
- The server's own gas task is kept on that same duration on every think.
- When a new warning starts, the warning task's countdown is set to the game state's warning
  time.
- The gas tasks (`Task_GasRelease_C` and `Task_GasWarning_C`) aren't in the SDK, so their fields
  are found by name at runtime. They're looked up every 10 s, at most 12 times, until found.
- It only runs while the match is `InProgress`.

**The blue wall before the first circle (`PreZoneCover`)**
- The problem: before the first circle, the game left the blue wall's *start* circle at 0 m. That
  can make every player count as "outside the zone", so they get the blue screen effect in the
  plane.
- Until the first warning or shrink: if the blue wall, or its start circle, doesn't reach all four
  map corners, PUBG_Match centres both on the map. The radius becomes the larger of the existing
  one and centre-to-corner + 50 m (5820 m).
- It is checked on every think until the first circle, logged, and changed at most 20 times.
- v6 no longer touches the game's smoothed (`Lerp…`) circle values. Since the server never
  updates them itself, a value written there stuck for the whole match, and the bots read the
  blue wall as never moving. That was the v5 bug.

**Known:** the server also runs the **warning** countdown at about 2× speed, so "300 s" really
lasts about 150 s. That isn't changed: only the shrink display is corrected.

| Key | Values | Default | Live | What it does |
|---|---|---|---|---|
| `ZoneFix` | 1/0 | 1 | 1 | Master switch for the shrink-time fix (and the task sync below). |
| `ReleaseScale` | 0.05–1 | 0.5 | 0.5 | Announced shrink time × this = what players are shown. 0.5 matches the server. 1.0 = no change. Values ≤ 0.05 or > 1 fall back to 0.5. |
| `SyncTasks` | 1/0 | 1 | 1 | Keeps the server's gas tasks on the game state's durations (warning countdown + shrink duration). |
| `PreZoneCover` | 1/0 | 1 | 1 | Before the first circle, makes sure the blue wall covers the whole map (above). |
| `ZoneLog` | 1/0 | 1 | 1 | Logs the blue wall, its start, the white circle, durations and gas state: every 10 s until the first circle, then every 60 s (max 40 lines). |

---

## 2. Plane `[Plane]`

**The problem:** PUBG_InGame builds the plane's path from settings.ini StartPos/EndPos as up to
154 points **bouncing back and forth** between start and end. The plane crosses that whole zig-zag
in its fixed flying time (~80 s), which looks like jitter, U-turns, curves and slowdowns. On top
of that, the game sent the plane's position to players only about once a second.

**The fix (as soon as a new plane appears; planes are checked every 0.5 s):**

1. **New route.**
   - A random direction, through a random point within `CrossRadius` of the map centre (spread
     evenly over that circle).
   - The line is cut where it meets the square `EdgeMin`…`EdgeMax`, so it starts and ends just
     off the coast.
   - Routes shorter than 60% of the map (clipping a corner) are thrown away. Up to 20 tries.
2. **Plane moved.** The plane is teleported to the route start, facing along the route. Riders
   are attached, so they come with it.
3. **Path replaced.** Two straight (linear) points at `Altitude`. With `SplineAbsolute=1` the path
   is pinned to map coordinates instead of following the plane.
4. **Path guard.** If something rewrites the path afterwards (PUBG_InGame keeps adding points),
   the straight route is put back and logged. At most 20 times.
5. **Smoother plane on players' screens** (`PlaneNetRate`). The plane's network update rate is
   raised: 30/s, minimum 10/s. Priority is raised to ≥ 3, and the plane is kept relevant to
   everyone.
6. **Jump open** (`OpenJump`). If the plane's jump area isn't open, PUBG_Match opens it. If
   something closes it again before the end, it's re-opened once.
7. **Eject at the end** (`EjectAtEnd`). When the plane is `EjectBeforeEnd` from the end of the
   route, everyone still aboard is thrown out with the game's own `EjectAll`, which gives a
   normal skydive. This happens once.

**Speed:** the plane always takes its own flying time (~80 s) to cross whatever path it has, about
350–400 km/h on these routes. `SpeedKmh` changes the plane's recorded flying time, but the plane
**ignores it**. In v2, setting it is what broke F-to-jump. **Leave it at 0.**

### F to jump: `JumpAssist`

**What the game does when a player presses F in the plane:**
1. The player's game sends `ServerStartInteractionByComponent` and `ServerTryInteractByComponent`
   (the log shows both).
2. The server agrees it's allowed: `AllowInteractBy` returns true.
3. Then it never actually ejects the player.

**What JumpAssist does about it:**
- It watches for that F request from anyone sitting in the plane: any server request with
  Interact/Leave/Exit/Eject/Jump/Parachute in its name, except Cancel and seat swaps.
- After `JumpAssistDelay`, if the player is still aboard and the jump is open, it does the jump
  itself, in this order:
  1. the plane's own jump handler (`OnInteractBy`). That gives the **normal skydive + parachute**.
     All successful test jumps took this path.
  2. If still aboard: give them the plane's parachute item, then leave the seat normally.
  3. Last resort: force them out of the seat. Note that in v4 this alone gave no skydive and no
     chute.
- Each step and its result are logged.

| Key | Values | Default | Live | What it does |
|---|---|---|---|---|
| `PlaneRoute` | 1/0 | 1 | 1 | Master switch for the new straight route. 0 = PUBG_InGame's own path, and no eject-at-end. |
| `MapSize` | cm | 816000 | — | Map size (Erangel). Used for the centre, the 60% minimum route length and the zone corner check. |
| `CrossRadius` | cm | 120000 | 120000 | The route passes within this distance of the map centre. Bigger = more varied routes, more near the edges. |
| `EdgeMin` / `EdgeMax` | cm | 30000 / 786000 | same | The square the route starts and ends on (300 m in from the map edge). |
| `Altitude` | cm | 150000 | 150000 | Flight height (1500 m). |
| `SpeedKmh` | km/h, 0 = off | 0 | 0 | Rewrites the plane's flying time for this speed. The plane ignores it, and it can break jumping. **Keep 0.** |
| `EjectAtEnd` | 1/0 | 1 | 1 | Throw everyone still aboard out near the far coast. Needed: the plane stops and hovers at the end of the route. |
| `EjectBeforeEnd` | cm | 80000 | 80000 | How far before the route end that happens (800 m), so nobody lands in the sea. |
| `OpenJump` | 1/0 | 1 | 1 | Make sure the plane's jump is open once it's on the new route, and re-open it once if something closes it. |
| `PlaneNetRate` | updates/s, 0 = game default | 30 | 30 | Plane position updates per second to players, plus higher priority and always-relevant. The game's own rate was 1/s. |
| `JumpAssist` | 1/0 | 1 | 1 | Do the jump for a player who pressed F and is still aboard (see above). |
| `JumpAssistDelay` | seconds | 0.3 | 0.3 | How long to give the game to do it itself first. It never has, so keep it short. |
| `JumpLog` | 1/0 | 1 | 1 | Log what players' games send the server while in the plane (first 3 of each request type, every F request; 200 lines max). For 6 s after an F press, also log the game's own jump-related calls with their yes/no results (150 lines max). |
| `PlaneLog` | 1/0 | 1 | 1 | Log the plane every 3 s: position, speed, metres off the route, jump open, in jump area. Also the original path, the new route and any path restores. |
| `SplineAbsolute` | 1/0 | 1 | 1 | Pin the new path to map coordinates instead of to the (moving) plane. |

---

## 3. Player movement `[Movement]`

**The problem:** some rocks and meshes differ between the server and players' games, so the
server keeps "correcting" players. That shows up as rubber-banding, or getting stuck at rocks.

**The fix:** H4TIUX's fix. When a player's movement update arrives (`ServerMove`,
`ServerMoveDual`, `ServerMoveDualHybridRootMotion`, `ServerMoveOld`), that player's movement
component gets `bIgnoreClientMovementErrorChecksAndCorrection = 1`. The server then accepts the
player's own position. This is logged for the first 50 players.

**Downside:** players are trusted completely, so a speed or teleport cheat would go straight
through. That's fine among friends, but it's the first thing to harden with a server-side
speed/teleport check.

| Key | Values | Default | Live | What it does |
|---|---|---|---|---|
| `TrustClientMovement` | 1/0 | 1 | 1 | The server trusts players' positions (no corrections). 0 = the game's normal checks. |

---

## 4. Map sections `[Map]`

**The problem:**
- PUBG_InGame force-loads the map's streaming sections, but skips **every name containing "32"**.
  That's meant to skip the weather variants (sections 320–332), but it also hits real sections
  like 32, 132 and 232.
- Other sections are left to distance streaming, which a server with no nearby camera may never
  trigger.
- Result: the server can be missing floors, rocks and loot spots that players can see. One
  example was a building with no loot.

**The fix:**
- 5 s after the world loads (to let PUBG_InGame do its own pass first), every streaming section
  that isn't loaded gets loaded. The exception is any section whose trailing number is
  `WeatherFirst`…`WeatherLast`. Each loaded section is set loaded + visible + never
  distance-unloaded, and loads in the background, so the server doesn't freeze.
- Every section it loads is logged by name. The total is reported 20 s later. Typical: "227–273
  of 333 loaded, 54–100 requested → 327 of 333"; the 6 left are the weather variants.

**Note:** this keeps the **whole map** loaded on the server, which uses more RAM than PUBG_InGame
alone.

| Key | Values | Default | Live | What it does |
|---|---|---|---|---|
| `LoadMissingTiles` | 1/0 | 1 | 1 | Load every map section the server skipped (except the weather range). |
| `WeatherFirst` / `WeatherLast` | section numbers | 320 / 332 | 320 / 332 | Sections with a trailing number in this range are left alone: weather variants such as Weather_Clear/Dark/Foggy/Rainy, Heightmap_x3_y3 and Benchmark. Same range H4TIUX skips. |

---

## Reading the log

**Normal lines:**

| Line | Meaning |
|---|---|
| `[PUBG_Match v6] loaded` / `ProcessEvent hooked` | Started fine |
| `new world - settings reloaded (...)` | A new match; shows which features are on |
| `map: section was NOT loaded ... loading it now` / `map: now 327 of 333` | Map section fix |
| `movement: trusting client position for ...` | Movement trust applied to a player |
| `zone: gas tasks - release found ... warning found ...` | Zone fix can sync the gas tasks |
| `zone: before the first circle the blue wall doesn't cover the map ... centring it` | PreZoneCover acted |
| `zone: warning N s` / `zone: shrink starts - game said N s, players now see N/2 s` | Zone fix working |
| `plane: spotted` → `original path ...` → `new route ...` → `new path 2 points` | Plane rerouted |
| `plane: at X,Y,Z  N km/h  0 m off the route  jump open, in jump area yes` | Plane healthy |
| `plane: path changed again ... restoring` | Something rewrote the path and it was put back |
| `plane: reached the far coast - ejecting everyone still aboard` | Eject at the end |
| `jump: F/interact from a player in the plane ...` → `jumped them out with the plane's own jump (normal skydive)` | JumpAssist did its job |

**Problem lines:**

| Line | Meaning |
|---|---|
| `jump: ... STILL in the plane` | All three jump steps failed |
| `MH_... failed` | The hook couldn't install |
| `no server game mode found (client?)` | Injected into the wrong process |




source
PUBG_Match v6 - source
======================

Match/MatchMain.cpp   all of PUBG_Match (one file): blue zone fix, pre-zone blue wall, plane route +
                      smoothing + jump assist + eject at end, client movement trust, missing map sections.
MATCH_SETTINGS.md     how it works + every PUBG_Match.ini key + how to read PUBG_Match_LOG.txt
SDK/ + SDK.hpp ...    Dumper-7 SDK for this build (same as the other mods)
minhook_src/          MinHook;  winshim/ = Windows.h shim for cross-compiling

Build (Linux, clang + MinGW-w64 13 posix):   ./build_match.sh   -> PUBG_Match.dll
Check:  x86_64-w64-mingw32-objdump -d PUBG_Match.dll | grep -c '%gs:0x58'   -> must print 0
Live settings: ../config/PUBG_Match.ini

