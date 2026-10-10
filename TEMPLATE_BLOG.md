# Deploy and Host Cobalt Tools on Railway

[cobalt](https://github.com/imputnet/cobalt) downloads media from YouTube, TikTok, Twitter/X, Instagram, and over 20 other platforms — no watermarks, no ads, no tracking, no account. This template deploys a complete self-hosted instance, API and web UI both, from the official images, unmodified. Railway's own reference template is already solid — 100 health score, 52 active deployments — so this one doesn't fix a broken deploy. It adds the one thing the reference is missing (a real healthcheck) and documents a startup behavior that's easy to mistake for a hung deploy if you've never seen it before.

## About Hosting Cobalt Tools

Two services, two official images: `ghcr.io/imputnet/cobalt` (the API, AGPL-3.0) and `ghcr.io/spotdemo4/cobalt-web` (the web UI). Neither is forked or patched here — every difference from the reference template is in configuration, not code.

The reference template leaves `healthcheckPath` unset on both services. With no healthcheck, Railway has no way to tell a genuinely broken deploy from a working one before routing traffic to it — the deploy just completes once the container starts, whether or not the app inside it is actually serving requests. That's the real gap, and it's a small one precisely because this template is otherwise already solid: there's no dead variable to find, no wrong port, nothing structurally wrong. Just an absent safety check.

## Common Use Cases

- **Downloading your own content for backup**: save your own uploaded videos/posts from platforms before they're taken down or lost
- **Archiving public media you have rights to use**: research, journalism, or fair-use commentary that needs a local copy
- **A private alternative to public cobalt instances**: public instances rate-limit aggressively and can disappear; self-hosting means it's yours and always available
- **Family or team media tooling**: share one private instance instead of everyone relying on random public mirrors

## Dependencies for Cobalt Tools Hosting

Running this template needs nothing beyond the two containers — no database, no external service. The two services do depend on each other: `web` needs `api`'s public URL at boot to build correctly.

### Deployment Dependencies

- [cobalt's Source Code](https://github.com/imputnet/cobalt) — AGPL-3.0
- [cobalt's Official Documentation](https://github.com/imputnet/cobalt/blob/main/docs/api.md)
- [cobalt-web's Source](https://github.com/spotdemo4/cobalt-web-docker) (the Docker packaging; the actual frontend source lives in `imputnet/cobalt`'s own `web/` directory)

### Implementation Details

The API's healthcheck path was confirmed directly in `api/src/core/api.js`: `app.get('/', ...)` returns a `200` with server-info JSON (version, enabled services, git commit) — registered with no authentication, before any API-key middleware. That's a clean, safe target for Railway's prober.

The web service's `/` took more care. Reading `spotdemo4/cobalt-web-docker`'s `start.sh` revealed something worth knowing before relying on this image: it doesn't ship a pre-built frontend. The Dockerfile only runs `pnpm install` and `svelte-kit sync` during the image build — the actual `vite build` runs live, every single time the container starts, triggered by `start.sh`'s `if [ ! -d ./web/build ]` check (which is always true, since nothing ever persists a previous build). Deployed and watched this happen directly: `"building cobalt web..."` followed by real Vite build output — chunk transformations, file size listings — for roughly 11 to 13 seconds before `static-web-server` finally starts listening and the healthcheck can pass. It's not a bug, exactly — the app works, it's just doing more at boot than it looks like it should. But if you're watching deploy logs expecting an instant "server started" line and instead see a wall of Vite output, it's easy to assume something's stuck. It isn't. This template sets an explicit `180` second healthcheck timeout specifically so a slower build under real traffic or a busier region doesn't get mistaken for a failed deploy.

One more thing worth confirming rather than assuming: does the frontend actually end up pointed at the right API? `WEB_DEFAULT_API` is validated and required at build time by cobalt's own `vite.config.ts` (the build throws if it's missing), and it gets baked directly into the compiled client bundle during that same on-boot build — there's no server-side runtime lookup to verify separately. Deployed both services, then fetched and searched every JS chunk the live frontend actually serves, and found the `api` service's real public domain string baked into one of them. That's the proof the two services are actually wired together, not just independently healthy.

## Why Deploy Cobalt Tools on Railway?

A self-hosted cobalt instance means no rate limits you don't control, no dependency on a public instance's uptime, and your request history staying yours. Railway handles the two-service networking (private DNS between `api` and `web`, public domains for both) and TLS without any manual configuration.

This template's contribution is narrow but real: the healthcheck the reference lacks, and documentation for the one piece of this stack's behavior (the on-boot frontend build) that would otherwise cost you a few minutes of "is this broken?" the first time you watch it deploy.

## What Was Verified

Both services were deployed live and checked directly, not assumed from reading source alone. `GET /` on `api` returns `200` with real server-info JSON, confirming the healthcheck target is both correct and functional. `GET /` on `web` returns `200` with the actual cobalt UI (`<title>cobalt</title>`). The on-boot build was timed directly from deploy logs: roughly 11 to 13 seconds from the first Vite log line to `static-web-server` reporting it's listening. And critically, the cross-service wiring was checked by fetching the live frontend's compiled JavaScript and confirming the `api` service's actual public domain is present in it — not inferred from the variable being set, but found in the bytes actually served to a browser.

## Frequently Asked Questions

### Why does the web service's deploy log show a full frontend build?
Because `ghcr.io/spotdemo4/cobalt-web` builds its SvelteKit frontend at container start rather than at image build time — confirmed directly in its `start.sh` and in live deploy logs. It typically takes 11–13 seconds. This is expected behavior, not a stuck or broken deploy.

### Why is the healthcheck timeout set to 180 seconds instead of left default?
Because the web service's on-boot build adds real, variable-length startup time before it can respond to a healthcheck. 180 seconds gives comfortable headroom beyond the ~13 seconds observed in testing, so a slower build under real-world load doesn't get flagged as a failed deploy.

### Does this template modify either cobalt image?
No. Both `ghcr.io/imputnet/cobalt` and `ghcr.io/spotdemo4/cobalt-web` are deployed exactly as published. Every difference from the reference template is in Railway service configuration — healthcheck paths, timeout, and documentation — not in the containers themselves.

### Is `CORS_WILDCARD=1` safe?
Yes — confirmed in cobalt's own source (`api/src/core/api.js`): the API has no cookie-based session to protect, so an open CORS policy doesn't expose a CSRF risk the way it would on an app with authenticated sessions. It's necessary here specifically because `api` and `web` are deployed as separate services with separate domains.

### Where can I find cobalt?
Source is on GitHub at [github.com/imputnet/cobalt](https://github.com/imputnet/cobalt). Use this template to deploy both services with a working healthcheck and a startup-timing-aware configuration, in one click.

Source for this template's docs: https://github.com/shruti060701/cobalt-tools-railway
