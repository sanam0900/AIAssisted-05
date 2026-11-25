# PART 1 — Understanding STILLFRAME

## What is the game?

STILLFRAME is a first-person shooter set in a transit system (subway platforms, escalators, turnstiles) and an office building. The player fights waves of three enemy archetypes — walker, rusher, marksman — using melee, throwable props, and three firearms (pistol, scatter, carbine). A level is cleared when all enemy waves are defeated or, in some levels, when the player reaches an exit.

The game runs entirely in the browser using **WebGPU** (not WebGL). There is no server, no external asset loading, and no audio files — sounds are synthesized on the fly via the Web Audio API. The entire game is a single HTML file of ~7 400 lines.

---

## The unique mechanic — Time Moves When You Move

The central mechanic is a **motion-driven time scale** implemented by the `TimeScaleDriver` class (line 6161).

> "Time flows at the speed you create."

By default the world ticks at **1.5 % of real speed** (`MIN = 0.015`) — enemies, bullets, and physics are nearly frozen. The time scale climbs toward 1.0 in direct proportion to three signals:

| Signal | Weight |
|--------|--------|
| Planar movement speed (WASD) | proportional to `WALK_SPEED` |
| Mouse-look angular speed | `LOOK_WEIGHT = 0.35`, saturates at `LOOK_FULL = 900 px/s` |
| Action "kicks" (shoot / throw / punch) | momentary impulse via `actionKick()` |

The final `timeScale` is `max(moveInput, lookInput, kickInput)`, smoothed with `approach()` so it never snaps. Enemies fire, move, and telegraph attacks only as fast as the player acts. Stopping completely freezes the battlefield — the player can read laser aim-lines before they fire.

The same scale feeds:
- the physics fixed-step accumulator (`Time` class, line 2056)
- the audio pitch / tempo (`AudioEngine.setTimeScale`)
- visual post-processing freeze/desaturation (`updatePost`, line 7216)
- bullet trail length (`trailLen = 0.6 + 5 * (1 - ts)`, line 7148)

---

## Subsystem map

| # | Subsystem | Lines | Responsibility |
|---|-----------|-------|----------------|
| 1 | **HTML / CSS** | 1–494 | UI layout: canvas, HUD overlay, crosshair, pause/results screens, settings forms |
| 2 | **Gfx** | 505–631 | WebGPU adapter + device creation, canvas resize observer, resolution scaling |
| 3 | **Mesh / buffers** | 632–664 | GPU vertex/index buffer allocation (`createBufferWithData`) |
| 4 | **WGSL shaders** | 666–1133 | 9 shader programs as JS string constants: `common_default`, `scene_default` (PCF shadows), `shadow_default`, `glass_default` (refraction), `fx_default` (particle segments), `post_default`, `outline_default` (depth+normal edge detect), `bloom_default` (5-level), `final_default` (ACES tonemap, grain, vignette, chromatic aberration) |
| 5 | **Renderer** | 1135–1833 | Creates all 9 WebGPU render pipelines; orchestrates the full frame: shadow → opaque → glass → fx → viewmodel → outline → bloom → final; manages per-frame instance and segment buffers |
| 6 | **Camera** | 1836–1874 | Yaw/pitch first-person camera; perspective + view matrix via wgpu-matrix CDN import |
| 7 | **Input** | 1876–2054 | Keyboard/mouse/gamepad; pointer lock; per-frame press/release tracking; `DEFAULT_BINDINGS` (WASD + mouse) |
| 8 | **Time / Loop** | 2056–2156 | `Time`: fixed-step accumulator at 120 Hz; `Loop`: `requestAnimationFrame` calling `frame` / `fixedStep` / `render` callbacks |
| 9 | **AudioEngine** | 2233–2507 | Web Audio API; procedural sound synthesis — `gunshot`, `lowpass`, `highpass`; `synthesizeAll()` builds all SFX from `SYNTHS` lookup table; synthesized music via oscillators + LFO |
| 10 | **LevelBuilder / Level data** | 2509–2837 | Box-based level DSL; `ACT1` = 5 levels (platform/car/turnstiles/escalator/lastTrain); `ACT2` = office theme levels |
| 11 | **Ui** | 3400–3777 | Builds all screens (title, levels, pause, results, replay) via `innerHTML`; `updateHud()`; localStorage save (`KEY = "stillframe.save"`) |
| 12 | **StaticWorld** | 3999–4203 | 2D spatial grid (cell = 2 m); axis-separated AABB movement with step-up (`moveCharacter`); sphere–capsule collision for props/shards; `raycast()`; `lineClear()` |
| 13 | **NavGrid / A\*** | 4305–4581 | A\* on 0.5-unit grid; `bake()` pre-computes walkable floor heights; `findPath()` with path smoothing |
| 14 | **World / Enemies** | 4792–6053 | Game state (enemies, props, projectiles, player); enemy state machine (idle → alert → pursue → stunned); shooting, melee, wave system; bullet–bullet clash mechanic |
| 15 | **TimeScaleDriver / FxSystem / Game** | 6161–7421 | `TimeScaleDriver` drives world tick rate from player input; `FxSystem` manages particles and glass shards; `Game` wires everything into the main loop |

---

## One-frame trace — pressing W for the first time

```
1. Browser fires keydown("KeyW")
   └─ Input.onKeyDown()  [line ~1900]
        sets heldKeys.add("move_forward")

2. Loop.requestAnimationFrame fires
   └─ Loop.tick()  [line ~2100]
        calls Game.frame(t)

3. Game.frame(t)  [line 6957]
   ├─ input.update(dt)       — copies held/pressed state, reads mouse delta
   ├─ player.update(input, dt, world, look, true)   — WASD integrates velocity
   ├─ player.toFrame(...)    — packages pos/vel/yaw/pitch + action flags
   ├─ world.applyPlayerFrame(f)   — writes player state into simulation
   └─ driver.update(dt, WALK_SPEED, planarSpeed, lookSpeed, airborne, busy)
        timeScale = max(moveInput, lookInput, kickInput)   [line 6161]
        t.timeScale = smoothed result

4. Time accumulator adds (realDt × timeScale) to sim budget
   └─ while budget ≥ FIXED_STEP (1/120 s):
        Loop calls Game.fixedStep()
          └─ world.step(FIXED_STEP)   [line 5221]
               — enemy AI, projectiles, physics, wave spawning

5. Loop calls Game.render(t)  [line 7038]
   ├─ handleEvents(world)    — plays SFX, spawns FX particles
   ├─ fx.update(visualDt)    — advances particle/shard positions
   ├─ camera.update()        — recomputes view + projection matrices
   ├─ drawWorld(...)         — submits instance draw calls to Renderer
   ├─ fx.draw(renderer)
   ├─ viewmodel.draw(...)
   └─ renderer.render(cam, vmCam, post, realTime)   [line 1135]
        shadow pass → opaque pass → glass pass →
        fx pass → viewmodel pass → outline pass →
        bloom (5 levels) → final (tonemap + grain) →
        GPUCommandEncoder.submit()
```

---

## Three design problems

### 1 — Everything in one 7 400-line file
All 15 subsystems share a single `index.html`. There are no ES modules, no imports, no directory structure. Adding a feature means searching the whole file. Any name collision silently overwrites a previous symbol (all classes/functions are `var` declarations in script scope). It is impossible to tree-shake, dead-code-eliminate, or unit test any subsystem in isolation.

### 2 — Shaders embedded as plain JS string literals
The nine WGSL programs live as template-literal string constants (e.g., `const scene_default = \`...WGSL...\``) with no file extension, no syntax highlighting, no WGSL language-server support, and no compile-time validation. A typo in a shader only surfaces as a WebGPU pipeline-creation error at runtime. The shader code is also mixed into the middle of the JS file, making it impossible to import an external WGSL linter or use editor tooling.

### 3 — CDN import at runtime with no version pinning
The only external dependency, `wgpu-matrix`, is loaded via a `type="importmap"` pointing at `https://cdn.jsdelivr.net/npm/wgpu-matrix@3/dist/3.x/wgpu-matrix.module.min.js`. The CDN URL contains no content-hash, so a CDN outage or a silent patch to that package can silently break the game. There is no `package.json`, no lockfile, and no offline fallback. Running the game without internet access is impossible.
