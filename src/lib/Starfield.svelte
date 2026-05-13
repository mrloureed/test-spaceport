<script lang="ts">
  import { T, useTask } from '@threlte/core';
  import * as THREE from 'three';

  interface Props {
    headX?: number; // normalized -1..1 (left/right from center)
    headY?: number; // normalized -1..1 (up/down from center)
  }

  let { headX = 0, headY = 0 }: Props = $props();

  const STAR_COUNT = 2500;
  const DEPTH = 800;
  const SPEED = 120; // units per second

  // Circular dot sprite so stars render as round points, not squares
  function makeCircleTexture(size = 64): THREE.Texture {
    const canvas = document.createElement('canvas');
    canvas.width = size;
    canvas.height = size;
    const ctx = canvas.getContext('2d')!;
    const r = size / 2;
    const gradient = ctx.createRadialGradient(r, r, 0, r, r, r);
    gradient.addColorStop(0, 'rgba(255,255,255,1)');
    gradient.addColorStop(0.4, 'rgba(255,255,255,0.8)');
    gradient.addColorStop(1, 'rgba(255,255,255,0)');
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, size, size);
    return new THREE.CanvasTexture(canvas);
  }

  // Build initial star positions spread through the tunnel
  const posArr = new Float32Array(STAR_COUNT * 3);

  function spawnStar(i: number, maxZ: number) {
    const r = 8 + Math.random() * 180;
    const theta = Math.random() * Math.PI * 2;
    posArr[i * 3] = r * Math.cos(theta);
    posArr[i * 3 + 1] = r * Math.sin(theta);
    posArr[i * 3 + 2] = -(Math.random() * maxZ);
  }

  for (let i = 0; i < STAR_COUNT; i++) {
    spawnStar(i, DEPTH);
  }

  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute('position', new THREE.BufferAttribute(posArr, 3));

  const material = new THREE.PointsMaterial({
    color: 0xffffff,
    size: 2.5,
    sizeAttenuation: true,
    map: makeCircleTexture(),
    transparent: true,
    alphaTest: 0.01,
    depthWrite: false,
  });

  // Camera ref — Threlte sets this when <T.PerspectiveCamera bind:ref> mounts
  let cam: THREE.PerspectiveCamera | undefined = $state();

  useTask((delta) => {
    // Move all stars toward the camera (increase z)
    for (let i = 0; i < STAR_COUNT; i++) {
      posArr[i * 3 + 2] += SPEED * delta;
      if (posArr[i * 3 + 2] > 8) {
        spawnStar(i, DEPTH);
      }
    }
    (geometry.attributes.position as THREE.BufferAttribute).needsUpdate = true;

    // Parallax: head right → view pans right (positive rotation.y)
    if (cam) {
      cam.rotation.x += (headY * 0.4 - cam.rotation.x) * 0.06;
      cam.rotation.y += (headX * 0.4 - cam.rotation.y) * 0.06;
    }
  });
</script>

<T.PerspectiveCamera bind:ref={cam} makeDefault fov={90} near={0.1} far={2000} />
<T.Points {geometry} {material} />
