# Lesson 06: The useWebcam Hook — Accessing the Camera

## The Pitch

**Tracy Jordan:**
> WE'RE TURNING ON THE CAMERA! This is what I was BORN for! Cameras LOVE me! But this isn't a regular camera, this is your LAPTOP CAMERA, and we're gonna capture it with something called `getUserMedia`! Sounds like a dating app! I like webcam access so much I want to take it behind the middle school and get it pregnant! But seriously — the camera can SEE you and your COMPUTER can SEE what the camera sees and that's how we're gonna make the game know when you're waving your hands around like a MANIAC!

**Jack Donaghy:**
> What we're building is a reusable, encapsulated hardware abstraction layer. In the enterprise world, we call this *modular component architecture*. The `useWebcam` hook will be a single point of entry for camera access across all four games. One hook, multiple consumers. That's leverage. That's efficiency. That's the kind of horizontal integration that makes shareholders weep with joy.

**Liz Lemon:**
> Okay, the webcam API. `getUserMedia`. Sounds simple. IT IS NOT. First, it's a Promise-based API, so it's async. Second, the user might DENY permission, so you need error handling. Third, when the component unmounts, you need to STOP the camera stream or it keeps running in the background — you'll see the green camera light staying on and the user thinks you're spying on them. And FOURTH, React's Strict Mode in development calls your effects TWICE, which means if you're not careful, you'll request the camera twice, get two streams, and only clean up one. This is my nightmare. This is LITERALLY my nightmare.

---

## The Theory

### The MediaDevices API

**Question:** How does a web application access the user's camera?

**Answer:** Through the `navigator.mediaDevices.getUserMedia()` API. This is a browser-native API (not a library) that:

1. Prompts the user for permission to access their camera (and/or microphone).
2. Returns a `Promise<MediaStream>` — a stream of video (and optionally audio) data.
3. The `MediaStream` can be bound to a `<video>` element using `videoElement.srcObject = stream`.

```ts
const stream = await navigator.mediaDevices.getUserMedia({
  video: true,   // Request video access
  audio: false,  // We don't need audio for our game
});
```

### Video Constraints

**Question:** Can we control the resolution and frame rate of the camera?

**Answer:** Yes, via *constraints*:

```ts
const stream = await navigator.mediaDevices.getUserMedia({
  video: {
    width: { ideal: 640 },
    height: { ideal: 480 },
    facingMode: "user",  // Front camera on mobile
  },
  audio: false,
});
```

- **`ideal`** means "get as close to this as possible." If the camera can't do 640×480, it picks the nearest resolution.
- **`exact`** means "this or fail." Use sparingly.
- **`facingMode: "user"`** selects the selfie camera on mobile devices.

For our game, 640×480 is sufficient. Higher resolutions mean more pixels to process for motion detection, which is wasteful.

### MediaStream lifecycle

**Question:** What happens to the camera when the user navigates away from the game?

**Answer:** The `MediaStream` stays active until explicitly stopped. The browser keeps the camera on (green light stays on). To stop it:

```ts
stream.getTracks().forEach((track) => track.stop());
```

Each `MediaStream` contains one or more `MediaStreamTrack` objects (video track, audio track). Calling `.stop()` on each track releases the hardware.

### Why a custom hook?

**Question:** Why encapsulate this in a React hook instead of doing it inline in a component?

**Answer:** 

1. **Reusability:** Every game needs camera access. A hook lets any component call `useWebcam()` instead of duplicating the setup/teardown logic.
2. **Encapsulation:** The hook manages the async lifecycle (requesting, receiving, stopping) internally. The consumer just gets the video ref and a ready status.
3. **Testability:** You can mock the hook in tests without touching the DOM or browser APIs.
4. **Cleanup safety:** The hook's `useEffect` return function handles stream cleanup. This is critical for React Strict Mode.

### React Strict Mode and effects

**Question:** Why does React Strict Mode complicate this?

**Answer:** In development mode with `<React.StrictMode>`, React intentionally:

1. **Mounts** the component.
2. **Unmounts** the component (calls cleanup).
3. **Remounts** the component.

This is React's way of stress-testing your effects to catch cleanup bugs. For `getUserMedia`, this means:

1. Mount → Request camera → Get stream A
2. Unmount → Cleanup → Stop stream A
3. Remount → Request camera → Get stream B

If your cleanup doesn't stop the tracks properly, stream A stays active (camera LED on, memory leaking). Your hook MUST handle this correctly.

### The `<video>` element and `autoPlay`

**Question:** Why do we bind the stream to a hidden `<video>` element?

**Answer:** The `<video>` element is the browser's native video decoder. When you set `videoElement.srcObject = stream`, the browser:

1. Decodes the video frames from the camera.
2. Makes them available as bitmap data.
3. Allows `<canvas>` to draw from the video using `ctx.drawImage(videoElement, ...)`.

The video element is *hidden* (`display: none` or off-screen) because we don't want to show the raw camera feed — we'll draw it to a canvas with game overlays. The video element is just a data source.

**Important attributes:**
- `autoPlay`: Start playing immediately when a stream is attached.
- `playsInline`: On iOS, prevents the video from going fullscreen.
- `muted`: Required for autoplay to work without user interaction in most browsers.

---

## The Assignment

### Step 1: Create the hook file

- Create `src/hooks/useWebcam.ts`.
- The hook should have the following signature:

```ts
interface UseWebcamResult {
  videoRef: React.RefObject<HTMLVideoElement | null>;
  isReady: boolean;
  error: string | null;
}

function useWebcam(): UseWebcamResult
```

### Step 2: Implement the hook logic

Inside the hook:

1. Create a `useRef<HTMLVideoElement | null>(null)` for the video element.
2. Create state for `isReady` (boolean, default `false`).
3. Create state for `error` (string | null, default `null`).
4. Use a `useEffect` that:
   - Calls `navigator.mediaDevices.getUserMedia({ video: { width: { ideal: 640 }, height: { ideal: 480 }, facingMode: "user" }, audio: false })`.
   - On success: binds the stream to the video element via `srcObject`, waits for the `loadedmetadata` event, then sets `isReady` to `true`.
   - On failure: sets the `error` state with a user-friendly message.
   - **Cleanup function:** Stops all tracks on the stream and nullifies `srcObject`.
5. Use an `AbortController` or a boolean `isMounted` flag to prevent state updates after unmount (handles the Strict Mode double-mount pattern).

### Step 3: Create a test component

- Update `src/App.tsx` to:
  - Call `useWebcam()`.
  - Render a hidden `<video>` element with the `videoRef` attached.
  - Display the camera status: "Loading camera...", "Camera ready!", or the error message.
  - Attach `autoPlay`, `playsInline`, and `muted` attributes to the `<video>` element.

### Step 4: Verify

- Run `npm run dev` and open the app.
- The browser should prompt for camera permission.
- After granting permission, the status should change to "Camera ready!".
- Open DevTools → check that the video element has a non-null `srcObject`.
- Navigate away from the page and back — the camera light should turn OFF and then back ON (cleanup works).
- Run `npx tsc --noEmit` — zero type errors.

### Deliverables

1. `src/hooks/useWebcam.ts` — a fully typed custom hook for camera access.
2. Updated `App.tsx` that uses the hook and displays camera status.
3. Proper cleanup of MediaStream tracks on unmount.
4. Works correctly with React Strict Mode (no leaked streams).
5. Zero type errors.

---

## Liz's Nightmares

> **Liz Lemon:** The webcam hook. Where hardware meets software meets my anxiety disorder.

1. **Not stopping tracks on cleanup:** If your `useEffect` cleanup doesn't call `track.stop()` on every track, the camera stays active after unmount. The user sees the green LED light, thinks they're being surveilled, and writes a one-star review. You MUST stop tracks. Every single one.

2. **Setting state after unmount:** In the Strict Mode double-mount scenario, the first mount's async `getUserMedia` call might resolve AFTER the component has already unmounted and remounted. If you call `setIsReady(true)` on the unmounted instance, React warns: "Can't perform a React state update on an unmounted component." Use an `isMounted` flag or AbortController to gate state updates.

3. **Forgetting `autoPlay` and `playsInline`:** Without `autoPlay`, the video element won't start playing when you set `srcObject`. Without `playsInline`, iOS Safari forces the video into fullscreen. Without `muted`, some browsers block autoplay entirely. All three are required. Every. Single. Time.

4. **Not handling permission denial:** If the user clicks "Block" on the camera prompt, `getUserMedia` rejects with a `NotAllowedError`. If you don't catch this, you get an unhandled promise rejection. Your error state should have a human-readable message like "Camera access was denied. Please allow camera access in your browser settings."

5. **Testing in HTTP instead of HTTPS:** `getUserMedia` only works on secure origins: `https://` or `localhost`. If you're testing on `http://192.168.1.x`, it will fail silently or throw a `NotAllowedError`. Vite's dev server on `localhost` works fine, but be aware of this if you ever try to test on another device on your network.

6. **Binding the ref before the video element mounts:** The `videoRef.current` is `null` until React renders the `<video>` element. If your `useEffect` runs before the video element exists (timing edge case), `videoRef.current.srcObject = stream` throws. Always null-check the ref inside the effect.
