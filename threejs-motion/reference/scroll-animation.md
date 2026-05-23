# Scroll-driven 3D motion

Three approaches, in rough order of how much control you give up.

## 1. drei `ScrollControls` — fastest, R3F-native

Best when the scrollable content lives *inside* the canvas or is tightly coupled to
it. drei syncs a scroll offset you can read every frame.

```tsx
'use client';
import { Canvas } from '@react-three/fiber';
import { ScrollControls, Scroll, useScroll } from '@react-three/drei';
import { useFrame } from '@react-three/fiber';
import { useRef } from 'react';

function Model() {
  const ref = useRef();
  const scroll = useScroll();
  useFrame(() => {
    // scroll.offset is 0..1 across the whole scroll range
    ref.current.rotation.y = scroll.offset * Math.PI * 2;
  });
  return (
    <mesh ref={ref}>
      <torusKnotGeometry />
      <meshStandardMaterial />
    </mesh>
  );
}

export default function Scene() {
  return (
    <Canvas>
      <ambientLight />
      <ScrollControls pages={3} damping={0.2}>
        <Model />
        <Scroll html>{/* normal HTML that scrolls in sync */}</Scroll>
      </ScrollControls>
    </Canvas>
  );
}
```

`damping` gives inertia for free. `pages` sets scroll length as a multiple of
viewport height.

## 2. Lenis + GSAP ScrollTrigger — most control

Best when the page is a normal scrolling document and the 3D scene reacts to it.
This is the common choice for marketing/portfolio sites with mixed DOM + 3D.

- **Lenis** provides smooth scroll.
- **GSAP ScrollTrigger** maps scroll position to tweens.
- The 3D scene reads values that GSAP writes.

Key rule: **GSAP must not fight the render loop.** Let GSAP tween a plain object
or a ref, and read that value inside `useFrame`. Do not tween three.js objects
directly while also mutating them in `useFrame` — pick one writer per property.

```tsx
// pattern only
const progress = useRef({ v: 0 });
useEffect(() => {
  const st = ScrollTrigger.create({
    trigger: '#section',
    start: 'top top',
    end: 'bottom bottom',
    scrub: true,
    onUpdate: (self) => { progress.current.v = self.progress; },
  });
  return () => st.kill();          // cleanup is mandatory
}, []);

useFrame(() => {
  mesh.current.position.y = progress.current.v * 10;
});
```

## 3. Raw scroll listener — only for something trivial

A passive `scroll` listener writing into a ref. Fine for one cheap effect; do not
build a whole scrollytelling page this way — no smoothing, jank on momentum
scrolling, and you reinvent ScrollTrigger badly.

## Cross-cutting rules

- **Never drive animation off React state per frame.** Scroll fires far faster
  than you want renders. Write to a ref; read it in `useFrame`.
- **Always lerp toward the target**, don't snap to it: `current += (target - current) * 0.1`. Raw scroll values are jittery.
- **Kill ScrollTriggers / remove listeners on unmount.** In Next.js dev, Strict
  Mode double-mounts; un-cleaned triggers stack up and multiply the effect.
- **Respect `prefers-reduced-motion`.** Heavy scroll-jacking is an accessibility
  problem — provide a reduced or static path.
- **Decouple scroll smoothing from the render loop.** Lenis has its own RAF; let
  R3F keep its own. Don't nest one inside the other.
