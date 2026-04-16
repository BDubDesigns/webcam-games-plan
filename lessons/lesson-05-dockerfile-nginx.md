# Lesson 05: Multi-Stage Dockerfile with Nginx

## The Pitch

**Tracy Jordan:**
> YO! We're putting our GAME in a WHALE! Docker has a whale as its logo and I think that's beautiful! A whale is just a bus for the ocean! We're putting our whole website inside a container and sending it to a server in GERMANY! That's called HETZNER! I don't know what that word means but it sounds like a German sneeze! I like containerization so much I want to take it behind the middle school and get it pregnant!

**Jack Donaghy:**
> What Tracy is mangling is this: we are containerizing our application for deployment. In the corporate world, we call this "standardizing the delivery mechanism." A Docker container is to software what a shipping container is to global trade — it doesn't matter what's inside, the exterior is always the same dimensions, always fits on the same trucks, ships, and cranes. Our Hetzner VPS running Coolify will pull this container and run it. No environment configuration, no dependency mismatches, no "it works on my machine." This is *supply chain optimization* at the deployment layer, and it's how every serious organization ships software.

**Liz Lemon:**
> Okay, Docker. Deep breaths. I can do this. So here's the thing — most Dockerfiles I've seen are TERRIBLE. They install Node.js, copy the entire project, run `npm install`, build it, and then serve it with a Node server. The result is a 1.2 GB image that includes your entire build toolchain, test frameworks, and `node_modules` in production. That's like packing your entire apartment to go to the grocery store. We're going to use a MULTI-STAGE build, which means we build the app in one container and then copy ONLY the static files into a tiny Nginx container. The final image will be like 25 MB. I'm stressed about it but also proud.

---

## The Theory

### What is Docker and why do we need it?

**Question:** Our app is just static HTML, CSS, and JS after building. Why not just copy files to a server?

**Answer:** You *could*, but then you'd need to:
1. Install and configure Nginx on the server manually.
2. Keep the server's Nginx config in sync with your app's requirements.
3. Handle different server OS versions, architectures, and configurations.
4. Document all of this for anyone else who needs to deploy.

Docker solves this by packaging your app AND its server (Nginx) into a single, reproducible artifact. The Dockerfile IS the documentation.

### Multi-Stage Builds

**Question:** What is a multi-stage Docker build?

**Answer:** A multi-stage build uses multiple `FROM` statements in a single Dockerfile. Each `FROM` starts a new "stage" with a fresh filesystem. You can copy files from one stage to another using `COPY --from=<stage>`.

```
Stage 1 (Builder):  Node.js image → install deps → build app → produces /dist
        │
        │  COPY --from=builder /dist → only the built static files
        ▼
Stage 2 (Production): Nginx image → serve static files → TINY final image
```

**Why this matters:**
- The builder stage needs Node.js, npm, `node_modules`, TypeScript, Vite, etc. (~500 MB+)
- The production stage only needs Nginx + your built files (~25 MB)
- Only the LAST stage becomes the final image. All builder stages are discarded.

### Nginx for static files

**Question:** Why Nginx instead of a Node.js server?

**Answer:** For serving static files, Nginx is superior in every way:

| Metric | Nginx | Node.js (express.static) |
|--------|-------|--------------------------|
| Memory usage | ~2 MB | ~30-50 MB |
| Concurrent connections | 10,000+ | ~1,000 (single-threaded) |
| Purpose-built for static files | ✅ | ❌ (general-purpose runtime) |
| Image size | ~25 MB (Alpine) | ~150 MB+ |

For a static Vite build (HTML, CSS, JS, images), Nginx is the standard production choice.

### SPA routing with Nginx

**Question:** Our React app uses client-side routing. What happens when a user navigates to `/game/window-cleaner` and refreshes the page?

**Answer:** Without configuration, Nginx looks for a file at `/game/window-cleaner/index.html`, which doesn't exist. It returns a 404.

The fix: configure Nginx to serve `index.html` for ALL routes that don't match a real file. This is called a "fallback" or "try_files" configuration:

```nginx
try_files $uri $uri/ /index.html;
```

This tells Nginx: "Try the exact path, then try it as a directory, and if neither exists, serve `index.html`." React Router then handles the URL client-side.

### Coolify and Hetzner

**Question:** How does Coolify fit into this?

**Answer:** Coolify is a self-hosted PaaS (Platform as a Service) that runs on your Hetzner VPS. When you push to your repository:

1. Coolify detects the push (via webhook or polling).
2. It pulls your repository.
3. It builds the Docker image using your `Dockerfile`.
4. It runs the container with the appropriate port mapping.
5. It handles SSL, reverse proxy, and domain routing.

Your Dockerfile is the contract between your code and Coolify. Write it correctly, and deployment is automatic.

---

## The Assignment

### Step 1: Create the Nginx configuration

- Create a file called `nginx.conf` at the project root.
- It should:
  - Listen on port 80.
  - Serve files from `/usr/share/nginx/html`.
  - Use `try_files $uri $uri/ /index.html` for SPA fallback routing.
  - Set appropriate cache headers:
    - For hashed assets (`/assets/*`): cache for 1 year (`Cache-Control: public, max-age=31536000, immutable`). Vite adds content hashes to filenames, so these are safe to cache aggressively.
    - For `index.html`: no cache (`Cache-Control: no-cache`). This ensures users always get the latest HTML that references the latest hashed assets.
  - Enable gzip compression for text-based assets.

### Step 2: Create the Dockerfile

- Create a `Dockerfile` at the project root.
- It should use two stages:

**Stage 1: Builder**
  - Use `node:20-alpine` as the base image.
  - Set the working directory to `/app`.
  - Copy `package.json` and `package-lock.json` first (for Docker layer caching).
  - Run `npm ci` to install dependencies.
  - Copy the rest of the source files.
  - Run `npm run build` to produce the production build in `dist/`.

**Stage 2: Production**
  - Use `nginx:stable-alpine` as the base image.
  - Copy the custom `nginx.conf` to `/etc/nginx/conf.d/default.conf`.
  - Copy the built files from the builder stage's `/app/dist` to `/usr/share/nginx/html`.
  - Expose port 80.
  - The default Nginx command (`nginx -g 'daemon off;'`) will start the server.

### Step 3: Create a `.dockerignore` file

- Create `.dockerignore` at the project root.
- It should ignore: `node_modules`, `dist`, `.git`, `.github`, `*.md`, `.env*`.
- This keeps the Docker build context small and fast.

### Step 4: Build and test locally (if Docker is available)

- Build the image: `docker build -t webcam-games .`
- Run the container: `docker run -p 8080:80 webcam-games`
- Visit `http://localhost:8080` and verify the app loads.
- Verify that refreshing on a deep route still serves the app (SPA routing works).

### Deliverables

1. `nginx.conf` with SPA routing, caching, and gzip configuration.
2. `Dockerfile` with a two-stage build (Node builder + Nginx production).
3. `.dockerignore` to exclude unnecessary files from the build context.
4. (If Docker is available) The app serves correctly from the container at port 8080.

---

## Liz's Nightmares

> **Liz Lemon:** Docker. Dockerfiles. Docker Compose. Docker buildx. Docker Desktop licensing. Docker Hub rate limits. I need a sandwich.

1. **Copying `node_modules` into the Docker image:** NEVER put `node_modules` in the build context. It's massive, platform-dependent (native modules compiled for macOS won't work on Linux), and completely unnecessary — `npm ci` installs them fresh inside the container. This is why `.dockerignore` exists. Without it, your build context could be 500 MB+ and take forever to send to the Docker daemon.

2. **Not leveraging Docker layer caching:** Docker caches each `RUN`, `COPY`, and `ADD` instruction as a layer. If a layer's inputs haven't changed, Docker reuses the cached version. By copying `package.json` and `package-lock.json` BEFORE copying source code, the `npm ci` layer is cached as long as dependencies don't change. If you copy everything at once, every code change invalidates the dependency installation cache, and `npm ci` runs from scratch every time. This turns a 5-second build into a 60-second build.

3. **Using `node:20` instead of `node:20-alpine`:** The standard Node image is ~350 MB. The Alpine variant is ~50 MB. For a builder stage that's discarded, it doesn't matter for final image size, but it matters for build speed (downloading the base image). And for the production stage, Alpine Nginx is ~25 MB vs. ~140 MB for standard Nginx.

4. **Forgetting `try_files` in Nginx:** Without it, refreshing the page on ANY route other than `/` returns a 404. Your user navigates to `/game`, hits F5, and sees "404 Not Found." They think the site is broken. They leave. They never come back. They tell their friends. Your game dies alone. Add `try_files`.

5. **Caching `index.html`:** If you set long cache headers on `index.html`, users will get a stale version that references old JavaScript bundles (which may have been deleted from the server). The fix: cache hashed assets aggressively, but NEVER cache `index.html`. Vite's build system already handles this correctly — asset filenames include content hashes (`main-abc123.js`), so new builds produce new filenames. As long as `index.html` is always fresh, it points to the right assets.

6. **Running the Nginx container as root:** By default, Nginx runs as root inside the container. For our project (served behind Coolify's reverse proxy), this is acceptable. But in a more locked-down environment, you'd configure Nginx to run as a non-root user. Just know this is a thing you might need to address later.
