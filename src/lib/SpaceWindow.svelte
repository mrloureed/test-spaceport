<script lang="ts">
  import { onDestroy } from 'svelte';
  import { Canvas } from '@threlte/core';
  import Starfield, { type Variant } from './Starfield.svelte';
  import { FaceDetector, FilesetResolver } from '@mediapipe/tasks-vision';

  interface Props {
    onclose?: () => void;
  }

  let { onclose }: Props = $props();

  const VARIANTS: Variant[] = ['warp', 'side'];
  const rawView = new URLSearchParams(window.location.search).get('view') ?? 'warp';
  const variant: Variant = (VARIANTS as string[]).includes(rawView) ? (rawView as Variant) : 'warp';

  let headX = $state(0);
  let headY = $state(0);
  let trackingStatus = $state<'idle' | 'loading' | 'active' | 'no-face' | 'error'>('idle');
  let trackingError = $state('');
  let showPreview = $state(false);

  let videoEl: HTMLVideoElement;
  let faceDetector: FaceDetector | null = null;
  let rafId = 0;
  let stream: MediaStream | null = null;

  async function enableTracking() {
    trackingStatus = 'loading';
    try {
      const vision = await FilesetResolver.forVisionTasks(
        'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision/wasm',
      );
      faceDetector = await FaceDetector.createFromOptions(vision, {
        baseOptions: {
          modelAssetPath:
            'https://storage.googleapis.com/mediapipe-models/face_detector/blaze_face_short_range/float16/1/blaze_face_short_range.tflite',
          delegate: 'GPU',
        },
        runningMode: 'VIDEO',
        minDetectionConfidence: 0.5,
      });

      stream = await navigator.mediaDevices.getUserMedia({
        video: { facingMode: 'user', width: { ideal: 640 }, height: { ideal: 480 } },
      });
      videoEl.srcObject = stream;
      await videoEl.play();

      trackingStatus = 'no-face';
      rafId = requestAnimationFrame(detect);
    } catch (e) {
      trackingStatus = 'error';
      trackingError = String(e);
    }
  }

  function detect() {
    if (faceDetector && videoEl?.readyState >= 2) {
      try {
        const result = faceDetector.detectForVideo(videoEl, performance.now());
        if (result.detections.length > 0) {
          const bb = result.detections[0].boundingBox!;
          const cx = bb.originX + bb.width / 2;
          const cy = bb.originY + bb.height / 2;
          headX = (cx / videoEl.videoWidth - 0.5) * 2;
          headY = (cy / videoEl.videoHeight - 0.5) * 2;
          trackingStatus = 'active';
        } else {
          trackingStatus = 'no-face';
        }
      } catch {
        // ignore transient errors during video seek/pause
      }
    }
    rafId = requestAnimationFrame(detect);
  }

  onDestroy(() => {
    cancelAnimationFrame(rafId);
    stream?.getTracks().forEach((t) => t.stop());
    faceDetector?.close();
  });
</script>

<div class="space-window">
  <!-- Hidden video for webcam input; always in DOM so bind:this resolves early -->
  <!-- eslint-disable-next-line svelte/valid-compile -->
  <video bind:this={videoEl} class="webcam" class:visible={showPreview} muted playsinline></video>

  <Canvas>
    <Starfield {headX} {headY} {variant} />
  </Canvas>

  <div class="hud">
    <div class="hud-left">
      {#if trackingStatus === 'idle'}
        <button class="hud-btn" onclick={enableTracking}>Enable face tracking</button>
      {:else if trackingStatus === 'loading'}
        <span class="hud-text dim">Loading model…</span>
      {:else if trackingStatus === 'active'}
        <span class="hud-text">
          <span class="dot green"></span>
          Tracking · x {headX.toFixed(2)} y {headY.toFixed(2)}
        </span>
        <button class="hud-btn small" onclick={() => (showPreview = !showPreview)}>
          {showPreview ? 'Hide cam' : 'Show cam'}
        </button>
      {:else if trackingStatus === 'no-face'}
        <span class="hud-text">
          <span class="dot yellow"></span>
          Camera active — no face detected
        </span>
        <button class="hud-btn small" onclick={() => (showPreview = !showPreview)}>
          {showPreview ? 'Hide cam' : 'Show cam'}
        </button>
      {:else}
        <span class="hud-text error">Tracking unavailable: {trackingError}</span>
      {/if}
    </div>

    <div class="hud-right">
      <span class="hud-text dim" style="margin-right:0.75rem">{variant}</span>
      <button class="hud-btn close" onclick={onclose}>✕ Close</button>
    </div>
  </div>
</div>

<style>
  .space-window {
    position: fixed;
    inset: 0;
    z-index: 100;
    background: #000;
    overflow: hidden;
  }

  /* Threlte renders a canvas as a direct child of Canvas's div */
  .space-window :global(canvas) {
    display: block;
    width: 100% !important;
    height: 100% !important;
  }

  .webcam {
    position: absolute;
    bottom: 3.5rem;
    right: 1rem;
    width: 180px;
    border-radius: 0.5rem;
    border: 1px solid #334155;
    transform: scaleX(-1); /* mirror for natural feel */
    display: none;
    z-index: 10;
  }

  .webcam.visible {
    display: block;
  }

  .hud {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0.5rem 1rem;
    background: rgba(0, 0, 0, 0.55);
    backdrop-filter: blur(6px);
    border-top: 1px solid rgba(255, 255, 255, 0.07);
    z-index: 10;
  }

  .hud-left {
    display: flex;
    align-items: center;
    gap: 0.75rem;
  }

  .hud-text {
    font-size: 0.78rem;
    color: #94a3b8;
    display: flex;
    align-items: center;
    gap: 0.4rem;
    font-family: monospace;
  }

  .hud-text.dim {
    color: #475569;
  }

  .hud-text.error {
    color: #f87171;
  }

  .dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    flex-shrink: 0;
  }

  .dot.green {
    background: #22c55e;
    box-shadow: 0 0 5px #22c55e;
  }

  .dot.yellow {
    background: #eab308;
  }

  .hud-btn {
    background: rgba(255, 255, 255, 0.08);
    border: 1px solid rgba(255, 255, 255, 0.15);
    border-radius: 0.375rem;
    color: #e2e8f0;
    font-size: 0.78rem;
    padding: 0.3rem 0.75rem;
    cursor: pointer;
    transition: background 0.15s;
  }

  .hud-btn:hover {
    background: rgba(255, 255, 255, 0.15);
  }

  .hud-btn.small {
    font-size: 0.7rem;
    padding: 0.2rem 0.5rem;
  }

  .hud-btn.close {
    color: #94a3b8;
  }
</style>
