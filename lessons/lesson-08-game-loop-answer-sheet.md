# Lesson 08 — Answer Sheet: The Game Loop — requestAnimationFrame in React

## The Implementation

### File: `src/hooks/useAnimationFrame.ts`

```ts
import { useEffect, useRef } from "react";

/**
 * The callback signature for the animation frame handler.
 *
 * @param deltaTime - Time in milliseconds since the last frame.
 *   Use this for frame-rate independent animations and logic.
 *   Example: On a 60Hz display, this is ~16.67ms. On a 120Hz display, ~8.33ms.
 */
type FrameCallback = (deltaTime: number) => void;

/**
 * useAnimationFrame — Custom hook for a robust requestAnimationFrame loop.
 *
 * Solves three critical problems:
 *   1. React Strict Mode: Properly cancels the loop on cleanup, preventing
 *      duplicate loops from the mount → unmount → remount cycle.
 *   2. Stale closures: Stores the callback in a ref so the loop always
 *      calls the latest version without restarting.
 *   3. Frame-rate independence: Passes delta time to the callback so game
 *      logic works correctly on 60Hz, 120Hz, and dropped-frame scenarios.
 *
 * @param callback - Function to call on every animation frame.
 * @param isActive - When false, the loop doesn't run. Defaults to true.
 *
 * Usage:
 *   useAnimationFrame((deltaTime) => {
 *     // Update game state using deltaTime
 *     // Draw to canvas
 *   }, isReady);
 */
export function useAnimationFrame(
  callback: FrameCallback,
  isActive: boolean = true,
): void {
  // Store the latest callback in a ref.
  // This is the "callback ref" pattern:
  //   - The rAF loop reads from this ref on every frame.
  //   - When the parent component re-renders with a new callback, we update
  //     the ref — but the loop itself doesn't restart.
  //   - This avoids both stale closures AND loop restarts.
  const callbackRef = useRef<FrameCallback>(callback);

  // Store the previous frame's timestamp for delta calculation.
  // Using a ref (not state) because this is loop-internal bookkeeping
  // that should NEVER trigger a React re-render.
  const previousTimeRef = useRef<number | null>(null);

  // Store the current animation frame ID for cleanup.
  const frameIdRef = useRef<number>(0);

  // Keep the callback ref up to date on every render.
  // This runs synchronously during render (before effects),
  // ensuring the ref is current before the next frame fires.
  useEffect(() => {
    callbackRef.current = callback;
  });

  // The main animation loop effect.
  // Dependencies: [isActive] — restarts the loop when activation changes.
  useEffect(() => {
    // Don't start the loop if inactive.
    if (!isActive) return;

    /**
     * The animation frame handler.
     * Called by the browser before each repaint (~60fps on most displays).
     *
     * @param timestamp - DOMHighResTimeStamp in milliseconds since page load.
     *   This is provided by requestAnimationFrame, NOT by us.
     */
    function tick(timestamp: DOMHighResTimeStamp): void {
      // Calculate delta time.
      // On the first frame, previousTimeRef.current is null, so delta is 0.
      // This prevents a huge delta spike on the first frame (which could be
      // hundreds of ms after the effect runs).
      const deltaTime =
        previousTimeRef.current !== null
          ? timestamp - previousTimeRef.current
          : 0;

      previousTimeRef.current = timestamp;

      // Call the latest callback from the ref.
      // This always gets the most recent version, even if the parent
      // component has re-rendered with a new callback closure.
      callbackRef.current(deltaTime);

      // Schedule the next frame.
      // This creates the loop: tick → rAF → tick → rAF → ...
      frameIdRef.current = requestAnimationFrame(tick);
    }

    // Start the loop by requesting the first frame.
    frameIdRef.current = requestAnimationFrame(tick);

    // Cleanup: cancel the pending animation frame.
    // This runs when:
    //   1. The component unmounts.
    //   2. isActive changes from true to false.
    //   3. React Strict Mode unmounts during its stress test.
    // cancelAnimationFrame ensures the next scheduled tick() never fires.
    return () => {
      cancelAnimationFrame(frameIdRef.current);
      // Reset the previous time so the next loop start gets a clean delta.
      previousTimeRef.current = null;
    };
  }, [isActive]);
}
```

### File: `src/components/GameCanvas.tsx` (updated)

```tsx
import { useRef } from "react";
import { useAnimationFrame } from "../hooks/useAnimationFrame";

/**
 * Props for the GameCanvas component.
 */
interface GameCanvasProps {
  videoRef: React.RefObject<HTMLVideoElement | null>;
  isReady: boolean;
}

/** Canvas dimensions in pixels. */
const CANVAS_WIDTH = 640;
const CANVAS_HEIGHT = 480;

/**
 * GameCanvas — Renders a continuous live camera feed on an HTML5 canvas.
 *
 * Uses the useAnimationFrame hook to create a 60fps rendering loop that:
 *   1. Clears the canvas.
 *   2. Draws the current video frame.
 *
 * The loop only runs when isReady is true (camera is active).
 * When the camera isn't ready, a static placeholder is displayed.
 */
export function GameCanvas({
  videoRef,
  isReady,
}: GameCanvasProps): React.JSX.Element {
  const canvasRef = useRef<HTMLCanvasElement | null>(null);

  // The rendering context is obtained once and reused.
  // We store it in a ref to avoid calling getContext("2d") every frame.
  const ctxRef = useRef<CanvasRenderingContext2D | null>(null);

  // Animation frame callback — called ~60 times per second when active.
  // The _deltaTime parameter is unused for now but will be essential for
  // time-based game logic (countdowns, particle physics, etc.).
  useAnimationFrame((_deltaTime: number) => {
    // Lazily initialize the context ref.
    // We do this here instead of in a separate useEffect because the
    // canvas ref is guaranteed to be non-null by the time the animation
    // loop runs (it's only active when isReady is true, which is after mount).
    if (!ctxRef.current) {
      ctxRef.current = canvasRef.current?.getContext("2d") ?? null;
    }

    const ctx = ctxRef.current;
    const video = videoRef.current;
    if (!ctx || !video) return;

    // Step 1: Clear the entire canvas.
    // This removes the previous frame. Without this, we'd draw on top of
    // the old frame. For a full-canvas video draw it's technically invisible,
    // but it's essential when we add the dirt layer and compositing.
    ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

    // Step 2: Draw the current video frame to the canvas.
    // drawImage(video) captures whatever frame the <video> element is
    // currently displaying. Because the video is playing continuously,
    // each call to drawImage gets the latest camera frame.
    ctx.drawImage(video, 0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
  }, isReady);

  return (
    <canvas
      ref={(node) => {
        canvasRef.current = node;
        // Draw placeholder immediately when the canvas mounts but camera isn't ready.
        if (node && !isReady) {
          const ctx = node.getContext("2d");
          if (ctx) {
            drawPlaceholder(ctx);
          }
        }
      }}
      width={CANVAS_WIDTH}
      height={CANVAS_HEIGHT}
      className="rounded-lg border border-slate-700"
    />
  );
}

/**
 * Draws a placeholder screen while waiting for the camera.
 */
function drawPlaceholder(ctx: CanvasRenderingContext2D): void {
  ctx.fillStyle = "#0f172a";
  ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
  ctx.fillStyle = "#94a3b8";
  ctx.font = "24px sans-serif";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillText("Waiting for camera...", CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2);
  ctx.textAlign = "start";
  ctx.textBaseline = "alphabetic";
}
```

### Understanding the hook's ref pattern

```
Render 1: callback = fn_v1 → callbackRef.current = fn_v1
  ↓ rAF fires → reads callbackRef.current → calls fn_v1

Render 2: callback = fn_v2 → callbackRef.current = fn_v2
  ↓ rAF fires → reads callbackRef.current → calls fn_v2 (NOT fn_v1!)

Render 3: callback = fn_v3 → callbackRef.current = fn_v3
  ↓ rAF fires → reads callbackRef.current → calls fn_v3

The loop NEVER restarts. Only the callback updates.
```

This is why the callback is NOT in the `useEffect` dependency array. The effect that runs the loop depends only on `isActive`. The callback is accessed via ref, which is always current.

### Testing the loop in DevTools

Open Chrome DevTools → Performance tab:

1. Click "Record" (⏺️).
2. Let it run for 2-3 seconds.
3. Click "Stop."
4. Look at the "Frames" section. Each frame should be ~16.67ms apart.
5. In the "Main" thread, you should see a repeating pattern of `requestAnimationFrame` callbacks.
6. There should be EXACTLY ONE loop — not two (Strict Mode check).

---

## Executive Summary

**Jack Donaghy:**

> The game loop is the *takt time* of our production facility. Let me explain what we've engineered here, because it's a masterpiece of operational design.

> 1. **`requestAnimationFrame` is demand-driven production.** Unlike `setInterval` — which is push-based ("produce every 16ms whether the customer wants it or not") — rAF is pull-based ("produce one frame right before the display needs it"). This eliminates overproduction (wasted frames the display can't show) and underproduction (jank from misaligned timing). It's the Kanban system of web rendering.

> 2. **The callback ref pattern is a decoupled organizational structure.** The animation loop is the factory floor — it runs continuously and doesn't care about management changes above it. The callback is the production order — it can change every render cycle. By accessing the callback through a ref, the factory floor (loop) doesn't need to stop and restart every time a new production order comes in. It just reads the latest order and executes. This is *operational continuity* — the hallmark of a resilient organization.

> 3. **Delta time is quality normalization.** Different monitors run at different refresh rates. A 60Hz display produces ~16.67ms deltas. A 120Hz display produces ~8.33ms deltas. A tab that was backgrounded might produce a 500ms delta on resume. By passing delta time to the callback, all game logic runs at the same *perceived speed* regardless of hardware. This is *process standardization across variable operating conditions* — exactly what Six Sigma was designed to achieve.

> 4. **Cleanup in `useEffect` is our shutdown protocol.** When the component unmounts — whether from navigation, Strict Mode testing, or a state change — `cancelAnimationFrame` immediately terminates the loop. No zombie callbacks. No orphaned processes. No leaked resources. This is a clean shutdown — the engineering equivalent of a factory that can halt production in under one second with zero work-in-progress waste.

> We now have a 60fps production line that's decoupled from React's render cycle, immune to stale closures, frame-rate independent, and properly lifecycle-managed. This is the engine that powers everything. Treat it with respect.
