# Lesson 07 — Answer Sheet: Canvas Rendering Fundamentals — useRef & useEffect with the Canvas API

## The Implementation

### File: `src/components/GameCanvas.tsx`

```tsx
import { useEffect, useRef } from "react";

/**
 * Props for the GameCanvas component.
 *
 * @property videoRef - Ref to the hidden <video> element from useWebcam.
 *   Used as the source for ctx.drawImage() to capture camera frames.
 * @property isReady - Whether the video stream is active and metadata loaded.
 *   When false, a placeholder is drawn instead of the camera feed.
 */
interface GameCanvasProps {
  videoRef: React.RefObject<HTMLVideoElement | null>;
  isReady: boolean;
}

/** Canvas dimensions in pixels. Matches our camera resolution. */
const CANVAS_WIDTH = 640;
const CANVAS_HEIGHT = 480;

/**
 * GameCanvas — Renders the camera feed onto an HTML5 canvas.
 *
 * Architecture:
 *   This component bridges React's declarative model with the imperative Canvas API.
 *   - useRef provides direct access to the <canvas> DOM element.
 *   - useEffect performs imperative drawing operations after the DOM is ready.
 *
 * Currently draws a single frame. The animation loop is added in Lesson 08.
 *
 * @param props - Video ref and readiness state from useWebcam hook.
 */
export function GameCanvas({ videoRef, isReady }: GameCanvasProps): React.JSX.Element {
  // Ref to the <canvas> DOM element.
  // This gives us direct access for imperative Canvas API calls.
  // useRef<HTMLCanvasElement | null>(null) — strict typing, null until mounted.
  const canvasRef = useRef<HTMLCanvasElement | null>(null);

  // Effect: Draw to the canvas when the video is ready.
  // Dependencies: [isReady] — re-run when camera readiness changes.
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    // getContext("2d") returns the 2D rendering context.
    // This is the object we use for ALL drawing operations.
    // It's safe to call multiple times — it returns the same context instance.
    const ctx = canvas.getContext("2d");
    if (!ctx) return;

    if (isReady && videoRef.current) {
      // --- Draw the current video frame ---
      // ctx.drawImage() is the Swiss Army knife of canvas drawing.
      // It accepts: <img>, <video>, <canvas>, ImageBitmap, OffscreenCanvas
      //
      // Signature: drawImage(source, dx, dy, dWidth, dHeight)
      //   source: the video element (captures the CURRENT displayed frame)
      //   dx, dy: destination x, y on the canvas (top-left corner)
      //   dWidth, dHeight: destination width and height (stretches if needed)
      ctx.drawImage(videoRef.current, 0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
    } else {
      // --- Draw placeholder while waiting for camera ---
      drawPlaceholder(ctx);
    }
  }, [isReady, videoRef]);

  return (
    <canvas
      ref={canvasRef}
      // IMPORTANT: Set width and height as HTML attributes, not CSS.
      // These set the canvas's internal pixel buffer resolution.
      // CSS width/height only affects display size and would cause blurriness
      // if it doesn't match the buffer size.
      width={CANVAS_WIDTH}
      height={CANVAS_HEIGHT}
      // Tailwind classes for visual presentation only.
      // rounded-lg: subtle border radius
      // border: thin border for visibility against dark background
      // The actual canvas content is drawn imperatively, not via CSS.
      className="rounded-lg border border-slate-700"
    />
  );
}

/**
 * Draws a placeholder screen on the canvas while waiting for the camera.
 *
 * This is called when isReady is false (camera still initializing).
 * We draw directly to the canvas rather than using a CSS overlay so that
 * the canvas always has content — no flash of empty/white space.
 *
 * @param ctx - The 2D rendering context of the canvas.
 */
function drawPlaceholder(ctx: CanvasRenderingContext2D): void {
  // Fill the entire canvas with a dark background.
  ctx.fillStyle = "#0f172a"; // slate-900 to match the page background
  ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  // Draw centered text.
  ctx.fillStyle = "#94a3b8"; // slate-400 for subtle text
  ctx.font = "24px sans-serif";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillText("Waiting for camera...", CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2);

  // Reset text alignment to defaults (good hygiene — don't leave state dirty).
  ctx.textAlign = "start";
  ctx.textBaseline = "alphabetic";
}
```

### File: `src/App.tsx` (updated)

```tsx
/**
 * App.tsx
 *
 * Root application component.
 * Integrates the webcam hook with the game canvas.
 *
 * Architecture:
 *   useWebcam() → provides videoRef + isReady status
 *   GameCanvas  → draws video frames (or placeholder) to a <canvas>
 *
 * The <video> element is hidden. It exists solely as a data source for
 * canvas drawing. Users never see the raw video feed.
 */
import { GameCanvas } from "./components/GameCanvas";
import { useWebcam } from "./hooks/useWebcam";

function App(): React.JSX.Element {
  const { videoRef, isReady, error } = useWebcam();

  return (
    <main className="flex min-h-screen flex-col items-center justify-center gap-6 bg-slate-900">
      <h1 className="text-4xl font-bold text-white">Webcam Games</h1>

      {/* Error display — shown instead of canvas if camera access fails */}
      {error ? (
        <p className="max-w-md text-center text-red-400">{error}</p>
      ) : (
        <>
          {/* Status indicator above the canvas */}
          <p className={isReady ? "text-green-400" : "text-yellow-400"}>
            {isReady ? "Camera ready!" : "Loading camera..."}
          </p>

          {/* Game canvas — draws video feed or placeholder */}
          <GameCanvas videoRef={videoRef} isReady={isReady} />
        </>
      )}

      {/*
        Hidden video element — data source for canvas drawing.
        This MUST be in the DOM for the video stream to decode frames.
        "hidden" removes it from visual layout but keeps it functional.
      */}
      <video ref={videoRef} autoPlay playsInline muted hidden />
    </main>
  );
}

export default App;
```

### Understanding the data flow

```
                  useWebcam hook
                       │
            ┌──────────┴──────────┐
            │                     │
        videoRef              isReady
            │                     │
            ▼                     ▼
    <video hidden>         GameCanvas component
    (decodes stream)              │
            │                     │
            │              useEffect fires
            │                     │
            └────────┬────────────┘
                     │
              ctx.drawImage(video)
                     │
                     ▼
              <canvas> element
           (visible to the user)
```

### Key TypeScript patterns used

**1. Strict null checking on refs:**

```ts
const canvas = canvasRef.current;  // HTMLCanvasElement | null
if (!canvas) return;               // Narrow to HTMLCanvasElement
const ctx = canvas.getContext("2d"); // CanvasRenderingContext2D | null
if (!ctx) return;                   // Narrow to CanvasRenderingContext2D
```

TypeScript's control flow analysis understands these checks. After the `if (!canvas) return` guard, `canvas` is guaranteed non-null for the rest of the scope.

**2. Interface for component props:**

```ts
interface GameCanvasProps {
  videoRef: React.RefObject<HTMLVideoElement | null>;
  isReady: boolean;
}
```

Using an explicit interface (vs. inline types) makes the component contract clear and the types reusable.

**3. Constants extracted from the component:**

```ts
const CANVAS_WIDTH = 640;
const CANVAS_HEIGHT = 480;
```

Constants outside the component prevent re-creation on every render and provide a single source of truth for canvas dimensions.

---

## Executive Summary

**Jack Donaghy:**

> What we've built here is the **rendering pipeline** — the factory floor where all visual output is manufactured. Let me explain the strategic significance using manufacturing metaphors, because that's what I do and I'm very good at it.

> 1. **The `<video>` element is our raw material supply chain.** It receives camera frames and decodes them. The user never sees it — just as a consumer never sees the factory that makes their microwave. It's infrastructure. It's hidden. It's essential.

> 2. **The `<canvas>` element is our finished goods display.** This is what the user sees. Every pixel is under our control. No CSS layout engine, no DOM reconciliation, no framework overhead between our drawing commands and the user's eyeballs. That's direct-to-consumer manufacturing. That's efficiency.

> 3. **The `useRef` + `useEffect` bridge is our operational interface.** React manages the component lifecycle (mounting, updating, unmounting). Canvas manages the pixels. `useRef` gives us a stable, render-immune handle to the DOM element. `useEffect` gives us a lifecycle hook to perform imperative work. Together, they solve the impedance mismatch between React's declarative world and Canvas's imperative world. This is like having a bilingual manager between your Japanese factory and your American sales team. Communication is clear. Errors are caught. Nobody accidentally draws on a null canvas.

> 4. **The placeholder state is customer experience design.** When the camera isn't ready, we don't show a blank white canvas — we show a styled placeholder with a message. This is the product equivalent of a loading screen at a luxury hotel. The customer knows something is happening. They trust the process. First impressions are everything.

> We now have raw materials (camera) flowing into a production facility (canvas) through a clean operational interface (useRef + useEffect). In the next lesson, we add the assembly line itself: the `requestAnimationFrame` game loop that runs at 60 frames per second. That's when manufacturing begins at scale.
