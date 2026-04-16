# Lesson 01: Initializing the Vite + React + TypeScript Project

## The Pitch

**Tracy Jordan:**
> Yo, LISTEN UP! We are about to build the most INCREDIBLE webcam game the world has ever seen! But first, we gotta lay down the foundation. You know what that's called? It's called a VEET project. Veet? Vite? I don't care how you say it, I like project scaffolding so much I want to take it behind the middle school and get it pregnant! We're building a TEMPLE of code, people, and the first brick is this right here!

**Jack Donaghy:**
> Lemon, what Tracy is attempting to articulate — with all the grace of a malfunctioning Roomba — is that every Six Sigma Black Belt project begins with a robust initialization phase. We call this the "Define" stage in our DMAIC framework. A properly scaffolded Vite project is to front-end development what a corner office on the 52nd floor is to executive culture: non-negotiable. We will leverage Vite's best-in-class HMR capabilities to maximize developer velocity and minimize time-to-first-meaningful-render. This is vertical integration of your toolchain, and it starts now.

**Liz Lemon:**
> Oh God, here we go. Okay, look. I know you're excited. I'm excited too. But can we just... can we please make sure we don't skip the TypeScript strict mode setup? Because I've seen what happens when you let `any` types creep into a codebase. It's like when Jenna tries to "casually" mention her film career — it starts small and then suddenly EVERYTHING is untyped and on fire. Let's just... let's do this right. Please.

---

## The Theory

### What is Vite and why does it matter?

Vite (French for "fast," pronounced "veet") is a next-generation front-end build tool. But *why* does it exist when we already have webpack, Parcel, and others?

**Question:** When you run `npm run dev` in a traditional webpack project with 500 modules, what happens before you see your app in the browser?

**Answer:** Webpack bundles *every single module* into one or more output files before the dev server can serve anything. As your project grows, this startup time grows linearly — or worse.

**Question:** How does Vite solve this?

**Answer:** Vite takes a fundamentally different approach:

1. **During development:** Vite serves your source files directly using native ES modules (ESM). The browser itself resolves `import` statements. Vite only transforms files *on demand* when the browser requests them. This means startup is nearly instant regardless of project size.

2. **During production:** Vite uses Rollup under the hood to produce a highly optimized, tree-shaken, code-split bundle. You get the best of both worlds.

### What does `create vite` actually scaffold?

When you run `npm create vite@latest`, the template generator creates:

- **`index.html`** — The entry point. Unlike CRA, this lives at the project root, not in a `public/` folder. Vite treats `index.html` as the true entry point and processes `<script>` tags within it.
- **`src/main.tsx`** — The React entry file that calls `ReactDOM.createRoot()`.
- **`src/App.tsx`** — The root component.
- **`vite.config.ts`** — Vite's configuration file, using the `@vitejs/plugin-react` plugin.
- **`tsconfig.json`** / **`tsconfig.app.json`** — TypeScript configuration files.
- **`package.json`** — Dependencies and scripts.

### Why strict TypeScript from Day 1?

**Question:** What does `"strict": true` in `tsconfig.json` actually enable?

**Answer:** It's a shorthand that enables a family of strict checks:

| Flag | What it catches |
|------|----------------|
| `strictNullChecks` | Prevents `null` and `undefined` from being assigned to non-nullable types |
| `strictFunctionTypes` | Enforces contravariant parameter types in function types |
| `strictBindCallApply` | Type-checks `bind`, `call`, and `apply` |
| `strictPropertyInitialization` | Ensures class properties are initialized in the constructor |
| `noImplicitAny` | Errors when TypeScript can't infer a type and would fall back to `any` |
| `noImplicitThis` | Errors when `this` has an implicit `any` type |
| `alwaysStrict` | Emits `"use strict"` in every file |

For a game that relies heavily on Canvas API pixel arrays (`Uint8ClampedArray`), typed coordinates, and nullable refs (`useRef<HTMLCanvasElement | null>`), strict mode catches entire categories of bugs at compile time.

### The `type: "module"` in package.json

**Question:** What does `"type": "module"` in your `package.json` do?

**Answer:** It tells Node.js to treat `.js` files as ES modules by default (using `import`/`export`) instead of CommonJS (`require`/`module.exports`). This aligns your Node-side tooling (Vite config, scripts) with the module system you're already using in your browser-side TypeScript code.

---

## The Assignment

Your task is to scaffold and configure a new Vite + React + TypeScript project. Follow these steps exactly:

### Step 1: Scaffold the project

- Use `npm create vite@latest` with the `react-ts` template.
- The project should be named `webcam-games`.
- Install all dependencies.

### Step 2: Clean up the boilerplate

- Delete the contents of `src/App.css`.
- Replace the contents of `src/App.tsx` with a minimal component that renders a single `<h1>` element with the text "Webcam Games".
- Delete `src/assets/react.svg` (the React logo).
- Clear out the default Vite styles from `src/index.css` — leave the file empty for now (Tailwind will take over in the next lesson).

### Step 3: Verify the tsconfig is strict

- Open `tsconfig.app.json` (this is the one Vite's template uses for your app code).
- Confirm that `"strict": true` is present in `compilerOptions`.
- If it isn't, add it.
- Additionally, add the following compiler options if they are not already present:
  - `"noUnusedLocals": true`
  - `"noUnusedParameters": true`
  - `"noFallthroughCasesInSwitch": true`
  - `"noUncheckedIndexedAccess": true`

### Step 4: Verify the dev server runs

- Run `npm run dev`.
- Confirm that the application renders "Webcam Games" in the browser at `http://localhost:5173`.
- Confirm there are zero TypeScript errors by running `npx tsc --noEmit`.

### Deliverables

After completing this lesson, your project should have:

1. A clean Vite + React + TypeScript scaffold with no boilerplate content.
2. A strict `tsconfig.app.json` configuration.
3. A minimal `App.tsx` component rendering a heading.
4. Zero type errors when running `tsc --noEmit`.

---

## Liz's Nightmares

> **Liz Lemon:** Okay, before you go all Tracy Jordan on this and just start smashing keys, let me tell you what's going to go wrong. Because it WILL go wrong. It always does.

1. **Using `npm init vite` instead of `npm create vite@latest`:** The `init` command is deprecated. Always use `create`. If you use the old one, you'll get a stale template and then you'll be debugging phantom TypeScript errors at 2 AM eating cheese from a sleeve of crackers. Don't be me.

2. **Forgetting to `cd` into the project before running `npm install`:** The scaffolder creates a new directory. If you run `npm install` in the parent directory, you'll install nothing and wonder why `react` is missing. I've done this. More than once. Fine, four times.

3. **Editing `tsconfig.json` instead of `tsconfig.app.json`:** Vite's current React-TS template uses a *project references* setup. The root `tsconfig.json` just references `tsconfig.app.json` and `tsconfig.node.json`. Your app's compiler options live in `tsconfig.app.json`. If you add `noUnusedLocals` to the root config, it won't apply to your app code and you'll think it's working when it's not. This is the tsconfig equivalent of a silent emotional breakdown.

4. **Not running `tsc --noEmit` separately from the dev server:** Vite's dev server uses esbuild for transpilation, which is *blazingly fast* but does NOT perform type checking. It strips the types and moves on. You can have 47 type errors and Vite will happily serve your app like nothing's wrong. You MUST run `tsc --noEmit` separately (or set up a CI check, which we will in Lesson 04) to actually catch type errors. This is not optional. This is survival.

5. **Leaving boilerplate CSS in place:** The default Vite template includes styles that center everything and add hover effects to the logo. If you don't clear these out now, they'll fight with Tailwind later, and debugging CSS specificity conflicts is my personal definition of hell.
