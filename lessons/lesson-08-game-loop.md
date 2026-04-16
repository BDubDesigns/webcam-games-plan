# Lesson 08: The Game Loop — requestAnimationFrame in React

## The Pitch

**Tracy Jordan:**
> THE GAME IS ALIVE! It's MOVING! We got this thing called `requestAnimationFrame` which is like telling the browser "HEY! Draw another picture! NOW ANOTHER ONE! NOW ANOTHER!" SIXTY TIMES A SECOND! It's like a flipbook but inside your computer! I like game loops so much I want to take it behind the middle school and get it pregnant! This is the HEARTBEAT of the game, people! Without this, we just got a PHOTOGRAPH, not a MOVIE!

**Jack Donaghy:**
> The game loop is the production cadence of our entire operation. In manufacturing, we call this the *takt time* — the rhythm at which units move through the production line. `requestAnimationFrame` synchronizes our rendering with the browser's display refresh rate (typically 60Hz). This is not a `setInterval` — we don't run on our own clock. We run on the *browser's* clock, which means zero wasted frames, zero screen tearing, and optimal power consumption. That's operational efficiency at the render pipeline level.

**Liz Lemon:**
> Okay, this is the lesson where everything gets COMPLICATED. Here's the problem: `requestAnimationFrame` is a browser API. It takes a callback and calls it once before the next repaint. To create a loop, you call `requestAnimationFrame` INSIDE the callback. Simple enough. BUT — we're in React. React Strict Mode double-mounts. `useEffect` runs after render. If we start a `requestAnimationFrame` loop in `useEffect` and don't cancel it in cleanup, we get TWO loops running simultaneously. Each one calls `requestAnimationFrame` inside itself, creating an exponentially growing zombie army of animation callbacks. I've seen this happen. It's not fun. We need to do this RIGHT.

---

## The Theory

### What is requestAnimationFrame?

**Question:** Why use `requestAnimationFrame` (rAF) instead of `setInterval`?

**Answer:**

| Feature | `setInterval(fn, 16)` | `requestAnimationFrame(fn)` |
|---------|----------------------|-----------------------------|
| Timing | Fixed interval (even if tab is hidden) | Synced to display refresh |
| Power efficiency | Runs even when tab is backgrounded | Paused when tab is hidden |
| Frame alignment | May not align with vsync | Always aligned with vsync |
| Drift | Accumulates timing errors | Precise to display refresh |
| Cancellation | `clearInterval(id)` | `cancelAnimationFrame(id)` |

`requestAnimationFrame` was designed specifically for animation. The browser controls *when* your callback fires, ensuring it runs at the optimal time for smooth visual updates.

### The basic rAF loop pattern

```ts
function loop(): void {
  // 1. Update game state
  // 2. Draw to canvas
  // 3. Request the NEXT frame
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop); // Start the loop
```

Each call to `requestAnimationFrame(loop)` schedules `loop` to run once before the next repaint. By calling `requestAnimationFrame` inside `loop`, we create a perpetual cycle that runs at ~60fps.

### The problem with rAF in React

**Question:** What goes wrong when you naively use rAF in a `useEffect`?

**Answer:** Consider this:

```ts
useEffect(() => {
  function loop() {
    draw();
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);
}, []);
```

**Problem 1: Strict Mode double-mount.** In development, React mounts → unmounts → remounts. If cleanup doesn't cancel the rAF, the first mount's loop keeps running alongside the second mount's loop. Two loops = double rendering = performance disaster.

**Problem 2: Stale closures.** The `loop` function closes over variables from the effect scope. If state changes after the effect runs, `loop` still sees the old values. This is the "stale closure" problem.

### The solution: useRef for the animation frame ID

We store the rAF ID in a ref and cancel it in the cleanup function:

```ts
useEffect(() => {
  let frameId: number;
  function loop() {
    draw();
    frameId = requestAnimationFrame(loop);
  }
  frameId = requestAnimationFrame(loop);
  
  return () => cancelAnimationFrame(frameId);
}, []);
```

The cleanup function calls `cancelAnimationFrame(frameId)`, which stops the loop dead. When Strict Mode remounts, a fresh loop starts.

### The useAnimationFrame custom hook

**Question:** Should we extract the animation loop into a reusable hook?

**Answer:** Yes. A `useAnimationFrame` hook:

1. Accepts a callback function (the "tick" or "frame" handler).
2. Starts a rAF loop on mount.
3. Cancels the loop on unmount.
4. Uses `useRef` for the callback to avoid stale closure issues.

The key insight: the *callback ref* pattern. Instead of putting the callback in the dependency array (which would restart the loop on every render), we store the latest callback in a ref. The rAF loop always reads from the ref, getting the most recent callback without restarting the loop.

### Delta time

**Question:** Why pass the timestamp to the frame callback?

**Answer:** `requestAnimationFrame` passes a `DOMHighResTimeStamp` to your callback. This is the time in milliseconds since the page loaded. By tracking the delta between frames, you can:

1. Make animations frame-rate independent (works on 60Hz and 120Hz displays).
2. Detect dropped frames (delta > 20ms on a 60Hz display).
3. Implement time-based game logic (e.g., "3 second countdown").

---

## The Assignment

### Step 1: Create the useAnimationFrame hook

- Create `src/hooks/useAnimationFrame.ts`.
- The hook should:
  - Accept a callback: `(deltaTime: number) => void`.
  - Accept a boolean `isActive` parameter (default true). When false, the loop doesn't run.
  - Use `useRef` to store the latest callback (avoiding stale closures).
  - Use `useRef` to store the previous timestamp (for delta calculation).
  - Use `useEffect` to start/stop the rAF loop based on `isActive`.
  - Properly cancel the animation frame in the cleanup function.

### Step 2: Update GameCanvas to use the animation loop

- Replace the single-frame draw in `GameCanvas` with a continuous loop using `useAnimationFrame`.
- The frame callback should:
  1. Clear the canvas.
  2. Draw the current video frame.
  3. (For now, that's it — motion detection and game logic come in later lessons.)
- The loop should only be active when `isReady` is true.

### Step 3: Verify

- Run `npm run dev`.
- The canvas should now show a LIVE camera feed (updating every frame), not a single snapshot.
- Wave your hand — you should see smooth motion.
- Open DevTools → Performance tab → record a few seconds. Verify:
  - rAF callbacks fire at ~16ms intervals (60fps).
  - There's only ONE loop running (not two from Strict Mode).
- Navigate away and back — the loop should stop and restart cleanly.
- Run `npx tsc --noEmit` and `npm run lint` — zero errors.

### Deliverables

1. `src/hooks/useAnimationFrame.ts` — reusable animation loop hook.
2. Updated `GameCanvas.tsx` using the hook for continuous video rendering.
3. Proper cleanup — no zombie animation loops in Strict Mode.
4. Frame-rate independent delta time calculation.
5. Zero type errors, zero lint errors.

---

## Liz's Nightmares

> **Liz Lemon:** The game loop. The heartbeat of the game. Also the source of approximately 80% of all React game development bugs. Let me walk you through the ways this can go wrong.

1. **Not cancelling the animation frame on cleanup:** If you forget `cancelAnimationFrame(frameId)` in your `useEffect` cleanup, React Strict Mode will create two loops. Then four. Then eight. Your browser tab will consume 100% CPU and Chrome will offer to kill it. I've been there. I was eating a Hot Pocket and my laptop fan started screaming.

2. **Stale closure on the callback:** If you pass the callback directly to `useEffect`'s dependency array, the loop restarts every time the callback changes (which is every render, because functions are recreated each render). If you DON'T include it in the dependency array, the loop uses the stale initial callback forever. The solution: store the latest callback in a ref. The loop reads from the ref, always getting the fresh version. ESLint's `exhaustive-deps` rule will complain — that's why the ref pattern exists.

3. **Drawing without clearing:** If you call `ctx.drawImage()` without first calling `ctx.clearRect()`, the new frame draws ON TOP of the old frame. For a full-canvas video draw, this is invisible (new frame covers old frame). But when we add the dirt layer and compositing, this will cause visual artifacts. Build the habit now: clear first, draw second.

4. **Assuming 60fps:** `requestAnimationFrame` targets the display's refresh rate, which is 60Hz on most monitors. But on 120Hz displays, it fires 120 times per second. On a busy tab, it might drop to 30fps. NEVER assume a fixed frame rate. Use delta time for all time-based logic. This is why we calculate `deltaTime` — it makes our game work correctly regardless of frame rate.

5. **Starting the loop before the video is ready:** If `isReady` is false and you start drawing `ctx.drawImage(video, ...)`, you'll draw a black rectangle because the video has no frames yet. The `isActive` parameter on our hook gates this: the loop only runs when the camera is ready. Don't skip this.

6. **Not using `useRef` for the previous timestamp:** If you store the previous timestamp in `useState`, every frame update triggers a re-render, which triggers a re-draw, which is recursive and catastrophic. Timestamps are internal bookkeeping, not rendering data. Always use a ref for loop-internal state that doesn't affect the UI.
