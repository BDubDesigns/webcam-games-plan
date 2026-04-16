# Lesson 09: Frame Differencing & Motion Detection

## The Pitch

**Tracy Jordan:**
> The computer can SEE you MOVING! It's like MAGIC but with MATH! We take a picture, then we take ANOTHER picture, and we see what CHANGED! If your hand moved, the pixels are DIFFERENT! It's called FRAME DIFFERENCING and I don't fully understand it but I DO understand that it's INCREDIBLE! I like motion detection so much I want to take it behind the middle school and get it pregnant! We're teaching the computer to have EYES!

**Jack Donaghy:**
> What Tracy is describing is our core competitive differentiator — the proprietary motion detection algorithm that powers our entire product line. In business terms, this is our *value chain analysis*. We receive raw pixel data (raw materials), apply a differencing algorithm (manufacturing process), and output motion regions (finished goods). The elegance of this approach is that it requires zero external libraries, zero machine learning models, and zero cloud APIs. It's pure canvas pixel manipulation. That's vertical integration at the compute level — we own the entire production process from input to output. No dependencies. No vendor lock-in. No supply chain vulnerabilities.

**Liz Lemon:**
> Okay so THIS is the lesson that will make or break the game. Motion detection sounds fancy, but the algorithm is actually straightforward. The NIGHTMARE part is performance. A 640×480 frame has 307,200 pixels. Each pixel has 4 values (R, G, B, A). That's 1.2 million numbers to compare PER FRAME at 60fps. If you try to process this at full resolution, your game drops to 5fps and your user thinks their computer is broken. That's why we DOWNSCALE to 10% before processing. 64×48 = 3,072 pixels. Much more manageable. The downscale happens on a HIDDEN offscreen canvas. The user never sees it. It's just for math. This is the kind of optimization that makes me feel slightly less stressed.

---

## The Theory

### What is frame differencing?

**Question:** How can we detect motion using only pixel data?

**Answer:** Motion detection via frame differencing is conceptually simple:

1. **Capture Frame A** (the "previous" frame).
2. **Capture Frame B** (the "current" frame).
3. **Compare every pixel:** For each pixel position, calculate the difference between Frame A and Frame B's color values.
4. **Threshold:** If the difference exceeds a threshold, that pixel has "motion."
5. **Aggregate:** Group motion pixels into regions.

```
Frame A (previous)    Frame B (current)     Difference (motion)
┌──────────────┐     ┌──────────────┐      ┌──────────────┐
│   ██         │     │      ██      │      │   ██ ██      │
│   ██         │     │      ██      │  →   │   ██ ██      │
│              │     │              │      │              │
└──────────────┘     └──────────────┘      └──────────────┘
Hand on left         Hand moved right      Both positions differ
```

### Pixel data structure: ImageData

**Question:** How do we access individual pixels on a canvas?

**Answer:** The Canvas API provides `getImageData()`:

```ts
const imageData = ctx.getImageData(0, 0, width, height);
const pixels = imageData.data; // Uint8ClampedArray
```

The `data` property is a `Uint8ClampedArray` — a flat array where every 4 consecutive elements represent one pixel:

```
Index:  0    1    2    3    4    5    6    7    8    ...
        R₀   G₀   B₀   A₀   R₁   G₁   B₁   A₁   R₂  ...
        ←── Pixel 0 ──→   ←── Pixel 1 ──→   ←── Pixel 2
```

- **R** (Red): 0–255
- **G** (Green): 0–255
- **B** (Blue): 0–255
- **A** (Alpha): 0–255 (255 = fully opaque)

To access pixel at position `(x, y)` in an image of width `w`:

```ts
const index = (y * w + x) * 4;
const r = pixels[index];      // Red
const g = pixels[index + 1];  // Green
const b = pixels[index + 2];  // Blue
const a = pixels[index + 3];  // Alpha
```

### Why downscale?

**Question:** Why process at 10% resolution instead of full resolution?

**Answer:** Performance math:

| Resolution | Pixels | Values (RGBA) | Per-frame comparison |
|-----------|--------|---------------|---------------------|
| 640×480 (100%) | 307,200 | 1,228,800 | ~1.2M comparisons |
| 320×240 (50%) | 76,800 | 307,200 | ~307K comparisons |
| 64×48 (10%) | 3,072 | 12,288 | ~12K comparisons |

At 60fps, full resolution means 73 million comparisons per second. At 10%, it's 737K. That's a 99% reduction in computation.

The downscale itself is free — `ctx.drawImage()` with smaller target dimensions does bilinear interpolation in hardware (GPU-accelerated). The reduction in pixel-level computation is massive.

### The threshold: dealing with noise

**Question:** Why can't we use a threshold of 0 (any change = motion)?

**Answer:** Webcams are noisy. Even a perfectly still scene produces slightly different pixel values each frame due to:

- Sensor noise (electrical interference)
- Auto-exposure adjustments
- Compression artifacts
- Lighting fluctuations

A threshold of ~30 out of 255 (~12%) is a good starting point. This ignores the constant low-level noise while detecting actual hand movement.

### The offscreen canvas pattern

**Question:** Why use a separate canvas for motion detection?

**Answer:** We need a canvas to:

1. Draw the video frame at 10% size (downscaling).
2. Call `getImageData()` to extract pixels.

This canvas should NOT be the main visible canvas because:
- It's a different resolution (64×48 vs 640×480).
- We don't want the user to see it.
- Drawing and extracting on the same canvas we display causes flickering.

An *offscreen canvas* is created in JavaScript (`document.createElement("canvas")`) and never added to the DOM. It exists purely for computation.

---

## The Assignment

### Step 1: Define the motion detection types

- Create `src/types/motion.ts` with these interfaces:

```ts
interface MotionRegion {
  x: number;       // 0-1 normalized position
  y: number;       // 0-1 normalized position
  intensity: number; // 0-1 normalized motion strength
}

interface MotionDetectionResult {
  regions: MotionRegion[];
  overallMotion: number;  // 0-1 overall motion score
}
```

### Step 2: Create the useMotionDetection hook

- Create `src/hooks/useMotionDetection.ts`.
- The hook should:
  - Accept a `videoRef` and an `isActive` boolean.
  - Internally create an offscreen canvas at 10% resolution (64×48).
  - Store the previous frame's pixel data in a ref.
  - Expose a `detectMotion()` function that:
    1. Draws the current video frame to the offscreen canvas (downscaled).
    2. Extracts pixel data via `getImageData()`.
    3. Compares current pixels to previous frame's pixels.
    4. Applies a per-pixel threshold (default: 30).
    5. Divides the canvas into a grid of regions (e.g., 8×6 cells).
    6. For each cell, calculates the percentage of pixels with motion.
    7. Returns a `MotionDetectionResult`.
    8. Stores the current frame as the new "previous" frame.

### Step 3: Configure motion detection constants

- Define constants at the top of the hook file:
  - `DETECTION_WIDTH = 64`
  - `DETECTION_HEIGHT = 48`
  - `PIXEL_DIFF_THRESHOLD = 30` (minimum RGB difference to count as motion)
  - `GRID_COLS = 8`
  - `GRID_ROWS = 6`

### Step 4: Integrate into GameCanvas

- Use the `useMotionDetection` hook in `GameCanvas`.
- In the animation frame callback, call `detectMotion()`.
- For debugging, visualize the motion: overlay a semi-transparent colored rectangle on each grid cell where motion is detected. Use red with alpha proportional to motion intensity.

### Step 5: Verify

- Run `npm run dev`.
- Wave your hand in front of the camera.
- You should see red overlay rectangles appearing where you move.
- The overlay should disappear when you hold still.
- Run `npx tsc --noEmit` and `npm run lint` — zero errors.

### Deliverables

1. `src/types/motion.ts` — TypeScript interfaces for motion detection.
2. `src/hooks/useMotionDetection.ts` — the complete motion detection hook.
3. Updated `GameCanvas.tsx` with motion visualization debug overlay.
4. Motion detection runs at 10% resolution on an offscreen canvas.
5. Zero type errors, zero lint errors.

---

## Liz's Nightmares

> **Liz Lemon:** Pixel manipulation. Where off-by-one errors aren't just bugs — they're psychedelic visual nightmares.

1. **Off-by-one in the RGBA index:** The pixel array is flat. Pixel `i` starts at index `i * 4`. If you forget the `* 4`, you'll read every 4th pixel and the motion map will be a kaleidoscope of wrong. I once spent an hour debugging this and the result looked like a Windows 95 screensaver.

2. **Not creating the offscreen canvas ONCE:** If you create a new offscreen canvas every frame (`document.createElement("canvas")`), you're allocating and garbage collecting a canvas 60 times per second. Create it once in a ref and reuse it. Garbage collection pauses cause frame drops, and frame drops cause jank.

3. **Forgetting to handle the first frame:** On the first call to `detectMotion()`, there's no previous frame to compare against. If you try to compare `current[i] - previous[i]` and previous is null, you crash. The first frame should always return zero motion and just store itself as the "previous" for next time.

4. **Not normalizing the motion regions to 0-1 coordinates:** The offscreen canvas is 64×48 pixels, but the game canvas is 640×480. If you return motion coordinates in offscreen-canvas pixels, the game logic has to know about the downscale ratio. Instead, normalize to 0-1 (fractional positions), and let the consumer multiply by whatever canvas size they're using. This decouples detection resolution from display resolution.

5. **Threshold too low → everything is motion:** Webcam noise is real. A threshold of 10 will detect "motion" in every frame even with a perfectly still camera. Start at 30 and tune from there. If you're getting false positives, go higher. You can make this configurable later.

6. **Comparing alpha channels:** The alpha channel of a camera feed is always 255 (fully opaque). Comparing alpha values wastes computation and adds nothing. Only compare R, G, and B.
