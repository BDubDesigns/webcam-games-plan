# Lesson 13 — Answer Sheet: Final Assembly — The WindowCleaner Game

## The Implementation

### File: `src/components/WindowCleanerGame.tsx`

```tsx
import { useCallback, useRef, useState } from "react";
import { useWebcam } from "../hooks/useWebcam";
import { useAnimationFrame } from "../hooks/useAnimationFrame";
import { useMotionDetection } from "../hooks/useMotionDetection";
import { DirtLayer, DIRT_WIN_THRESHOLD } from "../game/DirtLayer";
import { ParticleSystem } from "../game/ParticleSystem";
import type { GameState } from "../game/gameState";
import { transition } from "../game/gameState";

import {
  MIN_CLEAN_RADIUS,
  MAX_CLEAN_RADIUS,
  MOTION_THRESHOLD,
  PROGRESS_CHECK_INTERVAL,
  SHINE_RADIUS_MULTIPLIER,
  SHINE_BASE_OPACITY,
} from "../game/cleaningConfig";

// =============================================================================
// Constants
// =============================================================================

const CANVAS_WIDTH = 640;
const CANVAS_HEIGHT = 480;

/** Number of bubbles to emit when the player wins */
const WIN_BUBBLE_COUNT = 100;

// =============================================================================
// Component
// =============================================================================

/**
 * WindowCleanerGame — The complete Window Cleaner game component.
 *
 * Composes all game systems:
 *   - useWebcam: Camera access and lifecycle
 *   - useAnimationFrame: 60fps game loop
 *   - useMotionDetection: Frame differencing for input
 *   - DirtLayer: Procedural dirt generation and cleaning
 *   - ParticleSystem: Celebration bubble effects
 *   - GameState: Finite state machine
 *
 * Render pipeline per frame:
 *   1. Clear canvas
 *   2. Draw mirrored video (all states)
 *   3. State-specific rendering (ready screen / gameplay / win screen)
 *
 * User interaction:
 *   - Click to start (ready → playing)
 *   - Wave hands to clean (playing)
 *   - Click to play again (won → ready)
 */
export function WindowCleanerGame(): React.JSX.Element {
  // --- Hooks ---
  const { videoRef, isReady, error } = useWebcam();

  // --- State ---
  const [gameState, setGameState] = useState<GameState>("ready");
  const { detectMotion } = useMotionDetection(videoRef, gameState === "playing");

  // --- Refs (no re-renders needed for these) ---
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const ctxRef = useRef<CanvasRenderingContext2D | null>(null);
  const dirtLayerRef = useRef<DirtLayer | null>(null);
  const particleSystemRef = useRef<ParticleSystem>(new ParticleSystem());
  const frameCountRef = useRef<number>(0);
  const dirtRemainingRef = useRef<number>(1);

  // We store gameState in a ref so the animation frame callback always
  // has the latest value without restarting the loop.
  // (The useState value in the closure might be stale.)
  const gameStateRef = useRef<GameState>(gameState);
  gameStateRef.current = gameState;

  // --- Dirt Layer Initialization ---
  const getDirtLayer = useCallback((): DirtLayer => {
    if (!dirtLayerRef.current) {
      const layer = new DirtLayer(CANVAS_WIDTH, CANVAS_HEIGHT);
      layer.generate();
      dirtLayerRef.current = layer;
    }
    return dirtLayerRef.current;
  }, []);

  // --- Click Handler ---
  const handleCanvasClick = useCallback((): void => {
    const current = gameStateRef.current;

    if (current === "ready") {
      // Reset everything for a fresh game
      const dirtLayer = getDirtLayer();
      dirtLayer.reset();
      dirtLayer.generate();
      particleSystemRef.current.clear();
      dirtRemainingRef.current = 1;
      frameCountRef.current = 0;

      const next = transition(current, "playing");
      setGameState(next);
    } else if (current === "won") {
      const next = transition(current, "ready");
      setGameState(next);
    }
    // "playing" state: clicks do nothing (motion is the input)
  }, [getDirtLayer]);

  // --- Animation Frame Callback ---
  // This is the HEART of the game. Called ~60 times per second.
  // It handles all rendering and game logic based on the current state.
  useAnimationFrame((_deltaTime: number) => {
    // --- Setup ---
    if (!ctxRef.current) {
      ctxRef.current = canvasRef.current?.getContext("2d") ?? null;
    }
    const ctx = ctxRef.current;
    const video = videoRef.current;
    if (!ctx || !video) return;

    const state = gameStateRef.current;

    // === STEP 1: Clear canvas ===
    ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

    // === STEP 2: Draw mirrored video feed (all states) ===
    // The video is ALWAYS visible as the background layer.
    ctx.save();
    ctx.scale(-1, 1);
    ctx.translate(-CANVAS_WIDTH, 0);
    ctx.drawImage(video, 0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
    ctx.restore();

    // === STEP 3: State-specific rendering ===

    if (state === "ready") {
      // --- READY STATE ---
      // Show title screen over the camera feed.
      drawReadyScreen(ctx);
    } else if (state === "playing") {
      // --- PLAYING STATE ---
      const dirtLayer = getDirtLayer();

      // 3a: Detect motion
      const motionResult = detectMotion();

      // 3b: Clean dirt where motion is detected
      const cleaningPoints: CleaningPoint[] = [];

      for (const region of motionResult.regions) {
        if (region.intensity < MOTION_THRESHOLD) continue;

        // Mirror X to match mirrored video display
        const mirroredX = 1 - (region.x + region.width);
        const centerX = (mirroredX + region.width / 2) * CANVAS_WIDTH;
        const centerY = (region.y + region.height / 2) * CANVAS_HEIGHT;

        const radius =
          MIN_CLEAN_RADIUS +
          region.intensity * (MAX_CLEAN_RADIUS - MIN_CLEAN_RADIUS);

        dirtLayer.clean(centerX, centerY, radius);

        cleaningPoints.push({
          x: centerX,
          y: centerY,
          radius,
          intensity: region.intensity,
        });
      }

      // 3c: Draw shine effects at cleaning locations
      for (const point of cleaningPoints) {
        drawShineEffect(ctx, point.x, point.y, point.radius, point.intensity);
      }

      // 3d: Draw dirt overlay
      dirtLayer.draw(ctx);

      // 3e: Check progress (throttled)
      frameCountRef.current++;
      if (frameCountRef.current % PROGRESS_CHECK_INTERVAL === 0) {
        dirtRemainingRef.current = dirtLayer.getDirtRemaining();
      }

      // 3f: Draw progress HUD
      drawProgressHUD(ctx, dirtRemainingRef.current);

      // 3g: Check win condition
      if (dirtRemainingRef.current < DIRT_WIN_THRESHOLD) {
        // Immediately update the ref so that the NEXT rAF tick sees "won"
        // and doesn't re-trigger this branch. setGameState only schedules a
        // React re-render; the ref is what the frame loop reads.
        const next = transition(state, "won");
        gameStateRef.current = next;

        // Emit celebration bubbles
        particleSystemRef.current.emitCelebration(
          CANVAS_WIDTH,
          CANVAS_HEIGHT,
          WIN_BUBBLE_COUNT,
        );

        // Schedule a React re-render to reflect the state change in the UI
        setGameState(next);
      }
    } else if (state === "won") {
      // --- WON STATE ---
      const dirtLayer = getDirtLayer();

      // Draw remaining dirt (should be mostly clean)
      dirtLayer.draw(ctx);

      // Update and draw celebration bubbles
      const particles = particleSystemRef.current;
      particles.update();
      particles.draw(ctx);

      // Draw win screen overlay
      drawWinScreen(ctx);
    }
  }, isReady);

  // --- Render ---
  // If camera failed, show error. Otherwise, show the game canvas.
  if (error) {
    return (
      <div className="flex min-h-screen flex-col items-center justify-center gap-4 bg-slate-900">
        <h1 className="text-4xl font-bold text-white">Window Cleaner</h1>
        <p className="max-w-md text-center text-red-400">{error}</p>
      </div>
    );
  }

  return (
    <div className="flex min-h-screen flex-col items-center justify-center gap-4 bg-slate-900">
      <h1 className="text-3xl font-bold text-white">🫧 Window Cleaner</h1>

      <div className="relative">
        <canvas
          ref={(node) => {
            canvasRef.current = node;
            // Draw initial placeholder while camera loads
            if (node && !isReady) {
              const ctx = node.getContext("2d");
              if (ctx) {
                ctx.fillStyle = "#0f172a";
                ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
                ctx.fillStyle = "#94a3b8";
                ctx.font = "24px sans-serif";
                ctx.textAlign = "center";
                ctx.textBaseline = "middle";
                ctx.fillText(
                  "Initializing camera...",
                  CANVAS_WIDTH / 2,
                  CANVAS_HEIGHT / 2,
                );
                ctx.textAlign = "start";
                ctx.textBaseline = "alphabetic";
              }
            }
          }}
          width={CANVAS_WIDTH}
          height={CANVAS_HEIGHT}
          onClick={handleCanvasClick}
          className={`rounded-lg border border-slate-700 ${
            gameState === "playing" ? "cursor-none" : "cursor-pointer"
          }`}
        />
      </div>

      {/* Status bar below canvas */}
      <div className="flex items-center gap-4 text-sm text-slate-400">
        {isReady ? (
          <span className="flex items-center gap-1">
            <span className="inline-block h-2 w-2 rounded-full bg-green-400" />
            Camera active
          </span>
        ) : (
          <span className="flex items-center gap-1">
            <span className="inline-block h-2 w-2 animate-pulse rounded-full bg-yellow-400" />
            Loading camera...
          </span>
        )}
        <span>•</span>
        <span>
          {gameState === "ready" && "Click the canvas to start"}
          {gameState === "playing" && "Wave your hands to clean!"}
          {gameState === "won" && "Click to play again"}
        </span>
      </div>

      {/* Keep video mounted for canvas/motion processing; visually hide it off-screen instead of using `hidden` */}
      <video
        ref={videoRef}
        autoPlay
        playsInline
        muted
        className="absolute -left-[9999px] top-auto h-0 w-0 overflow-hidden opacity-0 pointer-events-none"
      />
    </div>
  );
}

// =============================================================================
// Types
// =============================================================================

interface CleaningPoint {
  x: number;
  y: number;
  radius: number;
  intensity: number;
}

// =============================================================================
// Drawing Functions
// =============================================================================

/**
 * Draws the "Ready" screen — title, instructions, and start prompt.
 */
function drawReadyScreen(ctx: CanvasRenderingContext2D): void {
  // Semi-transparent overlay
  ctx.fillStyle = "rgba(0, 0, 0, 0.6)";
  ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  ctx.save();

  // Title
  ctx.fillStyle = "#ffffff";
  ctx.font = "bold 42px sans-serif";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.shadowColor = "rgba(0, 0, 0, 0.5)";
  ctx.shadowBlur = 8;
  ctx.fillText("🪟 Window Cleaner", CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2 - 50);

  // Instructions
  ctx.shadowBlur = 0;
  ctx.fillStyle = "#94a3b8";
  ctx.font = "20px sans-serif";
  ctx.fillText(
    "Wave your hands in front of the camera",
    CANVAS_WIDTH / 2,
    CANVAS_HEIGHT / 2 + 10,
  );
  ctx.fillText(
    "to clean the dirty window!",
    CANVAS_WIDTH / 2,
    CANVAS_HEIGHT / 2 + 38,
  );

  // Start prompt
  ctx.fillStyle = "#38bdf8";
  ctx.font = "bold 24px sans-serif";
  ctx.fillText("▶ Click to Start", CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2 + 90);

  ctx.restore();
}

/**
 * Draws the win celebration screen.
 */
function drawWinScreen(ctx: CanvasRenderingContext2D): void {
  // Light overlay (less opaque than ready screen to show bubbles)
  ctx.fillStyle = "rgba(0, 0, 0, 0.3)";
  ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  ctx.save();

  // Victory text
  ctx.fillStyle = "#4ade80";
  ctx.font = "bold 48px sans-serif";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.shadowColor = "rgba(0, 0, 0, 0.5)";
  ctx.shadowBlur = 10;
  ctx.fillText(
    "🫧 Window Clean! 🫧",
    CANVAS_WIDTH / 2,
    CANVAS_HEIGHT / 2 - 30,
  );

  // Play again prompt
  ctx.shadowBlur = 0;
  ctx.fillStyle = "#ffffff";
  ctx.font = "24px sans-serif";
  ctx.fillText(
    "Click to Play Again",
    CANVAS_WIDTH / 2,
    CANVAS_HEIGHT / 2 + 30,
  );

  ctx.restore();
}

/**
 * Draws the cleaning progress HUD in the top-right corner.
 */
function drawProgressHUD(
  ctx: CanvasRenderingContext2D,
  dirtRemaining: number,
): void {
  const percentClean = Math.round((1 - dirtRemaining) * 100);
  const text = `${percentClean}% Clean`;

  ctx.save();
  ctx.font = "bold 20px sans-serif";

  const metrics = ctx.measureText(text);
  const padding = 10;
  const boxWidth = metrics.width + padding * 2;
  const boxHeight = 32;
  const boxX = CANVAS_WIDTH - boxWidth - 10;
  const boxY = 10;

  // Background
  ctx.fillStyle = "rgba(0, 0, 0, 0.6)";
  ctx.fillRect(boxX, boxY, boxWidth, boxHeight);

  // Text — green when near win, white otherwise
  ctx.fillStyle = percentClean >= 95 ? "#4ade80" : "#ffffff";
  ctx.textAlign = "left";
  ctx.textBaseline = "middle";
  ctx.fillText(text, boxX + padding, boxY + boxHeight / 2);

  ctx.restore();
}

/**
 * Draws a subtle white glow at the cleaning location for visual feedback.
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
```

### File: `src/App.tsx` (final version)

```tsx
/**
 * App.tsx — Root application component.
 *
 * Simply mounts the WindowCleanerGame.
 * In future phases, this will become a game router with multiple games.
 */
import { WindowCleanerGame } from "./components/WindowCleanerGame";

function App(): React.JSX.Element {
  return <WindowCleanerGame />;
}

export default App;
```

### Complete file tree

```
webcam-games/
├── .github/
│   └── workflows/
│       └── main.yml                    # CI: tsc + eslint + prettier
├── .dockerignore
├── .prettierrc
├── .prettierignore
├── Dockerfile                          # Multi-stage: Node builder → Nginx
├── nginx.conf                          # SPA routing + caching + gzip
├── eslint.config.js                    # Flat config with Prettier bridge
├── package.json
├── package-lock.json
├── tsconfig.json
├── tsconfig.app.json                   # Strict TS config
├── tsconfig.node.json
├── vite.config.ts                      # React + Tailwind plugins
├── index.html
├── public/
│   └── vite.svg
└── src/
    ├── components/
    │   └── WindowCleanerGame.tsx        # 🎮 Main game component
    ├── game/
    │   ├── Bubble.ts                    # Bubble particle class
    │   ├── DirtLayer.ts                 # Procedural dirt + cleaning
    │   ├── ParticleSystem.ts            # Particle lifecycle manager
    │   ├── cleaningConfig.ts            # Tunable game constants
    │   └── gameState.ts                 # State machine
    ├── hooks/
    │   ├── useAnimationFrame.ts         # rAF loop hook
    │   ├── useMotionDetection.ts        # Frame differencing hook
    │   └── useWebcam.ts                 # Camera access hook
    ├── types/
    │   └── motion.ts                    # Motion detection interfaces
    ├── App.tsx                          # Root component
    ├── App.css                          # Empty (Tailwind handles styles)
    ├── index.css                        # @import "tailwindcss"
    ├── main.tsx                         # ReactDOM entry
    └── vite-env.d.ts                    # Vite type declarations
```

### Verification checklist

```bash
# 1. Type checking — zero errors
npx tsc --noEmit

# 2. Linting — zero errors
npm run lint

# 3. Formatting — all files formatted
npm run format:check

# 4. Production build — succeeds
npm run build

# 5. Dev server — game is playable
npm run dev

# 6. Docker build (if available) — image builds
docker build -t webcam-games .

# 7. Docker run (if available) — serves correctly
docker run -p 8080:80 webcam-games
```

---

## Executive Summary

**Jack Donaghy:**

> Lemon, Tracy — gather round. Let me tell you what we've accomplished over the course of these 13 lessons, because I don't think either of you fully appreciates the magnitude of what's been built.

> We started with nothing. An empty directory. And through disciplined application of Six Sigma DMAIC methodology, we have produced a **production-grade, containerized, CI-gated, webcam-powered motion detection game** built on a strict TypeScript + React + Tailwind stack.

> Let me enumerate the enterprise competencies demonstrated by this codebase:

> 1. **Infrastructure as Code.** Our `Dockerfile`, `nginx.conf`, and GitHub Actions workflow define the entire deployment pipeline in version-controlled, reproducible files. There is zero tribal knowledge. A new developer clones the repo, runs `npm install`, and they're productive in minutes. That's *operational readiness*.

> 2. **Layered Architecture.** Hooks (infrastructure), game objects (business logic), and rendering functions (presentation) are cleanly separated. Each layer can be modified, tested, and reasoned about independently. This is the *separation of concerns* that allows organizations to scale without descending into chaos.

> 3. **Zero-Dependency Core Logic.** Our motion detection algorithm uses zero external libraries. Frame differencing on raw pixel arrays. No TensorFlow. No cloud APIs. No model downloads. The game works offline, loads instantly, and has zero supply chain vulnerabilities in its core path. That's *operational independence*.

> 4. **State Machine Governance.** Three states. Three transitions. No impossible combinations. No boolean flag soup. This is *process governance through engineering*, and it's the reason this game will never end up in a state where the player is simultaneously winning and losing.

> 5. **Performance Engineering.** 10% downscale for motion detection. Throttled progress checks. `willReadFrequently` hints. Offscreen canvases. Radial gradient soft-edge cleaning. Every optimization was calculated and justified. This is *Six Sigma process improvement* — measure, analyze, improve.

> 6. **Interview-Ready Architecture.** Every component — custom hooks, particle systems, canvas compositing, state machines, CI pipelines, Docker multi-stage builds — is a topic that comes up in senior front-end engineering interviews. This isn't a tutorial project. This is a *portfolio piece* that demonstrates systems thinking.

> Here's what I want you to remember: we didn't build a game. We built a **system that produces games**. The `useWebcam` hook, the `useAnimationFrame` loop, the `useMotionDetection` pipeline — these are reusable infrastructure. Phase 2 (Pose Matching), Phase 3 (Hammer Squish), and Phase 4 (Lane Runner) will all build on this foundation. Every hour invested in Phase 0 and Phase 1 pays compound dividends in every subsequent phase.

> That is leverage. That is vertical integration. That is what we do.

> Now if you'll excuse me, I have a scotch to pour and a quarterly earnings call to dominate.

**Tracy Jordan:**
> I DON'T KNOW WHAT HALF OF THAT MEANT BUT I'M VERY EXCITED! LET'S MAKE MORE GAMES! TRACY JORDAN OUT! 🎮🫧

**Liz Lemon:**
> I... I think we actually did it. The code compiles. The game works. The Docker image is 25 MB. The CI is green. I'm going to go eat a sandwich and cry a little bit. Happy tears. Mostly.
