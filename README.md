# mcp-servers

An index of every MCP (Model Context Protocol) server used in this homelab —
one server per app, each deployed and managed as its own separate stack.
This repo has no code, no compose file, and no config of its own; it exists
purely so there's one place that lists what's running and points at each
server's real source.

Each server is its own standing network service (streamable-HTTP, not
stdio) — deploy each one as its own Dockhand stack pointed at its own repo
below, with its own `.env`/`.env.dockhand` for credentials. That's a
deliberate choice, not an oversight: bundling all of these into one combined
stack would force one shared env file across all of them (Dockhand writes a
single `.env.dockhand` per stack), which means every container ends up
holding every other service's credentials in its own environment — a much
bigger blast radius than any of these servers need. Keeping them as separate
stacks means a problem in one container never exposes another's secrets.

## Servers

| Server | Source | Port | Auth |
|---|---|---|---|
| sonarr-mcp | [Barrow1990/sonarr-mcp-server](https://github.com/Barrow1990/sonarr-mcp-server) | 8931 | `SONARR_API_KEY` |
| radarr-mcp | [Barrow1990/radarr-mcp-server](https://github.com/Barrow1990/radarr-mcp-server) | 8932 | `RADARR_API_KEY` |
| bazarr-mcp | [Barrow1990/bazarr-mcp-server](https://github.com/Barrow1990/bazarr-mcp-server) | 8933 | `BAZARR_API_KEY` |
| prowlarr-mcp | [Barrow1990/prowlarr-mcp-server](https://github.com/Barrow1990/prowlarr-mcp-server) | 8934 | `PROWLARR_API_KEY` |
| dockhand-mcp | [strausmann/mcp-dockhand](https://github.com/strausmann/mcp-dockhand) (third-party) | 8935 (image default 8080) | Dockhand username/password |
| jellyfin-mcp | [jaredtrent/jellyfin-mcp](https://github.com/jaredtrent/jellyfin-mcp) (third-party) | 8936 (image default 8080) | `JELLYFIN_API_KEY` + required bearer token when bound off localhost |
| authentik-mcp | [Barrow1990/authentik-mcp-server](https://github.com/Barrow1990/authentik-mcp-server) | 8937 | `AUTHENTIK_API_TOKEN` (read-mostly; writes need `AUTHENTIK_ALLOW_WRITES=true` + `confirm=True` per call) |
| lubelogger-mcp | [hargata/lubelog_mcp](https://github.com/hargata/lubelog_mcp) (third-party, official) | 8938 (image default 8080) | LubeLogger username/password (Local Auth) or header token (Header Auth) |
| seerr-mcp | [jhomen368/overseerr-mcp](https://github.com/jhomen368/overseerr-mcp) (third-party) | 8939 (image default 8085) | `SEERR_API_KEY` — works against seerr-team/seerr, Jellyseerr, or Overseerr |
| streamystats-mcp | [Barrow1990/streamystats-mcp-server](https://github.com/Barrow1990/streamystats-mcp-server) | 8940 | `STREAMYSTATS_JELLYFIN_TOKEN` (a Jellyfin API key — Streamystats has no API key of its own) |
| reclaimerr-mcp | [Barrow1990/reclaimerr-mcp-server](https://github.com/Barrow1990/reclaimerr-mcp-server) | 8941 | `RECLAIMERR_API_TOKEN` (scoped) |

Ports are the convention used across this fleet to avoid collisions if
several ever do run on the same host — each server's own repo is the source
of truth for its actual default and how to override it.

## Third-party servers: verify before trusting

Four of these aren't built or tested here — check each project's own
README/releases for anything that's changed since this list was put
together:

- **dockhand-mcp**: session-based auth (username/password), not an API
  token like the rest of this fleet.
- **jellyfin-mcp**: a Jellyfin API key is admin-equivalent access to your
  media server — take its bearer-token requirement seriously.
- **lubelogger-mcp**: built by LubeLogger's own author, self-described as
  "experimental" and may change without notice.
- **seerr-mcp**: targets `seerr-team/seerr` but the API is shared with
  Jellyseerr/Overseerr, so it works against any of the three.

## License

MIT
