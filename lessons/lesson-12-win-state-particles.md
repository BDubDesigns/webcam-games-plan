# Lesson 12: Win State & Bubble Particle System

## The Pitch

**Tracy Jordan:**
> BUBBLES! We need BUBBLES! When you clean the window, BUBBLES should FLOAT UP like you're in a BATH! But a bath for your EYES! I once took a bubble bath in a hot tub at the Four Seasons and I passed out for three hours, so I know a LOT about bubbles! I like particle systems so much I want to take it behind the middle school and get it pregnant! When the player WINS, it should look like a CELEBRATION! Like New Year's Eve but with SOAP!

**Jack Donaghy:**
> The win state and its accompanying particle effect serve a precise strategic function: *positive reinforcement through visual reward*. In behavioral economics, this is the *variable reward mechanism* — the dopamine hit that keeps users engaged. A clean window with no celebration is a PowerPoint slide. A clean window with an animated bubble explosion is an *experience*. The particle system is our product's "unboxing moment" — the first impression of quality that turns a user into an advocate. Also, particle systems demonstrate mastery of object-oriented game architecture. In an interview, being able to whiteboard a particle system from scratch is what separates a mid-level candidate from a senior one.

**Liz Lemon:**
> Okay, particle systems sound fun. And they ARE fun... to watch. Building one in a `requestAnimationFrame` loop inside React while managing object lifetimes and avoiding memory leaks? LESS fun. Here's the challenge: each bubble is an object with position, velocity, size, opacity, and a lifetime. Every frame, we update every bubble, remove dead ones, and draw the survivors. If we're not careful with object creation and garbage collection, we'll create thousands of bubble objects per second, garbage collect them, and see frame rate hitches. We need object pooling — or at minimum, a clean lifecycle. Also, the win state itself needs to be a proper state machine, not a boolean flag. Ready → Playing → Won. Clean transitions. No going backwards. No zombie game loops running after the win screen. Please.

---

## The Theory

### Game State Machine

**Question:** Why use a state machine instead of boolean flags?

**Answer:** Boolean flags like `isPlaying`, `hasWon`, `isStarted` lead to impossible states. With three booleans, you have 2³ = 8 possible combinations, but most are invalid (`isPlaying && hasWon`?). A state machine has ONE current state:

```
type GameState = "ready" | "playing" | "won";
```

Valid transitions:
```
ready → playing   (user starts the game)
playing → won     (dirt drops below threshold)
won → ready       (user clicks "Play Again")
```

Invalid transitions are impossible by construction. No flags. No ambiguity.

### What is a particle system?

**Question:** What are particles and how do they work?

**Answer:** A particle system is a collection of many small, simple objects (particles) that together create a visual effect (bubbles, fire, smoke, sparkles).

Each particle has:
- **Position** (x, y)
- **Velocity** (vx, vy) — how fast it moves per frame
- **Size** (radius)
- **Opacity** (alpha, 0–1)
- **Lifetime** (how many frames/ms until it dies)
- **Age** (how many frames/ms it has existed)

Each frame:
1. **Update** every particle: move by velocity, increment age, reduce opacity.
2. **Remove** dead particles (age > lifetime or opacity ≤ 0 or off-screen).
3. **Spawn** new particles if needed.
4. **Draw** surviving particles.

### Bubble physics

**Question:** How do bubbles behave?

**Answer:** Bubbles:
- **Float upward** (negative vy — remember, Y increases downward on canvas).
- **Drift horizontally** with slight randomness (small random vx).
- **Wobble** — sinusoidal x-offset makes them look like they're floating, not flying.
- **Fade out** as they age (decreasing opacity).
- **Vary in size** — random initial radius for visual variety.
- **Have a glassy look** — semi-transparent with a highlight/shine.

### Drawing a bubble on canvas

**Question:** How do we make a circle look like a bubble?

**Answer:** A simple approach:

1. Draw a circle with a semi-transparent fill (the bubble body).
2. Draw a smaller white circle offset to the upper-left (the specular highlight).
3. Optionally add a stroke (border) with slight opacity.

```ts
// Bubble body
ctx.beginPath();
ctx.arc(x, y, radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(150, 220, 255, ${opacity * 0.3})`;
ctx.fill();
ctx.strokeStyle = `rgba(200, 240, 255, ${opacity * 0.5})`;
ctx.stroke();

// Specular highlight
ctx.beginPath();
ctx.arc(x - radius * 0.3, y - radius * 0.3, radius * 0.2, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255, 255, 255, ${opacity * 0.6})`;
ctx.fill();
```

### Memory management for particles

**Question:** How do we avoid creating garbage?

**Answer:** Two strategies:

1. **Reuse arrays:** Instead of `filter()` (which creates a new array), use an in-place compaction. Or just use `filter()` — for a few hundred bubbles, the GC cost is negligible. Don't over-optimize.

2. **Limit particle count:** Cap the total number of active particles. If the cap is reached, either stop spawning or remove the oldest particle before adding a new one.

---

## The Assignment

### Step 1: Define the game state

- Create `src/game/gameState.ts` with:
  - A `GameState` type: `"ready" | "playing" | "won"`.
  - Optionally, transition functions that validate state changes.

### Step 2: Create the Bubble particle

- Create `src/game/Bubble.ts` with a `Bubble` class or interface:
  - Properties: `x`, `y`, `vx`, `vy`, `radius`, `opacity`, `age`, `lifetime`.
  - An `update()` method that:
    - Moves the bubble by velocity.
    - Adds sinusoidal wobble to x position.
    - Increments age.
    - Reduces opacity based on age/lifetime ratio.
    - Returns `true` if alive, `false` if dead.
  - A `draw(ctx)` method that draws the bubble with body + highlight.

### Step 3: Create the ParticleSystem

- Create `src/game/ParticleSystem.ts`:
  - Manages an array of `Bubble` instances.
  - `emit(x, y, count)`: Creates `count` new bubbles at position (x, y) with random velocities and sizes.
  - `update()`: Updates all particles, removes dead ones.
  - `draw(ctx)`: Draws all particles.
  - `isActive()`: Returns true if any particles are alive.

### Step 4: Create the win screen

- When the game state transitions to `"won"`:
  - Emit a burst of bubbles from multiple points across the canvas.
  - Draw a "You Win!" message centered on the canvas.
  - Draw a "Play Again" prompt.
  - Continue the animation loop to animate the bubbles.
  - Stop motion detection (no longer needed).

### Step 5: Integrate the state machine

- Update `GameCanvas` (or create a new wrapper component) to:
  - Start in "ready" state with a "Click to Start" prompt.
  - Transition to "playing" on click.
  - Transition to "won" when dirt is below threshold.
  - Transition back to "ready" on click from "won" state.

### Step 6: Verify

- Run `npm run dev`.
- Click to start → dirt appears over camera feed.
- Wave hands → dirt clears, progress updates.
- When 95%+ clean → "You Win!" appears with bubble animation.
- Click "Play Again" → dirt resets, game restarts.
- Run `npx tsc --noEmit` and `npm run lint`.

### Deliverables

1. `src/game/gameState.ts` — game state type and transitions.
2. `src/game/Bubble.ts` — particle with physics and rendering.
3. `src/game/ParticleSystem.ts` — particle lifecycle management.
4. Win screen with bubble animation and "Play Again" flow.
5. Full game state machine: ready → playing → won → ready.
6. Zero type errors, zero lint errors.

---

## Liz's Nightmares

> **Liz Lemon:** Particle systems and state machines. The part where game development goes from "fun project" to "why did I choose this career."

1. **No upper bound on particles:** If you keep emitting bubbles without limiting the array, you'll eventually have thousands of particles updating and drawing every frame. Set a cap (e.g., 200 max). Your frame rate will thank you.

2. **Particles continuing after component unmount:** If the animation loop is still running when the component unmounts, bubble objects keep getting updated in memory with no canvas to draw to. The `useAnimationFrame` hook handles loop cleanup, but make sure the particle system doesn't hold references that prevent garbage collection.

3. **State machine transitions going backward:** If you don't guard transitions, a late-arriving motion detection result could cause a `playing → won → playing` flip. Once `won`, stay won until the user explicitly restarts. Check state BEFORE processing motion.

4. **"Play Again" not resetting the dirt layer:** If you transition from `won` back to `ready` but forget to call `dirtLayer.reset()` and `dirtLayer.generate()`, the player starts with a clean window and instantly wins again. Reset EVERYTHING on restart.

5. **Drawing the win screen without stopping cleaning logic:** If motion detection still runs during the win state, the player's movements keep "cleaning" a window that's already clean. Waste of CPU. Gate motion detection behind `state === "playing"`.

6. **Bubbles appearing behind the dirt layer:** Draw order matters. During the win state: video → dirt (should be mostly clean) → bubbles → win text. If bubbles draw before dirt, the remaining 5% of dirt covers the bubbles. Ugly.
