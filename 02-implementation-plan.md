# Implementation Plan - Webcam Games for Kids

**Date:** Research Phase 2 of 4
**Focus:** Step-by-step learning path and game-specific architecture

---

## Phase 0: Project Setup (Day 1)

### Minimal Project Structure
```
webcam-games/
├── index.html          # Main entry, fullscreen canvas
├── css/
│   └── style.css       # Minimal, fullscreen styles
├── js/
│   ├── core/
│   │   ├── camera.js   # Webcam access, canvas setup
│   │   ├── loop.js     # Game loop (requestAnimationFrame)
│   │   └── utils.js    # Math helpers, collision detection
│   ├── games/
│   │   ├── window-cleaner/  # Phase 1
│   │   ├── pose-match/      # Phase 2
│   │   ├── hammer-squish/   # Phase 3
│   │   └── lane-runner/     # Phase 4
│   └── main.js         # Router, game switching
├── assets/
│   ├── sprites/        # PNGs (dirt, hammer, bugs, coins)
│   └── sounds/         # Optional MP3s
└── lib/
    └── mediapipe/      # CDN links preferred
```

### CDN Dependencies (No Build Step)
```html
<!-- MediaPipe -->
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/pose/pose.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
```

---

## Phase 1: Window Cleaner Game (Week 1)

### Goal
Learn: Webcam access, canvas manipulation, pixel comparison, simple game state

### Architecture

```javascript
// Core classes
class Camera {
  async init() { /* getUserMedia */ }
  getFrame() { /* draw video to canvas, return ImageData */ }
}

class DirtLayer {
  constructor(width, height) {
    this.canvas = document.createElement('canvas');
    this.canvas.width = width / 10;  // Downscale for performance
    this.canvas.height = height / 10;
    this.ctx = this.canvas.getContext('2d');
    this.generateDirt();
  }

  generateDirt() {
    // Fill with brown noise/speckles
    // Store alpha values for each "dirt pixel"
  }

  clean(x, y, radius, amount) {
    // Reduce alpha in region
    // Return true if fully clean
  }

  isFullyClean() { /* check if all alpha < threshold */ }
}

class MotionDetector {
  constructor() {
    this.previousFrame = null;
    this.threshold = 30;  // Pixel diff threshold
  }

  detect(currentFrame) {
    // 1. Downscale current frame
    // 2. Compare with previousFrame
    // 3. Return array of "hot spots" (x, y, intensity)
    // 4. Store current as previous
  }
}

class WindowCleanerGame {
  constructor() {
    this.state = 'waiting'; // waiting | playing | won
    this.dirtLayer = new DirtLayer(640, 480);
    this.motionDetector = new MotionDetector();
    this.bubbles = []; // Particle array
  }

  update() {
    if (this.state !== 'playing') return;

    const frame = camera.getFrame();
    const motions = this.motionDetector.detect(frame);

    motions.forEach(({x, y, intensity}) => {
      if (intensity > THRESHOLD) {
        const cleaned = this.dirtLayer.clean(x, y, 20, intensity);
        if (cleaned) this.spawnBubbles(x, y);
      }
    });

    if (this.dirtLayer.isFullyClean()) {
      this.state = 'won';
    }

    // Update bubbles
    this.bubbles = this.bubbles.filter(b => b.update());
  }

  render(ctx) {
    // 1. Draw webcam feed
    ctx.drawImage(camera.video, 0, 0);

    // 2. Draw window frame overlay (CSS or canvas)

    // 3. Draw dirt layer (scaled up)
    ctx.drawImage(this.dirtLayer.canvas, 0, 0, 640, 480);

    // 4. Draw bubbles
    this.bubbles.forEach(b => b.render(ctx));

    // 5. Draw UI (percent clean, timer)
  }
}
```

### Key Technical Challenges

1. **Motion Threshold Tuning**
   - Start with: 30/255 pixel difference
   - Require: 100+ changed pixels to trigger clean
   - Test with: Low-quality webcam, slow movements

2. **Dirt Visualization**
   - Use: `ctx.globalAlpha` for transparency
   - Alternative: Composite with `destination-out` to "erase"

3. **Bubbles**
   - Simple particle system: x, y, radius, speedY
   - Float upward, fade out

---

## Phase 2: Pose Match Game (Week 2)

### Goal
Learn: MediaPipe Pose integration, landmark processing, pose comparison

### Architecture

```javascript
class PoseDetector {
  constructor() {
    this.pose = new Pose({
      locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/pose/${file}`
    });
    this.pose.setOptions({
      modelComplexity: 1,  // 0=light, 1=full, 2=heavy
      smoothLandmarks: true,
      minDetectionConfidence: 0.5,
      minTrackingConfidence: 0.5
    });
    this.currentPose = null;
  }

  async init() {
    this.pose.onResults((results) => {
      if (results.poseLandmarks) {
        this.currentPose = results.poseLandmarks;
      }
    });
    await this.pose.send({image: camera.video});
  }

  getPose() { return this.currentPose; }
}

// 33 Landmarks from MediaPipe Pose
const LANDMARKS = {
  NOSE: 0,
  LEFT_SHOULDER: 11, RIGHT_SHOULDER: 12,
  LEFT_ELBOW: 13, RIGHT_ELBOW: 14,
  LEFT_WRIST: 15, RIGHT_WRIST: 16,
  LEFT_HIP: 23, RIGHT_HIP: 24,
  LEFT_KNEE: 25, RIGHT_KNEE: 26,
  LEFT_ANKLE: 27, RIGHT_ANKLE: 28,
  // ... see full list in docs
};

class PoseMatcher {
  comparePose(current, target) {
    // Normalize both poses (scale-invariant)
    // Compare key angles: shoulder-elbow-wrist, hip-knee-ankle
    // Return: 0-1 similarity score
  }

  calculateAngle(a, b, c) {
    // Vector math: angle at point b
    // Used for: "Make your arm straight" / "Bend your knee"
  }
}

class PoseMatchGame {
  constructor() {
    this.targets = [
      { name: 'T-Pose', landmarks: [...], duration: 3000 },
      { name: 'Touch Toes', landmarks: [...], duration: 2000 },
      { name: 'Star Jump', landmarks: [...], duration: 1000 },
    ];
    this.currentTarget = 0;
    this.holdStart = null;
    this.score = 0;
  }

  update() {
    const pose = poseDetector.getPose();
    if (!pose) return;

    const target = this.targets[this.currentTarget];
    const similarity = poseMatcher.comparePose(pose, target.landmarks);

    if (similarity > 0.8) {
      if (!this.holdStart) this.holdStart = Date.now();

      if (Date.now() - this.holdStart > target.duration) {
        this.score++;
        this.currentTarget = (this.currentTarget + 1) % this.targets.length;
        this.holdStart = null;
      }
    } else {
      this.holdStart = null;
    }
  }

  render(ctx) {
    // 1. Draw webcam
    // 2. Draw skeleton overlay (connect landmarks)
    // 3. Draw target pose silhouette (ghost/transparent)
    // 4. Draw progress bar for "hold" timer
    // 5. Draw score
  }
}
```

### Key Technical Challenges

1. **Mirror Effect**
   - `ctx.scale(-1, 1)` flips canvas horizontally
   - Makes it feel like a mirror (kids understand easier)

2. **Pose Normalization**
   - Scale: Distance between shoulders = 1.0 unit
   - Center: Midpoint of hips = origin (0, 0)
   - This makes comparison work regardless of distance from camera

3. **Forgiving Detection**
   - Use angle ranges instead of exact positions
   - "Arm up" = angle between shoulder-hip and shoulder-wrist > 60°

---

## Phase 3: Hammer Squish Game (Week 3)

### Goal
Learn: Hand tracking, sprite rendering, collision detection, particle effects

### Architecture

```javascript
class HandTracker {
  constructor() {
    this.hands = new Hands({...});
    this.hands.setOptions({
      maxNumHands: 2,  // Could use both hands!
      modelComplexity: 1,
      minDetectionConfidence: 0.5,
      minTrackingConfidence: 0.5
    });
  }

  // 21 landmarks per hand
  // 0: wrist
  // 1-4: thumb
  // 5-8: index
  // 9-12: middle
  // 13-16: ring
  // 17-20: pinky

  getHandPosition(landmarks) {
    // Use wrist (0) or middle finger base (9)
    return {
      x: landmarks[9].x,
      y: landmarks[9].y,
      // Calculate rotation from wrist to middle tip
      angle: Math.atan2(
        landmarks[12].y - landmarks[9].y,
        landmarks[12].x - landmarks[9].x
      )
    };
  }
}

class Bug {
  constructor() {
    this.x = Math.random() * canvas.width;
    this.y = Math.random() * canvas.height;
    this.speedX = (Math.random() - 0.5) * 2;
    this.speedY = (Math.random() - 0.5) * 2;
    this.alive = true;
  }

  update() {
    this.x += this.speedX;
    this.y += this.speedY;

    // Bounce off edges
    if (this.x < 0 || this.x > canvas.width) this.speedX *= -1;
    if (this.y < 0 || this.y > canvas.height) this.speedY *= -1;
  }

  render(ctx) {
    // Draw bug sprite or emoji 🐛
  }
}

class Particle {
  // Squish effect: splatter, fade out
}

class HammerSquishGame {
  constructor() {
    this.hammer = {
      sprite: new Image(),
      width: 100,
      height: 100,
      offsetX: -50,  // Center on hand
      offsetY: -80
    };
    this.bugs = [];
    this.splatters = [];
    this.score = 0;
    this.spawnTimer = 0;
  }

  update() {
    // Update bugs
    this.bugs.forEach(bug => bug.update());

    // Spawn new bugs
    this.spawnTimer++;
    if (this.spawnTimer > 60 && this.bugs.length < 10) {
      this.bugs.push(new Bug());
      this.spawnTimer = 0;
    }

    // Check collisions
    const hand = handTracker.getHandPosition();
    if (hand) {
      this.bugs.forEach(bug => {
        if (bug.alive && this.checkCollision(hand, bug)) {
          bug.alive = false;
          this.score++;
          this.spawnSplatter(bug.x, bug.y);
        }
      });
    }

    // Remove dead bugs, update particles
    this.bugs = this.bugs.filter(b => b.alive);
    this.splatters = this.splatters.filter(s => s.update());
  }

  checkCollision(hand, bug) {
    const dx = hand.x - bug.x;
    const dy = hand.y - bug.y;
    return Math.sqrt(dx*dx + dy*dy) < 50;  // 50px hit radius
  }

  render(ctx) {
    // 1. Draw webcam
    // 2. Draw bugs
    this.bugs.forEach(b => b.render(ctx));

    // 3. Draw splatter effects
    this.splatters.forEach(s => s.render(ctx));

    // 4. Draw hammer at hand position
    const hand = handTracker.getHandPosition();
    if (hand) {
      ctx.save();
      ctx.translate(hand.x, hand.y);
      ctx.rotate(hand.angle + Math.PI/2);  // Adjust for sprite orientation
      ctx.drawImage(
        this.hammer.sprite,
        this.hammer.offsetX,
        this.hammer.offsetY,
        this.hammer.width,
        this.hammer.height
      );
      ctx.restore();
    }

    // 5. Draw score
  }
}
```

### Key Technical Challenges

1. **Hammer Rotation**
   - Use hand angle from wrist to middle finger
   - Add 90° offset depending on sprite orientation
   - Smooth rotation: `lerp(currentAngle, targetAngle, 0.2)`

2. **Z-Order**
   - Draw: Webcam → Splatters → Bugs → Hammer
   - Hammer should appear on top

3. **Performance**
   - Limit bugs to 10-15 on screen
   - Object pooling for particles

---

## Phase 4: Lane Runner Game (Week 4)

### Goal
Learn: Body position detection, lane-based logic, spawning patterns, game difficulty

### Architecture

```javascript
class LaneDetector {
  constructor() {
    this.lanes = [
      { x: 0.2, label: 'left' },    // 20% from left
      { x: 0.5, label: 'center' },  // Center
      { x: 0.8, label: 'right' }    // 80% from left
    ];
  }

  getPlayerLane(pose) {
    // Use hip center or nose x position
    const playerX = (pose[23].x + pose[24].x) / 2;  // Hip center

    if (playerX < 0.33) return 0;  // Left
    if (playerX < 0.66) return 1;  // Center
    return 2;  // Right
  }
}

class Obstacle {
  constructor(lane) {
    this.lane = lane;
    this.y = -50;  // Start above screen
    this.speed = 5;
    this.type = Math.random() > 0.5 ? 'coin' : 'monster';
  }

  update() {
    this.y += this.speed;
  }

  render(ctx) {
    const x = this.lane * (canvas.width / 3) + (canvas.width / 6);
    // Draw coin 🪙 or monster 👾 at x, this.y
  }
}

class LaneRunnerGame {
  constructor() {
    this.laneDetector = new LaneDetector();
    this.obstacles = [];
    this.score = 0;
    this.lives = 3;
    this.gameSpeed = 1;
    this.spawnTimer = 0;
  }

  update() {
    // Get player position
    const pose = poseDetector.getPose();
    if (!pose) return;

    const playerLane = this.laneDetector.getPlayerLane(pose);

    // Update obstacles
    this.obstacles.forEach(obs => {
      obs.update();

      // Check collision when obstacle reaches bottom
      if (obs.y > canvas.height - 100) {
        if (obs.lane === playerLane) {
          if (obs.type === 'coin') {
            this.score += 10;
          } else {
            this.lives--;
            // Screen shake effect
          }
          obs.hit = true;
        }
      }
    });

    // Remove off-screen and hit obstacles
    this.obstacles = this.obstacles.filter(
      o => o.y < canvas.height + 50 && !o.hit
    );

    // Spawn new obstacles
    this.spawnTimer++;
    const spawnRate = Math.max(30, 100 - this.score / 10);  // Gets faster
    if (this.spawnTimer > spawnRate) {
      const lane = Math.floor(Math.random() * 3);
      this.obstacles.push(new Obstacle(lane));
      this.spawnTimer = 0;
    }

    // Game over check
    if (this.lives <= 0) {
      this.state = 'gameover';
    }
  }

  render(ctx) {
    // 1. Draw webcam
    // 2. Draw lane lines (subtle UI)
    ctx.strokeStyle = 'rgba(255,255,255,0.3)';
    ctx.beginPath();
    ctx.moveTo(canvas.width/3, 0);
    ctx.lineTo(canvas.width/3, canvas.height);
    ctx.moveTo(2*canvas.width/3, 0);
    ctx.lineTo(2*canvas.width/3, canvas.height);
    ctx.stroke();

    // 3. Draw obstacles
    this.obstacles.forEach(o => o.render(ctx));

    // 4. Draw player indicator (glow under current lane)
    const playerLane = this.laneDetector.getPlayerLane(poseDetector.getPose());
    // Draw glowing highlight

    // 5. Draw UI (score, lives)
  }
}
```

### Key Technical Challenges

1. **Lane Dead Zones**
   - Leave 10% buffer on edges (0-0.1, 0.9-1.0) so kids don't have to be precise
   - Visual feedback: Highlight the active lane

2. **Speed Scaling**
   - Start slow (obstacle speed = 3px/frame)
   - Increase by 0.1 every 100 points
   - Cap at speed = 10 to keep it playable

3. **Fair Spawning**
   - Don't spawn in same lane twice in a row
   - Ensure coins outnumber monsters 2:1

---

## Shared Systems

### Game State Manager
```javascript
class GameManager {
  constructor() {
    this.games = {
      'window-cleaner': new WindowCleanerGame(),
      'pose-match': new PoseMatchGame(),
      'hammer-squish': new HammerSquishGame(),
      'lane-runner': new LaneRunnerGame()
    };
    this.currentGame = null;
  }

  switchGame(gameName) {
    this.currentGame = this.games[gameName];
    this.currentGame.init();
  }

  update() {
    if (this.currentGame) {
      this.currentGame.update();
    }
  }

  render(ctx) {
    if (this.currentGame) {
      this.currentGame.render(ctx);
    }
  }
}
```

### Camera Singleton
```javascript
// Shared across all games
const camera = new Camera();
await camera.init();
```

### Pose Detector Singleton
```javascript
// Shared across pose-based games
const poseDetector = new PoseDetector();
await poseDetector.init();
```

---

## Assets Needed

| Asset | Format | Source |
|-------|--------|--------|
| Dirt speckles | PNG with alpha | Generate procedurally or GIMP |
| Hammer | PNG | Draw or find free sprite |
| Bugs | PNG/emoji | 🐛 or custom sprite |
| Coins | PNG/emoji | 🪙 or custom |
| Monsters | PNG/emoji | 👾 or custom |
| Window frame | PNG | CSS border or image |
| Bubbles | Procedural | Canvas circles with gradient |
| Splatter | Procedural | Canvas radial gradient |

**Free sprite resources:**
- Kenney.nl (game assets)
- OpenGameArt.org
- Emojis as fallback (built-in, no assets needed)

---

## Credits Used

- **Search 1:** "tensorflow.js pose detection hand tracking 2025" - 2 credits
- **Search 2:** "javascript canvas pixel difference motion detection webcam" - 2 credits
- **Search 3:** "javascript game engine architecture game loop state management" - 2 credits
- **Total:** 6 credits

---

## Next Phase

Phase 3 will cover:
- Performance optimization strategies
- Testing on low-end devices
- Sound integration (optional)
- Deployment options (GitHub Pages, Netlify)
