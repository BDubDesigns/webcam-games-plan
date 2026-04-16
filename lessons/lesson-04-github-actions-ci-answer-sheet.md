# Lesson 04 — Answer Sheet: GitHub Actions CI Workflow

## The Implementation

### Step 1: Create the workflow directory

```bash
mkdir -p .github/workflows
```

### Step 2: Write the CI workflow

**`.github/workflows/main.yml`:**

```yaml
# =============================================================================
# CI Workflow — Quality Gate
# =============================================================================
# This workflow runs on every push to main and every pull request targeting main.
# It enforces three quality checks:
#   1. TypeScript type safety (tsc --noEmit)
#   2. Code quality (ESLint)
#   3. Code formatting (Prettier)
#
# If ANY check fails, the workflow fails and the PR is flagged.
# =============================================================================

name: CI

# --- Triggers ---
# When should this workflow run?
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

# --- Jobs ---
# A job is a set of steps that run on the same runner (virtual machine).
# We define a single job because our checks are fast and sequential.
# For larger projects, you might split type-check, lint, and format into
# parallel jobs — but for our project, the overhead of spinning up multiple
# runners outweighs the time saved.
jobs:
  quality-gate:
    name: Type Check, Lint & Format
    runs-on: ubuntu-latest

    steps:
      # Step 1: Checkout the repository
      # Without this, the runner has an empty filesystem.
      # actions/checkout@v4 clones the repo at the commit that triggered the workflow.
      - name: Checkout repository
        uses: actions/checkout@v4

      # Step 2: Set up Node.js
      # - Installs Node.js 20 on the runner
      # - Enables npm caching: dependencies are cached between runs based on
      #   package-lock.json hash. If the lock file hasn't changed, npm ci
      #   can skip downloading packages. This typically saves 30-60 seconds.
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      # Step 3: Install dependencies
      # npm ci (clean install):
      # - Deletes node_modules entirely
      # - Installs exact versions from package-lock.json
      # - Fails if package-lock.json is out of sync with package.json
      # - Faster than npm install (skips dependency resolution)
      # This ensures our CI environment matches production EXACTLY.
      - name: Install dependencies
        run: npm ci

      # Step 4: TypeScript type checking
      # tsc --noEmit:
      # - Runs the full TypeScript compiler
      # - Checks all types, interfaces, and generic constraints
      # - Does NOT emit JavaScript files (Vite handles that)
      # - Exit code 1 if ANY type error exists
      # This catches: type mismatches, null safety violations, missing properties,
      # incorrect function signatures, and more.
      - name: Type check
        run: npx tsc --noEmit

      # Step 5: ESLint
      # Runs our configured ESLint rules against all source files.
      # This catches: unused variables, React hook violations, suspicious patterns,
      # and any custom rules we add later.
      # Note: We use "npm run lint" to ensure we run the same command locally and in CI.
      - name: Lint
        run: npm run lint

      # Step 6: Prettier format check
      # prettier --check does NOT modify files — it only reports if they're formatted.
      # Exit code 1 if ANY file doesn't match Prettier's output.
      # This ensures: consistent formatting across all contributors, no formatting
      # debates in code review, and clean git diffs.
      - name: Check formatting
        run: npm run format:check
```

### Understanding the workflow structure

```
main.yml
├── name: "CI"                         # Displayed in the Actions tab
├── on:                                # Trigger configuration
│   ├── push: branches: [main]        # Run on pushes to main
│   └── pull_request: branches: [main] # Run on PRs targeting main
└── jobs:
    └── quality-gate:                  # Job identifier (used in status checks)
        ├── runs-on: ubuntu-latest     # Runner environment
        └── steps:                     # Sequential commands
            ├── Checkout               # Get the code
            ├── Setup Node.js          # Install runtime with caching
            ├── Install dependencies   # npm ci for deterministic installs
            ├── Type check             # tsc --noEmit
            ├── Lint                   # eslint
            └── Check formatting       # prettier --check
```

### Step 3: Validate the workflow

You can validate the YAML structure using an online YAML validator or your editor's YAML extension. Key things to verify:

1. **All indentation uses 2 spaces** (no tabs).
2. **`uses:` values have pinned versions** (`@v4`, not `@latest` or `@main`).
3. **`run:` commands are single-line strings** (no accidental multi-line issues).
4. **The `on:` trigger is correct** — both `push` and `pull_request` are listed.

### Step 4: Commit and push

```bash
# Stage the workflow file
git add .github/workflows/main.yml

# Commit
git commit -m "ci: add quality gate workflow (tsc, eslint, prettier)"

# Push to main (or your feature branch for a PR)
git push origin main
```

After pushing, navigate to:
```
https://github.com/<your-username>/webcam-games/actions
```

You should see the "CI" workflow running. Click into it to see each step's output.

**Expected result:** All steps should show green checkmarks ✅.

### Workflow visualization

```
Developer pushes to main
        │
        ▼
GitHub detects push event
        │
        ▼
Workflow "CI" triggers
        │
        ▼
Job "quality-gate" starts on ubuntu-latest
        │
        ├── ✅ Checkout repository
        ├── ✅ Setup Node.js 20 (with npm cache)
        ├── ✅ Install dependencies (npm ci)
        ├── ✅ Type check (tsc --noEmit)
        ├── ✅ Lint (eslint src/)
        └── ✅ Check formatting (prettier --check)
        │
        ▼
All checks pass → Green status on commit/PR
```

### File structure after this lesson

```
webcam-games/
├── .github/
│   └── workflows/
│       └── main.yml            # CI workflow
├── .prettierrc
├── .prettierignore
├── eslint.config.js
├── package.json
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── vite.config.ts
├── src/
│   ├── App.css
│   ├── App.tsx
│   ├── index.css
│   ├── main.tsx
│   └── vite-env.d.ts
└── node_modules/
```

---

## Executive Summary

**Jack Donaghy:**

> I'm going to tell you something that every Fortune 500 CEO knows but most developers don't: the cost of finding a defect increases *exponentially* the later it's found in the delivery pipeline. A type error caught at the compiler level costs seconds. That same error caught in production costs hours of debugging, a hotfix deployment, customer trust, and — God forbid — a Hacker News post.

> What we've built here is a **fully automated quality gate** — the "Control" phase of our DMAIC process. Let me quantify the value:

> 1. **Type checking in CI prevents type regressions.** When someone refactors a TypeScript interface and doesn't update all consumers, `tsc --noEmit` catches it *before* the merge. Without this, you merge a type bomb that detonates in someone else's code three PRs later. That's a supply chain defect — the most expensive kind.

> 2. **ESLint in CI catches React lifecycle violations.** The `exhaustive-deps` rule is going to be absolutely critical when we build the game loop in Phase 1. A missing dependency in `useEffect` creates a stale closure — the hook captures an old value and never updates. In a game running at 60fps, that means your motion detector processes the same frame forever. CI catches this before it ships.

> 3. **Prettier check in CI enforces zero-friction code review.** When every file is consistently formatted, reviewers can focus on *logic* instead of *aesthetics*. I've calculated that formatting debates consume approximately 15% of senior developer time in organizations without automated formatting. That's 15% of a $200K salary — $30,000 per year per developer — wasted on semicolons. We've eliminated that cost entirely.

> 4. **`npm ci` ensures environmental determinism.** "Works on my machine" is not a shipping strategy — it's an excuse. By using `npm ci` with a locked dependency tree, our CI environment is byte-for-byte identical to what we develop against. Zero variance. Zero surprises.

> This workflow runs in under 60 seconds and costs fractions of a cent per execution on GitHub's free tier. The ROI is infinite. This is not optional. This is operational excellence. This is what separates a well-managed enterprise from a college hackathon project.
