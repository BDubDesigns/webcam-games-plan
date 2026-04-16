# Lesson 03 — Answer Sheet: ESLint & Prettier — The Quality Gates

## The Implementation

### Step 1: Install dependencies

```bash
# Prettier: the opinionated code formatter
# eslint-config-prettier: disables ESLint rules that conflict with Prettier
npm install -D prettier eslint-config-prettier
```

### Step 2: Configure Prettier

**`.prettierrc`** (create at project root):

```json
{
  "semi": true,
  "singleQuote": false,
  "tabWidth": 2,
  "trailingComma": "all",
  "printWidth": 80,
  "bracketSpacing": true,
  "arrowParens": "always"
}
```

**Why these choices:**

- **`semi: true`** — Explicit semicolons prevent ASI (Automatic Semicolon Insertion) gotchas. This is especially important in minified production builds.
- **`singleQuote: false`** — JSX uses double quotes by convention (`<div className="...">`). Using double quotes in TS/JS keeps things consistent with JSX.
- **`trailingComma: "all"`** — Adding items to arrays, objects, and function parameters produces cleaner git diffs (only the new line shows as changed, not the previous line getting a comma added).
- **`arrowParens: "always"`** — `(x) => x` instead of `x => x`. When you add a type annotation later (`(x: number) => x`), the parens are already there. Reduces diff noise.

### Step 3: Create `.prettierignore`

**`.prettierignore`** (create at project root):

```
dist/
node_modules/
*.min.*
```

### Step 4: Update ESLint configuration

**`eslint.config.js`:**

```js
import js from "@eslint/js";
import globals from "globals";
import reactHooks from "eslint-plugin-react-hooks";
import reactRefresh from "eslint-plugin-react-refresh";
import tseslint from "typescript-eslint";
import eslintConfigPrettier from "eslint-config-prettier";

export default tseslint.config(
  // Ignore the dist directory globally
  { ignores: ["dist"] },

  // Base configurations
  {
    // Extend recommended configs for JS and TS
    extends: [js.configs.recommended, ...tseslint.configs.recommended],

    // Apply to TypeScript and TSX files only
    files: ["**/*.{ts,tsx}"],

    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
    },

    plugins: {
      // React Hooks plugin: enforces Rules of Hooks
      "react-hooks": reactHooks,
      // React Refresh plugin: ensures components are compatible with HMR
      "react-refresh": reactRefresh,
    },

    rules: {
      // --- React Hooks Rules ---

      // "rules-of-hooks": Ensures hooks are only called at the top level
      // and only from React function components or custom hooks.
      // This is non-negotiable. Breaking this rule causes runtime crashes.
      ...reactHooks.configs.recommended.rules,

      // --- React Refresh Rules ---

      // Ensures only React components are exported from modules for proper HMR.
      // allowConstantExport: true permits `export const CONSTANT = ...` alongside
      // component exports, which is common in utility-heavy game code.
      "react-refresh/only-export-components": [
        "warn",
        { allowConstantExport: true },
      ],
    },
  },

  // IMPORTANT: eslint-config-prettier MUST be last.
  // It disables all ESLint formatting rules that would conflict with Prettier.
  // If any config object comes after this, it could re-enable conflicting rules.
  eslintConfigPrettier,
);
```

**Key architectural decisions:**

1. **`eslintConfigPrettier` is the last entry.** This is the single most important thing in this file. It turns off every ESLint rule that overlaps with Prettier (indentation, quotes, semicolons, etc.). If it's not last, other configs can re-enable those rules downstream.

2. **`react-hooks` rules are spread from the plugin's recommended config.** This gives us:
   - `react-hooks/rules-of-hooks: "error"` — Hooks must be called at the top level, not inside conditions or loops.
   - `react-hooks/exhaustive-deps: "warn"` — Dependency arrays in `useEffect`, `useMemo`, and `useCallback` must include all referenced values.

3. **Files are scoped to `**/*.{ts,tsx}`.** TypeScript rules don't apply to `.js` config files.

### Step 5: Add scripts to package.json

Add these to the `"scripts"` section of `package.json`:

```jsonc
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "eslint src/",
    "format": "prettier --write \"src/**/*.{ts,tsx,css,json}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,css,json}\"",
    "preview": "vite preview"
  }
}
```

**Script breakdown:**

| Script | Purpose | When to use |
|--------|---------|-------------|
| `lint` | Run ESLint on all source files | During development, before committing |
| `format` | Auto-format all source files with Prettier | During development, before committing |
| `format:check` | Check formatting without changing files | In CI — fails if any file is unformatted |

### Step 6: Run everything

```bash
# Format the entire codebase first
npm run format

# Run ESLint
npm run lint

# Run TypeScript type checking
npx tsc --noEmit
```

**Expected output:**

```bash
$ npm run format
# Lists all formatted files (or says "X files unchanged")

$ npm run lint
# No output = no errors

$ npx tsc --noEmit
# No output = no type errors
```

### Final configuration file overview

```
webcam-games/
├── .prettierrc                 # Prettier configuration
├── .prettierignore             # Files Prettier should skip
├── eslint.config.js            # ESLint flat config with Prettier bridge
├── package.json                # Updated with lint/format scripts
├── ... (everything else unchanged)
```

---

## Executive Summary

**Jack Donaghy:**

> Let me tell you a story about quality control. In 1986, Motorola introduced Six Sigma — a methodology that reduces defects to 3.4 per million opportunities. They didn't achieve this by hoping their engineers would write good code. They achieved it by *building quality gates into the process itself*.

> That's exactly what we've done here:

> 1. **Prettier is our automated formatting press.** It takes raw, inconsistent developer output and produces uniform, specification-compliant code. There is no debate. There is no meeting. There is no "I prefer tabs." Prettier has decided, and Prettier is correct. This eliminates an entire category of code review friction — what I call "aesthetic defects" — that consume 20-30% of review cycles in undisciplined organizations.

> 2. **ESLint is our in-line quality inspector.** It catches defects at the point of production: unused variables (dead inventory), missing hook dependencies (process defects), and suspicious patterns (early warning indicators). The `react-hooks/exhaustive-deps` rule alone will save us from an entire class of stale-closure bugs when we build the game loop. That's not a lint rule — that's a *defect prevention system*.

> 3. **The separation of concerns between ESLint and Prettier — enforced by `eslint-config-prettier` — is organizational clarity.** Each tool has a defined scope. There is no overlap, no territorial conflict, no passive-aggressive interoffice memos about semicolons. This is the kind of operational efficiency that gets you invited to speak at Davos.

> 4. **The `format:check` script is our CI quality gate.** It doesn't fix problems — it *detects* them. In the next lesson, we'll wire this into GitHub Actions so that no unformatted code ever reaches the main branch. This is the difference between quality as an aspiration and quality as a guarantee.

> We have now established a zero-defect pipeline from developer keystroke to formatted, linted, type-checked code. The marginal cost of adding these tools is hours. The marginal cost of NOT adding them is weeks of debugging, refactoring, and interpersonal conflict. I've run the numbers. They don't lie.
