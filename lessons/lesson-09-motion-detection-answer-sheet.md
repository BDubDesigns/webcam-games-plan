# Lesson 09 — Answer Sheet: Frame Differencing & Motion Detection

## The Implementation

### File: `src/types/motion.ts`

```ts
/**
 * Represents a region of the canvas where motion was detected.
 *
 * Coordinates are normalized to 0–1 range, making them resolution-independent.
 * The consumer maps these to their canvas size:
 *   canvasX = region.x * canvasWidth
 *   canvasY = region.y * canvasHeight
 */
export interface MotionRegion {
  /** Normalized X position (0 = left edge, 1 = right edge) */
  x: number;
  /** Normalized Y position (0 = top edge, 1 = bottom edge) */
  y: number;
  /** Width of the region as a fraction of the total (e.g., 1/8 for 8 columns) */
  width: number;
  /** Height of the region as a fraction of the total (e.g., 1/6 for 6 rows) */
  height: number;
  /** Motion intensity in this region (0 = no motion, 1 = maximum motion) */
  intensity: number;
}

/**
 * Result of a single frame's motion detection pass.
 */
export interface MotionDetectionResult {
  /** Grid of regions with their motion intensities */
  regions: MotionRegion[];
  /** Overall motion score across the entire frame (0–1) */
  overallMotion: number;
}
```

### File: `src/hooks/useMotionDetection.ts`

```ts
import { useCallback, useRef } from "react";
import type { MotionDetectionResult, MotionRegion } from "../types/motion";

// =============================================================================
// Motion Detection Constants
// =============================================================================

/**
 * Processing resolution — 10% of 640×480.
 * This reduces pixel comparison from 307,200 pixels to 3,072 pixels per frame.
 * The downscale is done by drawImage() which uses hardware-accelerated interpolation.
 */
const DETECTION_WIDTH = 64;
const DETECTION_HEIGHT = 48;

/**
 * Per-pixel RGB difference threshold.
 * Any single-channel (R, G, or B) difference below this value is ignored.
 * This filters out webcam sensor noise, auto-exposure adjustments,
 * and compression artifacts.
 *
 * 30 out of 255 ≈ 12% — a good balance between sensitivity and noise rejection.
 */
const PIXEL_DIFF_THRESHOLD = 30;

/**
 * Grid dimensions for motion region aggregation.
 * The detection canvas is divided into GRID_COLS × GRID_ROWS cells.
 * Each cell reports its own motion intensity independently.
 * 8×6 = 48 cells → enough spatial resolution to locate a moving hand.
 */
const GRID_COLS = 8;
const GRID_ROWS = 6;

// =============================================================================
// Hook
// =============================================================================

/**
 * Return type for useMotionDetection.
 *
 * Exposes a single function that performs one frame of motion detection.
 * The hook manages all internal state (offscreen canvas, previous frame)
 * via refs so no re-renders are triggered.
 */
interface UseMotionDetectionReturn {
  detectMotion: () => MotionDetectionResult;
}

/**
 * useMotionDetection — Detects motion by comparing consecutive video frames.
 *
 * Algorithm:
 *   1. Draw the current video frame to a hidden offscreen canvas (downscaled to 10%).
 *   2. Extract pixel data via getImageData().
 *   3. Compare each pixel's RGB values against the previous frame.
 *   4. Apply threshold (ignore differences < 30/255 to filter noise).
 *   5. Divide the canvas into an 8×6 grid of regions.
 *   6. For each cell, calculate the fraction of pixels with motion.
 *   7. Return the results. Store current frame as "previous" for next call.
 *
 * Performance:
 *   - 64×48 = 3,072 pixels per frame.
 *   - 3 channel comparisons per pixel = 9,216 comparisons per frame.
 *   - At 60fps: ~553K comparisons/second — negligible CPU cost.
 *
 * @param videoRef - Ref to the <video> element (source of camera frames).
 */
export function useMotionDetection(
  videoRef: React.RefObject<HTMLVideoElement | null>,
): UseMotionDetectionReturn {
  // Offscreen canvas — created once, reused every frame.
  // This canvas exists purely in JavaScript; it's never added to the DOM.
  // We draw the video here at 10% size, then extract pixel data.
  const offscreenCanvasRef = useRef<HTMLCanvasElement | null>(null);
  const offscreenCtxRef = useRef<CanvasRenderingContext2D | null>(null);

  // Previous frame's pixel data — used for comparison.
  // null on the first frame (no comparison possible).
  const previousFrameRef = useRef<Uint8ClampedArray | null>(null);

  /**
   * Lazily initializes the offscreen canvas and its context.
   * Called on the first invocation of detectMotion().
   */
  function getOffscreenContext(): CanvasRenderingContext2D | null {
    if (!offscreenCtxRef.current) {
      const canvas = document.createElement("canvas");
      canvas.width = DETECTION_WIDTH;
      canvas.height = DETECTION_HEIGHT;
      offscreenCanvasRef.current = canvas;
      offscreenCtxRef.current = canvas.getContext("2d", {
        // willReadFrequently: true tells the browser we'll call getImageData()
        // often. This hint allows the browser to optimize the canvas for
        // CPU-side pixel reads (avoids GPU→CPU transfer penalties).
        willReadFrequently: true,
      });
    }
    return offscreenCtxRef.current;
  }

  /**
   * Performs one frame of motion detection.
   *
   * Call this once per animation frame. It:
   *   1. Draws the video to the offscreen canvas (downscaled).
   *   2. Extracts pixel data.
   *   3. Compares with the previous frame.
   *   4. Returns motion regions.
   *   5. Stores current frame for next comparison.
   *
   * @returns MotionDetectionResult with regions and overall motion score.
   */
  const detectMotion = useCallback((): MotionDetectionResult => {
    const video = videoRef.current;
    const ctx = getOffscreenContext();

    // If video or context isn't available, return zero motion.
    if (!video || !ctx) {
      return { regions: [], overallMotion: 0 };
    }

    // --- Step 1: Draw current video frame to offscreen canvas (downscaled) ---
    // drawImage() handles the downscaling internally via bilinear interpolation.
    // This is GPU-accelerated on most browsers — essentially free.
    ctx.drawImage(video, 0, 0, DETECTION_WIDTH, DETECTION_HEIGHT);

    // --- Step 2: Extract pixel data ---
    const imageData = ctx.getImageData(
      0,
      0,
      DETECTION_WIDTH,
      DETECTION_HEIGHT,
    );
    const currentPixels = imageData.data;

    // --- Step 3: Handle first frame ---
    // No previous frame to compare against. Store current and return zero motion.
    const previousPixels = previousFrameRef.current;
    if (!previousPixels) {
      // Clone the current pixel data (Uint8ClampedArray is a typed array;
      // slice() creates a proper copy, not a reference).
      previousFrameRef.current = new Uint8ClampedArray(currentPixels);
      return { regions: [], overallMotion: 0 };
    }

    // --- Step 4: Compare pixels and build per-cell motion counts ---
    // We calculate the cell size in pixels on the offscreen canvas.
    const cellWidth = DETECTION_WIDTH / GRID_COLS;
    const cellHeight = DETECTION_HEIGHT / GRID_ROWS;

    // Motion count per grid cell.
    // motionCounts[row][col] = number of pixels with motion in that cell.
    const motionCounts: number[][] = Array.from(
      { length: GRID_ROWS },
      () => new Array<number>(GRID_COLS).fill(0),
    );

    // Total pixels per cell (for calculating intensity as a fraction).
    const pixelsPerCell = Math.floor(cellWidth) * Math.floor(cellHeight);

    // Total motion across the entire frame (for overallMotion score).
    let totalMotionPixels = 0;
    const totalPixels = DETECTION_WIDTH * DETECTION_HEIGHT;

    // Iterate over every pixel in the offscreen canvas.
    for (let y = 0; y < DETECTION_HEIGHT; y++) {
      for (let x = 0; x < DETECTION_WIDTH; x++) {
        // Calculate the flat array index for this pixel.
        // Each pixel occupies 4 bytes: R, G, B, A.
        const i = (y * DETECTION_WIDTH + x) * 4;

        // Compare RGB channels between current and previous frame.
        // We use absolute difference on each channel independently.
        // Alpha (index + 3) is always 255 for camera frames — skip it.
        const diffR = Math.abs(
          (currentPixels[i] ?? 0) - (previousPixels[i] ?? 0),
        );
        const diffG = Math.abs(
          (currentPixels[i + 1] ?? 0) - (previousPixels[i + 1] ?? 0),
        );
        const diffB = Math.abs(
          (currentPixels[i + 2] ?? 0) - (previousPixels[i + 2] ?? 0),
        );

        // A pixel has "motion" if ANY channel exceeds the threshold.
        // This catches both color changes and brightness changes.
        const hasMotion =
          diffR > PIXEL_DIFF_THRESHOLD ||
          diffG > PIXEL_DIFF_THRESHOLD ||
          diffB > PIXEL_DIFF_THRESHOLD;

        if (hasMotion) {
          // Determine which grid cell this pixel belongs to.
          const col = Math.min(Math.floor(x / cellWidth), GRID_COLS - 1);
          const row = Math.min(Math.floor(y / cellHeight), GRID_ROWS - 1);

          // Non-null assertion is safe here because we initialized the array above.
          // But with noUncheckedIndexedAccess, we need to guard.
          const rowArray = motionCounts[row];
          if (rowArray) {
            rowArray[col] = (rowArray[col] ?? 0) + 1;
          }

          totalMotionPixels++;
        }
      }
    }

    // --- Step 5: Build motion regions ---
    const regions: MotionRegion[] = [];

    for (let row = 0; row < GRID_ROWS; row++) {
      for (let col = 0; col < GRID_COLS; col++) {
        const count = motionCounts[row]?.[col] ?? 0;
        // Intensity is the fraction of pixels in the cell that have motion.
        // 0 = no motion, 1 = every pixel changed.
        const intensity =
          pixelsPerCell > 0 ? Math.min(count / pixelsPerCell, 1) : 0;

        // Only include regions that have meaningful motion.
        // This reduces the size of the output array and avoids processing
        // dead zones in the game logic.
        if (intensity > 0.05) {
          regions.push({
            // Normalized coordinates (0–1 range)
            x: col / GRID_COLS,
            y: row / GRID_ROWS,
            width: 1 / GRID_COLS,
            height: 1 / GRID_ROWS,
            intensity,
          });
        }
      }
    }

    // --- Step 6: Store current frame as previous for next call ---
    // We reuse the existing buffer if possible (same length) to avoid allocation.
    previousFrameRef.current = new Uint8ClampedArray(currentPixels);

    return {
      regions,
      overallMotion:
        totalPixels > 0
          ? Math.min(totalMotionPixels / totalPixels, 1)
          : 0,
    };
  }, [videoRef]);

  return { detectMotion };
}
```

### File: `src/components/GameCanvas.tsx` (updated with debug overlay)

```tsx
import { useRef } from "react";
import { useAnimationFrame } from "../hooks/useAnimationFrame";
import { useMotionDetection } from "../hooks/useMotionDetection";
import type { MotionRegion } from "../types/motion";

interface GameCanvasProps {
  videoRef: React.RefObject<HTMLVideoElement | null>;
  isReady: boolean;
}

const CANVAS_WIDTH = 640;
const CANVAS_HEIGHT = 480;

/**
 * GameCanvas — Now with motion detection!
 *
 * The animation loop now:
 *   1. Clears the canvas.
 *   2. Draws the video frame.
 *   3. Runs motion detection.
 *   4. Draws a debug overlay showing motion regions.
 */
export function GameCanvas({
  videoRef,
  isReady,
}: GameCanvasProps): React.JSX.Element {
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const ctxRef = useRef<CanvasRenderingContext2D | null>(null);

  // Initialize the motion detection hook.
  const { detectMotion } = useMotionDetection(videoRef);

  useAnimationFrame(() => {
    if (!ctxRef.current) {
      ctxRef.current = canvasRef.current?.getContext("2d") ?? null;
    }

    const ctx = ctxRef.current;
    const video = videoRef.current;
    if (!ctx || !video) return;

    // Step 1: Clear
    ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

    // Step 2: Draw video
    ctx.drawImage(video, 0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

    // Step 3: Detect motion
    const motionResult = detectMotion();

    // Step 4: Debug overlay — visualize motion regions
    drawMotionDebugOverlay(ctx, motionResult.regions);
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

/**
 * Draws semi-transparent red rectangles over regions with detected motion.
 * This is a DEBUG visualization — it will be replaced with the dirt/cleaning
 * mechanic in later lessons.
 */
function drawMotionDebugOverlay(
  ctx: CanvasRenderingContext2D,
  regions: MotionRegion[],
): void {
  for (const region of regions) {
    // Map normalized coordinates (0–1) to canvas pixels.
    const x = region.x * CANVAS_WIDTH;
    const y = region.y * CANVAS_HEIGHT;
    const w = region.width * CANVAS_WIDTH;
    const h = region.height * CANVAS_HEIGHT;

    // Alpha is proportional to motion intensity.
    // More motion = more opaque red overlay.
    ctx.fillStyle = `rgba(255, 0, 0, ${region.intensity * 0.5})`;
    ctx.fillRect(x, y, w, h);
  }
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
```

### How the algorithm works visually

```
Video Frame (640×480)
        │
        ▼  drawImage() with downscale
Offscreen Canvas (64×48)
        │
        ▼  getImageData()
Pixel Array (3,072 pixels × 4 channels)
        │
        ├── Compare with Previous Frame
        │   └── Per-pixel: |current[R,G,B] - previous[R,G,B]| > 30?
        │
        ▼  Aggregate into grid
Motion Grid (8×6 cells)
        │
        ├── Cell (0,0): 45% of pixels moved → intensity: 0.45
        ├── Cell (1,0): 2% of pixels moved  → intensity: 0.02 (below 5% threshold, ignored)
        ├── Cell (2,0): 78% of pixels moved → intensity: 0.78
        │   ...
        │
        ▼  Filter (intensity > 0.05) & Normalize coordinates
MotionDetectionResult {
  regions: [
    { x: 0.000, y: 0.000, width: 0.125, height: 0.167, intensity: 0.45 },
    { x: 0.250, y: 0.000, width: 0.125, height: 0.167, intensity: 0.78 },
    ...
  ],
  overallMotion: 0.12
}
```

---

## Executive Summary

**Jack Donaghy:**

> What we've built is a **real-time signal processing pipeline** — and we did it with zero external dependencies. Let me explain the strategic significance.

> 1. **Frame differencing is zero-dependency intelligence.** No TensorFlow. No MediaPipe. No cloud APIs. No model downloads. No vendor contracts. We detect motion using pure mathematics on pixel arrays. This is the engineering equivalent of building your own power plant instead of buying electricity from the grid. Total operational independence.

> 2. **The downscale optimization is a 99% cost reduction.** By processing at 10% resolution, we reduced per-frame computation by two orders of magnitude — from 1.2 million comparisons to 12,000. The quality loss is negligible because we're detecting *motion*, not *faces*. We don't need high resolution to know that something moved. This is the operational equivalent of realizing that you don't need a $5,000 suit for every meeting — sometimes the $2,000 suit is more than sufficient.

> 3. **The grid aggregation is a data summarization layer.** Raw pixel-level motion data is noise. Grid-level motion data is signal. By aggregating 3,072 pixel comparisons into 48 region intensities, we've transformed unstructured data into actionable intelligence. In business terms, we've turned a 500-page financial report into a one-page executive summary. Same information. 100x more useful.

> 4. **Normalized coordinates are resolution-independent contracts.** The motion detection system doesn't know or care about the display canvas size. It outputs 0–1 coordinates. The game logic maps those to whatever canvas it's using. This is *loose coupling* — the same principle that allows a corporate headquarters to operate factories of different sizes with the same management framework.

> 5. **The `willReadFrequently` hint is performance engineering.** By telling the browser we'll call `getImageData()` every frame, we allow it to keep the canvas in CPU memory instead of GPU memory. This avoids the expensive GPU→CPU data transfer that would otherwise happen 60 times per second. It's a one-line optimization with a measurable performance impact. That's efficiency. That's what I do.
