# Docker Autoheal

[![CI Build](https://github.com/betterix/docker-autoheal/actions/workflows/github-build.yml/badge.svg)](https://github.com/betterix/docker-autoheal/actions/workflows/github-build.yml)
[![GitHub Release](https://img.shields.io/github/v/release/betterix/docker-autoheal?include_prereleases&label=release)](https://github.com/betterix/docker-autoheal/releases)
[![Container Image](https://img.shields.io/badge/ghcr.io-betterix%2Fdocker--autoheal-blue?logo=docker)](https://ghcr.io/betterix/docker-autoheal)

A lightweight, zero-dependency monitoring daemon that monitors Docker containers and automatically restarts any container that enters an `unhealthy` state according to Docker's native `HEALTHCHECK`.

---

## Features

* **Lightweight & Fast**: Built on minimal Alpine Linux with only `curl` and `jq`.
* **Multi-Arch**: Pre-built for `linux/amd64` and `linux/arm64`.
* **Crash-Resilient**: Designed to stay alive through transient Docker daemon restarts and socket timeouts.
* **Flexible Monitoring Modes**: Watch all containers, or only containers explicitly opting in via labels.
* **Multi-Channel Alerts**: Webhooks with safe JSON encoding, Apprise push alerts, and post-restart hook scripts.
* **Graceful Timeouts**: Configurable per-container stop timeouts (`autoheal.stop.timeout`) before forced termination.

---

## Container Image

Published to GitHub Container Registry:

```bash
ghcr.io/betterix/docker-autoheal:latest
```

---

## Quick Start

### 1. Docker Compose (Recommended)

Add Autoheal as a service in your `docker-compose.yml`:

```yaml
services:
  autoheal:
    image: ghcr.io/betterix/docker-autoheal:latest
    container_name: autoheal
    restart: always
    network_mode: none
    environment:
      AUTOHEAL_CONTAINER_LABEL: all
      AUTOHEAL_INTERVAL: 10
      AUTOHEAL_DEFAULT_STOP_TIMEOUT: 10
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

### 2. Docker CLI

#### Standard UNIX Socket (Monitor All Containers)

```bash
docker run -d \
  --name autoheal \
  --restart always \
  -e AUTOHEAL_CONTAINER_LABEL=all \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/betterix/docker-autoheal:latest
```

#### Opt-in Monitoring (Only Restart Containers with `autoheal=true`)

```bash
docker run -d \
  --name autoheal \
  --restart always \
  -e AUTOHEAL_CONTAINER_LABEL=autoheal \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/betterix/docker-autoheal:latest
```

Then add the label to any target container:

```yaml
services:
  web:
    image: nginx:alpine
    labels:
      autoheal: "true"
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost/ || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3
```

#### TCP / Remote Docker Daemon (with mTLS)

```bash
docker run -d \
  --name autoheal \
  --restart always \
  -e AUTOHEAL_CONTAINER_LABEL=all \
  -e DOCKER_SOCK=tcps://docker.lan:2376 \
  -v /path/to/certs:/certs:ro \
  ghcr.io/betterix/docker-autoheal:latest
```

> **Note**: For mTLS, place your certificates under `/certs` with names `ca.pem`, `client-cert.pem`, and `client-key.pem`.

---

## Configuration Reference

### Environment Variables

| Variable | Default | Description |
|---|---|---|
| `AUTOHEAL_CONTAINER_LABEL` | `autoheal` | Label name to watch (matches `label=true`). Set to `all` to watch all containers. |
| `AUTOHEAL_INTERVAL` | `5` | Polling interval in seconds between health status checks. |
| `AUTOHEAL_START_PERIOD` | `0` | Delay in seconds before running the first check loop on startup. |
| `AUTOHEAL_DEFAULT_STOP_TIMEOUT` | `10` | Timeout in seconds to allow a container to stop gracefully before SIGKILL. |
| `AUTOHEAL_ONLY_MONITOR_RUNNING` | `false` | When `true`, only checks containers in `running` state (ignores paused). |
| `DOCKER_SOCK` | `/var/run/docker.sock` | Path to Docker UNIX domain socket or TCP endpoint (`tcp://...`, `tcps://...`). |
| `CURL_TIMEOUT` | `30` | Timeout in seconds for individual Docker API HTTP requests. |
| `WEBHOOK_URL` | `""` | Optional HTTP URL to POST a JSON notification to when a container is restarted. |
| `WEBHOOK_JSON_KEY` | `text` | Key name for the message string in the JSON webhook payload. |
| `APPRISE_URL` | `""` | Optional Apprise API URL for push notifications. |
| `POST_RESTART_SCRIPT` | `""` | Path to an executable script to run after each restart attempt. |

### Container Labels

| Label | Example | Purpose |
|---|---|---|
| `<AUTOHEAL_CONTAINER_LABEL>=true` | `autoheal=true` | Marks container to be monitored when `AUTOHEAL_CONTAINER_LABEL` is not `all`. |
| `autoheal=False` | `autoheal=False` | Excludes container from being monitored even when watching `all`. |
| `autoheal.stop.timeout` | `autoheal.stop.timeout=30` | Overrides the stop timeout (in seconds) for that specific container. |

---

## Notifications & Hooks

### Webhook
When `WEBHOOK_URL` is set, a JSON POST request is sent on restart events:
```json
{
  "text": "Container web (a1b2c3d4e5f6) found to be unhealthy. Successfully restarted the container!"
}
```
You can customize the key (`text`) by setting `WEBHOOK_JSON_KEY=content` (e.g. for Discord webhooks).

### Apprise
When `APPRISE_URL` is set, Autoheal sends a formatted notification payload with `title` and `body`.

### Post-Restart Script
When `POST_RESTART_SCRIPT` is defined, the executable is called with positional arguments:
```bash
/path/to/script.sh "$CONTAINER_NAME" "$CONTAINER_SHORT_ID" "$CONTAINER_STATE" "$TIMEOUT"
```

---

## Local Development & Testing

```bash
# Lint shell scripts
docker run --rm -v "$PWD:/mnt" koalaman/shellcheck:stable /mnt/docker-entrypoint

# Build container image
docker build -t autoheal-local .

# Run locally
docker run -d \
  --name autoheal-local \
  -e AUTOHEAL_CONTAINER_LABEL=all \
  -v /var/run/docker.sock:/var/run/docker.sock \
  autoheal-local
```

For AI agents and developer architecture guidelines, see [AGENTS.md](AGENTS.md).

---

## License

MIT License — see [LICENSE](LICENSE) for details.
