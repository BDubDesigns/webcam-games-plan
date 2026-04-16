# Core Technologies Research - Webcam Games for Kids

**Date:** Research Phase 1 of 4
**Focus:** Pose detection, motion detection, and rendering libraries

---

## 1. Pose Detection Options

### Recommended: MediaPipe (Google)

**Best for:** All pose detection games (pose matching, lane runner, hand tracking)

**Options:**

| Solution | Keypoints | Speed | Best For |
|----------|-----------|-------|----------|
| **MediaPipe Pose** | 33 landmarks | 30+ FPS | Full body tracking |
| **MediaPipe Hands** | 21 landmarks per hand | Real-time | Hammer game, hand gestures |
| **MediaPipe Holistic** | Pose + Hands + Face | ~15-20 FPS | Combined tracking |

**Installation:**
```bash
npm install @mediapipe/pose @mediapipe/camera_utils
# or CDN for simple prototyping
```

**Why MediaPipe over TensorFlow.js directly:**
- Pre-optimized models
- Better performance on CPU
- Easier API
- Works offline after load

### Alternative: MoveNet (TensorFlow.js)

**Best for:** Maximum speed, lower-end devices

| Variant | Speed | Use Case |
|---------|-------|----------|
| Lightning | ~7ms | Fastest, mobile-friendly |
| Thunder | ~25-35ms | Higher accuracy |

**Trade-off:** MoveNet is faster but MediaPipe has better hand detail for the hammer game.

**Recommendation:** Start with MediaPipe Pose + Hands, fall back to MoveNet if performance issues.

---

## 2. Motion Detection (Window Cleaning Game)

### Approach: Frame Differencing with Canvas

**Core technique:**
```javascript
// Compare current frame vs previous frame
canvasContext.globalCompositeOperation = 'difference';
canvasContext.drawImage(video, 0, 0);
```

**Algorithm:**
1. Downscale frames to 10% size (reduces noise, improves performance)
2. Use `getImageData()` to access pixel arrays
3. Compare RGB values between frames
4. Apply threshold: ignore small changes (noise)
5. Score: Sum of significant pixel changes

**Handling Low-Quality Webcams:**
- **Downscaling:** Process at 64x48 or 100x75 instead of 640x480
- **Threshold:** Set `PIXEL_SCORE_THRESHOLD` (e.g., ignore changes < 30/255)
- **Blur tolerance:** Use `IMAGE_SCORE_THRESHOLD` (e.g., need 1000+ pixels to trigger)
- **Temporal smoothing:** Require motion for 3+ consecutive frames

**Dirt Visualization:**
- Canvas overlay with transparent PNG of dirt specks
- "Clean" by reducing alpha based on motion score in that region
- Bubbles: CSS animations or Canvas particles on clean detection

---

## 3. Rendering & Overlays

### HTML5 Canvas (Primary)

**For:**
- Drawing game elements (hammer, bugs, coins, lane runner)
- Compositing webcam feed + overlays
- Particle effects (bubbles, sparkles)

**Key APIs:**
- `globalCompositeOperation` for layering
- `drawImage()` for webcam + sprites
- `requestAnimationFrame()` for 60fps game loop

### CSS Overlays (Secondary)

**For:**
- UI elements (score, timer)
- Static overlays (window frame)
- Simple animations (bubbles floating up)

### Performance Tips:

1. **Use offscreen canvas** for static elements (dirt layer)
2. **Only redraw changed regions** (dirty rectangles)
3. **Scale down processing canvas**, display at full size

---

## 4. Hand Tracking Specifics (Hammer Game)

### MediaPipe Hands Output:

21 landmarks per hand:
```
0: wrist
1-4: thumb
5-8: index finger
9-12: middle finger
13-16: ring finger
17-20: pinky
```

### Hammer Positioning:
- Use landmark 9 (middle finger MCP) or 0 (wrist) as anchor
- Offset based on hand angle (landmark 5 to 17)
- Scale hammer sprite based on hand size (distance 0 to 9)

### Bug Collision:
- Simple circle collision: distance between hammer head and bug < threshold
- Or bounding box for performance

---

## 5. Learning Path Structure

### Phase 1: Webcam + Canvas Basics
- GetUserMedia API
- Drawing video to canvas
- Basic pixel manipulation

### Phase 2: Motion Detection
- Frame differencing
- Thresholding
- Window cleaning prototype

### Phase 3: Pose Detection
- MediaPipe integration
- Landmark visualization
- Simple pose matching

### Phase 4: Game Logic
- Hammer overlay tracking
- Lane runner movement detection
- Scoring and game states

---

## Next Research Topics

1. **Game architecture patterns** - Game loop, state management
2. **Asset pipeline** - Sprites, sound effects
3. **Performance optimization** - FPS targeting on low-end devices

**Credits used:** 4 (2 search queries)
