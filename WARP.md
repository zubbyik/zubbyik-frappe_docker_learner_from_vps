# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## What this repo is
This is a containerized setup for running and developing Frappe/ERPNext using Docker Compose. The repo is mostly Compose/YAML + container build files; application code lives either:
- in a “bench” created under `development/` (git-ignored), or
- in custom apps under `apps/` (example: `apps/library_management`).

Key docs (worth reading first for unfamiliar workflows):
- `docs/getting-started.md` (big-picture architecture + workflows)
- `docs/development.md` (devcontainer / bench creation)
- `docs/site-operations.md` (bench commands via containers)

## Common commands
### Quick “Play With Docker”-style local run (single file)
Uses `pwd.yml` and creates a site via the `create-site` one-shot container.

```sh
# Start
docker compose -f pwd.yml up -d

# Follow initial site creation
docker compose -f pwd.yml logs -f create-site

# Stop + wipe volumes (removes DB + site data)
docker compose -f pwd.yml down -v
```

### Main Compose stack (compose.yaml + overrides)
Important: `compose.yaml` defines the core Frappe services but does **not** include DB/Redis by default.

A common “self-contained” setup is MariaDB + Redis + no proxy:

```sh
cp example.env .env

docker compose --env-file .env \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  up -d
```

Useful ops (same compose args as above):

```sh
# Logs
docker compose ... logs -f backend

# Shell in backend (where `bench` is available)
docker compose ... exec backend bash

# Common bench operations
docker compose ... exec backend bench list-sites
docker compose ... exec backend bench --site <site> console
docker compose ... exec backend bench --site <site> migrate
```

### Development environment (Dev Container)
The supported dev workflow is to run a “bench tools” container (`frappe/bench`) and create your bench under `development/`.

```sh
# One-time setup
cp -R devcontainer-example .devcontainer

# Start the devcontainer compose stack
docker compose -f .devcontainer/docker-compose.yml up -d

# Get a shell inside the dev container
docker compose -f .devcontainer/docker-compose.yml exec frappe bash

# Inside the container, create bench + site (interactive)
cd /workspace/development
python installer.py

# Start Frappe dev processes
cd /workspace/development/frappe-bench
bench start
```

### Lint / formatting
CI runs pre-commit on all files (`.github/workflows/lint.yml`).

```sh
python -m pip install -U pre-commit
pre-commit run --all-files
```

Notes:
- Root `.pre-commit-config.yaml` includes a `shfmt` hook that needs a working Go toolchain locally.
- The custom app `apps/library_management` has its own pre-commit config.

```sh
cd apps/library_management
python -m pip install -U pre-commit
pre-commit run --all-files
```

### Tests
Tests are integration-style and use Docker Compose to start services and then execute checks (`tests/test_frappe_docker.py`).

```sh
python -m venv .venv
source .venv/bin/activate
pip install -r requirements-test.txt

# Full test suite
pytest

# Run a single test
pytest tests/test_frappe_docker.py::test_endpoints
```

Notes:
- The test harness (`tests/utils.py::Compose`) runs `docker compose -p test ...` and tears down volumes on exit.
- CI may inject `FRAPPE_VERSION` / `ERPNEXT_VERSION`; locally they can be omitted.

## High-level architecture & code layout
### Compose services (runtime)
`compose.yaml` defines the “core” Frappe runtime services:
- `configurator`: one-shot init that writes `common_site_config.json` values (db/redis/socketio config) into the shared `sites` volume.
- `backend`: the main Python web process.
- `frontend`: nginx reverse proxy (serves assets; routes to backend + websocket). Uses `resources/nginx-entrypoint.sh` and `resources/nginx-template.conf`.
- `websocket`: Node Socket.IO server.
- `queue-short`, `queue-long`: background workers.
- `scheduler`: scheduled jobs.

All of these share the `sites` volume (`/home/frappe/frappe-bench/sites`).

### Overrides
`overrides/` contains compose fragments to add/modify services for specific deployments (DB choice, redis, proxy/traefik, TLS, etc.).
Examples:
- `overrides/compose.mariadb.yaml` / `overrides/compose.postgres.yaml`
- `overrides/compose.redis.yaml`
- `overrides/compose.proxy.yaml`, `overrides/compose.https.yaml`

### Container images and builds
- `images/bench/Dockerfile`: the “bench tooling” image used for devcontainers.
- `images/production/Containerfile`: the main runtime image used by `compose.yaml` (`frappe/erpnext:*`).
- `images/custom/Containerfile` and `images/layered/Containerfile`: customizable build variants.
- `docker-bake.hcl`: Buildx Bake definitions (targets like `bench`, `erpnext`, `base`, `build`).

### Development area
- `development/` is intended for local benches/sites and is git-ignored.
- `development/installer.py` is the helper to create a bench + site and set required “container hostnames” for db/redis.

### Custom apps
- `apps/` contains custom app repositories checked into this repo.
- `apps/library_management` is a standard Frappe app scaffold:
  - `apps/library_management/library_management/hooks.py`: app metadata + hook points.
  - Python package config lives in `apps/library_management/pyproject.toml` (ruff config, etc.).

### Tests
- `tests/` contains Docker+Frappe integration tests.
- `tests/compose.ci.yaml` rewrites image references for CI.
- `tests/conftest.py` creates a temp `.env` (from `example.env`) and brings up the compose stack before tests.
