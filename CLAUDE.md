# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is — and what we're turning it into

This directory is a fork of [`bluesky-social/pds`](https://github.com/bluesky-social/pds), the *distribution* repo for self-hosting a Bluesky Personal Data Server. Upstream ships:

- A thin Node.js wrapper (`service/index.js`) that boots `@atproto/pds` and exposes a `/tls-check` endpoint for Caddy on-demand TLS.
- A `Dockerfile` that bundles the wrapper + the `pdsadmin/*` admin scripts into `ghcr.io/bluesky-social/pds`.
- An `installer.sh` that provisions a *single* PDS host with Docker + Caddy + a `pds.service` systemd unit + a `watchtower` auto-updater, all rooted at `/pds` (hard-coded — the installer rejects any other path).
- `pdsadmin.sh` and the `pdsadmin/` subcommands, which are thin `curl` wrappers over the PDS admin XRPC endpoints (`com.atproto.admin.*`, `com.atproto.server.createInviteCode`, etc.) authenticated with HTTP Basic `admin:$PDS_ADMIN_PASSWORD` from `/pds/pds.env`.

**The work in this directory has two goals beyond just tracking upstream:**

1. **Multi-domain production deployment via the SGC playbook.** The single-host, fixed-`/pds`-path, watchtower-driven model from upstream is incompatible with how SGC manages services. We're moving operational ownership to the sister Ansible role at `~/sgc/ansible/roles/pds-ar/` (development home), which gets published as `P3X-118/pds-ar` and consumed from `~/sgc/SGC/requirements.yml`. See `~/sgc/CLAUDE.md` for the playbook conventions.
2. **A dead-simple admin UX** for the small set of operations we actually do: list/create/delete accounts, reset passwords, takedown/untakedown, mint invite codes, request crawls. Today these live as bash scripts in `pdsadmin/`; the UX should call the same XRPC endpoints rather than reinventing protocol logic.

When in doubt about whether a change belongs upstream or in our SGC layer: changes to the *PDS protocol surface* (Node wrapper, Dockerfile, admin XRPC calls) belong here and ideally get pushed upstream; changes to *how an instance is deployed/configured/secured* belong in `pds-ar`.

## Repo layout

```
service/               # Node.js entrypoint that runs @atproto/pds
  index.js             # Boots PDS + exposes /tls-check (Caddy on-demand TLS hook)
  package.json         # Pinned to a specific @atproto/pds release (current source of truth for PDS version)
pdsadmin/              # Bash subcommands invoked by /usr/local/bin/pdsadmin
  account              # list / create / delete / takedown / untakedown / reset-password
  create-invite-code   # POSTs to com.atproto.server.createInviteCode
  request-crawl        # POSTs com.atproto.sync.requestCrawl to relay hosts
  pdshelp              # Help text for the `pdsadmin help` command
pdsadmin.sh            # Legacy host-side dispatcher; downloads sub-scripts from GitHub at runtime.
                       # Inside the container, the pdsadmin/* scripts are copied straight to
                       # /usr/local/bin and dispatched directly (no network fetch).
Dockerfile             # node:20.13.1-alpine3.18; pnpm install --production; copies pdsadmin/* to /usr/local/bin
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

# Run pdsadmin against a deployed instance (host-side, expects /pds/pds.env)
sudo pdsadmin help
sudo pdsadmin account list
sudo pdsadmin account create alice@example.com alice.example.com
sudo pdsadmin create-invite-code

# Override the env file location (useful for multi-tenant on one host)
sudo PDS_ENV_FILE=/sgc/sgc-pds-foo/pds.env pdsadmin account list
```

There is no test suite, lint config, or CI beyond the image build workflow (`.github/workflows/build-and-push-ghcr.yaml`). Don't invent one without asking.

## Architecture notes worth knowing before editing

- **`/tls-check` is load-bearing.** Caddy's on-demand TLS asks PDS whether to issue a cert for a given subdomain. The handler in `service/index.js` returns 200 only for the configured `PDS_HOSTNAME` or for handles that resolve to an account in `serviceHandleDomains`. Breaking this means no certs get issued for new user handles.
- **Admin auth is a single shared secret** (`PDS_ADMIN_PASSWORD` in `pds.env`, used as HTTP Basic password by every `pdsadmin` subcommand). Every admin XRPC call goes over `https://${PDS_HOSTNAME}/xrpc/...` — the admin scripts hit the *public* hostname, not localhost. Any new admin tooling should follow the same pattern: it's just authenticated XRPC, no privileged side channels.
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
