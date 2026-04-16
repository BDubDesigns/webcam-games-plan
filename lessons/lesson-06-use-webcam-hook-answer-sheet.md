# Lesson 06 — Answer Sheet: The useWebcam Hook — Accessing the Camera

## The Implementation

### File: `src/hooks/useWebcam.ts`

```ts
import { useEffect, useRef, useState } from "react";

/**
 * Result object returned by the useWebcam hook.
 *
 * @property videoRef - Ref to attach to a <video> element. The hook binds
 *   the camera stream to this element's srcObject.
 * @property isReady - True once the video stream is active and metadata
 *   has loaded (dimensions are known). Safe to draw to canvas.
 * @property error - Human-readable error message if camera access fails.
 *   Null when there's no error.
 */
interface UseWebcamResult {
  videoRef: React.RefObject<HTMLVideoElement | null>;
  isReady: boolean;
  error: string | null;
}

/**
 * Camera constraints for getUserMedia.
 *
 * - 640x480 is sufficient for motion detection (we'll downscale further).
 * - "user" facing mode selects the front camera on mobile devices.
 * - Audio is disabled — we're building a visual game, not a podcast.
 */
const VIDEO_CONSTRAINTS: MediaStreamConstraints = {
  video: {
    width: { ideal: 640 },
    height: { ideal: 480 },
    facingMode: "user",
  },
  audio: false,
};

/**
 * useWebcam — Custom hook for accessing the user's camera.
 *
 * This hook encapsulates the entire lifecycle of a webcam stream:
 *   1. Requests camera permission via getUserMedia.
 *   2. Binds the resulting MediaStream to a <video> element.
 *   3. Reports readiness once the video metadata has loaded.
 *   4. Cleans up (stops all tracks) on unmount.
 *
 * Handles React Strict Mode correctly:
 *   In development, Strict Mode mounts → unmounts → remounts components.
 *   The cleanup function stops the first stream, and the remount starts a new one.
 *   The `isCancelled` flag prevents state updates from the stale first mount.
 *
 * Usage:
 *   const { videoRef, isReady, error } = useWebcam();
 *   return <video ref={videoRef} autoPlay playsInline muted hidden />;
 */
export function useWebcam(): UseWebcamResult {
  // Ref to the <video> element. The consumer must attach this to a <video> tag.
  const videoRef = useRef<HTMLVideoElement | null>(null);

  // Stream ref: keeps a reference to the active MediaStream for cleanup.
  // We use a ref (not state) because we don't want to re-render when the stream changes.
  const streamRef = useRef<MediaStream | null>(null);

  // Ready state: true once the video stream is active and metadata is loaded.
  const [isReady, setIsReady] = useState<boolean>(false);

  // Error state: human-readable error message if something goes wrong.
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    // Flag to prevent state updates after this effect's cleanup has run.
    // This handles two scenarios:
    //   1. React Strict Mode: first mount is cleaned up, but its async
    //      getUserMedia might still resolve and try to update state.
    //   2. Fast unmount: component unmounts before getUserMedia resolves.
    let isCancelled = false;

    /**
     * Starts the camera stream and binds it to the video element.
     */
    async function startCamera(): Promise<void> {
      try {
        // Request camera access. This prompts the user for permission
        // on first call. Subsequent calls may reuse the permission grant.
        const stream =
          await navigator.mediaDevices.getUserMedia(VIDEO_CONSTRAINTS);

        // If the effect was cleaned up while we were waiting for the user
        // to grant permission, stop the stream immediately and bail out.
        if (isCancelled) {
          stream.getTracks().forEach((track) => track.stop());
          return;
        }

        // Store the stream reference for cleanup.
        streamRef.current = stream;

        // Bind the stream to the video element.
        // The video element decodes the camera frames and makes them
        // available for canvas drawing via ctx.drawImage(video, ...).
        const video = videoRef.current;
        if (video) {
          video.srcObject = stream;

          // Wait for the video metadata to load.
          // "loadedmetadata" fires when the browser knows the video dimensions
          // and duration. At this point, it's safe to draw the video to canvas.
          await new Promise<void>((resolve) => {
            video.onloadedmetadata = () => {
              resolve();
            };
          });

          // Final cancelled check after the metadata load await.
          if (!isCancelled) {
            setIsReady(true);
          }
        }
      } catch (err) {
        // Don't update state if we've been cancelled.
        if (isCancelled) return;

        // Map the error to a human-readable message.
        if (err instanceof DOMException) {
          switch (err.name) {
            case "NotAllowedError":
              setError(
                "Camera access was denied. Please allow camera access in your browser settings.",
              );
              break;
            case "NotFoundError":
              setError(
                "No camera found. Please connect a camera and try again.",
              );
              break;
            case "NotReadableError":
              setError(
                "Camera is in use by another application. Please close other apps using the camera.",
              );
              break;
            case "OverconstrainedError":
              setError(
                "Camera does not support the requested resolution. Try a different camera.",
              );
              break;
            default:
              setError(`Camera error: ${err.message}`);
          }
        } else {
          setError("An unexpected error occurred while accessing the camera.");
        }
      }
    }

    startCamera();

    // Cleanup function: runs when the component unmounts or the effect re-runs.
    // This MUST stop all media tracks to release the camera hardware.
    return () => {
      isCancelled = true;

      // Stop every track on the stream.
      // A MediaStream can have multiple tracks (video, audio).
      // We requested video only, but we stop all tracks defensively.
      if (streamRef.current) {
        streamRef.current.getTracks().forEach((track) => {
          track.stop();
        });
        streamRef.current = null;
      }

      // Clear the video element's source.
      if (videoRef.current) {
        videoRef.current.srcObject = null;
      }

      // NOTE: We intentionally do NOT call setIsReady(false) or setError(null)
      // here. Setting state during cleanup (which runs on unmount, including
      // Strict Mode's mount→unmount cycle) can trigger "state update on an
      // unmounted component" warnings. On remount, the fresh useState calls
      // reinitialize the state automatically, so resetting here is unnecessary.
    };
  }, []); // Empty dependency array: run once on mount, clean up on unmount.

  return { videoRef, isReady, error };
}
```

### File: `src/App.tsx`

```tsx
/**
 * App.tsx
 *
 * Root application component.
 * Demonstrates the useWebcam hook by displaying camera status.
 *
 * The <video> element is hidden — it serves as a data source for canvas
 * drawing in future lessons. We don't display the raw camera feed directly.
 */
import { useWebcam } from "./hooks/useWebcam";

function App(): React.JSX.Element {
  const { videoRef, isReady, error } = useWebcam();

  return (
    <main className="flex min-h-screen flex-col items-center justify-center gap-4 bg-slate-900">
      <h1 className="text-4xl font-bold text-white">Webcam Games</h1>

      {/* Status indicator */}
      {error ? (
        <p className="text-red-400">{error}</p>
      ) : isReady ? (
        <p className="text-green-400">Camera ready!</p>
      ) : (
        <p className="text-yellow-400">Loading camera...</p>
      )}

      {/*
        Hidden video element.
        This is the "data source" for our game canvas — not a visual element.
        
        Required attributes:
        - ref: connects the hook's stream binding
        - autoPlay: starts playing immediately when srcObject is set
        - playsInline: prevents iOS Safari from forcing fullscreen
        - muted: required for autoplay to work without user gesture in most browsers
        
        The "hidden" attribute removes it from the visual layout entirely.
        Alternative: className="absolute -left-[9999px]" to keep it in the DOM
        but off-screen (some browsers throttle hidden video elements).
      */}
      <video ref={videoRef} autoPlay playsInline muted hidden />
    </main>
  );
}

export default App;
```

### Why specific design decisions were made

**1. `isCancelled` flag vs. AbortController:**

We use a simple boolean flag instead of `AbortController` because `getUserMedia` doesn't accept an `AbortSignal`. The flag guards state updates after cleanup. An `AbortController` would be overkill here.

**2. `streamRef` instead of state for the stream:**

The `MediaStream` object doesn't affect rendering — it's only used for cleanup. Storing it in a ref avoids unnecessary re-renders when the stream is assigned.

**3. Error handling with `DOMException` names:**

The `getUserMedia` API throws specific `DOMException` types. We switch on `err.name` rather than `err.message` because names are standardized across browsers while messages vary.

**4. Empty dependency array `[]`:**

The effect should run once on mount and clean up on unmount. All values used inside (`videoRef`, `streamRef`) are refs, which are stable across renders. The `setIsReady` and `setError` state setters are also stable (guaranteed by React). No dependencies needed.

---

## Executive Summary

**Jack Donaghy:**

> What we've built is not "a webcam hook." We've built a **hardware abstraction layer** — a clean interface between unreliable physical hardware and our application logic. Let me explain why this matters in terms of organizational architecture.

> 1. **Single Responsibility Principle as organizational design.** This hook does ONE thing: manage camera access. It doesn't know about games, canvases, or motion detection. In a well-run corporation, the VP of Hardware (this hook) doesn't attend the VP of Game Logic's meetings. They communicate through a clean interface: `videoRef` and `isReady`. That's efficient. That's Jack Welch-era GE.

> 2. **Defensive cleanup is risk management.** The `isCancelled` flag and comprehensive track stopping are not "nice to have" — they're the engineering equivalent of a golden parachute. When React's Strict Mode stress-tests your cleanup, this hook passes flawlessly. No leaked streams, no zombie camera access, no user trust violations. In regulatory terms, this is compliance-by-design.

> 3. **Error categorization is customer experience strategy.** We don't show users a raw `DOMException`. We translate hardware failures into human-readable guidance. "Camera access was denied. Please allow camera access in your browser settings." This is the difference between a product and a prototype. Products communicate; prototypes crash.

> 4. **The hook pattern is horizontal leverage.** Every game in our portfolio — Window Cleaner, Pose Match, Hammer Squish, Lane Runner — will call `useWebcam()`. One implementation, four consumers. That's a 75% reduction in duplicated camera management code. In Six Sigma terms, we've standardized the process and eliminated variation. The result? Predictable quality across every product line.

> This is Phase 1's foundation stone. Every subsequent lesson builds on this hook. Build it right, and everything downstream is easier. Build it wrong, and you're debugging camera leaks at 2 AM. I don't debug at 2 AM. I have people for that. But since we're a lean operation — build it right.
