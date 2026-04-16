# Lesson 13: Final Assembly — The WindowCleaner Game

## The Pitch

**Tracy Jordan:**
> THIS IS IT! The FINAL LESSON! We're putting ALL the pieces together like a VOLTRON ROBOT made of CODE! We got the camera, the canvas, the dirt, the bubbles, the motion detection — now we're combining them into ONE MEGA GAME COMPONENT! I like final assembly so much I want to take it behind the middle school and get it pregnant! This is the SEASON FINALE, people! Like when I starred in "Who Dat Ninja" — everything comes together at the end!

**Jack Donaghy:**
> Lemon, this is the *Integration Phase* — the final and most critical stage of our Six Sigma DMAIC process. We've Defined (Phase 0), Measured (motion detection), Analyzed (frame differencing), and Improved (cleaning mechanics, particles). Now we Control — assembling every component into a unified, production-grade product. In manufacturing, this is where the engine, transmission, body, and electronics become a car. Individually, they're components. Together, they're a BMW. The art of integration is ensuring that each component's interface is clean, responsibilities are clear, and the composition is maintainable. This is what separates a senior engineer from a junior one: juniors build features. Seniors compose systems.

**Liz Lemon:**
> Okay. Deep breath. We're assembling EVERYTHING. And the thing about "everything" is that "everything" has a lot of moving parts. We have hooks that depend on refs, refs that depend on DOM elements, effects that depend on state, state that depends on user input, and rendering that depends on all of the above. The order of operations matters. The cleanup order matters. The render order matters. If we get the layering wrong, the dirt draws under the video. If we get the state transitions wrong, the game loop runs during the win screen. If we get the cleanup wrong, we leak camera streams when the user navigates away. But — and this is the hopeful part — we've been building each piece with clean interfaces and proper lifecycle management for 12 lessons. If we did our job right, composition should be... smooth? Is that possible? I don't trust it.

---

## The Theory

### Component composition architecture

**Question:** How should we structure the final game?

**Answer:** Our architecture separates concerns into three layers:

```
┌─────────────────────────────────────────────┐
│  App.tsx                                     │
│  └── WindowCleanerGame (game orchestrator)   │
│       ├── useWebcam (camera access)          │
│       ├── useAnimationFrame (game loop)      │
│       ├── useMotionDetection (input)         │
│       ├── DirtLayer (game object)            │
│       ├── ParticleSystem (effects)           │
│       └── GameState (state machine)          │
└─────────────────────────────────────────────┘
```

**Layer 1: Hooks** — Manage browser APIs and React lifecycle.
- `useWebcam`: Camera access, stream lifecycle.
- `useAnimationFrame`: rAF loop, cleanup.
- `useMotionDetection`: Frame differencing, offscreen canvas.

**Layer 2: Game Objects** — Pure TypeScript classes with no React dependency.
- `DirtLayer`: Procedural generation, cleaning, progress tracking.
- `ParticleSystem` / `Bubble`: Celebration particles.
- `GameState`: State machine.

**Layer 3: Rendering** — Canvas drawing functions, compositing.
- Video rendering (mirrored).
- Dirt compositing.
- Particle drawing.
- HUD overlays (progress, win screen, ready screen).

### The render pipeline order

**Question:** What's the correct drawing order for each game state?

**Answer:**

**State: `ready`**
1. Clear canvas
2. Draw mirrored video
3. Draw "ready" overlay (title + instructions + "Click to Start")

**State: `playing`**
1. Clear canvas
2. Draw mirrored video
3. Detect motion → clean dirt at mirrored positions
4. Draw shine effects at cleaning locations
5. Draw dirt overlay
6. Draw progress HUD

**State: `won`**
1. Clear canvas
2. Draw mirrored video
3. Draw remaining dirt (mostly clean)
4. Update + draw bubble particles
5. Draw win screen overlay

### Cleanup choreography

**Question:** When the game component unmounts, what needs to be cleaned up?

**Answer:** In order of importance:

1. **Cancel animation frame** — Stops the game loop. (Handled by `useAnimationFrame`.)
2. **Stop camera tracks** — Releases camera hardware. (Handled by `useWebcam`.)
3. **Clear particle system** — Prevents particle objects from lingering. (GC handles this, but explicit clear is tidy.)

Because our hooks manage their own cleanup in `useEffect` return functions, the component just needs to unmount. React calls the cleanup functions in reverse order of effect registration. This is one of the key benefits of the hooks architecture — cleanup is co-located with setup.

### File organization

**Question:** What's the final file tree for the game?

**Answer:**

```
src/
├── components/
│   └── WindowCleanerGame.tsx    ← Main game component (this lesson)
├── game/
│   ├── Bubble.ts                ← Particle class
│   ├── DirtLayer.ts             ← Dirt management
│   ├── ParticleSystem.ts        ← Particle lifecycle
│   ├── cleaningConfig.ts        ← Tunable constants
│   └── gameState.ts             ← State machine
├── hooks/
│   ├── useAnimationFrame.ts     ← rAF loop
│   ├── useMotionDetection.ts    ← Frame differencing
│   └── useWebcam.ts             ← Camera access
├── types/
│   └── motion.ts                ← Motion detection types
├── App.tsx                      ← Root component
├── main.tsx                     ← Entry point
├── index.css                    ← Tailwind import
└── vite-env.d.ts                ← Vite types
```

---

## The Assignment

### Step 1: Create the WindowCleanerGame component

- Create `src/components/WindowCleanerGame.tsx`.
- This is the **main game component** that composes everything.
- It should:
  - Use `useWebcam()` for camera access.
  - Use `useState<GameState>` for state management.
  - Use `useRef` for: canvas element, rendering context, DirtLayer, ParticleSystem, frame counter, dirt remaining.
  - Use `useAnimationFrame` with a comprehensive frame callback.
  - Use `useMotionDetection` for input processing.
  - Handle canvas click events for state transitions.
  - Render the `<canvas>` and hidden `<video>` elements.

### Step 2: Implement the frame callback

The frame callback should be a single function that handles ALL game states:

```
function onFrame(deltaTime: number): void {
  // 1. Get context + video (with null guards)
  // 2. Clear canvas
  // 3. Draw mirrored video
  // 4. Switch on gameState:
  //    - "ready": draw ready screen
  //    - "playing": detect motion → clean → shine → dirt → progress HUD → check win
  //    - "won": dirt → update/draw particles → win screen
}
```

### Step 3: Implement the click handler

- On click when `ready`: reset dirt, generate new dirt, clear particles, transition to `playing`.
- On click when `playing`: do nothing (motion is the input).
- On click when `won`: transition to `ready`.

### Step 4: Update App.tsx

- Replace the current GameCanvas + useWebcam usage with a single `<WindowCleanerGame />` component.
- App.tsx should be minimal — just layout and the game component.

### Step 5: Add the cursor style

- When the game state is `ready` or `won`, change the canvas cursor to `pointer` (indicating clickability).
- When `playing`, set cursor to `none` (the player's hands are the input, not the mouse).

### Step 6: Full verification

Run all checks:

```bash
# Type checking
npx tsc --noEmit

# Linting
npm run lint

# Formatting
npm run format:check

# Dev server
npm run dev

# Production build
npm run build
```

Test the game flow:
1. Page loads → camera initializes → "Ready" screen appears.
2. Click → dirt generates → game starts.
3. Wave hands → dirt clears, progress updates, shine effects visible.
4. Clean 95%+ → "You Win!" with bubble celebration.
5. Click → back to ready screen → can play again.

### Deliverables

1. `src/components/WindowCleanerGame.tsx` — the complete, self-contained game component.
2. Updated `src/App.tsx` — clean root component using WindowCleanerGame.
3. Full game loop: ready → playing → won → ready.
4. All checks pass: `tsc --noEmit`, `lint`, `format:check`, `build`.
5. Zero type errors, zero lint errors, clean production build.

---

## Liz's Nightmares

> **Liz Lemon:** The final assembly. Where twelve lessons of careful work either come together beautifully or collapse into a pile of runtime errors. No pressure. No pressure AT ALL.

1. **Importing from the wrong path:** With files in `hooks/`, `game/`, `components/`, and `types/`, it's easy to typo an import path. TypeScript catches this at compile time, but the error message ("Cannot find module '../game/DirtLyer'") can be cryptic if you don't spot the typo. Use your editor's autocomplete. Trust it more than your typing.

2. **Calling `setGameState` inside the animation frame:** `setState` triggers a re-render. If you call it every frame (e.g., setting `playing` when already `playing`), you trigger 60 re-renders per second. Only call `setGameState` when the state actually CHANGES. Guard it: `if (gameState === "playing" && shouldWin) setGameState("won")`.

3. **Drawing in the wrong order:** The canvas has no z-index. The last thing drawn is on top. If you draw the dirt AFTER the win screen, the dirt covers the "You Win!" text. If you draw bubbles BEFORE the dirt, dirt covers the bubbles. Drawing order IS z-order. Write it down. Review it. Test it.

4. **Not resetting ALL state on "Play Again":** When transitioning from `won` to `ready` to `playing`, you must reset: dirt layer (reset + generate), particle system (clear), frame counter (reset to 0), dirt remaining (reset to 1). If you forget the frame counter, the progress check fires immediately (frame 30 of the last game) and may show stale progress.

5. **Leaking the game loop during hot module replacement (HMR):** Vite's HMR preserves state across file edits. If you edit a file while the game is running, the old component unmounts and the new one mounts. The `useAnimationFrame` cleanup should handle this, but if you're storing the rAF ID outside of the hook (bad practice), the old loop might survive HMR. Keep ALL loop state inside hooks. Period.

6. **Building for production without testing:** `npm run build` runs `tsc -b && vite build`. The `tsc -b` step does a full type check. The `vite build` step bundles and minifies. If either fails, you have a problem. Always run `npm run build` before deploying. A green dev server means nothing if the production build fails.

7. **Forgetting this is the culmination of 13 lessons:** If something doesn't work, the bug could be in ANY of the layers: webcam hook, animation loop, motion detection, dirt layer, particle system, state machine, or rendering order. Debug systematically. Check each layer independently. Use `console.log` shamelessly. There is no shame in `console.log`. There IS shame in shipping a broken game.
