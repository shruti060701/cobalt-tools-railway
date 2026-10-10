# Template Composer setup checklist

Two services, both from official prebuilt images — no Dockerfile, no custom start command on either.

## Service: `api`
- **Image**: `ghcr.io/imputnet/cobalt:latest`
- **Icon**: `https://res.cloudinary.com/dh2nt6hgh/image/upload/v1759035050/cobalt_logo_zlnnac.png` (matches reference)
- **Networking**: generate a domain, target port `9000`
- **Healthcheck**: path `/` — confirmed in `api/src/core/api.js` (`app.get('/', ...)` returns `200` server-info JSON, registered before any auth middleware, no API key required)
- **Restart policy**: `ON_FAILURE`, max retries `10` (matches reference)

### `api` variables
| Name | Default Value | Description (paste into Composer) | Optional |
|---|---|---|---|
| `API_LISTEN_ADDRESS` | `0.0.0.0` | Address the API server binds to. | No |
| `API_PORT` | `9000` | Port the API listens on. | No |
| `PORT` | `9000` | Must match `API_PORT` and the domain's target port. | No |
| `API_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}/` | The API's own public URL — used to build download tunnel links. | No |
| `CORS_WILDCARD` | `1` | Allows the `web` service (a different domain) to call this API from the browser. | No |
| `DURATION_LIMIT` | `10800` | Max video duration in seconds cobalt will process. | No |
| `FORCE_LOCAL_PROCESSING` | `never` | Processing mode. | No |
| `PROCESSING_PRIORITY` | `10` | Process niceness for media processing. | No |
| `RATELIMIT_MAX` | `20` | Max requests per rate-limit window, per IP. | No |
| `RATELIMIT_WINDOW` | `60` | Rate-limit window in seconds. | No |
| `SESSION_RATELIMIT_MAX` | `10` | Max session creations per window, per IP. | No |
| `SESSION_RATELIMIT_WINDOW` | `60` | Session rate-limit window in seconds. | No |
| `TUNNEL_LIFESPAN` | `90` | How long a download tunnel stays valid, in seconds. | No |
| `TUNNEL_RATELIMIT_MAX` | `40` | Max tunnel requests per window, per IP. | No |
| `TUNNEL_RATELIMIT_WINDOW` | `60` | Tunnel rate-limit window in seconds. | No |

## Service: `web`
- **Image**: `ghcr.io/spotdemo4/cobalt-web:latest`
- **Icon**: same as `api`
- **Networking**: generate a domain, target port `8787`
- **Healthcheck**: path `/` — serves the built app shell directly, `200`, no auth
- **Healthcheck timeout**: `180` seconds — **do not leave this at a short default**. See note below.
- **Restart policy**: `ON_FAILURE`, max retries `10`

### `web` variables
| Name | Default Value | Description (paste into Composer) | Optional |
|---|---|---|---|
| `PORT` | `8787` | Must match the domain's target port. | No |
| `WEB_DEFAULT_API` | `https://${{api.RAILWAY_PUBLIC_DOMAIN}}/` | The `api` service's public URL. Baked into the frontend at boot — requires both services to be in the same project with matching service names for this reference to resolve. | No |
| `WEB_HOST` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | This service's own public URL, used for canonical links. | No |
| `LOG_LEVEL` | `info` | Log verbosity for the static file server. | No |

### Important: the web service builds itself at every boot
`ghcr.io/spotdemo4/cobalt-web`'s entrypoint runs a full `vite build` on container start — the image ships without a pre-built frontend. Confirmed live: this took ~11–13 seconds per boot in testing. Deploy logs will show Vite build chunk output (`transforming...`, file size listings) before the server starts — this is expected, not a stuck or failed deploy. The `180s` healthcheck timeout exists specifically to give this room under slower/busier conditions than the test environment.

## Verified live (2026-10-10)
- `api`: `GET /` → `200`, real server-info JSON (version, enabled services, git commit)
- `web`: `GET /` → `200`, real cobalt UI (`<title>cobalt</title>`)
- Confirmed the `web` service's built JS bundle actually contains the `api` service's live domain (fetched and grepped the served chunks directly) — proving `WEB_DEFAULT_API` correctly flows through the on-boot build, not just that each service independently passes its healthcheck
- `web` boot timing, from first Vite log line to `static-web-server` listening: ~11–13s
