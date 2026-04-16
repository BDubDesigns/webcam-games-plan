# Lesson 11 — Answer Sheet: Cleaning Mechanics — Motion Erases Dirt

## The Implementation

### File: `src/game/cleaningConfig.ts`

```ts
/**
 * cleaningConfig.ts — Tunable constants for the cleaning mechanic.
 *
 * These values control how responsive, aggressive, and satisfying the
 * cleaning feels. Adjust them during playtesting.
 *
 * Principle: All magic numbers live HERE, not scattered across components.
 * This makes tuning a one-file operation, not a codebase-wide search.
 */

/** Minimum cleaning radius in pixels (for low-intensity motion). */
export const MIN_CLEAN_RADIUS = 15;

/** Maximum cleaning radius in pixels (for high-intensity motion). */
export const MAX_CLEAN_RADIUS = 60;

/**
 * Minimum motion intensity (0–1) required to trigger cleaning.
 * Below this threshold, motion is ignored (filters out noise/breathing).
 * 0.1 = 10% of pixels in a grid cell must change to trigger cleaning.
 */
export const MOTION_THRESHOLD = 0.1;

/**
 * How often (in frames) to check dirt progress for the win condition.
 * 30 frames at 60fps ≈ every 0.5 seconds.
 * Higher = less CPU cost, lower = more responsive progress display.
 */
export const PROGRESS_CHECK_INTERVAL = 30;

/**
 * Radius of the visual "shine" feedback effect at cleaning locations.
 * Slightly larger than the cleaning radius for a glow effect.
 */
export const SHINE_RADIUS_MULTIPLIER = 1.3;

/**
 * Opacity of the shine feedback effect (0–1).
 * Scales with motion intensity: actual opacity = BASE * intensity.
 */
export const SHINE_BASE_OPACITY = 0.3;
```

### Updated `src/components/GameCanvas.tsx`

```tsx
import { useCallback, useRef } from "react";
import { useAnimationFrame } from "../hooks/useAnimationFrame";
import { useMotionDetection } from "../hooks/useMotionDetection";
import { DirtLayer, DIRT_WIN_THRESHOLD } from "../game/DirtLayer";
import {
  MIN_CLEAN_RADIUS,
  MAX_CLEAN_RADIUS,
  MOTION_THRESHOLD,
  PROGRESS_CHECK_INTERVAL,
  SHINE_RADIUS_MULTIPLIER,
  SHINE_BASE_OPACITY,
} from "../game/cleaningConfig";
import type { MotionRegion } from "../types/motion";

interface GameCanvasProps {
  videoRef: React.RefObject<HTMLVideoElement | null>;
  isReady: boolean;
}

const CANVAS_WIDTH = 640;
const CANVAS_HEIGHT = 480;

/**
 * GameCanvas — Full cleaning mechanic with mirrored video feed.
 *
 * Rendering order each frame:
 *   1. Clear canvas
 *   2. Draw mirrored video feed (so player sees themselves naturally)
 *   3. Detect motion → clean dirt at mirrored positions
 *   4. Draw shine effects at cleaning locations
 *   5. Draw dirt overlay
 *   6. Draw progress HUD
 */
export function GameCanvas({
  videoRef,
  isReady,
}: GameCanvasProps): React.JSX.Element {
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const ctxRef = useRef<CanvasRenderingContext2D | null>(null);
  const dirtLayerRef = useRef<DirtLayer | null>(null);
  const frameCountRef = useRef<number>(0);
  const dirtRemainingRef = useRef<number>(1);

  const { detectMotion } = useMotionDetection(videoRef);

  const getDirtLayer = useCallback((): DirtLayer => {
    if (!dirtLayerRef.current) {
      const layer = new DirtLayer(CANVAS_WIDTH, CANVAS_HEIGHT);
      layer.generate();
      dirtLayerRef.current = layer;
    }
    return dirtLayerRef.current;
  }, []);

  useAnimationFrame(() => {
    if (!ctxRef.current) {
      ctxRef.current = canvasRef.current?.getContext("2d") ?? null;
    }

    const ctx = ctxRef.current;
    const video = videoRef.current;
    if (!ctx || !video) return;

    const dirtLayer = getDirtLayer();

    // === Step 1: Clear ===
    ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

    // === Step 2: Draw mirrored video ===
    // save() → scale(-1, 1) → translate(-width, 0) → drawImage → restore()
    // This flips the video horizontally so the player sees a mirror image.
    // Without this, waving your right hand moves the left hand on screen.
    ctx.save();
    ctx.scale(-1, 1);
    ctx.translate(-CANVAS_WIDTH, 0);
    ctx.drawImage(video, 0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
    ctx.restore();

    // === Step 3: Detect motion and clean dirt ===
    const motionResult = detectMotion();

    // Collect cleaning positions for the shine effect (step 4).
    const cleaningPoints: Array<{
      x: number;
      y: number;
      radius: number;
      intensity: number;
    }> = [];

    for (const region of motionResult.regions) {
      // Skip regions below the motion threshold (noise filter).
      if (region.intensity < MOTION_THRESHOLD) continue;

      // Map motion coordinates to canvas space.
      // IMPORTANT: Mirror the X coordinate to match the mirrored video.
      // Raw motion x=0.8 (right side of camera) → mirrored x=0.2 (left of display).
      const mirroredX = 1 - (region.x + region.width);
      const centerX = (mirroredX + region.width / 2) * CANVAS_WIDTH;
      const centerY = (region.y + region.height / 2) * CANVAS_HEIGHT;

      // Map intensity to cleaning radius.
      // Higher intensity = larger cleaning area.
      const radius =
        MIN_CLEAN_RADIUS +
        region.intensity * (MAX_CLEAN_RADIUS - MIN_CLEAN_RADIUS);

      // Clean the dirt at this position.
      dirtLayer.clean(centerX, centerY, radius);

      // Remember this point for the shine effect.
      cleaningPoints.push({
        x: centerX,
        y: centerY,
        radius,
        intensity: region.intensity,
      });
    }

    // === Step 4: Draw shine effects at cleaning locations ===
    // A semi-transparent white circle that gives visual feedback
    // indicating where the player is actively cleaning.
    for (const point of cleaningPoints) {
      drawShineEffect(ctx, point.x, point.y, point.radius, point.intensity);
    }

    // === Step 5: Draw dirt overlay ===
    dirtLayer.draw(ctx);

    // === Step 6: Check progress and draw HUD ===
    frameCountRef.current++;
    if (frameCountRef.current % PROGRESS_CHECK_INTERVAL === 0) {
      dirtRemainingRef.current = dirtLayer.getDirtRemaining();
    }
    drawProgressHUD(ctx, dirtRemainingRef.current);
  }, isReady);

  return (
    <canvas
      ref={(node) => {
        canvasRef.current = node;
        if (node && !isReady) {
          const ctx = node.getContext("2d");
          if (ctx) drawPlaceholder(ctx);
        }
      }}
      width={CANVAS_WIDTH}
      height={CANVAS_HEIGHT}
      className="rounded-lg border border-slate-700"
    />
  );
}

// =============================================================================
// Drawing Helpers
// =============================================================================

/**
 * Draws a subtle white glow at the cleaning location.
 * Gives the player visual feedback that their motion is having an effect.
 */
function drawShineEffect(
  ctx: CanvasRenderingContext2D,
  x: number,
  y: number,
  radius: number,
  intensity: number,
): void {
  const shineRadius = radius * SHINE_RADIUS_MULTIPLIER;
  const opacity = SHINE_BASE_OPACITY * intensity;

  const gradient = ctx.createRadialGradient(x, y, 0, x, y, shineRadius);
  gradient.addColorStop(0, `rgba(255, 255, 255, ${opacity})`);
  gradient.addColorStop(1, "rgba(255, 255, 255, 0)");

  ctx.fillStyle = gradient;
  ctx.beginPath();
  ctx.arc(x, y, shineRadius, 0, Math.PI * 2);
  ctx.fill();
}

/**
 * Draws the progress display in the top-right corner.
 * Shows "XX% Clean" with a dark background for readability.
 */
function drawProgressHUD(
  ctx: CanvasRenderingContext2D,
  dirtRemaining: number,
): void {
  const percentClean = Math.round((1 - dirtRemaining) * 100);
  const text = `${percentClean}% Clean`;

  ctx.font = "bold 20px sans-serif";
  const metrics = ctx.measureText(text);
  const textWidth = metrics.width;
  const padding = 10;
  const x = CANVAS_WIDTH - textWidth - padding * 2 - 10;
  const y = 10;

  // Dark background rectangle for readability against any video content.
  ctx.fillStyle = "rgba(0, 0, 0, 0.6)";
  ctx.fillRect(x, y, textWidth + padding * 2, 32);

  // Progress text
  ctx.fillStyle =
    dirtRemaining < DIRT_WIN_THRESHOLD ? "#4ade80" : "#ffffff";
  ctx.textAlign = "left";
  ctx.textBaseline = "middle";
  ctx.fillText(text, x + padding, y + 16);

  // Reset text properties
  ctx.textAlign = "start";
  ctx.textBaseline = "alphabetic";
}

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

const CANVAS_WIDTH_FOR_HUD = 640; // Used by drawProgressHUD closure
```

Wait — there's a scoping issue. Let me clean that up. The `CANVAS_WIDTH` constant is available in the module scope, so `drawProgressHUD` can access it directly. Remove the `CANVAS_WIDTH_FOR_HUD` line. The code above uses the module-level `CANVAS_WIDTH` constant correctly in `drawProgressHUD`.

**Delete the last line** (`const CANVAS_WIDTH_FOR_HUD = 640;`) — it was an error. The helper functions access `CANVAS_WIDTH` from the module scope.

---

## Executive Summary

**Jack Donaghy:**

> We have now completed what I call the *interaction refinement phase*. In product development, this is the difference between a prototype and a product. Let me explain the value we've added.

> 1. **The mirror effect is user experience alignment.** When a child waves their right hand and the on-screen avatar's left hand moves, they're confused. The mirror transform eliminates this cognitive dissonance. In marketing, we call this *reducing friction in the customer journey*. The user doesn't have to think about orientation — they just move and clean. That's intuitive design. That's what Apple does. That's what we do.

> 2. **Configurable constants are operational agility.** All tuning parameters live in one file (`cleaningConfig.ts`). During playtesting — and there WILL be playtesting, because we are professionals — the designer adjusts `MIN_CLEAN_RADIUS` from 15 to 20 and instantly the game feels different. No code changes. No refactoring. No meetings. Just data-driven iteration. This is the engineering equivalent of a factory where you can change the product specifications by adjusting a dial on the control panel.

> 3. **The shine effect is perceived value.** The white glow at cleaning locations costs us approximately 0.1ms per frame in rendering time. But it adds an *experiential dimension* — the player SEES their action having an effect. It's tactile. It's satisfying. In consumer products, this is the "premium feel" — the click of a Mercedes door, the weight of an iPhone. Cost: minimal. Perceived value: massive.

> 4. **The progress HUD is transparency.** "43% Clean" tells the player exactly where they stand. No ambiguity. No frustration. This is the same principle as a progress bar in a file download — humans need to see progress to stay motivated. In Six Sigma, this is *Visual Management* — making the process visible to all stakeholders.

> We're approaching the finish line. The game is playable, responsive, and visually polished. Two lessons remain: the win state and the final assembly. We're in the home stretch, and we're running at Six Sigma quality levels. Maintain discipline.
