# Performance, the render loop, and cleanup

## Frame budget

60fps means **16.6ms per frame** for everything — JS, render, browser compositing.
Treat ~10ms of JS as the working ceiling. If a scene drops frames, profile before
guessing; the bottleneck is usually draw calls or overdraw, not "three.js being slow".

## The render loop

- **R3F:** `useFrame((state, delta) => { ... })`. Multiply motion by `delta` so it
  is framerate-independent. Never `setState` inside it.
- **Vanilla:** a single `requestAnimationFrame` loop calling `renderer.render`.
  Use a `THREE.Clock` for delta.

`useFrame` runs outside React's render — mutating `mesh.current.rotation` there is
correct and cheap. Mutating state is the trap.

## Draw calls — the usual bottleneck

- **Instancing:** many copies of one mesh → `InstancedMesh` (vanilla) or
  `<Instances>`/`<Instance>` (drei). Hundreds of objects collapse to one draw call.
- **Reuse geometries and materials.** Creating a new `MeshStandardMaterial` per
  object multiplies memory and draw cost. Define once, share the reference.
- **Merge static geometry** that never moves independently.

## Asset weight

- Models: prefer **GLB**, run through **Draco** or **meshopt** compression.
  `gltfjsx` converts a GLB into a typed JSX component.
- Textures: **KTX2 / Basis** compressed, power-of-two sizes, mipmaps on. A 4K PNG
  texture is often the real reason a scene is slow to load.
- `drei` `<Preload all />` warms the GPU so the first interaction isn't a stutter.

## Disposal — the silent memory leak

three.js does **not** garbage-collect GPU resources. Anything created imperatively
must be released:

```js
geometry.dispose();
material.dispose();
texture.dispose();
renderTarget.dispose();
renderer.dispose();     // vanilla, on teardown
```

- **R3F:** objects declared as JSX are disposed automatically when they unmount.
  Anything you `new` yourself inside an effect is your responsibility — dispose it
  in the effect's cleanup return.
- **Vanilla:** every `new` is a future `dispose`. Walk the scene graph on teardown.
- A leaking SPA route eventually exhausts GPU memory and the context is lost
  (black canvas, `WebGL context lost` in console).

## WebGPU vs WebGL

WebGPU is production-ready in recent three.js and broadly supported, but R3F's
WebGPU support is still maturing. **Default to the WebGL2 renderer** unless there's
a specific reason to switch; three.js falls back to WebGL2 automatically anyway.

## Common bugs checklist

- **Black screen:** no light, camera inside/behind geometry, or canvas parent has
  no height.
- **Everything flat/unlit:** using `MeshBasicMaterial` (ignores light) when you
  meant `MeshStandardMaterial`.
- **Z-fighting (flickering surfaces):** two faces coplanar, or camera near/far
  range too wide — tighten `near`/`far`.
- **Blurry on retina:** clamp `dpr` — in R3F, `<Canvas dpr={[1, 2]} />`.
- **Animation speeds up on fast monitors:** motion not multiplied by `delta`.
- **Memory climbs on route changes:** missing `.dispose()` calls.
