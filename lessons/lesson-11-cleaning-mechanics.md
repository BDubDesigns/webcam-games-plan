# Lesson 11: Cleaning Mechanics — Motion Erases Dirt

## The Pitch

**Tracy Jordan:**
> The game is WORKING! You wave your hands and the dirt DISAPPEARS! It's like I'm a WIZARD! A DIGITAL JANITOR WIZARD! But we gotta make it FEEL good — you know, like when you clean a real window and you can see the squeegee making that satisfying stripe? We need to make it feel PHYSICAL! I like haptic feedback in visual form so much I want to take it behind the middle school and get it pregnant!

**Jack Donaghy:**
> What we're refining here is the *user interaction model* — the interface between human gesture and digital response. In market research, this is called *perceived responsiveness*. The latency between a user's hand movement and the visible dirt erasure must be below 100 milliseconds to feel "instant." Fortunately, our 60fps pipeline delivers sub-17ms response time. But responsiveness alone isn't enough. We need *satisfaction engineering*: the cleaning radius must scale with motion intensity, the erasure must have soft edges for visual polish, and the system must gracefully handle edge cases like rapid motion and stationary hands. This is where good products become *great* products.

**Liz Lemon:**
> Okay, so we technically already have cleaning working from Lesson 10. But it's... rough. The cleaning radius is static, the motion-to-clean mapping is basic, and there are a bunch of edge cases we haven't handled. What happens when the user holds perfectly still? What happens when they wave REALLY fast? What about when they move their whole body and the entire frame is "motion"? We need to tune this. Tuning is the unsexy part of game development that nobody talks about but EVERYONE notices when it's wrong. It's like seasoning food — nobody says "wow, this has the perfect amount of salt" but they DEFINITELY notice when it's wrong.

---

## The Theory

### The cleaning feedback loop

**Question:** What makes cleaning feel satisfying?

**Answer:** Three factors:

1. **Immediacy:** Dirt disappears the same frame motion is detected. Zero perceptible delay.
2. **Proportionality:** Bigger/faster movements clean larger areas. Small movements clean small areas.
3. **Persistence:** Cleaned areas stay clean. Progress is never lost. The player sees cumulative results.

### Motion intensity to cleaning radius mapping

**Question:** How should we map motion intensity (0–1) to cleaning radius?

**Answer:** A linear map is fine for a start, but we can improve it:

```
Too sensitive:  radius = intensity * 100  → light breathing cleans the window
Too insensitive: radius = intensity * 10  → player has to flail wildly
Goldilocks:     radius = MIN_RADIUS + intensity * (MAX_RADIUS - MIN_RADIUS)
```

We should also set a minimum intensity threshold — below this, no cleaning happens. This prevents the constant low-level webcam noise from slowly cleaning the window.

### Handling overlapping motion regions

**Question:** What happens when the user moves their entire body and every grid cell has motion?

**Answer:** Every cell triggers a `clean()` call. This is correct behavior — if the whole frame has motion, the whole dirt layer gets cleaned. But it means the player can "cheat" by waving wildly. For our kid-friendly game, this is fine. For a more challenging version, you could limit cleaning to a maximum number of regions per frame.

### Mirror effect

**Question:** Should the camera feed be mirrored?

**Answer:** Yes! When a user sees themselves on screen, they expect mirror behavior — wave your right hand and the right hand on screen should wave. By default, the camera feed is NOT mirrored. We need to flip it horizontally.

This is done with a canvas transform:

```ts
ctx.save();
ctx.scale(-1, 1);           // Flip horizontally
ctx.translate(-width, 0);   // Shift back into view
ctx.drawImage(video, 0, 0, width, height);
ctx.restore();
```

**Important:** This also means motion region coordinates need to be mirrored. A motion at normalized x=0.8 (right side of raw frame) should clean at x=0.2 (left side of mirrored display).

---

## The Assignment

### Step 1: Add mirror support

- Update the video drawing in `GameCanvas.tsx` to mirror the camera feed horizontally.
- When mapping motion regions to cleaning positions, mirror the X coordinate: `mirroredX = 1 - (region.x + region.width)`.

### Step 2: Refine the cleaning mechanic

- Create `src/game/cleaningConfig.ts` with tunable constants:
  - `MIN_CLEAN_RADIUS`: Minimum cleaning radius (e.g., 15 pixels).
  - `MAX_CLEAN_RADIUS`: Maximum cleaning radius (e.g., 60 pixels).
  - `MOTION_THRESHOLD`: Minimum motion intensity to trigger cleaning (e.g., 0.1).
  - `CLEAN_STRENGTH`: Multiplier for how aggressively dirt is erased (e.g., 1.0).

- Update the cleaning logic to:
  - Skip regions below the `MOTION_THRESHOLD`.
  - Map intensity to radius using: `MIN_CLEAN_RADIUS + intensity * (MAX_CLEAN_RADIUS - MIN_CLEAN_RADIUS)`.

### Step 3: Add a progress display

- Display the current dirt percentage as text on the canvas (top-right corner).
- Format as "XX% clean" (e.g., "43% clean").
- Use a readable font with a dark background rectangle behind the text for contrast.

### Step 4: Add visual feedback for cleaning

- When dirt is being actively cleaned (motion detected), draw a subtle glow/highlight at the cleaning location.
- A simple approach: draw a semi-transparent white circle at each cleaning point, with opacity proportional to intensity. This gives a "wiping shine" effect.

### Step 5: Verify

- Run `npm run dev`.
- Confirm the camera feed is mirrored (wave your right hand → on-screen right hand waves).
- Confirm cleaning works: dirt disappears where you move.
- Confirm the progress display updates.
- Confirm small movements produce small cleans, big movements produce big cleans.
- Run `npx tsc --noEmit` and `npm run lint`.

### Deliverables

1. `src/game/cleaningConfig.ts` with tunable constants.
2. Mirrored camera feed in the game canvas.
3. Mirrored motion-to-cleaning coordinate mapping.
4. Intensity-based cleaning radius.
5. On-canvas progress display.
6. Visual feedback (shine effect) at cleaning locations.
7. Zero type errors, zero lint errors.

---

## Liz's Nightmares

> **Liz Lemon:** Fine-tuning. The part where you change one number and everything breaks.

1. **Mirroring the video but not the motion coordinates:** If you mirror the video display (so the player sees themselves naturally) but DON'T mirror the motion coordinates, cleaning happens on the WRONG SIDE of the screen. The player waves their right hand, and dirt disappears on the LEFT. It's disorienting and wrong. Mirror BOTH the video AND the cleaning positions.

2. **Mirroring the dirt layer too:** The dirt layer should NOT be mirrored. It's already in "display space" (same orientation as what the user sees). Only the VIDEO and the MOTION COORDINATES need mirroring. If you mirror the dirt layer, it flips every frame and you get a disco strobe effect.

3. **Progress check causing frame drops:** Remember, `getDirtRemaining()` calls `getImageData()` which is expensive. If your progress check interval is too low (every frame), you'll see consistent 5-10ms frame time spikes. Every 30 frames is fine for visual feedback. The user won't notice the half-second delay.

4. **Cleaning radius of 0 at low intensity:** If `MIN_CLEAN_RADIUS` is 0 and intensity is very low, you get a cleaning radius of basically nothing. The player waves gently and nothing happens. They think the game is broken. Always have a minimum radius above zero (15px is good).

5. **Text rendering on canvas without background:** White text on a live video feed is often unreadable depending on the background scene. Always draw a semi-transparent dark rectangle behind canvas text. This is basic UX and I shouldn't have to say it but I've seen production games ship without it.

6. **Not using `ctx.save()` and `ctx.restore()` around the mirror transform:** If you set `ctx.scale(-1, 1)` and don't restore, EVERYTHING drawn after that is mirrored. Your dirt layer is mirrored. Your text is mirrored. Your progress display reads "naelc %34". Always save before transforming and restore after.
