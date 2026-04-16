# Lesson 03: ESLint & Prettier — The Quality Gates

## The Pitch

**Tracy Jordan:**
> LISTEN! I just tried to commit some code and a ROBOT told me I had a "dangling comma." That's not even a THING! You can't dangle a comma! But apparently there's this thing called a "LINTER" — which sounds like something you pull out of a dryer — and it YELLS at you when your code isn't pretty enough. I like code formatting so much I want to take it behind the middle school and get it pregnant!

**Jack Donaghy:**
> Tracy, that "robot" is called ESLint, and it's the most important quality assurance mechanism in your entire pipeline. Let me explain this in terms of vertical integration. In manufacturing, you don't wait until the product is on the shelf to discover defects — you build quality into the production line. ESLint is your production line inspector. Prettier is your finish line quality check. Together, they enforce *zero-defect methodology* at the developer level, before code ever reaches version control. This is not bureaucracy. This is discipline. And discipline is what separates NBC from a public access channel in Schenectady.

**Liz Lemon:**
> Okay, I'm actually on Jack's side for once, which is terrifying. But here's the thing — ESLint and Prettier have overlapping concerns, and if you configure them wrong, they'll FIGHT each other. ESLint will say "add a semicolon" and Prettier will say "remove it" and you'll be stuck in an infinite loop of auto-fixing like some kind of developer Groundhog Day. We need to set them up so they don't conflict. I've lost entire evenings to this. ENTIRE EVENINGS, people.

---

## The Theory

### ESLint: What and Why

**Question:** What is ESLint and why can't TypeScript alone replace it?

**Answer:** ESLint is a *static analysis* tool for JavaScript and TypeScript. While TypeScript catches *type errors*, ESLint catches *code quality issues*:

| Tool | Catches | Example |
|------|---------|---------|
| TypeScript | Type mismatches, null safety, interface violations | `const x: number = "hello"` |
| ESLint | Unused variables, suspicious patterns, React hook rule violations | Using `useEffect` without dependency array |

**Critical for our project:** ESLint's `react-hooks/exhaustive-deps` rule catches missing dependencies in `useEffect` — which is exactly the kind of bug that causes stale closures in our game loop and memory leaks when unmounting.

### Prettier: What and Why

**Question:** Why not just use ESLint for formatting too?

**Answer:** ESLint *can* enforce formatting rules, but it's slow at it and the rules are limited. Prettier is an **opinionated code formatter** that:

1. Parses your code into an AST (Abstract Syntax Tree).
2. Reprints it from scratch according to its rules.
3. Produces consistent output regardless of what you wrote.

This means two developers can write the same logic in wildly different styles, and after Prettier, the output is identical. It eliminates *all* formatting debates. Zero meetings about tabs vs. spaces. Zero.

### The Separation of Concerns

**Question:** How do we prevent ESLint and Prettier from conflicting?

**Answer:** The modern approach uses `eslint-config-prettier`, which **disables all ESLint rules that would conflict with Prettier**. The workflow becomes:

1. **Prettier** handles formatting: indentation, line length, quotes, semicolons, trailing commas.
2. **ESLint** handles code quality: unused vars, hook rules, suspicious patterns.
3. **`eslint-config-prettier`** turns off any ESLint formatting rules so they don't clash.

### ESLint Flat Config (v9+)

**Question:** What is the "flat config" format?

**Answer:** ESLint v9 introduced a new configuration format:

- **Old format:** `.eslintrc.json` with `extends`, `plugins`, `overrides` — deeply nested, hard to reason about.
- **New format:** `eslint.config.js` (or `.mjs`/`.ts`) — a flat array of config objects. Each object can specify `files`, `rules`, `plugins`, and `languageOptions`. Composition is explicit: later objects override earlier ones.

Vite's React-TS template already scaffolds an `eslint.config.js` using flat config. We'll build on that.

---

## The Assignment

### Step 1: Install Prettier and the ESLint bridge

- Install `prettier` as a dev dependency.
- Install `eslint-config-prettier` as a dev dependency. This disables ESLint rules that conflict with Prettier.

### Step 2: Configure Prettier

- Create a `.prettierrc` file at the project root with these settings:
  - `semi`: `true` (always use semicolons)
  - `singleQuote`: `false` (use double quotes — matches JSX convention)
  - `tabWidth`: `2`
  - `trailingComma`: `"all"` (add trailing commas everywhere valid — improves git diffs)
  - `printWidth`: `80`
  - `bracketSpacing`: `true`
  - `arrowParens`: `"always"` (always wrap arrow function parameters in parens)

### Step 3: Update the ESLint configuration

- Open the existing `eslint.config.js` file.
- Import `eslint-config-prettier` and add it as the **last** entry in the config array. This ensures Prettier's rule overrides take precedence.
- Ensure the configuration includes:
  - TypeScript-aware parsing (should already be there from the Vite template).
  - React Hooks linting (`react-hooks/rules-of-hooks` and `react-hooks/exhaustive-deps`).
  - React Refresh linting (should already be there).

### Step 4: Add formatting and linting scripts

- Add these scripts to `package.json`:
  - `"lint"`: Runs ESLint on the `src/` directory.
  - `"format"`: Runs Prettier to format all `.ts`, `.tsx`, `.css`, and `.json` files.
  - `"format:check"`: Runs Prettier in check mode (reports unformatted files without fixing them — for CI).

### Step 5: Create ignore files

- Create a `.prettierignore` file that ignores `dist/`, `node_modules/`, and `*.min.*` files.

### Step 6: Run and verify

- Run `npm run format` to format the entire codebase.
- Run `npm run lint` to lint the codebase.
- Both commands should complete with zero errors.
- Run `npx tsc --noEmit` to confirm no type errors were introduced.

### Deliverables

1. Prettier configured with a `.prettierrc` file.
2. ESLint config updated with `eslint-config-prettier` as the last config entry.
3. `package.json` has `lint`, `format`, and `format:check` scripts.
4. `.prettierignore` file exists.
5. Running `lint`, `format:check`, and `tsc --noEmit` all pass with zero errors.

---

## Liz's Nightmares

> **Liz Lemon:** Formatting tools. Sounds harmless. Like a nice warm blanket. IT'S A TRAP.

1. **Putting `eslint-config-prettier` anywhere other than LAST in the config array:** The entire point of this config is to disable ESLint formatting rules. If another config comes after it and re-enables those rules, they'll conflict with Prettier again. It MUST be the last entry. This is not a suggestion. This is a commandment.

2. **Mixing `.eslintrc.json` (old format) with `eslint.config.js` (flat config):** If you have both files, ESLint gets confused and uses one or the other depending on the version and flags. The Vite template creates `eslint.config.js`. Use that. Delete any `.eslintrc` files if they exist.

3. **Forgetting `"react-hooks/exhaustive-deps": "warn"`:** This rule is the ONLY THING standing between you and stale closure bugs in `useEffect`. When we build the game loop, we'll have `useEffect` hooks that depend on callback refs and frame data. If the dependency array is wrong, your game loop will capture a stale snapshot of state and you'll spend three hours wondering why your motion detector is always one frame behind. This rule catches that. Keep it on. Please. I'm begging you.

4. **Running Prettier before ESLint:** In your editor, configure it to run Prettier on save, and ESLint separately. If you run ESLint with `--fix` first and then Prettier, they can fight. The correct order is: ESLint fixes code quality issues, Prettier formats the result. Most editors handle this correctly when both extensions are installed with the right settings.

5. **Not creating `.prettierignore`:** Without it, Prettier will try to format everything in your project, including `node_modules`, `dist/`, and any generated files. It'll either crash, take forever, or produce garbage diffs in your version control. Always ignore build outputs.
