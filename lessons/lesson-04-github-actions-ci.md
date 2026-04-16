# Lesson 04: GitHub Actions CI Workflow

## The Pitch

**Tracy Jordan:**
> I just pushed some code and then a GHOST checked it! There's a ROBOT GHOST living in GitHub and it runs your tests! It's called "GitHub Actions" but it should be called "GitHub GHOSTS" because they do stuff when you're not looking! I like continuous integration so much I want to take it behind the middle school and get it pregnant! FROM NOW ON, no bad code gets through — the ghost won't let it!

**Jack Donaghy:**
> What Tracy is describing, with his trademark eloquence, is a Continuous Integration pipeline — the single most important operational process in modern software development. Think of it this way: without CI, you're running a factory where every product goes straight from the assembly line to the customer with zero inspection. That's not a business — that's a liability. GitHub Actions provides us with an automated quality gate that runs on every push. Type checking. Linting. Format verification. If any of these fail, the merge is blocked. This is Six Sigma's "Control" phase incarnate — we're not hoping for quality, we're *enforcing* it through process.

**Liz Lemon:**
> Oh thank God. Finally, something that catches problems before they become MY problems. You know what I've been doing for the last three lessons? MANUALLY running `tsc --noEmit` and `npm run lint` and `npm run format:check`. Like an ANIMAL. If we automate this, I can go back to stress-eating in peace. But — and this is important — the workflow file has to be EXACTLY right. One wrong indentation in a YAML file and the whole thing breaks. YAML. The format where whitespace is structural. Because apparently someone looked at Python and said "what if we made this MORE fragile?"

---

## The Theory

### What is GitHub Actions?

**Question:** What problem does CI (Continuous Integration) solve?

**Answer:** Without CI, code quality depends entirely on developers remembering to run checks locally. This is unreliable because:

1. Developers forget.
2. Developers have different local environments.
3. Developers sometimes commit "just this once" without running checks.

CI runs automated checks on a *remote, consistent* environment every time code is pushed. If checks fail, the PR is flagged. No human discipline required.

### GitHub Actions Architecture

**Question:** How does a GitHub Actions workflow work?

**Answer:** A workflow is a YAML file in `.github/workflows/` that defines:

1. **Trigger (`on`):** What event starts the workflow — `push`, `pull_request`, etc.
2. **Jobs:** One or more jobs that run in parallel (or sequentially if dependencies are declared).
3. **Runner (`runs-on`):** The virtual machine environment — `ubuntu-latest`, `windows-latest`, etc.
4. **Steps:** Sequential commands within a job — checkout code, install dependencies, run commands.

```
Event (push) → Workflow triggers → Job starts on runner → Steps execute sequentially
```

### Key Concepts

**Question:** What is `actions/checkout`?

**Answer:** GitHub Actions runners start with an empty filesystem. `actions/checkout@v4` clones your repository into the runner so your code is available for subsequent steps.

**Question:** What is `actions/setup-node`?

**Answer:** It installs a specific Node.js version on the runner. This ensures consistency — your CI uses the same Node version regardless of when it runs or which runner machine it gets.

**Question:** Why `npm ci` instead of `npm install`?

**Answer:**

| Command | Behavior |
|---------|----------|
| `npm install` | Reads `package.json`, resolves dependencies, may update `package-lock.json` |
| `npm ci` | Reads `package-lock.json` ONLY, installs exact versions, fails if lock file is out of sync |

`npm ci` is designed for CI environments because:
- It's faster (skips dependency resolution).
- It's deterministic (exact versions from lock file).
- It catches lock file drift (fails if `package-lock.json` doesn't match `package.json`).

### What our CI should check

For our webcam game, the CI pipeline needs to verify three things:

1. **Type safety:** `tsc --noEmit` — catches type errors without emitting JavaScript.
2. **Code quality:** `eslint src/` — catches unused variables, hook violations, etc.
3. **Formatting:** `prettier --check` — catches unformatted code without modifying it.

If ANY of these fail, the CI run fails, and the push/PR is flagged.

---

## The Assignment

### Step 1: Create the workflow directory

- Create the directory `.github/workflows/` at the project root.

### Step 2: Write the CI workflow

- Create a file `.github/workflows/main.yml`.
- The workflow should be named `"CI"`.
- It should trigger on:
  - `push` to the `main` branch.
  - All `pull_request` events targeting the `main` branch.
- It should define a single job called `quality-gate` that:
  - Runs on `ubuntu-latest`.
  - Uses Node.js version 20 (via `actions/setup-node@v4` with caching enabled for `npm`).
  - Performs these steps in order:
    1. Checkout the repository.
    2. Set up Node.js with npm caching.
    3. Install dependencies using `npm ci`.
    4. Run TypeScript type checking: `npx tsc --noEmit`.
    5. Run ESLint: `npm run lint`.
    6. Run Prettier format check: `npm run format:check`.

### Step 3: Validate the YAML locally

- You can validate the YAML structure by reading through it carefully. Ensure:
  - Indentation is consistent (2 spaces, no tabs).
  - All `uses:` actions reference specific versions (e.g., `@v4`, not `@latest`).
  - Each `run:` command is a single line or uses the `|` multi-line syntax.

### Step 4: Commit and test

- Commit the workflow file to your repository.
- Push to `main` (or open a PR targeting `main`).
- Navigate to the "Actions" tab in your GitHub repository.
- Verify that the workflow runs and all steps pass (green checkmarks).

### Deliverables

1. `.github/workflows/main.yml` exists with a valid CI workflow.
2. The workflow runs type-checking, linting, and format-checking.
3. All three checks pass in the GitHub Actions UI.

---

## Liz's Nightmares

> **Liz Lemon:** YAML. We're writing YAML. The configuration format where a misplaced space turns your CI pipeline into a dumpster fire. I love my job. I love my job. I love my job.

1. **YAML indentation errors:** YAML uses spaces for structure (not tabs, NEVER tabs). If your `steps:` key is indented with 3 spaces instead of 2, the parser won't necessarily error — it might interpret the structure differently and run the wrong commands. Use an editor with YAML support. Always. I once spent 45 minutes debugging a workflow because of a single extra space. I'm not over it.

2. **Using `npm install` instead of `npm ci`:** In CI, `npm install` might resolve different dependency versions than your local environment if `package-lock.json` is stale. `npm ci` uses the lock file as the source of truth and fails fast if there's a mismatch. It's also faster because it deletes `node_modules` and does a clean install every time.

3. **Not pinning action versions:** Using `actions/checkout@main` instead of `actions/checkout@v4` means your workflow could break at any time when the action updates. Always pin to a major version (`@v4`) at minimum. For maximum security, pin to the full commit SHA, but `@v4` is acceptable for most projects.

4. **Running `tsc` without `--noEmit`:** Without this flag, `tsc` will try to emit JavaScript files. In a Vite project, Vite handles the build — `tsc` is only for type checking. If you forget `--noEmit`, you might get errors about output directory configuration that have nothing to do with your types.

5. **Forgetting to add the workflow file to git:** The `.github/` directory needs to be committed and pushed. If it's in your `.gitignore` (unlikely but possible), the workflow won't exist on GitHub. And if it doesn't exist on GitHub, it doesn't run. And if it doesn't run, we're back to "manually checking everything" which is how I developed my stress-eating habit in the first place.

6. **Not triggering on `pull_request`:** If you only trigger on `push` to `main`, PRs won't show CI status. Reviewers won't know if the code passes checks until after it's merged. That's like checking if the milk is expired AFTER you've poured it on your cereal.
