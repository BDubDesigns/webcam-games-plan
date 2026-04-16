# Lesson 10 — Answer Sheet: The Dirt Layer — Procedural Noise & Canvas Compositing

## The Implementation

### File: `src/game/DirtLayer.ts`

```ts
/**
 * DirtLayer — Manages the procedural dirt overlay for the Window Cleaner game.
 *
 * Architecture:
 *   The dirt exists on its own offscreen canvas, separate from the main display.
 *   This separation is critical because:
 *     1. We need to use `destination-out` compositing to erase dirt.
 *        If the dirt were on the same canvas as the video, erasing dirt
 *        would also erase the video feed.
 *     2. The dirt state must persist across frames. The main canvas is
 *        cleared and redrawn every frame, but the dirt canvas maintains
 *        its state until explicitly cleaned.
 *
 * Lifecycle:
 *   1. Create an instance with canvas dimensions.
 *   2. Call generate() once to fill with procedural dirt.
 *   3. Each frame: call clean() for motion regions, then draw() to composite.
 *   4. Periodically call getDirtRemaining() to check win condition.
 */

// =============================================================================
// Constants
// =============================================================================

/** Probability (0–1) that any given pixel has dirt. 0.7 = 70% coverage. */
const DIRT_DENSITY = 0.7;

/** Dirt color ranges (brownish/grime) */
const DIRT_R_MIN = 80;
const DIRT_R_MAX = 120;
const DIRT_G_MIN = 60;
const DIRT_G_MAX = 80;
const DIRT_B_MIN = 30;
const DIRT_B_MAX = 50;

/** Alpha range for dirt opacity (higher = more opaque) */
const DIRT_ALPHA_MIN = 100;
const DIRT_ALPHA_MAX = 200;

/** Win condition: if dirt remaining drops below this, the window is "clean" */
export const DIRT_WIN_THRESHOLD = 0.05;

// =============================================================================
// Class
// =============================================================================

export class DirtLayer {
  /** The offscreen canvas containing the dirt overlay */
  private readonly canvas: HTMLCanvasElement;
  /** The 2D context for the dirt canvas */
  private readonly ctx: CanvasRenderingContext2D;
  /** Canvas width in pixels */
  private readonly width: number;
  /** Canvas height in pixels */
  private readonly height: number;

  /**
   * Creates a new DirtLayer.
   *
   * @param width - Width of the dirt canvas (should match display canvas).
   * @param height - Height of the dirt canvas (should match display canvas).
   */
  constructor(width: number, height: number) {
    this.width = width;
    this.height = height;

    // Create an offscreen canvas — exists in JS memory, not in the DOM.
    this.canvas = document.createElement("canvas");
    this.canvas.width = width;
    this.canvas.height = height;

    const ctx = this.canvas.getContext("2d", {
      // willReadFrequently: we'll call getImageData() for progress checking.
      willReadFrequently: true,
    });

    if (!ctx) {
      throw new Error("Failed to create 2D context for dirt layer canvas");
    }

    this.ctx = ctx;
  }

  /**
   * Generates procedural dirt by setting individual pixel values.
   *
   * Algorithm:
   *   1. Create an ImageData buffer for the entire canvas.
   *   2. For each pixel, randomly decide if dirt exists (based on DIRT_DENSITY).
   *   3. If dirt: set to a random brownish color with varying opacity.
   *   4. If no dirt: set alpha to 0 (fully transparent).
   *   5. Write the buffer back to the canvas with putImageData().
   *
   * This should be called ONCE at game start. Calling it again resets the dirt.
   */
  generate(): void {
    // Create a blank ImageData buffer (all zeros = fully transparent black).
    const imageData = this.ctx.createImageData(this.width, this.height);
    const pixels = imageData.data;

    for (let i = 0; i < pixels.length; i += 4) {
      // Roll the dice: does this pixel have dirt?
      if (Math.random() < DIRT_DENSITY) {
        // Dirt pixel: brownish color with random variation.
        // randomInRange returns a value between min and max (inclusive).
        pixels[i] = randomInRange(DIRT_R_MIN, DIRT_R_MAX); // R
        pixels[i + 1] = randomInRange(DIRT_G_MIN, DIRT_G_MAX); // G
        pixels[i + 2] = randomInRange(DIRT_B_MIN, DIRT_B_MAX); // B
        pixels[i + 3] = randomInRange(DIRT_ALPHA_MIN, DIRT_ALPHA_MAX); // A
      }
      // Else: pixels are already 0,0,0,0 (transparent) — no action needed.
    }

    // Write the pixel data to the canvas in one operation.
    // putImageData() is a direct pixel buffer write — no compositing applied.
    this.ctx.putImageData(imageData, 0, 0);
  }

  /**
   * Erases dirt at the given position using a soft-edged radial gradient.
   *
   * Uses `globalCompositeOperation = "destination-out"`:
   *   - Where the gradient is drawn, existing dirt pixels have their alpha reduced.
   *   - The gradient itself is NOT visible — it acts as an eraser.
   *   - The radial gradient creates a soft edge (fully erased at center,
   *     partially erased at edges) for a natural "wiping" effect.
   *
   * @param x - X position on the canvas (in canvas pixels, not normalized).
   * @param y - Y position on the canvas (in canvas pixels, not normalized).
   * @param radius - Radius of the cleaning area in pixels.
   */
  clean(x: number, y: number, radius: number): void {
    // Save the current canvas state (composite operation, styles, transforms).
    this.ctx.save();

    // Set composite operation to "destination-out".
    // This means: "Where I draw, erase the existing content."
    // The drawn shape itself does not appear — it acts purely as a mask.
    this.ctx.globalCompositeOperation = "destination-out";

    // Create a radial gradient for soft edges.
    // Inner circle (radius 0): fully opaque → erases completely.
    // Outer circle (radius): fully transparent → no erasure at edges.
    const gradient = this.ctx.createRadialGradient(x, y, 0, x, y, radius);
    gradient.addColorStop(0, "rgba(255, 255, 255, 1)"); // Full erasure at center
    gradient.addColorStop(0.5, "rgba(255, 255, 255, 0.5)"); // Partial at midpoint
    gradient.addColorStop(1, "rgba(255, 255, 255, 0)"); // No erasure at edge

    // Draw the gradient circle.
    this.ctx.fillStyle = gradient;
    this.ctx.beginPath();
    this.ctx.arc(x, y, radius, 0, Math.PI * 2);
    this.ctx.fill();

    // Restore the previous canvas state.
    // This resets globalCompositeOperation back to "source-over" (the default).
    // CRITICAL: If we forget this, all subsequent draws will ERASE instead of ADD.
    this.ctx.restore();
  }

  /**
   * Calculates how much dirt remains on the canvas.
   *
   * Reads every pixel's alpha value and compares the sum against the
   * maximum possible alpha. This tells us what fraction of the original
   * dirt is still visible.
   *
   * PERFORMANCE NOTE: getImageData() is expensive (forces GPU→CPU sync).
   * Don't call this every frame. Call it every ~30 frames or after a
   * cleaning action.
   *
   * @returns A value from 0 (completely clean) to 1 (fully dirty).
   */
  getDirtRemaining(): number {
    const imageData = this.ctx.getImageData(0, 0, this.width, this.height);
    const pixels = imageData.data;

    let totalAlpha = 0;

    // Alpha is at every 4th position: indices 3, 7, 11, 15, ...
    for (let i = 3; i < pixels.length; i += 4) {
      totalAlpha += pixels[i] ?? 0;
    }

    // Maximum possible alpha: every pixel at 255.
    const maxAlpha = this.width * this.height * 255;

    return maxAlpha > 0 ? totalAlpha / maxAlpha : 0;
  }

  /**
   * Composites the dirt layer onto a target canvas context.
   *
   * Draws the entire dirt canvas onto the target context using the default
   * "source-over" composite operation. Since dirt pixels have varying alpha,
   * the video feed shows through partially — creating the "dirty glass" effect.
   *
   * @param targetCtx - The rendering context of the main display canvas.
   */
  draw(targetCtx: CanvasRenderingContext2D): void {
    // drawImage can accept another canvas as the source.
    // This draws the dirt overlay on top of whatever is already on the target canvas.
    // Pixels with alpha < 255 blend with the background (video feed).
    targetCtx.drawImage(this.canvas, 0, 0);
  }

  /**
   * Resets the dirt layer by clearing the canvas entirely.
   * Call generate() after this to create new dirt.
   */
  reset(): void {
    this.ctx.clearRect(0, 0, this.width, this.height);
  }
}

// =============================================================================
// Helpers
// =============================================================================

/**
 * Returns a random integer between min and max (inclusive).
 */
function randomInRange(min: number, max: number): number {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}
```

### Updated `src/components/GameCanvas.tsx`

```tsx
import { useRef } from "react";
import { useAnimationFrame } from "../hooks/useAnimationFrame";
import { useMotionDetection } from "../hooks/useMotionDetection";
import { DirtLayer } from "../game/DirtLayer";

interface GameCanvasProps {
  videoRef: React.RefObject<HTMLVideoElement | null>;
  isReady: boolean;
}

const CANVAS_WIDTH = 640;
const CANVAS_HEIGHT = 480;

/** Radius of the cleaning area in pixels when motion is detected */
const CLEAN_RADIUS = 40;

/** How often (in frames) to check dirt progress. Every 30 frames ≈ every 0.5s */
const PROGRESS_CHECK_INTERVAL = 30;

/**
 * GameCanvas — Now with a dirt layer!
 *
 * Renders:
 *   1. Live camera feed (bottom layer)
 *   2. Procedural dirt overlay (top layer)
 *
 * Motion detection erases dirt where the player moves.
 */
export function GameCanvas({
  videoRef,
  isReady,
}: GameCanvasProps): React.JSX.Element {
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const ctxRef = useRef<CanvasRenderingContext2D | null>(null);

  // Dirt layer — created once, persists across renders.
  const dirtLayerRef = useRef<DirtLayer | null>(null);

  // Frame counter for throttling progress checks.
  const frameCountRef = useRef<number>(0);

  // Motion detection hook.
  const { detectMotion } = useMotionDetection(videoRef);

  // Initialize dirt layer lazily on first frame.
  function getDirtLayer(): DirtLayer {
    if (!dirtLayerRef.current) {
      const layer = new DirtLayer(CANVAS_WIDTH, CANVAS_HEIGHT);
      layer.generate();
      dirtLayerRef.current = layer;
    }
    return dirtLayerRef.current;
  }

  useAnimationFrame(() => {
    if (!ctxRef.current) {
      ctxRef.current = canvasRef.current?.getContext("2d") ?? null;
    }

    const ctx = ctxRef.current;
    const video = videoRef.current;
    if (!ctx || !video) return;

    const dirtLayer = getDirtLayer();

    // Step 1: Clear the main canvas
    ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

    // Step 2: Draw the video feed (bottom layer)
    ctx.drawImage(video, 0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

    // Step 3: Detect motion and clean dirt
    const motionResult = detectMotion();
    for (const region of motionResult.regions) {
      // Map normalized coordinates (0–1) to canvas pixels
      const centerX = (region.x + region.width / 2) * CANVAS_WIDTH;
      const centerY = (region.y + region.height / 2) * CANVAS_HEIGHT;

      // Scale cleaning radius by motion intensity
      // More motion = larger cleaning area
      const radius = CLEAN_RADIUS * region.intensity;

      dirtLayer.clean(centerX, centerY, radius);
    }

    // Step 4: Draw the dirt layer on top
    dirtLayer.draw(ctx);

    // Step 5: Check progress periodically (not every frame — too expensive)
    frameCountRef.current++;
    if (frameCountRef.current % PROGRESS_CHECK_INTERVAL === 0) {
      const remaining = dirtLayer.getDirtRemaining();
      // For now, log it. Win state handling comes in Lesson 12.
      if (remaining < 0.05) {
        // eslint-disable-next-line no-console
        console.log("Window is clean! 🎉");
      }
    }
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

### Visual architecture

```
What the user sees (composited on main canvas):

  ┌─────────────────────────────┐
  │  Video feed (camera)        │  ← Layer 1: drawImage(video)
  │  ┌───────────────────────┐  │
  │  │  Brown dirt overlay    │  │  ← Layer 2: dirtLayer.draw(ctx)
  │  │  with "holes" where   │  │
  │  │  player has cleaned   │  │
  │  │  ██████░░░░░██████    │  │  ░ = cleaned (video visible)
  │  │  ██████████████████   │  │  █ = dirt (semi-transparent brown)
  │  │  ████░░░░░░░░██████   │  │
  │  └───────────────────────┘  │
  └─────────────────────────────┘

Internal structure:

  Main Canvas (640×480)
    └── Every frame:
        1. ctx.clearRect()
        2. ctx.drawImage(video)           ← Live camera
        3. motionDetection.detect()       ← Find motion
        4. dirtLayer.clean(x, y, r)       ← Erase dirt at motion
        5. dirtLayer.draw(ctx)            ← Overlay remaining dirt

  Dirt Canvas (640×480, offscreen)
    └── Persistent state:
        - Created once, never cleared (except by clean())
        - Pixels with alpha > 0 = dirt
        - Pixels with alpha = 0 = clean (video shows through)
```

---

## Executive Summary

**Jack Donaghy:**

> We have just built the **core product experience** — the dirty window and the cleaning mechanic. Let me contextualize this achievement.

> 1. **Procedural generation is manufacturing without inventory.** We don't ship a PNG dirt texture. We generate it algorithmically at runtime. This means zero asset loading time, infinite variation (every game has unique dirt), and zero storage cost. In business terms, we've achieved *on-demand manufacturing* — we produce exactly what we need, when we need it, with zero inventory.

> 2. **Canvas compositing is our production assembly line.** Each frame, we composite three layers: clear canvas → video feed → dirt overlay. This is a defined, repeatable, quality-controlled assembly process. The layers are isolated — video doesn't know about dirt, dirt doesn't know about video. They're composed at the final assembly stage. This is *modular manufacturing*.

> 3. **`destination-out` is our precision tooling.** The composite operation erases dirt with mathematical precision. The radial gradient creates soft edges — no harsh boundaries, no visible artifacts. This is the kind of finish quality that distinguishes a Kabletown premium product from a basic cable offering.

> 4. **Throttled progress checking is resource-aware operations.** We don't check dirt progress every frame (too expensive). We check every 30 frames. The user perceives instant feedback, but we've reduced the computational cost by 97%. This is *capacity planning* — allocating expensive resources (getImageData) only when necessary.

> The game is now *playable*. Wave your hands, clean the window. In the next two lessons, we add the win condition and the celebratory particle effects. Because in business, you don't just deliver the product — you deliver the *experience*.
