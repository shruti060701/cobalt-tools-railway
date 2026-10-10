# Cobalt Tools on Railway

Deploy a complete, self-hosted [cobalt](https://github.com/imputnet/cobalt) — API + web interface — for downloading media from YouTube, TikTok, Twitter/X, Instagram, and 20+ other platforms, with no watermarks, no tracking, and no ads. Two services, both running the official images (`ghcr.io/imputnet/cobalt`, AGPL-3.0, and `ghcr.io/spotdemo4/cobalt-web`), unmodified.

The reference `cobalt-tools-complete-setup` template is already solid — it has a 100 health score and 52 active deployments, and nothing here required forking or patching either image. What this template adds is what was actually missing: a real healthcheck on both services, and documentation of a startup behavior that's easy to mistake for a stuck deploy if you don't know it's expected.

## What's different from the official Railway template

| | Reference template | This template |
|---|---|---|
| **Healthcheck** | None configured on either service | `/` on both — confirmed, via each service's own source, to return `200` unauthenticated: the API returns its server-info JSON, the web service serves the actual app shell |
| **Web service boot time** | Undocumented | Documented (see below) + a `healthcheckTimeout` of `180s` set explicitly, so a slower cold start under load doesn't get mistaken for a failed deploy |

### The web service builds itself on every single boot — not just the first

This isn't a bug so much as a surprising fact about the `cobalt-web` image worth knowing before you see it in your own logs: `ghcr.io/spotdemo4/cobalt-web` does **not** ship a pre-built frontend. Its entrypoint (`start.sh`) runs a full Vite/SvelteKit production build (`vite build`, `svelte-kit sync`, prerendering) at container **start**, every time, because the image was never actually finished at build time — only `pnpm install` and `svelte-kit sync` run during the Docker build; the real `vite build` happens live, on boot.

In practice this added about 11–13 seconds to every cold start in testing — not dramatic, but real, and it means the first few seconds of every deploy's logs are full of Vite build output (`transforming...`, chunk size listings) rather than a server startup message. That's expected, not a crash. This template sets a `180s` healthcheck timeout specifically so a slower build under real load (shared CPU, a busier region) doesn't get flagged as unhealthy before it's actually had a chance to finish.

## Services

- **`api`** — the cobalt backend. Handles all actual media extraction/processing.
- **`web`** — the frontend UI. Talks to `api` over Railway's private network at build time (`WEB_DEFAULT_API` is baked into the built frontend during that same runtime build, so it must be set correctly before the first boot).

## After deploying

Open the `web` service's public domain — that's the UI. Paste a link, download. No login, no account.

## Variables

**`api`**

| Variable | Default | Purpose |
|---|---|---|
| `API_URL` | public domain | Must be set to the API's own public URL |
| `CORS_WILDCARD` | `1` | Allows the separately-domained `web` service to call this API from the browser. Confirmed safe in cobalt's own source — the API has no cookie-based session to protect, so a wildcard origin isn't a CSRF exposure here. |
| `DURATION_LIMIT` | `10800` | Max video duration (seconds) cobalt will process |
| `FORCE_LOCAL_PROCESSING` | `never` | Processing mode |
| `RATELIMIT_MAX` / `RATELIMIT_WINDOW` | `20` / `60` | Per-IP rate limiting |
| `SESSION_RATELIMIT_MAX` / `_WINDOW` | `10` / `60` | Session-creation rate limiting |
| `TUNNEL_LIFESPAN`, `TUNNEL_RATELIMIT_MAX`, `TUNNEL_RATELIMIT_WINDOW` | `90` / `40` / `60` | Download tunnel limits |
| `PROCESSING_PRIORITY` | `10` | Process niceness |

**`web`**

| Variable | Default | Purpose |
|---|---|---|
| `WEB_DEFAULT_API` | `api` service's public domain | Where the frontend sends requests. Baked in at container boot — changing it requires a redeploy. |
| `WEB_HOST` | `web` service's own public domain | Used for canonical URLs/meta tags |
| `LOG_LEVEL` | `info` | `static-web-server` log verbosity |

## License

`imputnet/cobalt` is [AGPL-3.0](https://github.com/imputnet/cobalt/blob/main/LICENSE). This template deploys the official image, unmodified.
