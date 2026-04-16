# Lesson 10: The Dirt Layer — Procedural Noise & Canvas Compositing

## The Pitch

**Tracy Jordan:**
> We're making the SCREEN DIRTY! On PURPOSE! I usually make things dirty by accident but this time it's PART OF THE GAME! We're going to draw a bunch of GRIME and MUCK and FILTH on the canvas and then the PLAYER has to clean it off by waving their hands! I like procedural dirt generation so much I want to take it behind the middle school and get it pregnant! This is like that time I was in a car wash but THE CAR IS YOUR COMPUTER!

**Jack Donaghy:**
> The dirt layer is our *product differentiation*. Any developer can show a webcam feed. WE are overlaying a fully procedural, alpha-controlled grime texture that the user physically interacts with. That's an *experiential product* — not a feature, but an experience. The technical foundation is the `globalCompositeOperation` API, which controls how new drawing operations interact with existing canvas content. In corporate terms, it's our *vertical integration of the rendering pipeline*. We own the compositing. We own the experience. We own the market.

**Liz Lemon:**
> Okay, `globalCompositeOperation`. Deep breaths. This is the Canvas API's most powerful and most confusing feature. There are 26 composite operations, and if you pick the wrong one, your dirt either disappears instantly, becomes indestructible, or turns into modern art. We need `destination-out`, which means "erase existing content where new content is drawn." Think of it like a magic eraser — wherever you draw, the existing pixels get their alpha reduced to zero. But here's the thing — this affects the ENTIRE canvas, not just the dirt. So we put the dirt on its OWN canvas layer and composite it separately. Because if you try to use `destination-out` on the same canvas as the video feed, you'll erase the video too, and then you'll be staring at a black rectangle wondering why you chose this career.

---

## The Theory

### What is the dirt layer?

**Question:** How does the "dirty window" effect work in our game?

**Answer:** Conceptually:

1. The camera feed shows the player's live video (the "window" behind the dirt).
2. On top of the video, we render a semi-transparent "dirt" overlay.
3. Where the player moves their hands (motion detected), the dirt is erased.
4. When all the dirt is gone, the player wins.

Technically, the dirt is a separate canvas (offscreen) filled with semi-transparent brown/gray pixels. We composite this dirt canvas ON TOP of the video canvas every frame.

### Procedural dirt generation

**Question:** How do we create realistic-looking dirt without an image asset?

**Answer:** Procedural generation creates the dirt pattern using random numbers:

1. **Iterate over every pixel** in the dirt canvas.
2. **For each pixel**, generate a random value to determine if dirt exists there.
3. **Vary the alpha** — some areas are more opaque (thick grime), others are more transparent (light dust).
4. **Use a brownish color** with varying opacity.

This produces a noise-like pattern that looks like real dirt/grime when viewed at normal scale.

### Canvas layering architecture

**Question:** Why not draw dirt directly on the main canvas?

**Answer:** Because we need to:

1. **Selectively erase dirt** without affecting the video feed underneath.
2. **Track the dirt state** frame-to-frame (how much dirt remains).
3. **Composite dirt on top of video** in a controlled way.

Architecture:

```
Layer 3 (top):    UI overlay (score, timer) — drawn last
Layer 2:          Dirt canvas (offscreen) — composited onto main canvas
Layer 1 (bottom): Video feed — drawn to main canvas first
```

The dirt canvas maintains its own persistent state. Each frame:
1. Draw video to main canvas.
2. Draw dirt canvas onto main canvas (semi-transparent overlay).

### globalCompositeOperation: `destination-out`

**Question:** How does `destination-out` work?

**Answer:** `globalCompositeOperation` controls how newly drawn pixels interact with existing pixels on the canvas.

`destination-out` means: **Where the new drawing overlaps with existing content, erase the existing content. The new drawing itself is not visible.**

```
Before:    ████████████  (dirt with alpha = 200)
Draw:           ██       (circle at center)
After:     ████    ████  (dirt erased where circle was drawn)
```

This is exactly what we need for the cleaning mechanic. We draw a circle (or any shape) on the dirt canvas where motion is detected. The dirt under that circle is erased. The circle itself doesn't appear.

### Tracking dirt progress

**Question:** How do we know when the window is clean?

**Answer:** After each cleaning pass, we read the dirt canvas's pixel data and sum the alpha values. When the total alpha falls below a threshold (e.g., 5% of the maximum possible), the window is "clean enough" and the player wins.

```ts
const dirtData = dirtCtx.getImageData(0, 0, width, height);
let totalAlpha = 0;
for (let i = 3; i < dirtData.data.length; i += 4) {
  totalAlpha += dirtData.data[i]; // Alpha is every 4th value
}
const maxAlpha = (width * height) * 255;
const dirtRemaining = totalAlpha / maxAlpha; // 0–1
```

---

## The Assignment

### Step 1: Create the DirtLayer class/module

- Create `src/game/DirtLayer.ts`.
- It should export a class or a set of functions that manage the dirt canvas.
- The dirt layer should:
  - Create an offscreen canvas at the same resolution as the main canvas (640×480).
  - Have a `generate()` method that fills the canvas with procedural dirt.
  - Have a `clean(x, y, radius)` method that erases dirt at the given position.
  - Have a `getDirtRemaining()` method that returns a 0–1 value.
  - Have a `draw(targetCtx)` method that composites the dirt onto another canvas context.

### Step 2: Implement procedural dirt generation

- The `generate()` method should:
  - Use `getImageData` / `putImageData` to set individual pixels.
  - For each pixel, generate a random chance of dirt (e.g., 70% chance of dirt per pixel).
  - Dirt color: brownish (R: 80–120, G: 60–80, B: 30–50).
  - Dirt alpha: vary between 100 and 200 (partial transparency so video shows through slightly).
  - Non-dirt pixels: alpha = 0 (fully transparent).

### Step 3: Implement the clean method

- The `clean(x, y, radius)` method should:
  - Set `globalCompositeOperation = "destination-out"` on the dirt canvas.
  - Draw a radial gradient circle centered at `(x, y)` with the given radius.
  - The gradient should be fully opaque at center, fading to transparent at edges.
  - This creates a soft "wiping" effect rather than a hard circle edge.
  - Restore the composite operation to `"source-over"` after drawing.

### Step 4: Implement dirt progress tracking

- The `getDirtRemaining()` method should:
  - Read the dirt canvas pixels.
  - Sum all alpha values.
  - Return the fraction of maximum possible alpha (0 = clean, 1 = fully dirty).

### Step 5: Integrate into GameCanvas

- Instantiate the dirt layer (use a ref so it persists across renders).
- Call `generate()` once on initialization.
- In the animation frame:
  1. Draw video.
  2. For each motion region, call `clean()` with the region's center mapped to canvas coordinates.
  3. Draw the dirt layer on top.

### Step 6: Verify

- Run `npm run dev`.
- You should see the camera feed with a brownish dirt overlay.
- Wave your hand — dirt should be erased where you move.
- The dirt should stay erased (persistent state).
- Run `npx tsc --noEmit` and `npm run lint`.

### Deliverables

1. `src/game/DirtLayer.ts` — complete dirt management module.
2. Procedural dirt generation with varying opacity.
3. Soft-edged cleaning using `destination-out` + radial gradient.
4. `getDirtRemaining()` for win condition tracking.
5. Integration in `GameCanvas.tsx`.
6. Zero type errors, zero lint errors.

---

## Liz's Nightmares

> **Liz Lemon:** Canvas compositing. Where every drawing operation potentially destroys everything you've already drawn. It's like cooking with explosives.

1. **Forgetting to reset `globalCompositeOperation`:** After setting it to `"destination-out"` for cleaning, if you don't set it back to `"source-over"`, EVERY subsequent draw operation will erase content instead of adding it. You'll draw your video and it'll erase the dirt. You'll draw text and it'll erase the video. Everything is erasing everything else. It's chaos. ALWAYS save/restore or explicitly reset.

2. **Cleaning on the wrong canvas:** If you accidentally use `destination-out` on the MAIN canvas instead of the dirt canvas, you'll erase the video feed. The screen goes black. The user panics. You panic. Everyone's panicking. The dirt layer has its OWN canvas. Clean THAT one.

3. **Generating dirt every frame:** The `generate()` method creates the initial dirt. It should be called ONCE, not every frame. If you call it every frame, the dirt regenerates instantly and the player can never clean it. They'll wave forever and wonder why they're not making progress. Store the dirt canvas in a ref and generate on initialization.

4. **Hard-edged cleaning circles:** If you use `fillRect` or `arc` with solid fill for cleaning, the cleaned areas have sharp edges. Real wiping has soft edges. Use a `createRadialGradient` with an opacity gradient from center (opaque) to edge (transparent) for a natural-looking clean.

5. **Reading pixel data for progress check every frame:** `getImageData()` is expensive — it forces a GPU→CPU sync. Don't call `getDirtRemaining()` every frame. Call it every 30 frames (every half second) or when a cleaning action occurs. The user won't notice the delay but your frame rate will thank you.

6. **Alpha accumulation errors:** When you have partially transparent dirt (alpha = 150) and you partially erase it (reduce alpha by 100), the remaining alpha is 50. If your progress check sums all alphas, small residual values add up. A 640×480 canvas has 307,200 pixels. If each has alpha = 5 (almost invisible), the total alpha is 1.5 million — which looks like 0.5% dirt remaining. Set your "clean enough" threshold to account for this noise. 5% remaining is a reasonable win condition.
