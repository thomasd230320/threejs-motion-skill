# Vanilla three.js skeleton

Use this only when the project has no React. Otherwise prefer R3F.

## Minimal scene with a clean render loop and teardown

```js
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

function createScene(container) {
  const scene = new THREE.Scene();

  const camera = new THREE.PerspectiveCamera(
    50,
    container.clientWidth / container.clientHeight,
    0.1,
    100
  );
  camera.position.set(0, 0, 5);

  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(container.clientWidth, container.clientHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // clamp dpr
  container.appendChild(renderer.domElement);

  scene.add(new THREE.AmbientLight(0xffffff, 0.6));
  const dir = new THREE.DirectionalLight(0xffffff, 1);
  dir.position.set(5, 5, 5);
  scene.add(dir);

  const geometry = new THREE.BoxGeometry(1, 1, 1);
  const material = new THREE.MeshStandardMaterial({ color: 0xff8800 });
  const mesh = new THREE.Mesh(geometry, material);
  scene.add(mesh);

  const controls = new OrbitControls(camera, renderer.domElement);
  const clock = new THREE.Clock();
  let frameId;

  function animate() {
    frameId = requestAnimationFrame(animate);
    const delta = clock.getDelta();
    mesh.rotation.y += delta;          // delta = framerate-independent
    controls.update();
    renderer.render(scene, camera);
  }
  animate();

  function onResize() {
    const { clientWidth: w, clientHeight: h } = container;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  }
  window.addEventListener('resize', onResize);

  // Teardown — call this when the scene is removed.
  return function dispose() {
    cancelAnimationFrame(frameId);
    window.removeEventListener('resize', onResize);
    controls.dispose();
    geometry.dispose();
    material.dispose();
    renderer.dispose();
    renderer.domElement.remove();
  };
}
```

## Notes

- **Import addons from `three/addons/...`** (the modern path), not the older
  `three/examples/jsm/...`. Both resolve to the same files; stay consistent.
- With a CDN / no bundler, use an **import map** so `three` and `three/addons/`
  resolve. Without one, bare specifiers fail in the browser.
- Every `new` of a geometry, material, texture, or renderer has a matching
  `.dispose()` in the teardown function. This is the #1 source of leaks.
- Clamp `setPixelRatio` to 2 — uncapped retina ratios quadruple the pixel count
  for no visible gain.
- Use `clock.getDelta()` for motion so animation speed is independent of the
  monitor's refresh rate.

## Scroll, vanilla-style

Keep a `targetScroll` updated by a passive `scroll` listener, and lerp the actual
value toward it inside the animate loop:

```js
let target = 0, current = 0;
window.addEventListener('scroll', () => {
  target = window.scrollY / (document.body.scrollHeight - innerHeight);
}, { passive: true });

// inside animate():
current += (target - current) * 0.1;   // smoothing
mesh.rotation.x = current * Math.PI * 2;
```

For anything more than one effect, use Lenis + GSAP ScrollTrigger instead of
hand-rolling — see `scroll-animation.md`.
