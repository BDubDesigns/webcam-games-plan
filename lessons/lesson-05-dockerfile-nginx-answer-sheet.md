# Lesson 05 — Answer Sheet: Multi-Stage Dockerfile with Nginx

## The Implementation

### Step 1: Nginx configuration

**`nginx.conf`** (create at project root):

```nginx
# =============================================================================
# Nginx Configuration — Static SPA Serving
# =============================================================================
# This configuration serves our Vite-built React application as static files.
# Key features:
#   1. SPA fallback routing (try_files → index.html)
#   2. Aggressive caching for hashed assets (1 year)
#   3. No caching for index.html (always serves latest version)
#   4. Gzip compression for text-based assets
# =============================================================================

server {
    # Listen on port 80 inside the container.
    # Coolify's reverse proxy handles HTTPS termination externally.
    listen 80;
    server_name _;

    # Root directory where Vite's build output is served from.
    # This is where we COPY --from=builder the dist/ contents.
    root /usr/share/nginx/html;
    index index.html;

    # --- Gzip Compression ---
    # Compresses responses before sending to the client.
    # Reduces bandwidth by 60-80% for text-based assets.
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml
        application/rss+xml
        image/svg+xml;

    # --- Hashed Assets (JS, CSS, images from Vite build) ---
    # Vite appends content hashes to filenames: main-abc123.js
    # If the content changes, the hash changes, so the filename changes.
    # This means we can cache these files FOREVER — a stale cache is impossible
    # because the filename itself is the cache key.
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable";
        access_log off;
    }

    # --- SPA Fallback Routing ---
    # For any URL that doesn't match a real file:
    #   1. Try the exact URI ($uri)
    #   2. Try the URI as a directory ($uri/)
    #   3. Fall back to /index.html
    # This allows React Router to handle client-side routes like /game/window-cleaner
    # without Nginx returning a 404.
    location / {
        try_files $uri $uri/ /index.html;

        # index.html must NEVER be cached.
        # It's the entry point that references hashed assets.
        # If a user has a stale index.html, it might reference JS/CSS files
        # that no longer exist on the server (because a new build changed the hashes).
        # no-cache means: "you can store it, but revalidate with the server every time."
        add_header Cache-Control "no-cache";
    }
}
```

### Step 2: The Dockerfile

**`Dockerfile`** (create at project root):

```dockerfile
# =============================================================================
# Multi-Stage Dockerfile — Webcam Games
# =============================================================================
# Stage 1 (builder): Install dependencies + build the Vite production bundle
# Stage 2 (production): Serve the static files with Nginx
#
# Why multi-stage?
#   - Builder stage: ~500 MB (Node.js, npm, node_modules, TypeScript, Vite)
#   - Production stage: ~25 MB (Nginx Alpine + static HTML/CSS/JS)
#   - Only the production stage becomes the final image.
#   - The builder stage is discarded after the build completes.
# =============================================================================

# ---------------------------------------------------------------------------
# Stage 1: Builder
# ---------------------------------------------------------------------------
# We use Node.js on Alpine Linux for a small base image.
# This stage installs dependencies, builds the app, and is then discarded.
FROM node:20-alpine AS builder

# Set the working directory inside the container.
# All subsequent commands run relative to /app.
WORKDIR /app

# Copy ONLY package.json and package-lock.json first.
# This is a Docker layer caching optimization:
#   - If these files haven't changed, Docker reuses the cached npm ci layer.
#   - Source code changes (which happen frequently) don't invalidate the
#     dependency installation cache (which is slow and changes rarely).
COPY package.json package-lock.json ./

# Install dependencies using npm ci (clean install).
# npm ci:
#   - Uses package-lock.json as the source of truth
#   - Faster than npm install (skips dependency resolution)
#   - Fails if package-lock.json is out of sync with package.json
#   - Produces a deterministic node_modules (same input → same output)
RUN npm ci

# NOW copy the rest of the source code.
# This layer is invalidated on every code change, but the npm ci layer above
# is still cached (because package.json and package-lock.json didn't change).
COPY . .

# Build the production bundle.
# "npm run build" runs "tsc -b && vite build" which:
#   1. Type-checks the entire codebase (tsc -b)
#   2. Bundles, tree-shakes, minifies, and outputs to dist/ (vite build)
# The output in dist/ is a fully static site: HTML, CSS, JS, and assets.
RUN npm run build

# ---------------------------------------------------------------------------
# Stage 2: Production
# ---------------------------------------------------------------------------
# We use the official Nginx Alpine image for the smallest possible footprint.
# This stage contains ONLY Nginx + our built static files.
FROM nginx:stable-alpine AS production

# Copy our custom Nginx configuration.
# This replaces the default Nginx config with our SPA-aware configuration
# that includes: try_files fallback, gzip compression, and cache headers.
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy the built static files from the builder stage.
# COPY --from=builder means "copy from the 'builder' stage above."
# We copy dist/ contents to Nginx's default serving directory.
COPY --from=builder /app/dist /usr/share/nginx/html

# Document that this container listens on port 80.
# This is informational — it doesn't actually open the port.
# The port mapping happens at runtime (docker run -p 8080:80).
EXPOSE 80

# The default command for the nginx image is:
#   nginx -g "daemon off;"
# This starts Nginx in the foreground (required for Docker containers).
# We don't need to specify CMD because the base image already has it.
```

### Step 3: Docker ignore file

**`.dockerignore`** (create at project root):

```
# Dependencies — installed fresh inside the container via npm ci.
# Platform-specific native modules (macOS vs. Linux) make this essential.
node_modules

# Build output — generated inside the container during the build stage.
dist

# Version control — not needed for building the image.
.git
.github

# Documentation — not part of the application.
*.md

# Environment files — contain secrets, should never be baked into images.
.env
.env.*

# Editor and OS files
.vscode
.idea
.DS_Store
Thumbs.db
```

**Why each entry matters:**

| Entry | Why excluded | Size saved |
|-------|-------------|------------|
| `node_modules` | Platform-dependent, reinstalled via `npm ci` | ~200-500 MB |
| `dist` | Regenerated during build | ~5 MB |
| `.git` | Version history not needed in container | ~10-100 MB |
| `.github` | CI workflows not needed in container | Minimal |
| `.env*` | Security — never bake secrets into images | Critical |

### Step 4: Build and test locally

```bash
# Build the Docker image
# -t webcam-games: tags the image with a name
# . : uses the current directory as the build context
docker build -t webcam-games .

# Run the container
# -p 8080:80: maps host port 8080 to container port 80
# --name wg: gives the container a name for easy management
docker run -p 8080:80 --name wg webcam-games

# In another terminal, test it:
curl http://localhost:8080
# Should return the index.html content

# Test SPA routing — a deep route should still return index.html:
curl http://localhost:8080/game/window-cleaner
# Should return the SAME index.html (React Router handles the route)

# Check the image size:
docker images webcam-games
# Expected: ~25-35 MB

# Clean up:
docker stop wg && docker rm wg
```

**Expected image size comparison:**

| Approach | Image size |
|----------|-----------|
| Single-stage with Node.js | ~500-800 MB |
| Multi-stage with Nginx Alpine | ~25-35 MB |
| Savings | ~95% reduction |

### File structure after this lesson

```
webcam-games/
├── .dockerignore               # Excludes node_modules, dist, .git, etc.
├── .github/
│   └── workflows/
│       └── main.yml
├── .prettierrc
├── .prettierignore
├── Dockerfile                  # Multi-stage: Node builder → Nginx production
├── nginx.conf                  # SPA routing, caching, gzip
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

> Let me frame what we've just accomplished in terms that the board of directors would understand.

> We have created a **standardized, reproducible deployment artifact** that reduces our production infrastructure footprint by 95%. In Six Sigma terms, this is a *process capability improvement* of unprecedented magnitude. Let me break down the value drivers:

> 1. **Multi-stage builds are supply chain optimization.** The builder stage is our factory — it has all the heavy machinery (Node.js, npm, TypeScript compiler, Vite bundler) needed to manufacture the product. But we don't ship the factory to the customer. We ship only the finished goods. The production stage is our distribution container: 25 MB of Nginx and static files. That's lean manufacturing applied to Docker images.

> 2. **Docker layer caching is inventory management.** By separating the `package.json` copy from the source code copy, we've created a caching boundary that saves 30-60 seconds on every build where dependencies haven't changed. Over hundreds of builds per month, that's hours of CI time saved. Time is money, and money is what separates us from PBS.

> 3. **Nginx is the correct production server.** Using a Node.js server to serve static files is like hiring a Harvard MBA to hand out flyers. Nginx was purpose-built for this task. It handles 10,000+ concurrent connections on 2 MB of RAM. Our game could go viral on Reddit and Nginx wouldn't even break a sweat. That's engineering headroom, and headroom is what allows you to capitalize on unexpected market opportunities.

> 4. **The `.dockerignore` file is our security perimeter.** By explicitly excluding `.env` files, `node_modules`, and `.git` history, we ensure that no secrets, no development dependencies, and no version control metadata ever make it into the production image. This isn't just best practice — it's *fiduciary responsibility*.

> 5. **Coolify integration is turnkey.** Our Dockerfile follows the standard Docker contract: build an image, expose a port, done. Coolify (or any Docker-compatible PaaS) can build and deploy this with zero additional configuration. That's what I call *operational leverage* — we've done the work once and it pays dividends on every deployment from now until the heat death of the universe.

> Phase 0 is complete. We have a fully scaffolded, strictly typed, linted, formatted, CI-gated, containerized application ready for production deployment. We haven't written a single line of game logic, and we're already operating at a higher standard than 90% of the startups in this city. That's not an accident. That's strategy.
