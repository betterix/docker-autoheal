# AGENTS.md — Developer & AI Agent Guide

This document provides architectural context, operational knowledge, development conventions, and maintenance guidelines for AI agents and human contributors working on `docker-autoheal`.

---

## 1. Project Overview

`docker-autoheal` is a lightweight, zero-dependency monitoring daemon for Docker containers. It detects containers marked as `unhealthy` by Docker's native `HEALTHCHECK` mechanism and automatically restarts them via the Docker Engine API.

### Core Philosophy
* **Minimal footprint**: Based on lightweight Alpine Linux with only `curl` and `jq`.
* **Zero external runtime dependencies**: Does not require Python, Node.js, or heavy runtimes.
* **Resilient**: Must never crash due to transient network hiccups, temporary Docker daemon restarts, or socket timeouts.

---

## 2. Codebase Architecture

```
docker-autoheal/
├── Dockerfile                      # Builds the minimal Alpine runtime container
├── docker-entrypoint               # Main daemon script executed inside container
├── .dockerignore                   # Build context exclusions (keeps builds fast)
├── .github/
│   ├── dependabot.yml              # Daily dependency updates with multi-ecosystem-groups
│   └── workflows/
│       └── github-build.yml        # CI/CD: ShellCheck linting & multi-arch Buildx to ghcr.io
├── tests/
│   ├── docker-compose.yml          # Test services (healthy, unhealthy, missing label)
│   ├── docker-compose.autoheal.yml # Test harness service
│   ├── tests.sh                    # Test runner
│   └── watch-autoheal/             # Test harness listening to Docker restart events
└── README.md                       # User-facing documentation
```

### Component Details

1. **[`docker-entrypoint`](docker-entrypoint)**:
   * **Endpoint detection**: Parses `DOCKER_SOCK` (`/var/run/docker.sock`, `tcp://...`, or `tcps://...` with TLS certificates under `/certs`).
   * **Query loop**: Periodically calls Docker API `/containers/json?filters={...}` using `curl`.
   * **Parsing**: Uses `jq` to parse container IDs, names, states, and optional stop timeout labels (`autoheal.stop.timeout`).
   * **Restart action**: Calls Docker API `POST /containers/{id}/restart?t={timeout}`.
   * **Notifications**: Dispatches background webhooks (`WEBHOOK_URL`), Apprise alerts (`APPRISE_URL`), or external post-restart scripts (`POST_RESTART_SCRIPT`).
   * **Signal handling**: Traps `TERM` and `INT` to gracefully shut down (exit code 143).

2. **[`Dockerfile`](Dockerfile)**:
   * Uses `alpine:latest`.
   * Installs `curl` and `jq`.
   * Copies `docker-entrypoint` with `--chmod=755`.
   * Includes internal `HEALTHCHECK` command (`pgrep -f autoheal || exit 1`).
   * Runs `docker-entrypoint autoheal` as the entrypoint/cmd.

3. **[`.github/workflows/github-build.yml`](.github/workflows/github-build.yml)**:
   * Runs on pull requests and pushes to `main` (and tags).
   * Lints all shell scripts with ShellCheck (`ludeeus/action-shellcheck`).
   * Builds multi-platform images (`linux/amd64`, `linux/arm64`).
   * Pushes to GitHub Container Registry (`ghcr.io`) only on push/tag events, not on pull requests.

4. **[`.github/dependabot.yml`](.github/dependabot.yml)**:
   * Manages daily grouped updates across `github-actions` and `docker` ecosystems using `multi-ecosystem-groups: all-updates`.

---

## 3. Environment Variables Reference

| Variable | Default | Purpose |
|---|---|---|
| `AUTOHEAL_CONTAINER_LABEL` | `autoheal` | Label to watch (matches `label=true`). Use `all` to watch all containers. |
| `AUTOHEAL_INTERVAL` | `5` | Polling interval in seconds between health checks. |
| `AUTOHEAL_START_PERIOD` | `0` | Delay in seconds before the first health check loop starts. |
| `AUTOHEAL_DEFAULT_STOP_TIMEOUT` | `10` | Default timeout in seconds for container stop before killing during restart. |
| `AUTOHEAL_ONLY_MONITOR_RUNNING` | `false` | If `true`, only inspect containers with status `running` (ignores paused). |
| `DOCKER_SOCK` | `/var/run/docker.sock` | Path to Docker UNIX socket or TCP address (`tcp://...` / `tcps://...`). |
| `CURL_TIMEOUT` | `30` | `--max-time` seconds for Docker API curl requests. |
| `WEBHOOK_URL` | `""` | URL to POST JSON notification on container restart. |
| `WEBHOOK_JSON_KEY` | `text` | Key name for the message string in the webhook payload. |
| `APPRISE_URL` | `""` | Apprise API endpoint for push notifications. |
| `POST_RESTART_SCRIPT` | `""` | Path to executable script called after restart with container arguments. |

---

## 4. Engineering & Scripting Rules for AI Agents

When editing or proposing changes in this repository, always follow these rules:

### A. POSIX Shell Compatibility
* The entrypoint script uses `#!/usr/bin/env sh` and runs in Alpine BusyBox `ash`.
* **Do NOT use bashisms**:
  * Do NOT use `${var:0:12}` substring slicing — use `$(printf '%.12s' "$var")`.
  * Do NOT use `${var:1}` — use `${var#/}` to strip leading slashes.
  * Do NOT use `[[ ... ]]` — use standard `[ ... ]` test expressions.
  * Do NOT use bash arrays `arr=(...)` — use space-separated lists or POSIX positional parameters.
* Always quote variable expansions to prevent word-splitting and glob expansion (`"$var"`).

### B. Safe JSON Generation
* **Never hand-roll JSON strings or heredocs with variable interpolation**:
  * Bad: `cat <<EOF { "key": "$msg" } EOF` (breaks when `$msg` contains quotes, newlines, or backslashes).
  * Good: `jq -nc --arg key "$KEY" --arg msg "$TEXT" '{($key): $msg}'` (guarantees 100% valid JSON escaping).

### C. Daemon Resilience (Never Crash on Transient Failures)
* The entrypoint uses `set -e` and `set -o pipefail`.
* **Never run pipelines like `get_container_info | jq ...` unguarded**:
  * If the Docker daemon temporarily restarts or socket times out, an unguarded pipe will fail and terminate the Autoheal process immediately.
  * Always capture the API output first or guard with `if ...; then ...; else log_warning; fi`.

### D. Signal Handling
* Always trap signals cleanly (`trap 'term_handler' TERM INT`).
* Never call `kill $$` inside a trap handler, as it sends signals back into the same process and triggers loops.

### E. Image & Build Context Hygiene
* Keep `Dockerfile` clean without storing sensitive-looking keys in `ENV` instructions (Docker build linter flags `*_KEY` in `ENV`).
* Keep `.dockerignore` up to date (`.git`, `*.md`, `LICENSE`, `tests`) so local git history is never transferred into build contexts.

---

## 5. Verification Commands

Before committing any changes, verify the following:

```bash
# 1. ShellCheck (must pass with 0 errors and 0 warnings)
docker run --rm -v "$PWD:/mnt" koalaman/shellcheck:stable /mnt/docker-entrypoint

# 2. Shell syntax validation
sh -n docker-entrypoint

# 3. Docker build test
docker build -t autoheal-local .
```
