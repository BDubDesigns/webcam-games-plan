# Lesson 01 — Answer Sheet: Initializing the Vite + React + TypeScript Project

## The Implementation

### Step 1: Scaffold the project

```bash
# Create the project using Vite's React + TypeScript template
npm create vite@latest webcam-games -- --template react-ts

# Navigate into the new project directory
cd webcam-games

# Install all dependencies
npm install
```

### Step 2: Clean up boilerplate

**`src/App.css`** — Clear all contents. The file should be empty:

```css
/* Intentionally left empty. Tailwind CSS will handle all styling. */
```

**`src/App.tsx`** — Replace with a minimal component:

```tsx
/**
 * App.tsx
 *
 * Root application component.
 * Currently renders a placeholder heading.
 * This will be replaced with the game router and canvas in future lessons.
 */
function App(): React.JSX.Element {
  return (
    <main>
      <h1>Webcam Games</h1>
    </main>
  );
}

export default App;
```

**Delete the React logo:**

```bash
rm src/assets/react.svg
```

> **Note:** If `src/assets/` is now empty, you can leave the directory or remove it. It's fine either way — we'll use it later for game assets.

**`src/index.css`** — Clear all contents:

```css
/* Intentionally left empty. Tailwind CSS will be configured in Lesson 02. */
```

### Step 3: Configure strict TypeScript

**`tsconfig.app.json`** — Ensure these compiler options are present:

```jsonc
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",

    /* Linting — this is where the real discipline lives */
    "strict": true,                        // Enables all strict type-checking options
    "noUnusedLocals": true,                // Error on declared but unused local variables
    "noUnusedParameters": true,            // Error on unused function parameters
    "noFallthroughCasesInSwitch": true,    // Error on switch case fallthrough without break
    "noUncheckedIndexedAccess": true       // Adds `undefined` to index signature results
  },
  "include": ["src"]
}
```

**Why `noUncheckedIndexedAccess`?**

This is critical for our game. When we do pixel manipulation:

```ts
const pixels: Uint8ClampedArray = imageData.data;
const red = pixels[i];  // Without this flag: number
                         // With this flag: number | undefined
```

This forces you to handle the case where an index might be out of bounds. For a game that iterates over pixel arrays by computing indices manually (`i * 4`, `i * 4 + 1`, etc.), this catches off-by-one errors at compile time.

### Step 4: Verify everything works

```bash
# Start the dev server — should show "Webcam Games" at http://localhost:5173
npm run dev

# In a separate terminal, run the type checker
# This should exit with code 0 and produce no errors
npx tsc --noEmit
```

**Expected output of `tsc --noEmit`:**

```
(no output — silence means success)
```

If you see errors, read them carefully. Common ones at this stage:

- **"Cannot find module 'react'"** — You forgot to run `npm install`.
- **"'React' refers to a UMD global..."** — Your `jsx` option should be `"react-jsx"`, not `"react"`.
- **Unused import warnings** — If you have `import './App.css'` in `App.tsx`, that's fine. CSS imports don't trigger unused-import errors.

### Final project structure

```
webcam-games/
├── index.html
├── package.json
├── tsconfig.json              # References tsconfig.app.json and tsconfig.node.json
├── tsconfig.app.json          # YOUR strict config lives here
├── tsconfig.node.json         # Config for Vite config file (Node context)
├── vite.config.ts
├── public/
│   └── vite.svg               # Can remove later, harmless for now
├── src/
│   ├── App.css                # Empty
│   ├── App.tsx                # Minimal "Webcam Games" component
│   ├── index.css              # Empty
│   ├── main.tsx               # ReactDOM.createRoot entry point
│   └── vite-env.d.ts          # Vite client type declarations
└── node_modules/
```

---

## Executive Summary

**Jack Donaghy:**

> Let me be crystal clear about what we've accomplished here, because I can see from your face that you think this was "just" running a scaffolding command. That kind of thinking is why you're not in the C-suite.

> What we've done is execute a **Lean Six Sigma "Define" phase** with surgical precision. We have:

> 1. **Selected a best-in-class build tool** (Vite) that leverages native ES modules for development and Rollup for production. This is not a trend — this is operational excellence. Webpack is a legacy encumbrance. We're optimizing for developer throughput per unit of cognitive load.

> 2. **Enforced strict TypeScript from inception.** In business, we call this "setting the quality bar at the point of manufacture, not at the point of inspection." The `noUncheckedIndexedAccess` flag alone will prevent an entire class of runtime errors when we start manipulating pixel arrays. That's not a compiler option — that's a *risk mitigation strategy*.

> 3. **Eliminated technical debt before it accrues.** By removing boilerplate CSS, dead assets, and unnecessary code, we've ensured zero impedance mismatch when we integrate Tailwind CSS and Canvas rendering. In Six Sigma terms, we've reduced our Defects Per Million Opportunities (DPMO) to zero at this stage.

> 4. **Established a verification protocol.** Running `tsc --noEmit` as a separate validation step is the engineering equivalent of a QA gate. Vite's dev server is optimized for speed, not correctness — it will serve you a flaming dumpster fire and smile while doing it. The type checker is your quality assurance department. Never ship without it.

> This is not a "hello world." This is the foundation of a production system. Treat it accordingly. Now if you'll excuse me, I have a meeting with the board about leveraging our synergistic core competencies in the streaming media vertical.
