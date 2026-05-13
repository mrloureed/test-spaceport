<script lang="ts">
  import { T, useTask } from '@threlte/core';
  import * as THREE from 'three';

  export type Variant = 'warp' | 'side';

  interface Props {
    headX?: number; // normalized -1..1 (left/right from center)
    headY?: number; // normalized -1..1 (up/down from center)
    variant?: Variant;
  }

  let { headX = 0, headY = 0, variant = 'warp' }: Props = $props();

  const STAR_COUNT = 2500;
  const DEPTH = 800;
  const SPEED = 120;       // warp: units/sec toward camera
  const SIDE_SPEED = 220;  // side: units/sec streaming across

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

  const posArr = new Float32Array(STAR_COUNT * 3);

  // Warp: stars in a cylinder rushing toward camera
  function spawnWarp(i: number, maxZ: number) {
    const r = 8 + Math.random() * 180;
    const theta = Math.random() * Math.PI * 2;
    posArr[i * 3]     = r * Math.cos(theta);
    posArr[i * 3 + 1] = r * Math.sin(theta);
    posArr[i * 3 + 2] = -(Math.random() * maxZ);
  }

  // Side: stars in a wide plane at various depths, streaming left→right
  // Z < 0 = depth from window glass. Perspective makes near stars (small |Z|)
  // fly across the screen much faster than distant ones.
  function spawnSide(i: number, randomX = true) {
    posArr[i * 3]     = randomX ? (Math.random() * 2 - 1) * 600 : 650;
    posArr[i * 3 + 1] = (Math.random() * 2 - 1) * 160;
    posArr[i * 3 + 2] = -(40 + Math.random() * 320);
  }

  for (let i = 0; i < STAR_COUNT; i++) {
    if (variant === 'side') spawnSide(i);
    else spawnWarp(i, DEPTH);
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
    if (variant === 'side') {
      // Stars stream in -X direction (past the side window)
      for (let i = 0; i < STAR_COUNT; i++) {
        posArr[i * 3] -= SIDE_SPEED * delta;
        if (posArr[i * 3] < -650) spawnSide(i, false); // recycle to right edge
      }
      if (cam) {
        // Head up/down → translate camera Y (look higher/lower through window)
        cam.position.x = 0;
        cam.position.z = 0;
        cam.position.y += (-headY * 45 - cam.position.y) * 0.08;
        // Head left/right → pan to look toward front/back of ship
        cam.rotation.x += (-headY * 0.06 - cam.rotation.x) * 0.08;
        cam.rotation.y += ( headX * 0.35 - cam.rotation.y) * 0.08;
      }
    } else {
      // Warp: stars rush toward camera along Z
      for (let i = 0; i < STAR_COUNT; i++) {
        posArr[i * 3 + 2] += SPEED * delta;
        if (posArr[i * 3 + 2] > 8) spawnWarp(i, DEPTH);
      }
      if (cam) {
        const targetX =  headX * 50;
        const targetY = -headY * 35;
        cam.position.x += (targetX - cam.position.x) * 0.08;
        cam.position.y += (targetY - cam.position.y) * 0.08;
        cam.rotation.y += ( headX * 0.12 - cam.rotation.y) * 0.08;
        cam.rotation.x += (-headY * 0.08 - cam.rotation.x) * 0.08;
      }
    }
    (geometry.attributes.position as THREE.BufferAttribute).needsUpdate = true;
  });
</script>

<T.PerspectiveCamera bind:ref={cam} makeDefault fov={90} near={0.1} far={2000} />
<T.Points {geometry} {material} />
