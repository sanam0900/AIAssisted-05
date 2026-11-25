# PART 2 — Restructure

## What changed

The single `index.html` (7421 lines, one `<script type="module">`) is now a Vite + TypeScript npm project.

## Project layout

```
stillframe/
├── index.html              ← HTML + CSS only; <script src="/src/main.ts">
├── src/
│   ├── main.ts             ← All game logic (TypeScript)
│   ├── wgsl.d.ts           ← Type declaration for *.wgsl?raw imports
│   └── shaders/
│       ├── common.wgsl     ← Shared structs: FrameUniforms, Instance, VertexIn/Out
│       ├── scene.wgsl      ← Opaque geometry pass (PCF shadows, fog)
│       ├── shadow.wgsl     ← Shadow depth pass
│       ├── glass.wgsl      ← Refractive glass pass
│       ├── fx.wgsl         ← Particle/bullet-trail segments
│       ├── post_common.wgsl← PostUniforms struct + fullscreen triangle vertex shader
│       ├── outline.wgsl    ← Depth+normal edge detection
│       ├── bloom.wgsl      ← 5-level bloom (prefilter / down / up)
│       └── final.wgsl      ← ACES tonemap, grain, vignette, chromatic aberration
├── package.json
├── tsconfig.json
├── vite.config.ts
└── .vscode/
    ├── launch.json         ← F5 launches Chrome on dev server
    └── tasks.json          ← vite:dev / vite:preview / vite:build tasks
```

## Why this split

Responsibility boundary follows the renderer pipeline:
- `common.wgsl` — shared structs referenced by scene, shadow, and glass passes
- Each subsequent shader file is exactly one render pass
- `post_common.wgsl` — the fullscreen triangle vertex shader reused by outline, bloom, and final
- `main.ts` — all JS/TS logic unchanged; imports shaders via `?raw` (Vite built-in)

## Structural changes made

| Before | After |
|--------|-------|
| CDN import of `wgpu-matrix` at runtime | npm dependency `wgpu-matrix@^3.4.2` (lockfile pinned) |
| 9 WGSL programs as JS string literals | 9 `.wgsl` files in `src/shaders/`; imported as raw strings via `?raw` |
| Single monolithic HTML file | `index.html` (HTML/CSS) + `src/main.ts` (logic) |
| No build step | `npm run build` → `dist/` (Vite + esbuild) |
| No source maps | `sourcemap: true` — breakpoints in `.ts` files work from VS Code |

## Verifying identical behaviour

The game logic in `src/main.ts` is the extracted JS with two mechanical changes:
1. Import line: CDN URL → `import { mat4, vec3 } from "wgpu-matrix"`
2. Shader variables: template literals → `import shader from "./shaders/X.wgsl?raw"`

The concatenation pattern `common_default + scene_default` is preserved unchanged — the renderer still combines the shared struct definitions with each pass shader at pipeline-creation time, exactly as before.

The production build (`npm run build`) succeeds with zero type errors.
The game still shows the STILLFRAME title screen in the browser — same HTML, same JS, same WebGPU behaviour.

## Known bugs deferred to Part 3

Bugs found during restructure are logged in `BUGS.md` but not fixed here — a refactor that silently changes behaviour is impossible to review.
