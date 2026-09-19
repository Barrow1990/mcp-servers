# mcp-servers

One `docker-compose.yml` and one `.env` for every MCP (Model Context
Protocol) server in this homelab — the custom-built ones in this GitHub org
plus the third-party ones adopted after research found better-maintained
alternatives.

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

This repo carries no source code — Compose config plus one env template,
pulling pre-built images from each project's own registry.

## One file, but no container sees more than its own variables

Every value lives in a single `.env`, but `docker-compose.yml` deliberately
does **not** use `env_file:` (which would dump the whole file into every
container). Instead each service's `environment:` block names its own
variables explicitly, e.g.:

```yaml
sonarr-mcp:
  environment:
    SONARR_URL: ${SONARR_URL:?}
    SONARR_API_KEY: ${SONARR_API_KEY:?}
    MCP_AUTH_TOKEN: ${SONARR_MCP_AUTH_TOKEN:-}
```

`${VAR}` substitution is resolved once, per-variable, at `docker compose`
parse time — it doesn't hand the container a file to read, so a variable
never referenced in a given service's block simply never reaches that
container. Verified directly: `docker compose config` on this repo shows
`sonarr-mcp`'s resolved environment contains exactly `SONARR_URL`,
`SONARR_API_KEY`, `MCP_AUTH_TOKEN`, `MCP_HOST`, `MCP_PORT` — no
`AUTHENTIK_API_TOKEN`, no `DOCKHAND_PASSWORD`, nothing from any other
service. Run `docker compose config` yourself after editing `.env` any time
you want to re-check this.

The `MCP_AUTH_TOKEN` env var name is hardcoded identically in every
custom-built server's own code, so `.env` gives each one a distinct source
variable (`SONARR_MCP_AUTH_TOKEN`, `RADARR_MCP_AUTH_TOKEN`, ...) and the
compose file maps each to the generic name that specific container expects
— that mapping is what keeps the tokens from colliding into one shared
secret across services.

The `:?` on required variables makes `docker compose up` fail fast with a
clear "variable is not set" error if you forget one, rather than starting a
container with an empty credential. `:-` (empty default) is used only for
genuinely optional variables.

## Does this fit how Dockhand expects to be used?

**Only if Dockhand feeds `docker compose` a plain `.env`.** Compose's `${VAR}`
substitution reads exclusively from a literal `.env` in the project
directory (or `--env-file <path>` / real shell environment variables) —
never from an arbitrarily-named file. The per-service `docker-compose.yml`
files in this same GitHub org (sonarr, radarr, etc.) were written on the
understanding that Dockhand writes its UI-configured values to a separate
`.env.dockhand` file, which is why those use `env_file: [.env, .env.dockhand]`
instead of substitution — `env_file:` can load an arbitrarily-named file;
substitution can't.

This repo hasn't been verified against a live Dockhand deployment, so
before relying on it:

1. Deploy this stack in Dockhand and check whether the values you set in
   its UI actually reach the containers (`docker compose config` on the
   deployed host, or just check a container's env — `docker exec sonarr-mcp
   env`).
2. If they don't — i.e. Dockhand is writing to `.env.dockhand` and Compose
   is silently substituting empty defaults or failing on the `:?` vars —
   the fix is either: edit `.env` directly in this repo's checkout instead
   of through Dockhand's UI (you lose the UI convenience, keep the
   isolation), or check Dockhand's stack settings for whether the env
   filename/path it targets is configurable, and point it at `.env` if so.

If that turns out not to work cleanly, the fallback is the same trade-off
as before: deploy each server as its own separate Dockhand stack (pointed
at its own repo, with its own `.env.dockhand`) instead of this combined
one — isolation either way, just at the stack level instead of the
variable level.

## Setup

```bash
cp .env.example .env   # fill in every service's real values
docker compose up -d --pull always
```

## Third-party servers: verify before trusting

Four of these aren't built or tested here — check each project's own
README/releases for anything that's changed since this repo was put
together:

- **dockhand-mcp**: authenticates with your Dockhand **username/password**
  (session cookie, auto-relogin on 401), not an API token.
- **jellyfin-mcp**: the bearer token is **required** once bound to `0.0.0.0`
  (which this compose does) — don't skip `JELLYFIN_HTTP_TOKEN`. A Jellyfin
  API key is admin-equivalent, so this container is worth taking as
  seriously as `authentik-mcp`.
- **lubelogger-mcp**: built by LubeLogger's own author, self-described as
  "experimental" and may "break without prior notice." `.env.example` uses
  its "Local Auth" mode (full account access); a narrower "Header Auth"
  mode exists but its exact variable names weren't confirmable — check
  <https://github.com/hargata/lubelog_mcp/wiki/Header-Auth> if you want that
  instead.
- **seerr-mcp**: targets `seerr-team/seerr` but the API is unchanged from
  Jellyseerr/Overseerr, so it works against any of the three.

## Security notes (apply to the whole stack)

- None of these ports should ever be reachable from outside your LAN/VLAN —
  no reverse proxy, no port-forward. Every one of these tokens grants real
  access to a real system behind it.
- `authentik-mcp` additionally requires `AUTHENTIK_ALLOW_WRITES=true` *and*
  `confirm=True` on the call itself before its one write tool
  (`set_user_active`) will do anything.
- If you're not currently using one of these 11 servers, don't run it —
  comment it out of `docker-compose.yml` rather than leaving unused surface
  area up.

## License

MIT
