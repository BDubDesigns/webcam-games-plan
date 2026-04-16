# Lesson 07: Canvas Rendering Fundamentals — useRef & useEffect with the Canvas API

## The Pitch

**Tracy Jordan:**
> We're PAINTING on the COMPUTER! There's this thing called a CANVAS and it's not the kind you buy at Michael's Arts and Crafts — it's an HTML element where you can draw ANYTHING! Rectangles! Circles! PICTURES! You can even draw your webcam on it like some kind of DIGITAL MIRROR! I like the Canvas API so much I want to take it behind the middle school and get it pregnant! We're about to make Bob Ross look like an AMATEUR!

**Jack Donaghy:**
> The HTML5 Canvas API is the rendering layer of our entire product portfolio. Think of it as our manufacturing floor — where raw materials (camera frames, pixel data, game sprites) are assembled into the finished product the user sees. Now, the challenge — and this is where most developers fail — is that the Canvas API is fundamentally *imperative*. It's a sequence of drawing commands: "move here, draw this, fill that." React, by contrast, is *declarative*: "here's what the UI should look like." Bridging these two paradigms requires `useRef` for direct DOM access and `useEffect` for lifecycle management. This is the operational interface between our declarative framework and our imperative rendering engine.

**Liz Lemon:**
> Oh boy. Okay. So here's the fundamental tension that's going to haunt us for the rest of this project: React wants to OWN the DOM. It manages elements, updates them, reconciles them. But Canvas doesn't work that way. You can't declare "draw a red circle at (100, 200)" in JSX and have React figure out the diff. You have to get a *reference* to the actual DOM canvas element, grab its 2D rendering context, and issue imperative drawing commands. It's like React and Canvas are two roommates who have completely different approaches to household chores, and you're the mediator making sure nobody burns the apartment down. That mediator is `useRef` + `useEffect`. Let's learn how to be that mediator without having a breakdown.

---

## The Theory

### What is the HTML5 Canvas?

**Question:** What is a `<canvas>` element and how does it differ from other HTML elements?

**Answer:** A `<canvas>` element is a *bitmap drawing surface* in the DOM. Unlike `<div>` or `<p>`, which contain structured content, a canvas is essentially a rectangle of pixels. You draw on it using JavaScript.

Key concepts:

1. **The element vs. the context:** The `<canvas>` element is the DOM node. To draw on it, you call `canvas.getContext("2d")` to get a `CanvasRenderingContext2D` object. This context provides all the drawing methods.
2. **Immediate mode rendering:** Canvas is "immediate mode" — once you draw something, it becomes pixels. There's no scene graph, no DOM tree of shapes. If you want to change something, you clear the canvas and redraw everything from scratch.
3. **Resolution:** The canvas has two sizes — its CSS display size and its internal pixel buffer size. If these don't match, content looks blurry. Set both explicitly.

### useRef for Canvas access

**Question:** Why do we need `useRef` for canvas?

**Answer:** React's declarative model doesn't give you direct access to DOM elements. `useRef` creates a persistent reference that survives re-renders:

```tsx
const canvasRef = useRef<HTMLCanvasElement | null>(null);
// Later, in JSX: <canvas ref={canvasRef} />
// In an effect: const ctx = canvasRef.current?.getContext("2d");
```

**Important:** The ref's `.current` property is `null` until React mounts the component and attaches the DOM element. Always null-check before using.

### useEffect for imperative setup

**Question:** Why `useEffect` for canvas operations?

**Answer:** `useEffect` runs *after* React has rendered the DOM. This guarantees:

1. The `<canvas>` element exists in the DOM.
2. `canvasRef.current` is no longer `null`.
3. We can safely call `getContext("2d")` and start drawing.

### The Canvas coordinate system

**Question:** How does the canvas coordinate system work?

**Answer:**

```
(0, 0) ——————————————→ X (width)
  |
  |
  |
  |
  ▼
  Y (height)
```

- Origin `(0, 0)` is the **top-left** corner.
- X increases to the right.
- Y increases **downward** (opposite of math convention).
- All coordinates are in pixels.

### Key Canvas API methods we'll use

| Method | Purpose |
|--------|---------|
| `ctx.drawImage(source, x, y, w, h)` | Draw an image/video/canvas at position |
| `ctx.clearRect(x, y, w, h)` | Clear a rectangular area |
| `ctx.fillRect(x, y, w, h)` | Fill a rectangle with current fill style |
| `ctx.getImageData(x, y, w, h)` | Extract pixel data as `ImageData` |
| `ctx.putImageData(data, x, y)` | Write pixel data back to canvas |
| `ctx.save()` / `ctx.restore()` | Save/restore canvas state (transforms, styles) |
| `ctx.globalCompositeOperation` | How new draws composite with existing content |

### Drawing a video frame to canvas

**Question:** How do we get a camera frame onto the canvas?

**Answer:** The `drawImage` method accepts a `<video>` element as a source:

```ts
ctx.drawImage(videoElement, 0, 0, canvas.width, canvas.height);
```

This captures the *current frame* of the video and draws it to the canvas. By calling this every animation frame, we get a live camera feed rendered on the canvas.

---

## The Assignment

### Step 1: Create a Canvas component

- Create `src/components/GameCanvas.tsx`.
- The component should accept a `videoRef` prop (the ref from `useWebcam`) and an `isReady` prop.
- It should render a `<canvas>` element with:
  - A ref (`canvasRef`).
  - Width of 640 and height of 480 (set as attributes, not CSS).
  - Tailwind classes for styling (centered, with a subtle border).

### Step 2: Draw the video feed to the canvas

- Use a `useEffect` that:
  - Gets the 2D context from the canvas ref.
  - Gets the video element from the video ref.
  - Only proceeds if both are available AND `isReady` is true.
  - Draws the current video frame to the canvas using `ctx.drawImage()`.
  - For now, draw a SINGLE frame (we'll add the animation loop in Lesson 08).

### Step 3: Display a "waiting" state

- When `isReady` is false, draw a placeholder on the canvas instead:
  - Fill the canvas with a dark color.
  - Draw text saying "Waiting for camera..." centered on the canvas.

### Step 4: Integrate into App.tsx

- Import and render `GameCanvas` in `App.tsx`, passing the `videoRef` and `isReady` from `useWebcam`.

### Step 5: Verify

- Run `npm run dev`.
- You should see a single snapshot of the camera feed on the canvas (it won't update — that's the game loop lesson).
- Run `npx tsc --noEmit` — zero type errors.
- Run `npm run lint` — zero lint errors.

### Deliverables

1. `src/components/GameCanvas.tsx` — a canvas component that draws a video frame.
2. Proper `useRef` + `useEffect` pattern for canvas access.
3. Null-safe access to both canvas and video refs.
4. Updated `App.tsx` integrating the new component.
5. Zero type errors, zero lint errors.

---

## Liz's Nightmares

> **Liz Lemon:** Canvas and React. Two great tastes that taste terrible together if you don't manage the lifecycle correctly.

1. **Setting canvas size via CSS instead of attributes:** If you do `<canvas style={{ width: 640, height: 480 }}>`, you're setting the *display* size but not the *buffer* size. The buffer defaults to 300×150, and everything looks stretched and blurry. Set `width` and `height` as HTML attributes: `<canvas width={640} height={480}>`. CSS can *additionally* scale the display, but the attribute sets the pixel buffer.

2. **Calling `getContext("2d")` every frame:** `getContext` returns the same context object every time, but calling it has overhead. Get it once and store it (or let the effect closure hold it). Don't call it in a loop.

3. **Drawing before the video is ready:** If you call `ctx.drawImage(video, ...)` before the video has loaded metadata, you get a blank or broken frame. That's why we have the `isReady` flag from the webcam hook. Always gate canvas drawing on `isReady === true`.

4. **Forgetting to handle the `null` case on refs:** TypeScript with `noUncheckedIndexedAccess` and strict null checks means `canvasRef.current` is `HTMLCanvasElement | null`. You MUST null-check before calling `.getContext()`. The compiler will yell at you. Thank it. It's protecting you from runtime errors.

5. **Not understanding immediate mode:** Canvas is not retained mode. There's no "canvas DOM" where you add and remove shapes. Every frame, you clear and redraw EVERYTHING. If you draw a circle and then want to move it, you don't "move the circle" — you clear the canvas, draw the circle at the new position. This is a fundamental mental model shift from DOM-based UI development, and it's critical to understand before we build the game loop.
