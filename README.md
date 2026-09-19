# mcp-servers

A single `docker-compose.yml` that brings up every homelab MCP (Model
Context Protocol) server discussed alongside this one — the custom-built
ones in this GitHub org plus the third-party ones adopted after research
turned up better-maintained alternatives.

| Server | Source | Port |
|---|---|---|
| `sonarr-mcp` | [Barrow1990/sonarr-mcp-server](https://github.com/Barrow1990/sonarr-mcp-server) | 8931 |
| `radarr-mcp` | [Barrow1990/radarr-mcp-server](https://github.com/Barrow1990/radarr-mcp-server) | 8932 |
| `bazarr-mcp` | [Barrow1990/bazarr-mcp-server](https://github.com/Barrow1990/bazarr-mcp-server) | 8933 |
| `prowlarr-mcp` | [Barrow1990/prowlarr-mcp-server](https://github.com/Barrow1990/prowlarr-mcp-server) | 8934 |
| `dockhand-mcp` | [strausmann/mcp-dockhand](https://github.com/strausmann/mcp-dockhand) (third-party) | 8935 |
| `jellyfin-mcp` | [jaredtrent/jellyfin-mcp](https://github.com/jaredtrent/jellyfin-mcp) (third-party) | 8936 |
| `authentik-mcp` | [Barrow1990/authentik-mcp-server](https://github.com/Barrow1990/authentik-mcp-server) | 8937 |
| `lubelogger-mcp` | [hargata/lubelog_mcp](https://github.com/hargata/lubelog_mcp) (third-party, official) | 8938 |
| `seerr-mcp` | [jhomen368/overseerr-mcp](https://github.com/jhomen368/overseerr-mcp) (third-party) | 8939 |
| `streamystats-mcp` | [Barrow1990/streamystats-mcp-server](https://github.com/Barrow1990/streamystats-mcp-server) | 8940 |
| `reclaimerr-mcp` | [Barrow1990/reclaimerr-mcp-server](https://github.com/Barrow1990/reclaimerr-mcp-server) | 8941 |

This repo carries **no source code of its own** — it's Compose config +
per-service env-file templates, pulling pre-built images from each project's
own registry. It does not build anything.

## Does this fit how Dockhand expects to be used? Read this before deploying.

**Short answer: yes for a plain `docker compose up -d`, but not as a single
Dockhand-managed stack if you want per-service secrets — and you do.**

Dockhand's per-stack config UI writes one `.env.dockhand` file *per stack*.
Every server in this repo needs its own distinct credentials (a different
API key/token per app), so if this whole repo were pointed at as **one**
Dockhand stack, Dockhand would have nowhere to put 11 different secrets
through its normal UI — you'd end up hand-editing files in this repo's git
checkout instead, which defeats the point of managing it through Dockhand at
all.

This repo solves that by giving each service its **own** env file under
`env/` (`env/sonarr.env`, `env/radarr.env`, ...) instead of one shared
`.env` — so nothing here leaks one service's credentials into another
container. But that split only helps for a manual `docker compose up -d`;
Dockhand's single-stack env override still can't reach into per-service
files inside one stack's checkout the way its UI is designed to work.

**Recommendation:**
- **If you use Dockhand**: keep each server as its own separate Dockhand
  stack pointed at its own repo (`sonarr-mcp-server`, `radarr-mcp-server`,
  ...), exactly as already set up — that's the actual fit for Dockhand's
  one-stack-one-env-file model, and it isolates credentials per container.
  Use *this* repo only as a reference list of what's running and their
  images, or to `docker compose up -d` the whole fleet manually outside
  Dockhand (e.g. for a fresh box, or local testing) when you don't need
  Dockhand's per-stack UI for it.
- **If you don't use Dockhand for these at all** (plain `docker compose`):
  this repo works as intended — copy each `env/*.env.example` to
  `env/<service>.env`, fill in real values, `docker compose up -d`, done.

## Setup

```bash
cp .env.example .env                                  # only JELLYFIN_HTTP_TOKEN — see below
for f in env/*.env.example; do cp "$f" "${f%.example}"; done
# now edit .env and every env/*.env with real URLs/keys
docker compose up -d --pull always
```

### Why `.env` at the root AND `env/<service>.env` per service?

`env_file:` (used for every service's own credentials, in `env/`) only
injects variables into *that container's* runtime environment — it can't
feed a `command:` argument, because `command:` is resolved by `docker
compose` itself before any container starts. `jellyfin-mcp` is the one
service in this stack that needs its bearer token passed as a CLI flag
(`--http-token`, confirmed from its own docker-compose.yml) rather than read
as a plain env var, so that one value — `JELLYFIN_HTTP_TOKEN` — has to live
in this repo's root `.env`, which `docker compose` *does* read for `${VAR}`
substitution. Every other credential in this stack stays in its own
`env/<service>.env` file exactly where you'd expect it.

## Third-party servers: what to verify yourself

These four aren't built or tested by this repo — before trusting them
against real credentials, check each project's own README/releases for
anything that's changed since this repo was put together:

- **dockhand-mcp**: authenticates with your Dockhand **username/password**
  (session cookie, auto-relogin on 401), not an API token — different from
  every *arr-family server in this stack.
- **jellyfin-mcp**: the bearer token is **required** once bound to
  `0.0.0.0` (which this compose does, to be LAN-reachable) — don't skip
  `JELLYFIN_HTTP_TOKEN`. A Jellyfin API key is admin-equivalent, so this
  container is worth taking as seriously as `authentik-mcp`.
- **lubelogger-mcp**: built by LubeLogger's own author, self-described as
  "experimental" and may "break without prior notice." `env/lubelogger.env`
  uses its "Local Auth" mode (full account access); a narrower "Header
  Auth" mode exists but its exact variable names weren't confirmable — check
  <https://github.com/hargata/lubelog_mcp/wiki/Header-Auth> if you want that
  instead.
- **seerr-mcp**: targets `seerr-team/seerr` but the API is unchanged from
  Jellyseerr/Overseerr, so it works against any of the three.

## Security notes (apply to the whole stack, not just this repo)

- None of these ports should ever be reachable from outside your LAN/VLAN —
  no reverse proxy, no port-forward. Every one of these tokens grants real
  access to a real system behind it.
- Set `MCP_AUTH_TOKEN` (where the service supports it — see each
  `env/*.env.example`) rather than leaving a server open to anything that
  can reach the port.
- `authentik-mcp` additionally requires `AUTHENTIK_ALLOW_WRITES=true` *and*
  `confirm=True` on the call itself before its one write tool
  (`set_user_active`) will do anything — see that repo's own README for why.
- If you're not currently using one of these 11 servers, don't run it —
  every extra container here is extra surface area for zero benefit.

## License

MIT
