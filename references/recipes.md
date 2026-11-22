# Recipes

These are starting points written against three r18x (`three/webgpu`, `three/tsl`). The TSL API still
moves between releases, so check imports against the installed version (`node_modules/three/src/Three.TSL.js`)
before trusting a snippet. Older releases named `RenderPipeline` `PostProcessing`.

---

## 1. Shell: fixed canvas + spacer + Lenis → store

```astro
<!-- layouts/Base.astro -->
<html lang="en" style={`--scroll-viewports:${totalViewports}`}>
  <body class="bg-black text-white">
    <canvas id="gl" class="fixed inset-0 h-svh w-screen" aria-hidden="true"></canvas>
    <div class="pointer-events-none" style="height:calc(var(--scroll-viewports) * 100svh)"></div>
    <main class="fixed inset-0 pointer-events-none"><slot /></main> <!-- overlay text, opt-in pointer-events -->
    <div data-preloader>…</div>
  </body>
</html>
```

```ts
// gl/stores.ts
import { map } from 'nanostores';
export const scroll = map({ viewports: 0, progress: 0, velocity: 0 });
export const preloader = map({ progress: 0, warm: 0, open: false });
export const env = map({
  reducedMotion: matchMedia('(prefers-reduced-motion: reduce)').matches,
  phone: matchMedia('(width < 768px), (aspect-ratio < 4/3)').matches,
});

// gl/scroll.ts
import Lenis from 'lenis';
export const lenis = new Lenis({ lerp: 0.09, smoothWheel: true, autoRaf: false });
lenis.on('scroll', (l) => {
  const vh = innerHeight;
  scroll.set({ viewports: l.scroll / vh, progress: l.progress, velocity: l.velocity });
});
// drive lenis from the same RAF as the renderer (see app.ts) so they never drift
```

## 2. Damped scroll camera (the "silky" part)

```ts
import { MathUtils } from 'three/webgpu';
const CAM = { distancePerViewport: 2.45, damping: 6 };
let travel = 0;
export function updateCamera(camera, dt) {
  dt = Math.min(dt, 1 / 30);                     // never jump after a hitch
  const target = scroll.get().viewports * CAM.distancePerViewport;
  travel = env.get().reducedMotion ? target : MathUtils.damp(travel, target, CAM.damping, dt);
  camera.position.set(start.x, start.y - travel, start.z);
  camera.rotation.x = -MathUtils.degToRad(pitch);
  // secondary motion: pointer/tilt parallax, also damped
  camera.position.x += parallax.x; camera.position.y += parallax.y;
}
```

Handheld wobble (summed sines with co-prime frequencies, scaled by a damped "handheld" weight):
```ts
const a = phase * Math.PI * 2;
offset.set(
  0.003 * Math.sin(a * 2 + .5) + 0.002 * Math.sin(a * 3),
  0.002 * Math.sin(a * 5 + 2) + 0.0015 * Math.sin(a * 7 + 1) + 0.00075 * Math.sin(a * 13),
).multiplyScalar(handheld);
```

## 3. Timeline: stages, wipes and snap points in viewport units

```ts
// gl/timeline.ts
export const stages = [
  { id: 'intro',    length: 2.5 },
  { id: 'work',     length: 8 },   // e.g. 4 projects × 2vh
  { id: 'about',    length: 4 },
  { id: 'contact',  length: 3 },
];
export const wipeLength = 0.6;     // overlap where post "handover" glitch peaks
let acc = 0;
export const table = stages.map(s => ({ ...s, start: (acc += s.length) - s.length }));
export const totalViewports = acc;
export const local = (id, vp) => { const s = table.find(t => t.id === id); return clamp01((vp - s.start) / s.length); };
export const handover = (vp) => Math.max(...table.slice(1).map(s => 1 - smoothstep(0, wipeLength / 2, Math.abs(vp - s.start))));
// snapping: on Lenis scroll end, if |velocity| small, lenis.scrollTo(nearestStageStart * innerHeight, { duration: 1.2 })
```

## 4. Renderer + loop (WebGPU with automatic WebGL2 fallback)

```ts
import * as THREE from 'three/webgpu';
const renderer = new THREE.WebGPURenderer({ canvas, antialias: false, forceWebGL: new URLSearchParams(location.search).has('webgl') });
await renderer.init();
renderer.setPixelRatio(Math.min(devicePixelRatio, env.get().phone ? 1.5 : 2));
const Pipeline = THREE.RenderPipeline ?? THREE.PostProcessing;
const post = new Pipeline(renderer);
post.outputNode = buildPost(scene, camera);           // §5
await renderer.compileAsync(scene, camera);          // pre-warm → preloader.warm = 1
const clock = new THREE.Timer();
renderer.setAnimationLoop((t) => {
  clock.update(t); const dt = clock.getDelta();
  lenis.raf(t);
  scenes.forEach(s => s.update(dt));
  postFx.update(dt);
  post.render();
});
document.addEventListener('visibilitychange', () => renderer.setAnimationLoop(document.hidden ? null : loop));
```

## 5. TSL tape + CRT post chain (original implementation of the look)

```ts
import { pass, uniform, uv, vec2, vec3, vec4, float, Fn, sin, fract, dot, mix, smoothstep, step, length, time, screenSize } from 'three/tsl';
import { bloom } from 'three/addons/tsl/display/BloomNode.js';

export const fx = {
  bulge: uniform(0.12), chroma: uniform(0.0035), grain: uniform(0.07), lines: uniform(700),
  scanline: uniform(0.12), vignette: uniform(0.7), hum: uniform(0.08), tracking: uniform(0),
  frame: uniform(0), glow: uniform(0.6),
};
const hash = Fn(([p]) => fract(sin(dot(p, vec2(12.9898, 78.233))).mul(43758.5453)));

export function buildPost(scene, camera) {
  const tex = pass(scene, camera).getTextureNode();
  const crt = Fn(() => {
    const p = uv().sub(0.5);
    const cuv = p.mul(float(1).add(fx.bulge.mul(dot(p, p)))).add(0.5);           // screen curvature
    const line = cuv.y.mul(fx.lines).floor();
    const band = smoothstep(0.07, 0.0, cuv.y.sub(fract(time.mul(0.07))).abs()).mul(fx.tracking);
    const jitter = hash(vec2(line, fx.frame)).sub(0.5).mul(0.002).add(band.mul(0.015));
    const s = cuv.add(vec2(jitter, 0));
    const col = vec3(                                                              // chroma bleed
      tex.sample(s.add(vec2(fx.chroma, 0))).r,
      tex.sample(s).g,
      tex.sample(s.sub(vec2(fx.chroma, 0))).b,
    ).toVar();
    col.mulAssign(float(1).sub(fx.scanline.mul(sin(s.y.mul(fx.lines).mul(Math.PI)).mul(0.5).add(0.5))));
    col.mulAssign(sin(s.y.mul(6.2832).add(time.mul(2.0))).mul(fx.hum).add(1));     // hum bar
    col.addAssign(hash(s.mul(screenSize).add(fx.frame)).sub(0.5).mul(fx.grain));   // grain
    col.mulAssign(mix(float(1).sub(fx.vignette), float(1), smoothstep(0.75, 0.2, length(p))));
    const inside = step(0, cuv.x).mul(step(cuv.x, 1)).mul(step(0, cuv.y)).mul(step(cuv.y, 1));
    return vec4(col.mul(inside), 1);
  })();
  return crt.add(bloom(crt, fx.glow, 0.4, 0.2));
}
// per frame: fx.frame.value = Math.floor(elapsed * 30);  // 30 fps "tape" cadence
// fx.tracking.value = damp(fx.tracking.value, handover(scroll.viewports), 8, dt);
// fx.grain.value = 0.07 + handover * 0.25;   // glitch burst between scenes
// reduced motion → tracking, grain, hum = 0
```

Layer flags pattern: keep a `layers` store (`{ crt: true, tape: true, bloom: true }`) and rebuild `outputNode` only when a flag changes. Animate values through uniforms, never through rebuilds.

## 6. GPGPU particles (WebGL version)

The core ideas:
- Sample N = size² points on a mesh with `MeshSurfaceSampler`, then store them in a float `DataTexture` (original positions).
- `GPUComputationRenderer` holds two variables, `uCurrentPosition` and `uCurrentVelocity`, each depending on both.
- The velocity shader damps velocity (`*= 0.7`), pulls each particle toward its original position (`+= dir * 0.0003`), and repels it from the mouse within radius 0.1, scaled by `uMouseSpeed`.
- The position shader adds velocity to position.
- Rendering uses `THREE.Points` with UVs indexing into the position texture, additive blending, and alpha from velocity (fast particles glow).
- Mouse input raycasts against the source mesh (`three-mesh-bvh` for speed) to get a world-space `uMouse`; speed decays by `*= 0.85` per frame.
- Post uses a modified UnrealBloomPass ("MotionBloomPass") at threshold 0.2 and strength 0.8.

**WebGPU/TSL equivalent (compute shaders, preferred for new work):**
```ts
import { instancedArray, Fn, instanceIndex, uniform, float, If, normalize, length, max } from 'three/tsl';
const N = home.length / 3;
const homeBuf = instancedArray(home, 'vec3');            // Float32Array from MeshSurfaceSampler
const pos = instancedArray(N, 'vec3'), vel = instancedArray(N, 'vec3');
const uMouse = uniform(new THREE.Vector3(1e3)), uMouseSpeed = uniform(0);
const init = Fn(() => { pos.element(instanceIndex).assign(homeBuf.element(instanceIndex)); })().compute(N);
const step = Fn(() => {
  const p = pos.element(instanceIndex), v = vel.element(instanceIndex), h = homeBuf.element(instanceIndex);
  v.mulAssign(0.7);
  v.addAssign(normalize(h.sub(p).add(1e-6)).mul(0.0003));
  const d = length(p.sub(uMouse));
  If(d.lessThan(0.1), () => { v.addAssign(normalize(p.sub(uMouse)).mul(float(1).sub(d.div(0.1))).mul(0.0023).mul(uMouseSpeed)); });
  p.addAssign(v);
})().compute(N);
await renderer.computeAsync(init);
const mat = new THREE.SpriteNodeMaterial({ transparent: true, depthWrite: false, blending: THREE.AdditiveBlending });
mat.positionNode = pos.toAttribute();
mat.scaleNode = float(0.004);
mat.colorNode = /* tint */ vec4(0.81, 0.65, 0.24, max(length(vel.toAttribute()).mul(100), 0.04).min(0.8));
const points = new THREE.Sprite(mat); points.count = N; scene.add(points);
// loop: renderer.compute(step)
```

## 7. Depth-relit image (TSL/WebGPU)

1. Generate a depth map (Depth Anything v2/3; white = near).
2. Convert the 8-bit values to float, blur slightly to remove banding, and store them as a half-float `DataTexture` (`DataUtils.toHalfFloat`).
3. Normals from the depth gradient (central differences) plus a weaker gradient of image luminance for micro detail:
   `n = normalize(vec3(-slope.x, -slope.y, 1) + vec3(-detail.x, -detail.y, 0))`.
4. Soft self-shadows: march `SHADOW_STEPS` samples from the pixel toward the light in UV space. At each step, compare the sampled depth to the rising ray depth, `blocked = clamp((blocker - ray)/softness)`, and keep the max as occlusion.
5. Material: `MeshPhongNodeMaterial({ specular: 0 })` with `colorNode = image`, `normalNode = fakeNormals`, `aoNode = 1 - occlusion`. A `PointLight` follows the damped pointer.

## 8. Kinetic image cylinders (R3F, works the same in vanilla three)

- Use a very low FOV perspective camera (`fov 7`, z 70) for a near-orthographic look.
- Stack N open `CylinderGeometry(r, r, 2, 100, 1, true)` "billboards", each rotated `i * π/2` on Y. Between them, place thin "banner" cylinders tilted `0.085` on Z. Tilt the whole group `(-0.15, 0, -0.2)`.
- Draw all images side by side into one canvas and use it as a `CanvasTexture` with `RepeatWrapping` on S, mapped around the cylinder.
- Animate with `texture.offset.x += speed * dt` or by rotating each group. Scroll velocity from Lenis adds to the speed, then decays.

## 9. Gaussian splats

Use `@sparkjsdev/spark` (`SplatMesh` for .ply/.spz/.splat). Capture scans with Polycam, Luma or Scaniverse, then convert and compress to `.spz`. Fade splat opacity in with `inOutCubic` over 1.5 s after mount. On phones, show splats only on dedicated pages, since they are heavy.

## 10. Preloader gate

```ts
// progress = loaded/total from THREE.LoadingManager; warm = 1 after compileAsync
preloader.listen(({ progress, warm }) => {
  if (progress >= 1 && warm >= 1 && !preloader.get().open) {
    preloader.setKey('open', true);
    animate('[data-preloader]', { opacity: 0, duration: 800, ease: 'inOutCubic' }); // anime.js v4
  }
});
setTimeout(() => preloader.setKey('progress', 1), 20000); // never trap the user
```

## 11. Hardening checklist
- [ ] `prefers-reduced-motion`: damping off, tape noise/tracking/hum off, no auto camera drift
- [ ] Phone preset: different camera start/pitch/fov, DPR ≤ 1.5, fewer particles, no splats on home
- [ ] `renderer.onDeviceLost` → reload once, then fall back to WebGL, then to a static page
- [ ] Pause the loop when the tab is hidden; cap dt; dispose scenes on route change
- [ ] Assets: GLB with Draco/meshopt, KTX2 textures, AVIF images, preloaded woff2 fonts
- [ ] Keyboard and skip-link access to all content; the canvas is `aria-hidden`
