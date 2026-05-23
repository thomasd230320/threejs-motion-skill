---
name: threejs-motion
description: "Use this skill whenever the user is building 3D graphics, WebGL scenes, or scroll-driven motion on the web with three.js or React Three Fiber (R3F). Triggers include: any mention of three.js, react-three-fiber, @react-three/fiber, drei, GLTF/GLB models, WebGL/WebGPU scenes, 3D hero sections, scroll animations tied to 3D, camera rigs, or shader/material work for the web. Especially use when the project is a Next.js app and the 3D scene must coexist with SSR. Covers project setup, the Next.js 'use client'/SSR boundary, scroll-linked animation patterns, the render loop, performance budgets, and disposal/cleanup. Do NOT use for non-3D CSS animation, 2D canvas work, or game engines like Unity/Unreal."
---

# three.js & React Three Fiber — motion and scroll work

## Overview

This skill covers building 3D scenes on the web, with a bias toward **scroll-driven
animation inside Next.js**. It assumes R3F as the primary path and vanilla three.js
as the fallback. Read the relevant reference file before writing code — the pitfalls
sections exist because these mistakes are easy to make and expensive to debug.

## Pinned versions (verify before relying on them)

As of mid-2026 the current major versions are:

| Package | Version | Notes |
|---|---|---|
| `three` | r0.17x | WebGPU renderer is production-ready since r171; WebGL2 is still the safe default |
| `@react-three/fiber` | v9.x | **v9 pairs with React 19**; v8 pairs with React 18 — mismatching them breaks the reconciler |
| `@react-three/drei` | v10.x | helper components; large, so import named members only |
| `@react-three/postprocessing` | latest | effects; gate behind a quality check on low-end devices |

Always run `npm view <pkg> version` to confirm — do not hard-code these numbers into
generated `package.json` files without checking.

## Decision: R3F or vanilla?

- **Project already uses React / Next.js** → use R3F. Do not hand-roll a renderer.
- **No React in the project** → vanilla three.js. Adding React just for 3D is a real
  bundle cost (~40KB+ on top of three.js's ~155KB gzipped) and not worth it.
- **A single isolated widget on an otherwise non-React page** → vanilla.

## Quick reference

| Task | Where to look |
|---|---|
| Next.js setup, SSR boundary, the `Canvas` client component | `reference/nextjs-setup.md` |
| Scroll-linked motion (drei ScrollControls vs. Lenis+GSAP vs. raw) | `reference/scroll-animation.md` |
| Render loop, performance budget, disposal, common bugs | `reference/performance.md` |
| Vanilla three.js skeleton (no React) | `reference/vanilla-skeleton.md` |

## Non-negotiable rules

1. **Never put `<Canvas>` in a Server Component.** It must live in a file with
   `'use client'` at the top. Importing three.js into server-rendered code throws
   `window is not defined`. See `reference/nextjs-setup.md`.
2. **Animate inside the render loop, not React state.** Per-frame updates go in
   `useFrame` (R3F) or the `requestAnimationFrame` callback (vanilla). Calling
   `setState` every frame causes a re-render storm and tanks performance.
3. **Dispose what you create.** Geometries, materials, textures, and render targets
   created imperatively are not garbage-collected. Vanilla code must call `.dispose()`.
   R3F disposes JSX-declared objects automatically — manual ones still need cleanup.
4. **Match `@react-three/fiber` major to the React major.** v9↔React 19, v8↔React 18.
5. **Budget the frame.** Target 16.6ms total. Prefer instancing over many meshes,
   reuse geometries/materials, and gate post-processing on device capability.

## Workflow

1. Confirm React-or-not, and the framework version, before scaffolding.
2. Read the reference file matching the task.
3. Scaffold the scene; keep the 3D scene component isolated and lazy-loaded.
4. Wire animation through the render loop.
5. Verify cleanup on unmount and check the frame budget before declaring done.
