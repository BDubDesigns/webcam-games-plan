# 📚 Webcam Games Curriculum — Phase 0 & Phase 1

> *"I want to go to there."* — Liz Lemon

A step-by-step, 30-Rock-flavored curriculum for building a webcam-powered motion detection game from scratch using **Vite + React + strict TypeScript + Tailwind CSS + Canvas API**.

---

## Phase 0: Project Setup & DevOps

| # | Lesson | Key Concepts | Files |
|---|--------|-------------|-------|
| 01 | [Vite + React + TypeScript Init](./lesson-01-vite-react-typescript-init.md) | Scaffolding, strict tsconfig, `noUncheckedIndexedAccess` | [Answer Sheet](./lesson-01-vite-react-typescript-init-answer-sheet.md) |
| 02 | [Tailwind CSS Setup](./lesson-02-tailwind-css-setup.md) | Tailwind v4, `@tailwindcss/vite` plugin, utility-first CSS | [Answer Sheet](./lesson-02-tailwind-css-setup-answer-sheet.md) |
| 03 | [ESLint & Prettier](./lesson-03-eslint-prettier.md) | Flat config, `eslint-config-prettier`, react-hooks rules | [Answer Sheet](./lesson-03-eslint-prettier-answer-sheet.md) |
| 04 | [GitHub Actions CI](./lesson-04-github-actions-ci.md) | Workflow YAML, `npm ci`, quality gate pipeline | [Answer Sheet](./lesson-04-github-actions-ci-answer-sheet.md) |
| 05 | [Dockerfile + Nginx](./lesson-05-dockerfile-nginx.md) | Multi-stage builds, SPA routing, layer caching, gzip | [Answer Sheet](./lesson-05-dockerfile-nginx-answer-sheet.md) |

---

## Phase 1: Window Cleaner Game

| # | Lesson | Key Concepts | Files |
|---|--------|-------------|-------|
| 06 | [useWebcam Hook](./lesson-06-use-webcam-hook.md) | `getUserMedia`, MediaStream, Strict Mode cleanup | [Answer Sheet](./lesson-06-use-webcam-hook-answer-sheet.md) |
| 07 | [Canvas Fundamentals](./lesson-07-canvas-fundamentals.md) | `useRef`, `useEffect`, `getContext("2d")`, `drawImage` | [Answer Sheet](./lesson-07-canvas-fundamentals-answer-sheet.md) |
| 08 | [The Game Loop](./lesson-08-game-loop.md) | `requestAnimationFrame`, callback ref pattern, delta time | [Answer Sheet](./lesson-08-game-loop-answer-sheet.md) |
| 09 | [Motion Detection](./lesson-09-motion-detection.md) | Frame differencing, `getImageData`, downscaling, thresholds | [Answer Sheet](./lesson-09-motion-detection-answer-sheet.md) |
| 10 | [The Dirt Layer](./lesson-10-dirt-layer.md) | Procedural generation, `globalCompositeOperation`, offscreen canvas | [Answer Sheet](./lesson-10-dirt-layer-answer-sheet.md) |
| 11 | [Cleaning Mechanics](./lesson-11-cleaning-mechanics.md) | Mirror transform, radial gradients, configurable tuning | [Answer Sheet](./lesson-11-cleaning-mechanics-answer-sheet.md) |
| 12 | [Win State & Particles](./lesson-12-win-state-particles.md) | State machine, bubble physics, particle system architecture | [Answer Sheet](./lesson-12-win-state-particles-answer-sheet.md) |
| 13 | [Final Assembly](./lesson-13-final-assembly.md) | Component composition, render pipeline, production build | [Answer Sheet](./lesson-13-final-assembly-answer-sheet.md) |

---

## Architecture Overview

```
src/
├── components/
│   └── WindowCleanerGame.tsx        ← Main game component
├── game/
│   ├── Bubble.ts                    ← Particle class
│   ├── DirtLayer.ts                 ← Dirt generation + cleaning
│   ├── ParticleSystem.ts            ← Particle lifecycle
│   ├── cleaningConfig.ts            ← Tunable constants
│   └── gameState.ts                 ← State machine
├── hooks/
│   ├── useAnimationFrame.ts         ← rAF game loop
│   ├── useMotionDetection.ts        ← Frame differencing
│   └── useWebcam.ts                 ← Camera access
├── types/
│   └── motion.ts                    ← Motion interfaces
└── App.tsx                          ← Root component
```

---

## How to Use This Curriculum

1. **Read the Lesson file** — understand the theory and requirements.
2. **Build it yourself** — implement the assignment WITHOUT looking at the answer sheet.
3. **Check the Answer Sheet** — compare your implementation with the reference.
4. **Run the checks** — `tsc --noEmit`, `npm run lint`, `npm run format:check`.
5. **Move to the next lesson** — each lesson builds on the previous one.

---

## Cast of Characters

| Character | Role | Teaches |
|-----------|------|---------|
| 🎭 **Tracy Jordan** | Chaotic hype man | Big-picture concepts, features, and game ideas |
| 🎩 **Jack Donaghy** | Corporate Six Sigma overlord | Architecture, best practices, CI/CD, deployment |
| 😰 **Liz Lemon** | Stressed-out pragmatist | Edge cases, memory leaks, React lifecycle nightmares |

---

*"Never go with a hippie to a second location."* — Jack Donaghy
