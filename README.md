# Firecrawl + SearXNG on Incus

Self-hosted [Firecrawl](https://github.com/firecrawl/firecrawl) (web scraping/crawling API)
with a self-hosted [SearXNG](https://github.com/searxng/searxng) metasearch backend, deployed
on an IncusOS host via [`incus-compose`](https://github.com/lxc/incus-compose), reachable only
over Tailscale.

## Architecture

| Service | Image | Role |
|---|---|---|
| `redis` | `docker.io/redis:alpine` | Rate limiting / caching |
| `rabbitmq` | `docker.io/rabbitmq:3-management` | AMQP broker for Firecrawl's NUQ queue system |
| `nuq-postgres` | `ghcr.io/firecrawl/nuq-postgres:latest` | Queue/job persistence (NUQ schema pre-baked into the image) |
| `nuq-init` | `ghcr.io/firecrawl/nuq-postgres:latest` | One-shot: replays the NUQ schema, then exits. `api` waits on it completing successfully |
| `playwright-service` | `ghcr.io/firecrawl/playwright-service:latest` | Headless browser rendering for JS-heavy pages |
| `searxng` | `docker.io/searxng/searxng:latest` | Metasearch — used both by Firecrawl's `/search` endpoint and directly by agents for lightweight overview searches |
| `api` | `ghcr.io/firecrawl/firecrawl:latest` | Main Firecrawl process — runs as a single "harness" container that also spawns internal workers |

Only `api` and `searxng` are reachable from outside the Incus project, and only over
Tailscale (never the public internet). Everything else — `redis`, `rabbitmq`,
`nuq-postgres`, `nuq-init`, `playwright-service` — is internal-only.

**Why both Firecrawl and SearXNG are exposed:** agents can hit SearXNG directly for cheap
search-and-skim (titles/snippets/URLs across many candidates), reserving Firecrawl's
`/scrape` for the few pages actually worth reading in full. Firecrawl's own `SEARXNG_ENDPOINT`
config still points at the same SearXNG instance for when a one-shot search-and-scrape via
Firecrawl's `/search` is more convenient.

## Prerequisites

- Incus 7.0+ on the target host, reachable over HTTPS (`core.https_address` set — required
  by `incus-compose` even for local use, since it caches images in a separate project and
  copies them into yours on `up`).
- [`incus-compose`](https://github.com/lxc/incus-compose) installed on the client machine
  you manage Incus from (not on the IncusOS host itself).
- An Incus HTTPS remote pointed at the IncusOS host, set as the **default** remote
  (`incus remote switch <name>` — check with `incus remote get-default`).
- OCI registry remotes added — **see the naming gotcha below, this is the part most likely
  to trip you up.**

### Remote naming: names must match the registry hostname, including `.io`

`incus-compose` resolves each image's leading path segment (e.g. `docker.io` in
`docker.io/redis:alpine`) directly against an Incus remote of that exact name. It is *not*
a hostname lookup — it's a literal string match against your configured remote names. This
means:

```
incus remote add --protocol oci docker.io https://docker.io
incus remote add --protocol oci ghcr.io https://ghcr.io
```

Shortened names like `docker` or `ghcr` (dropping the `.io`) will *not* resolve, even though
they point at the correct URL — you'll get errors like:

```
image source error getting image server for docker.io: The remote "docker.io" doesn't exist
```

If you already added remotes under shortened names, rename them rather than editing every
image reference in `compose.yaml`:

```
incus remote rename docker docker.io
incus remote rename ghcr ghcr.io
```

Verify with `incus remote list` before running `incus-compose up`.

## Repository structure

```
.
├── compose.yaml          # Main service definitions (Compose spec)
├── compose.incus.yaml    # Incus-specific overrides (Tailscale-scoped ports)
├── .env.example          # Template — copy to .env and fill in real secrets
├── .env                  # Real secrets (gitignored — never commit)
└── searxng/
    ├── settings.yml.example  # Template — copy to settings.yml and fill in real keys
    └── settings.yml          # SearXNG config; JSON format must stay enabled.
                              # Holds secret_key + Brave/Exa/CORE API keys
                              # (gitignored — never commit)
```

## Configuration

Copy `.env.example` to `.env` and fill in:

```
POSTGRES_USER=firecrawl
POSTGRES_PASSWORD=<strong-password>
POSTGRES_DB=firecrawl
BULL_AUTH_KEY=<strong-secret>
```

Generate strong values with:
```
openssl rand -hex 32
```
Hex-only output sidesteps `.env` parsing pitfalls (`$` triggers variable interpolation,
`#` starts a comment) and `BULL_AUTH_KEY` in particular gets pasted directly into a URL
path (`/admin/<key>/queues`), so avoiding `/` and spaces there matters too.

`compose.incus.yaml` is gitignored (it holds your real Tailscale IP); the committed
template `compose.incus.yaml.example` ships with a placeholder.
Replace it with your own IncusOS host's Tailscale IP for both the `api` (port 3002) and
`searxng` (port 8888→8080) mappings. Binding to the Tailscale address specifically — rather
than `0.0.0.0` — is what keeps these services off the public internet, so don't drop the
host-IP prefix from those port mappings.

Both mappings use **long-form ports with `x-incus-compose.nat: true`** (kernel NAT proxy
mode, incus-compose 1.1.0+). This is faster than the default userspace proxy (which
routes through the host's loopback and appears to the service as `127.0.0.1`).

**Gotcha — `api`'s port lives ONLY in the overlay, not in `compose.yaml`.** Compose
*merges* port lists across files; it does not replace. `compose.yaml` declaring
`ports: ["3002:3002"]` alongside the overlay's NAT entry produced two devices both named
`proxy-3002`, and the userspace `127.0.0.1` entry won the name collision — breaking
start with `Connect IP "127.0.0.1" must be one of the instance's static IPv4 addresses`.
This was harmless pre-NAT (both entries were userspace); the NAT flip exposed it. The
port is declared only in `compose.incus.yaml` now; don't re-add it to `compose.yaml`.

Other proxy-NAT facts worth knowing (Incus server is 7.3):
- NAT proxy needs Incus 7.2+ (or 7.0.1 LTS); below that the port is skipped with a
  warning. Requesting `nat` on an older server fails visibly, not silently.
- Instances here have no static NIC IP, so the connect address is `0.0.0.0` and Incus
  resolves the real instance IP via ARP/NDP detection at runtime (`device show` shows
  `connect: tcp:0.0.0.0:<port>`, which is expected — not a misconfig).
- NAT mode routes to the instance's real address, so host-local access to the host's
  own tailnet IP may behave differently than loopback-style userspace proxying; nothing
  in this stack depends on host-local access (use a tailnet peer or the container
  bridge IP).

Copy `searxng/settings.yml.example` to `searxng/settings.yml` and fill in the `secret_key`
(`openssl rand -hex 32`) plus your Brave/Exa/CORE API keys. Make sure `search.formats`
includes `json` — SearXNG defaults to HTML-only and returns 403 on JSON API requests
otherwise. Since SearXNG has a real second consumer here (agents, not just Firecrawl),
`limiter: true` is worth keeping on.

### Getting that config into the container: why it isn't `configs:`

Two independent traps stack here, and each one fails *silently*:

1. **Compose `configs:` is non-overwriting in incus-compose.** It resolves configs into an
   `InstanceFile` but never sets `Overwrite`, and the push then `Lstat`s the target and skips
   if the file already exists — logging only at debug level. So `configs:` only ever works for
   targets that don't already exist.
2. **`/etc/searxng` is masked by a tmpfs.** The image declares `VOLUME /etc/searxng`, which
   Incus honors by mounting a tmpfs there. Files are pushed over SFTP into the *stopped*
   rootfs, so at boot the tmpfs mounts over the path and hides them. The entrypoint then
   sees no `settings.yml` and generates a stock one — with a fresh random `secret_key` on
   every recreate. This defeats `configs:`, secrets, *and* a single-file seed aimed at
   `/etc/searxng`.

The fix is to stop fighting the tmpfs: point `SEARXNG_SETTINGS_PATH` at a path that isn't
under any declared `VOLUME` (`/etc/searxng-repo/settings.yml`) and seed the file there.
SearXNG's settings loader accepts a *file* for that variable and it takes precedence over
the `/etc/searxng` fallback.

A **single-file** seeded bind is deliberate. It's the only path in incus-compose that sets
`Overwrite: true`, and the push runs inside `start()` on every start. A seeded *directory*
bind instead creates an Incus storage volume that is populated only on first creation, which
means host edits silently stop taking effect until you delete the volume. `seed: true` is
mandatory either way: plain bind mounts are refused unless the client and the Incus host are
the same machine, and this stack is driven over an HTTPS remote.

**But `incus-compose up -d` alone does not re-apply an edited `settings.yml`.** The push
happens inside `start()`, and `up` skips `start()` for an instance that is already running
(`start instance searxng-1: error: resource is already running`) — so the container keeps
the old file, with no indication anything was skipped. Stop the service first:

```
incus-compose stop searxng
incus-compose up -d
```

Then confirm the new content actually landed before concluding your change didn't work:

```
incus-compose exec searxng -- cat /etc/searxng-repo/settings.yml
```

Corollary: `/etc/searxng-repo/settings.yml` is overwritten from the repo on every *start*,
so don't edit it in place with `incus-compose exec` — your changes are silently reverted the
next time the container starts. Edit `searxng/settings.yml` and do the stop/up cycle above.

Note that `/etc/searxng` still exists in the container as an entrypoint-generated stock
config on a tmpfs. It is unused and safe to ignore — but it means "there's a settings.yml in
`/etc/searxng`" is not evidence that your config is live. Check `/etc/searxng-repo` instead.

## Usage

```
# Start everything (detached — always use -d for anything long-running;
# foreground mode's behavior on a killed/interrupted client process is
# not something to rely on for lifecycle control)
incus-compose up -d

# Check status (add -a to include stopped instances — nuq-init is
# expected to be stopped once it has completed)
incus-compose ps -a

# Follow logs
incus-compose logs -f api

# Stop and remove containers (keeps volumes and cached images)
incus-compose down

# Also delete volumes — this drops the Postgres queue data in `pgdata`
incus-compose down --volumes

# Full teardown: remove the whole Incus project
incus-compose down --project
```

Prefer `incus-compose` over plain `incus` for day-to-day work — it scopes every operation to
the `firecrawl` project, so a mistake can't reach unrelated projects on the host. For
anything it doesn't wrap directly, `incus-compose incus ...` runs a raw incus command in
that same project context:

```
incus-compose incus delete -f searxng-1
```

## Verifying it works

```
curl http://<tailscale-ip>:3002/
# → {"message":"Firecrawl API",...}

curl -X POST http://<tailscale-ip>:3002/v1/crawl \
  -H 'Content-Type: application/json' \
  -d '{"url": "https://firecrawl.dev"}'

# 403 here means the JSON format never applied — your config isn't live
curl "http://<tailscale-ip>:8888/search?q=test&format=json"
```

To confirm SearXNG is actually running your config rather than an entrypoint-generated one,
check that the `secret_key` in the container matches the one in `searxng/settings.yml`:

```
incus-compose exec searxng -- grep secret_key /etc/searxng-repo/settings.yml
```

A `secret_key` you don't recognise means the file in the container is stale — the push was
skipped rather than overwritten. If the seed doesn't land at all, `SEARXNG_SETTINGS_PATH`
points at a nonexistent file and SearXNG raises an `EnvironmentError` and refuses to boot,
rather than silently serving defaults — check `incus-compose logs searxng`. Either way the
failure is visible; the old `configs:` approach was the one that failed silently.

## Incus-specific gotchas

These are the things that will bite you if you edit `compose.yaml`. Each one fails in a way
that doesn't obviously point at its cause.

- **Don't add a healthcheck to `rabbitmq`.** The `ic-healthd` sidecar latches the first
  failed probe into `unhealthy` and never re-probes, and RabbitMQ needs ~12s+ to boot. Any
  probe here therefore blocks `api` permanently, since `api` depends on it. Firecrawl
  reconnects to AMQP on its own, so the healthcheck buys nothing. Healthchecks on
  faster-booting services (`nuq-postgres`) are fine — note its generous `start_period`.
- **`PSQL_PAGER: cat` on the Postgres services is load-bearing.** incus-compose gives the
  entrypoint a TTY, so `psql` running the initdb scripts pipes its output into `less` and
  blocks forever. Without it, initdb never finishes and the NUQ schema is never created —
  with no error, just a hang.
- **`nuq-init` exists to recover from a half-finished initdb.** An interrupted init leaves
  Postgres up but the NUQ schema missing, which surfaces much later as confusing `api`
  errors rather than as a startup failure. `nuq-init` idempotently replays the image's own
  `010-nuq.sql` on every `up`, so that state can't persist. It reuses the `nuq-postgres`
  image because that already ships both `psql` and the SQL file, and `api` gates on it via
  `service_completed_successfully`.
- **SearXNG config can't be delivered by `configs:`** — see
  [the section above](#getting-that-config-into-the-container-why-it-isnt-configs).
- **Slow engines need an explicit per-engine `timeout`.** SearXNG's global
  `outgoing.request_timeout` default is 3.0s, and an engine that doesn't ship its own
  override inherits it. `core.ac.uk` is the case in point: `api.core.ac.uk` routinely takes
  8–15s to answer, so *every* CORE query timed out and the engine looked broken —
  `unresponsive_engines: [["core.ac.uk", "timeout"]]` with an otherwise healthy 200. Hence
  the `timeout: 20.0` on that engine in `searxng/settings.yml`.

  Two things to know before copying this pattern to other engines. First, SearXNG uses the
  **maximum** engine timeout in a search as the deadline for that whole search, so a slow
  engine drags out every search it participates in — acceptable here only because CORE is
  `science`-category and never runs on general queries. Second, `outgoing.max_request_timeout`
  is commented out (i.e. `None`) in the stock settings, which is what lets a per-engine value
  exceed the default at all; setting it would silently cap every engine back down.

  Diagnosing this class of problem: `curl <host>:8888/config` reports the effective `timeout`
  per engine, which is the fastest way to tell "engine is misconfigured" from "engine is
  merely slow".
- **CORE's own API is flaky, independent of the above.** `api.core.ac.uk` intermittently
  returns HTTP 500 wrapping an Azure `503 ... exceeded the limits of its provisioned
  capacity` — reproducible with plain `curl`, so it is upstream capacity throttling, not
  anything in this stack. Expect an occasional `[["core.ac.uk", "HTTP error"]]` in
  `unresponsive_engines`; retrying the query works. Don't chase it as a config bug.

## Design choices

- **`nuq-postgres` uses Firecrawl's own prebuilt image** (`ghcr.io/firecrawl/nuq-postgres`)
  rather than stock Postgres — it bakes in the NUQ schema init script. Currently amd64-only
  upstream; matters only if the host is arm64.
- **`api` runs as a single harness process** that internally manages the API server plus all
  workers, rather than as separate worker containers. No `command` override is set — the
  image's own default is relied on here, so this is worth re-checking if upstream changes
  its entrypoint.
- **Resource limits and log rotation** (`cpus`, `mem_limit`, `logging.options.max-size`)
  are set deliberately, not decorative — Playwright's Chromium instances are memory-hungry,
  and unbounded logs on a long-running self-hosted stack will eventually fill the disk.

## Keeping this in sync with upstream

Firecrawl's real `docker-compose.yaml` is the source of truth — this repo is a hand-adapted
subset of it (prebuilt images instead of local builds, SearXNG added, Incus-specific tuning
split into `compose.incus.yaml`). When Firecrawl changes required env vars, service
dependencies, or default commands, those changes won't propagate here automatically. Worth
periodically diffing against
[`firecrawl/firecrawl`'s `docker-compose.yaml`](https://github.com/firecrawl/firecrawl/blob/main/docker-compose.yaml)
rather than assuming this file stays current on its own.
