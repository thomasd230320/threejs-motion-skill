# Next.js + R3F setup and the SSR boundary

## The core problem

three.js touches `window`, `document`, and WebGL. Next.js renders components on the
server by default, where none of those exist. The whole game is keeping 3D code on
the client side of the boundary.

## Install

```bash
npm install three @react-three/fiber @react-three/drei
npm install -D @types/three   # if using TypeScript
```

Confirm the fiber major matches your React major: fiber v9 ↔ React 19, v8 ↔ React 18.
A mismatch produces obscure reconciler errors, not a clean message.

## Project layout that works

```
app/
  page.tsx              <- Server Component (default). NO three.js imports here.
  components/
    Scene.tsx            <- 'use client'. Holds <Canvas> and the 3D tree.
    SceneLoader.tsx       <- 'use client'. dynamic() wrapper, ssr: false.
```

## The Canvas component — must be a client component

```tsx
// app/components/Scene.tsx
'use client';

import { Canvas } from '@react-three/fiber';
import { OrbitControls } from '@react-three/drei';

export default function Scene() {
  return (
    <Canvas camera={{ position: [0, 0, 5], fov: 50 }}>
      <ambientLight intensity={0.6} />
      <directionalLight position={[5, 5, 5]} />
      <mesh>
        <boxGeometry args={[1, 1, 1]} />
        <meshStandardMaterial color="orange" />
      </mesh>
      <OrbitControls />
    </Canvas>
  );
}
```

## Loading it without SSR

Even though `Scene.tsx` is a client component, Next.js will still try to
server-render it during the initial pass. Wrap it with `dynamic(..., { ssr: false })`
so it only ever runs in the browser:

```tsx
// app/components/SceneLoader.tsx
'use client';

import dynamic from 'next/dynamic';

const Scene = dynamic(() => import('./Scene'), {
  ssr: false,
  loading: () => <div style={{ height: '100vh' }} />, // reserve layout space
});

export default Scene;
```

Then in `app/page.tsx` (a Server Component) just import `SceneLoader`.

## Pitfalls

- **`window is not defined` at build time** — three.js leaked into a Server
  Component. Move it behind `'use client'` and `ssr: false`.
- **Hydration mismatch / flicker** — the `loading` placeholder has different
  dimensions than the canvas. Give the placeholder the same height so layout is
  stable.
- **Canvas has zero height** — `<Canvas>` fills its parent; the parent needs an
  explicit height (`100vh`, a fixed px value, etc.), or nothing shows.
- **Double-mount in dev** — React 19 Strict Mode mounts twice. R3F handles this,
  but any manual `useEffect` setup must have matching cleanup or you leak GL
  contexts.
- **Importing `three/examples/jsm/...`** — that path still works but the modern
  path is `three/addons/...`. Pick one and be consistent; drei usually removes the
  need for either.
