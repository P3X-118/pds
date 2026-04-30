# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is — and what we're turning it into

This directory is a fork of [`bluesky-social/pds`](https://github.com/bluesky-social/pds), the *distribution* repo for self-hosting a Bluesky Personal Data Server. Upstream ships:

- A thin Node.js wrapper (`service/index.js`) that boots `@atproto/pds` and exposes a `/tls-check` endpoint for Caddy on-demand TLS.
- A `Dockerfile` that bundles the wrapper + the [`goat`](https://github.com/bluesky-social/goat) admin CLI binary into `ghcr.io/bluesky-social/pds`. Upstream's bash `pdsadmin/*` scripts have been **retired** in this fork in favor of `goat pds admin`.
- An `installer.sh` that provisions a *single* PDS host with Docker + Caddy + a `pds.service` systemd unit + a `watchtower` auto-updater, all rooted at `/pds` (hard-coded — the installer rejects any other path). Not used in SGC deployments.

**The work in this directory has three goals beyond just tracking upstream:**

1. **Multi-domain production deployment via the SGC playbook.** The single-host, fixed-`/pds`-path, watchtower-driven model from upstream is incompatible with how SGC manages services. We're moving operational ownership to the sister Ansible role at `~/sgc/ansible/roles/pds-ar/` (development home), which gets published as `P3X-118/pds-ar` and consumed from `~/sgc/SGC/requirements.yml`. See `~/sgc/CLAUDE.md` for the playbook conventions.
2. **A web admin UX (`PDS-Pro`)** that fronts goat's admin command surface with operator-level OAuth (Okta primary; Google, Microsoft, Facebook, X planned via `goth`). Lives in a separate repo `P3X-118/PDS-Pro`, deployed via its own ansible role `pds-pro-ar`. One admin instance fronts multiple PDS instances. Operator OAuth secrets are file-based per-environment, derived from `sgc_pgsk` per the SGC pattern. Audit log keyed on OAuth subject.
3. **Tracking upstream cleanly.** Weekly automated sync (`~/sgc/bin/pds-sync-upstream.sh`) opens a PR into `sgc-dev` with new upstream commits cherry-picked. `main` mirrors `bluesky-social/pds` with `no_push` enforced; customizations flow `sgc-dev` → `sgc`.

When in doubt about whether a change belongs upstream or in our SGC layer: changes to the *PDS protocol surface* (Node wrapper, Dockerfile, /tls-check) belong here and ideally get pushed upstream; changes to *how an instance is deployed/configured/secured* belong in `pds-ar`; admin UX work belongs in `PDS-Pro`.

## Repo layout

```
service/               # Node.js entrypoint that runs @atproto/pds
  index.js             # Boots PDS + exposes /tls-check (Caddy on-demand TLS hook)
  package.json         # Pinned to a specific @atproto/pds release (current source of truth for PDS version)
Dockerfile             # node:22-alpine; multi-stage build that also compiles goat from source and installs it at /usr/local/bin/goat
compose.yaml           # 3-service stack (caddy + pds + watchtower), all network_mode: host, bind-mounts /pds
installer.sh           # Single-host bootstrapper for Ubuntu/Debian — NOT used in SGC deployments
update.sh              # Pulls a fresh compose.yaml and `systemctl restart pds`
PUBLISH.md             # Release procedure for tagging new image versions
```

## The release / version flow

The `@atproto/pds` npm version pinned in `service/package.json` is the source of truth for what PDS code actually runs. The release flow (from `PUBLISH.md`):

1. `cd service && pnpm update @atproto/pds@<version>`
2. Commit to `main` with message `pds v<version>` — the `build-and-push-ghcr` workflow builds and pushes `ghcr.io/bluesky-social/pds:sha-<sha>` automatically.
3. Smoke test the `sha-*` tag.
4. `git tag v<version> && git push --tags` — this triggers tagged image builds (`<version>`, `<major>.<minor>`). End users track the `0.4` floating tag; `watchtower` picks up changes overnight in the upstream model.

We don't use `watchtower` in SGC deployments — the playbook will pin a specific image tag and update on a controlled cadence.

## Common commands

```bash
# Build the PDS Docker image locally (matches what GHCR builds on push)
docker build -t pds-local .

# Bump the pinned PDS version (only step that changes runtime behavior)
cd service && pnpm update @atproto/pds@<version>

# Run goat admin commands against a deployed instance (admin password auto-loaded from /pds/pds.env)
sudo docker exec pds goat pds admin account list
sudo docker exec pds goat pds admin account create --handle alice.example.com --email alice@example.com --password <pw>
sudo docker exec pds goat pds admin create-invites -n 5

# Run goat from outside the container (point at any PDS, pass admin password explicitly)
goat pds admin --pds-host https://pds.example.com --admin-password <pw> account list
```

There is no test suite, lint config, or CI beyond the image build workflow (`.github/workflows/build-and-push-ghcr.yaml`). Don't invent one without asking.

## Architecture notes worth knowing before editing

- **`/tls-check` is load-bearing.** Caddy's on-demand TLS asks PDS whether to issue a cert for a given subdomain. The handler in `service/index.js` returns 200 only for the configured `PDS_HOSTNAME` or for handles that resolve to an account in `serviceHandleDomains`. Breaking this means no certs get issued for new user handles.
- **Admin auth is a single shared secret** (`PDS_ADMIN_PASSWORD` in `pds.env`, used as HTTP Basic password by every `goat pds admin` call). Every admin XRPC call goes over `https://${PDS_HOSTNAME}/xrpc/...` — the public hostname, not localhost. The `PDS-Pro` web UX is the place where per-operator identity gets layered on top via OAuth, with the shared admin secret kept server-side and never reaching the browser.
- **goat's env file lookup is hardcoded to `/pds/pds.env`** (in `NewPDSAdminClient`). For SGC's `/sgc/sgc-pds-<instance>/pds.env` layout, either pass `--admin-password` explicitly or land an upstream PR adding `PDS_ENV_FILE` env var support. The `PDS-Pro` web UX will use the explicit-password approach.
- **Upstream installer assumptions we're discarding in SGC:**
  - Hard-codes data dir to `/pds`. We use `/sgc/sgc-pds-<instance>` per the SGC pathing convention.
  - Uses `network_mode: host` for all three containers. SGC services run on Traefik-routed Docker networks (`revproxy_service_networks`).
  - Bundles its own Caddy. SGC routes through Traefik instead — `pds-ar`'s systemd template already wires Traefik labels.
  - Bundles `watchtower`. SGC pins versions via the role's `requirements.yml` entry.
- **Secret generation in `installer.sh`** uses `openssl rand --hex 16` for `PDS_JWT_SECRET` / `PDS_ADMIN_PASSWORD` and a `secp256k1` keygen for `PDS_PLC_ROTATION_KEY_K256_PRIVATE_KEY_HEX`. The `pds-ar` role should derive these the SGC way (`sgc_pgsk` + `password_hash` with a per-service salt) for the symmetric secrets, but the PLC rotation key is an asymmetric ECDSA key — that has to be generated once per instance and persisted, not derived on every run.

## Sister Ansible role: `~/sgc/ansible/roles/pds-ar/`

This is the role that operationalizes PDS in SGC. **It is currently a stub** — only `Initial PDS Ansible role` has been committed, on a single `master` branch. It does *not* yet follow the SGC 3-branch model (`main`/`sgc-dev`/`sgc`) described in `~/sgc/CLAUDE.md`, and it has known issues to fix before it's production-ready:

- `defaults/main.yml`: `pds_container_image` has a stray space (`"docker.io/legitservices/ pds:latest"`) and points at the wrong registry — should be `ghcr.io/bluesky-social/pds:<pinned tag>`.
- `defaults/main.yml`: `pds_gid` is set to `sgc_playbook_uid` (typo, should be `sgc_playbook_gid`).
- `templates/pds.service.j2`: references `pds_container_labels_traefik` which is never defined in `defaults/main.yml`.
- `pds_environment_variables` only sets `PDS_HOSTNAME` and `PDS_PORT` — the full upstream env contract (JWT secret, admin password, PLC key, blobstore path, AppView/PLC/report URLs, crawler list) needs to be templated in.
- No `postgres_autom_itemized` integration yet — PDS defaults to SQLite and we'll likely keep it that way per-instance, but if/when we add Postgres, follow the pattern in `~/sgc/CLAUDE.md`.
- No `MIGRATION.md` for promoting an existing `installer.sh`-deployed PDS into the SGC layout (data dir move, env regeneration without invalidating the existing PLC rotation key).

When working on the role, follow the 4-step "Adding a New Ansible Role" pattern in `~/sgc/CLAUDE.md` for wiring it into `~/sgc/SGC/requirements.yml`, `setup.yml`, and `group_vars/mash_servers`. The role-specific comment markers (`# role-specific:pds` / `# /role-specific:pds`) are mandatory — `bin/optimize.py` depends on them.

## Sibling project: `P3X-118/PDS-Pro` (planned)

A small Go web app that wraps `goat pds admin` with operator-level OAuth (Okta primary; Google/Microsoft/Facebook/X via `goth`). Server-rendered HTML (templ + htmx). One admin instance fronts multiple PDS instances. Allowlist-based authorization initially; claim-based later. Per-action audit log on the OAuth subject. Deployed via its own ansible role `~/sgc/ansible/roles/pds-pro-ar/` following the SGC 3-branch model.
