# Lesson 12 — Answer Sheet: Win State & Bubble Particle System

## The Implementation

### File: `src/game/gameState.ts`

```ts
/**
 * GameState — Finite state machine for the Window Cleaner game.
 *
 * States:
 *   "ready"   → Initial state. Waiting for user to start.
 *   "playing" → Game is active. Motion detection + cleaning enabled.
 *   "won"     → Window is clean. Celebrating with bubbles.
 *
 * Valid transitions:
 *   ready → playing   (user clicks "Start")
 *   playing → won     (dirt below threshold)
 *   won → ready       (user clicks "Play Again")
 *
 * Using a union type instead of an enum for zero runtime cost
 * and better TypeScript inference.
 */
export type GameState = "ready" | "playing" | "won";

/**
 * Validates and performs a state transition.
 * Returns the new state if the transition is valid, or the current state if not.
 *
 * This prevents invalid transitions like "won" → "playing" (must go through "ready").
 *
 * @param current - The current game state.
 * @param next - The desired next state.
 * @returns The resulting state after the transition attempt.
 */
export function transition(current: GameState, next: GameState): GameState {
  const validTransitions: Record<GameState, GameState[]> = {
    ready: ["playing"],
    playing: ["won"],
    won: ["ready"],
  };

  const allowed = validTransitions[current];
  if (allowed.includes(next)) {
    return next;
  }

  // Invalid transition — return current state unchanged.
  // In development, you might want to console.warn here.
  return current;
}
```

### File: `src/game/Bubble.ts`

```ts
/**
 * Bubble — A single particle in the celebration effect.
 *
 * Physics:
 *   - Floats upward (negative vy because canvas Y increases downward)
 *   - Drifts horizontally with slight randomness
 *   - Wobbles via sinusoidal x-offset for natural floating motion
 *   - Fades out as it ages
 *   - Has a glassy appearance with specular highlight
 */

/** Wobble frequency — controls how fast the bubble oscillates horizontally */
const WOBBLE_FREQUENCY = 0.05;
/** Wobble amplitude — controls how far the bubble drifts side to side */
const WOBBLE_AMPLITUDE = 1.5;

export class Bubble {
  /** Current X position (canvas pixels) */
  x: number;
  /** Current Y position (canvas pixels) */
  y: number;
  /** Horizontal velocity (pixels per frame) */
  private readonly vx: number;
  /** Vertical velocity (pixels per frame, negative = upward) */
  private readonly vy: number;
  /** Bubble radius (pixels) */
  private readonly radius: number;
  /** Current opacity (0–1) */
  opacity: number;
  /** Number of frames this bubble has existed */
  private age: number;
  /** Maximum number of frames before the bubble dies */
  private readonly lifetime: number;
  /** Starting X for wobble calculation */
  private readonly startX: number;

  /**
   * Creates a new bubble.
   *
   * @param x - Initial X position
   * @param y - Initial Y position
   * @param config - Optional overrides for physics parameters
   */
  constructor(
    x: number,
    y: number,
    config?: {
      vx?: number;
      vy?: number;
      radius?: number;
      lifetime?: number;
    },
  ) {
    this.x = x;
    this.y = y;
    this.startX = x;
    this.vx = config?.vx ?? (Math.random() - 0.5) * 2; // -1 to 1
    this.vy = config?.vy ?? -(1 + Math.random() * 2); // -1 to -3 (upward)
    this.radius = config?.radius ?? 5 + Math.random() * 15; // 5 to 20
    this.opacity = 1;
    this.age = 0;
    this.lifetime = config?.lifetime ?? 90 + Math.floor(Math.random() * 60); // 90-150 frames
  }

  /**
   * Updates the bubble's position, wobble, and opacity.
   *
   * @returns true if the bubble is still alive, false if it should be removed.
   */
  update(): boolean {
    this.age++;

    // Move by velocity
    this.x += this.vx;
    this.y += this.vy;

    // Add sinusoidal wobble for natural floating motion.
    // sin(age * frequency) creates a smooth oscillation.
    // This is added ON TOP of the base velocity for a compound motion.
    this.x = this.startX + this.vx * this.age +
      Math.sin(this.age * WOBBLE_FREQUENCY) * WOBBLE_AMPLITUDE * this.radius;

    // Fade out based on age/lifetime ratio.
    // Starts fading at 50% lifetime for a gradual effect.
    const lifeRatio = this.age / this.lifetime;
    if (lifeRatio > 0.5) {
      this.opacity = 1 - (lifeRatio - 0.5) * 2; // Linear fade from 1 to 0
    }

    // Dead if: too old, invisible, or floated off-screen (y < -50)
    return this.age < this.lifetime && this.opacity > 0 && this.y > -50;
  }

  /**
   * Draws the bubble on the canvas.
   *
   * Appearance:
   *   - Semi-transparent body with a light blue tint
   *   - Thin border for definition
   *   - Small white specular highlight for glassy look
   *
   * @param ctx - The 2D rendering context of the canvas.
   */
  draw(ctx: CanvasRenderingContext2D): void {
    if (this.opacity <= 0) return;

    ctx.save();
    ctx.globalAlpha = this.opacity;

    // --- Bubble body ---
    // Semi-transparent light blue circle
    ctx.beginPath();
    ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
    ctx.fillStyle = "rgba(150, 220, 255, 0.3)";
    ctx.fill();

    // --- Bubble border ---
    // Slightly more opaque blue for definition
    ctx.strokeStyle = "rgba(200, 240, 255, 0.5)";
    ctx.lineWidth = 1;
    ctx.stroke();

    // --- Specular highlight ---
    // Small white circle in the upper-left quadrant simulates light reflection
    const highlightX = this.x - this.radius * 0.3;
    const highlightY = this.y - this.radius * 0.3;
    const highlightRadius = this.radius * 0.2;

    ctx.beginPath();
    ctx.arc(highlightX, highlightY, highlightRadius, 0, Math.PI * 2);
    ctx.fillStyle = "rgba(255, 255, 255, 0.7)";
    ctx.fill();

    ctx.restore();
  }
}
```

### File: `src/game/ParticleSystem.ts`

```ts
import { Bubble } from "./Bubble";

/**
 * ParticleSystem — Manages the lifecycle of multiple Bubble particles.
 *
 * Responsibilities:
 *   1. Creating (emitting) new bubbles at specified positions.
 *   2. Updating all active bubbles each frame.
 *   3. Removing dead bubbles to prevent memory growth.
 *   4. Drawing all active bubbles to the canvas.
 *
 * Performance:
 *   - Maximum particle count is capped to prevent frame rate degradation.
 *   - Dead particles are removed via filter (GC-friendly for small arrays).
 *   - For our use case (~100-200 bubbles), this is more than sufficient.
 */

/** Maximum number of simultaneous active particles */
const MAX_PARTICLES = 200;

export class ParticleSystem {
  /** Array of active bubble particles */
  private particles: Bubble[] = [];

  /**
   * Emits a burst of bubbles at the given position.
   *
   * @param x - Center X position for the emission.
   * @param y - Center Y position for the emission.
   * @param count - Number of bubbles to create.
   */
  emit(x: number, y: number, count: number): void {
    for (let i = 0; i < count; i++) {
      // Respect the particle cap
      if (this.particles.length >= MAX_PARTICLES) break;

      // Randomize position slightly around the emission point
      const offsetX = (Math.random() - 0.5) * 60;
      const offsetY = (Math.random() - 0.5) * 40;

      this.particles.push(new Bubble(x + offsetX, y + offsetY));
    }
  }

  /**
   * Emits bubbles across the entire canvas — used for the win celebration.
   *
   * Creates multiple emission points spread across the canvas
   * for a full-screen bubble effect.
   *
   * @param canvasWidth - Width of the canvas in pixels.
   * @param canvasHeight - Height of the canvas in pixels.
   * @param totalCount - Total number of bubbles to emit.
   */
  emitCelebration(
    canvasWidth: number,
    canvasHeight: number,
    totalCount: number,
  ): void {
    const emissionPoints = 5; // Number of spawn locations
    const perPoint = Math.ceil(totalCount / emissionPoints);

    for (let i = 0; i < emissionPoints; i++) {
      // Distribute emission points across the bottom half of the canvas
      const x = (canvasWidth / (emissionPoints + 1)) * (i + 1);
      const y = canvasHeight * (0.6 + Math.random() * 0.3); // Bottom 40%
      this.emit(x, y, perPoint);
    }
  }

  /**
   * Updates all active particles and removes dead ones.
   * Call this once per animation frame.
   */
  update(): void {
    // Update each particle and filter out dead ones.
    // Bubble.update() returns false when the bubble should be removed.
    this.particles = this.particles.filter((particle) => particle.update());
  }

  /**
   * Draws all active particles to the canvas.
   *
   * @param ctx - The 2D rendering context of the canvas.
   */
  draw(ctx: CanvasRenderingContext2D): void {
    for (const particle of this.particles) {
      particle.draw(ctx);
    }
  }

  /**
   * Returns true if any particles are still alive (animating).
   * Used to know when the celebration animation is complete.
   */
  isActive(): boolean {
    return this.particles.length > 0;
  }

  /**
   * Removes all particles. Used when resetting the game.
   */
  clear(): void {
    this.particles = [];
  }

  /**
   * Returns the current number of active particles.
   * Useful for debugging and performance monitoring.
   */
  get count(): number {
    return this.particles.length;
  }
}
```

### Win state rendering

The following functions should be used in your `GameCanvas` component to render the win state:

```ts
/**
 * Draws the "You Win!" celebration screen.
 *
 * Renders:
 *   1. A semi-transparent dark overlay to dim the camera feed.
 *   2. Large "Window Clean!" text.
 *   3. A "Click to Play Again" prompt.
 */
function drawWinScreen(ctx: CanvasRenderingContext2D): void {
  // Semi-transparent overlay to dim the background
  ctx.fillStyle = "rgba(0, 0, 0, 0.4)";
  ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  // "Window Clean!" text
  ctx.save();
  ctx.fillStyle = "#4ade80"; // green-400
  ctx.font = "bold 48px sans-serif";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.shadowColor = "rgba(0, 0, 0, 0.5)";
  ctx.shadowBlur = 10;
  ctx.fillText("🫧 Window Clean! 🫧", CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2 - 30);

  // "Play Again" prompt
  ctx.fillStyle = "#ffffff";
  ctx.font = "24px sans-serif";
  ctx.shadowBlur = 0;
  ctx.fillText("Click to Play Again", CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2 + 30);
  ctx.restore();
}

/**
 * Draws the "Ready" screen before the game starts.
 */
function drawReadyScreen(ctx: CanvasRenderingContext2D): void {
  // Semi-transparent overlay
  ctx.fillStyle = "rgba(0, 0, 0, 0.5)";
  ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  ctx.save();
  ctx.fillStyle = "#ffffff";
  ctx.font = "bold 36px sans-serif";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillText("Window Cleaner", CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2 - 40);

  ctx.font = "24px sans-serif";
  ctx.fillStyle = "#94a3b8";
  ctx.fillText(
    "Wave your hands to clean the window!",
    CANVAS_WIDTH / 2,
    CANVAS_HEIGHT / 2 + 10,
  );

  ctx.fillStyle = "#38bdf8"; // sky-400
  ctx.fillText("Click to Start", CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2 + 50);
  ctx.restore();
}
```

### State machine integration in GameCanvas

The key changes to `GameCanvas`:

```ts
// State management
const [gameState, setGameState] = useState<GameState>("ready");
const particleSystemRef = useRef<ParticleSystem>(new ParticleSystem());

// Click handler for state transitions
function handleCanvasClick(): void {
  if (gameState === "ready") {
    // Reset dirt and start playing
    const dirtLayer = getDirtLayer();
    dirtLayer.reset();
    dirtLayer.generate();
    particleSystemRef.current.clear();
    dirtRemainingRef.current = 1;
    setGameState("playing");
  } else if (gameState === "won") {
    // Return to ready state
    setGameState("ready");
  }
}

// In the animation frame callback:
useAnimationFrame(() => {
  // ... setup ctx, video ...

  // Step 1: Clear
  ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  // Step 2: Draw mirrored video (always, in all states)
  ctx.save();
  ctx.scale(-1, 1);
  ctx.translate(-CANVAS_WIDTH, 0);
  ctx.drawImage(video, 0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
  ctx.restore();

  // Step 3: State-specific rendering
  if (gameState === "playing") {
    // Motion detection + cleaning + dirt overlay + HUD
    // (same as Lesson 11)
    // ...

    // Check win condition
    if (dirtRemainingRef.current < DIRT_WIN_THRESHOLD) {
      particleSystemRef.current.emitCelebration(CANVAS_WIDTH, CANVAS_HEIGHT, 100);
      setGameState("won");
    }
  }

  if (gameState === "playing" || gameState === "won") {
    // Draw dirt (in won state, it's mostly clean but may have traces)
    dirtLayer.draw(ctx);
  }

  if (gameState === "won") {
    // Update and draw bubbles
    particleSystemRef.current.update();
    particleSystemRef.current.draw(ctx);
    drawWinScreen(ctx);
  }

  if (gameState === "ready") {
    drawReadyScreen(ctx);
  }
}, isReady);

// Add onClick to canvas
return (
  <canvas
    ref={/* ... */}
    onClick={handleCanvasClick}
    // ...
  />
);
```

---

## Executive Summary

**Jack Donaghy:**

> We have now completed three critical architectural components. Let me frame each in terms of its business value.

> 1. **The state machine is governance.** In any well-run organization, there are clear stages: proposal → approval → execution → review. You don't go from execution back to proposal without going through review. Our game state machine enforces the same discipline: ready → playing → won → ready. No shortcuts. No impossible states. No bugs from inconsistent flags. This is *process governance through architecture*, and it's the foundation of operational reliability.

> 2. **The particle system is our premium product tier.** A game without celebration effects is functional. A game WITH celebration effects is *delightful*. The bubble particle system — floating, wobbling, fading, sparkling — turns a task completion into a moment of joy. In consumer product terms, this is the difference between a Honda Civic and a BMW 5 Series. Both get you there. Only one makes you feel something when you arrive.

> 3. **Object-oriented particle architecture is interview-ready code.** The `Bubble` class encapsulates its own physics and rendering. The `ParticleSystem` manages the collection lifecycle. This is textbook *single responsibility principle* — each class does one thing well. In a technical interview, whiteboarding this architecture demonstrates: OOP fundamentals, game loop integration, memory management awareness, and render pipeline understanding. That's four senior-level competencies from one feature.

> 4. **The celebration emission pattern is designed, not random.** We don't dump 100 bubbles at a single point — we distribute them across 5 emission points on the lower half of the canvas. This creates a rising curtain of bubbles instead of a concentrated blob. That's *visual design through mathematics*. It costs us 10 extra lines of code and transforms the experience.

> One lesson remains: the final assembly. Everything we've built — webcam hook, canvas rendering, game loop, motion detection, dirt layer, cleaning mechanics, particle system, state machine — gets composed into a single, clean, production-ready game component. This is the *integration phase* of Six Sigma. Where individual components become a product.
