# Grocy compose

This directory contains a hardened Docker Compose service definition for running **Grocy** — a self-hosted grocery & household management app.

The service uses **SQLite** (the default backend) with all state stored in `./data`. It attaches to an external `proxy` network so a separately defined reverse proxy (e.g. Caddy) can reach it without exposing ports on the host.

> **Note on `user:`** — this service omits the `user:` directive because the image uses [linuxserver s6-overlay](https://github.com/linuxserver/docker-baseimage-ubuntu), which must start as root to chown `/config` to `PUID:PGID` before dropping privileges. The effective runtime UID/GID is controlled via the `PUID`/`PGID` environment variables instead. The required capabilities (`CHOWN`, `DAC_OVERRIDE`, `SETGID`, `SETUID`) are added back explicitly and all others remain dropped.

## Layout

- `./data` → `/config` (SQLite database, grocy config, and app state)

## Usage

### 1. Create the proxy network (once, on the host)

The service attaches to an external Docker network named `proxy`. Create it before first run if it does not already exist:

```sh
docker network create proxy
```

If your reverse proxy network has a different name, override `networks:` in your consumer `compose.yml`.

### 2. Prepare storage

Create the host directory and set ownership to the container's effective user (`10020:10021`):

```sh
cd custom/path
mkdir -p data
chown -R 10020:10021 data
```

### 3. Basic usage example

```yaml
# custom/path/compose.yml
services:
  grocy:
    extends:
      file: path/to/secure-deployments/apps/grocy/compose/compose.yml
      service: grocy
    environment:
      TZ: Europe/Amsterdam
```

Point your reverse proxy at `http://grocy/` (using the service name as the hostname on the shared `proxy` network).

### Grocy application settings

Grocy exposes runtime configuration via environment variables. Common ones to set in your consumer compose:

| Variable | Default | Description |
|---|---|---|
| `GROCY_CURRENCY` | `USD` | Currency symbol shown in the UI |
| `GROCY_CULTURE` | `en` | Locale / language (`de`, `fr`, `nl`, …) |
| `GROCY_MODE` | `production` | Set to `demo` for a read-only demo instance |

See the [grocy configuration reference](https://github.com/grocy/grocy#configuration) for all available options.
