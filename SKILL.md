---
name: cinematic-webgl-site
description: Build an award-style, scroll-driven 3D website (Astro + Three.js WebGPU/TSL + Lenis + anime.js) with a fixed full-screen canvas, damped scroll-to-camera choreography, Gaussian-splat / GPGPU-particle / depth-relit-image scenes, and a VHS/CRT post-processing stack. Use when the user wants a "stunning", "silky", "Awwwards-like" or immersive scroll-animated site, or asks to add cinematic WebGL scenes, scroll-driven camera moves, retro tape/CRT post effects, or particle/splat hero sections.
---

# Cinematic WebGL site playbook

Build original scenes, copy, fonts and assets.

## 1. Architecture overview

| Thought it was | Actually is |
|---|---|
| "The magic is WASM" | The `.wasm` files are **Draco** (mesh decompression) and **Basis/KTX2** (texture transcoding) decoders shipped with three.js loaders. They only decode assets; no animation logic lives in them. |
| "Astro does the animation" | Astro 6 is only the static HTML shell plus `<ClientRouter />` view transitions. All motion is client JS. |
| A normal scrolling page | `<html style="--scroll-viewports: 18.1">`: one **tall invisible spacer** (≈18 viewport heights) plus a **fixed full-screen canvas**. Scroll position is a number that drives a camera; nothing in the DOM actually scrolls past you. |

**Recommended stack:**
- **Astro v6.4** (static, ClientRouter view transitions) + **Tailwind v4** (CSS `@theme` tokens)
- **three.js r187** with `three/webgpu`: `WebGPURenderer`, **TSL** node shaders, node post-processing (`pass()`, `BloomNode`, TRAA, MRT)
- Rendering inside a **Web Worker** on an `OffscreenCanvas`. Frames return as `ImageBitmap` to a `bitmaprenderer` canvas, so the main thread stays free for scroll and DOM.
- **Lenis 1.3** smooth scroll; **anime.js 4.5** for DOM tweens; **nanostores** for shared state (scroll, route, preloader, tilt), mirrored into the worker
- **Gaussian splats** (custom `.vsplat` format plus loaders for PLY/SPLAT/KSPLAT/SPZ) for photoreal 3D scans; GLB models (Draco/meshopt) and KTX2 textures; HDR environment
- Tiered fallback: `webgpu → webgl → none` (none redirects to a static bio page). Sessionstorage remembers a downgraded tier after device loss. WebGPU **pipelines are pre-warmed** in 3 workers from `/pipeline-manifest.json`, so the first scroll doesn't stutter on shader compiles.
- Device-orientation tilt on touch, behind an explicit "Allow motion sensors" prompt; `prefers-reduced-motion` respected everywhere
- PostHog analytics, Cloudflare Pages hosting

## 2. Why it feels so satisfying (the actual secrets)

1. **Everything is damped, never set.** Per frame: `cam = damp(cam, target, λ, dt)` (`THREE.MathUtils.damp`, λ≈6-8). Lenis smooths the input, then damping smooths the 3D response, which gives two layers of inertia.
2. **Scroll is measured in viewports, not pixels.** `scrolledViewports = scrollY / innerHeight`, and camera travel = `scrolledViewports * distancePerViewport` (≈2.45 units). The choreography is a timeline of *stages* and *wipes* defined in viewport units, with **snap points** at each stage.
3. **One continuous world.** No page reloads for the canvas; route changes (Astro ClientRouter) only swap DOM while the WebGL scene transitions.
4. **Heavy, tasteful post.** A configurable chain: fisheye/bulge → camcorder handheld wobble → tape blur/noise/tracking lines/chroma bleed → CRT glow, hum bar, flicker, vignette, screen curvature → bloom. All values are uniforms, eased per frame and tied to scroll (e.g. a "handover" grain burst between scenes).
5. **Loading is designed, not hidden.** A preloader gate waits for `progress` plus shader `warmProgress`, then fades. Assets are compressed (Draco, KTX2, AVIF) and shaders pre-compiled.
6. **Restrained, typographic UI.** Black field, dot-matrix display fonts, mono/system body text, tiny UI with custom easings. The 3D does the shouting.
7. **Reliability engineering.** Device-lost recovery, chunk-load-error reload, tier downgrade, phone-specific camera presets, and DPR caps.

## 3. Our build recipe

Default stack: **Astro + three (`three/webgpu`, auto-falls back to WebGL2) + Lenis + anime.js + nanostores + Tailwind v4**. Add `@sparkjsdev/spark` only if we use Gaussian splats; GLSL effects can be ported to TSL.

```
src/
  layouts/Base.astro        # fixed <canvas id="gl">, spacer div, ClientRouter, preloader
  pages/index.astro         # DOM sections that overlay scenes (text, labels)
  gl/
    app.ts                  # renderer (WebGPU→WebGL fallback), loop, resize, DPR cap
    stores.ts               # nanostores: scroll, preloader, pointer, tilt, reducedMotion
    scroll.ts               # Lenis → scroll store (viewports), snap points
    timeline.ts             # stage/wipe table in viewport units → per-scene progress 0..1
    scenes/*.ts             # each scene: setup(), update(progress, dt), dispose()
    post/tape-crt.ts        # TSL post chain with uniform-driven layers
    fx/gpgpu-particles.ts   # particles (see references)
    fx/depth-relight.ts     # 2D image → relit surface
  styles/global.css         # @theme tokens, fonts, easings
```

Build order (each step must look good before the next):
1. Shell: fixed canvas, spacer height from timeline, Lenis, scroll store, damped camera over a placeholder scene. **Feel test the scroll first.**
2. Timeline: stages, wipes, snap points; DOM sections fade in at stage progress (anime.js or CSS vars).
3. Hero scene (pick one or two): GPGPU particle model, depth-relit photo, kinetic image cylinders, or a splat scan.
4. Post stack: bloom, then CRT/tape layers, each behind a flag; scroll-linked "handover" glitches between stages.
5. Preloader gate with real progress, pre-warm (`renderer.compileAsync(scene, camera)`), fonts preloaded.
6. Hardening: reduced motion (cut damping to instant, disable tape noise), phone presets, DPR cap ≤ 2, pause when tab is hidden, device-lost fallback, static fallback page.
7. Optional, later: move rendering to a Worker with OffscreenCanvas (only if the main thread is janky).

## 4. Design tokens (ours, in the same spirit)

- Background `#000`/`#050505`, text `#fff`, one or two signal accents (e.g. phosphor green `#1f0`, ice `#cde7e9`)
- Display: **Doto** (Google Fonts, OFL, dot-matrix). Pair with a mono or system sans for body. Don't use OffBit unless we license it ourselves.
- Easings: reveal `cubic-bezier(.25,1,.5,1)` .5s · menu `cubic-bezier(.22,1,.36,1)` .35s · page fade `cubic-bezier(.645,.045,.355,1)` 1s
- Tight leading on display type (0.66-0.95), CRT text glow `drop-shadow(0 0 1.5px #ffffffcf)`, step-end blink cursors

## 5. Rules

- The canvas is `position:fixed; inset:0`. Content scrolls over a spacer and must never fight the canvas for layout.
- Never write `camera.position = scrollY * k` directly; always damp toward a target with frame `dt` (clamped ≤ 1/30).
- Every effect parameter is a `uniform()` updated per frame, never a shader recompile.
- Gate DOM reveals and the scene start on the preloader store, not on `load`.
- Measure: 60 fps on a mid laptop, under 3 MB first load before 3D assets, compressed assets only.

See `references/recipes.md` for the code: Lenis↔scroll store, damped camera, timeline, TSL CRT/tape post, GPGPU particles, depth relighting, kinetic cylinders, and the preloader gate.
