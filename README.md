# Secure deployments

A library of hardened service definitions for Docker Compose (and eventually Podman Quadlet). Use these as base configurations in your own deployments via the `extends` keyword — don't run them directly from this repo.

## Available apps

| App | Description |
|-----|-------------|
| [caddy](apps/caddy/compose/README.md) | Caddy reverse proxy / web server |
| jellyfin | Jellyfin media server |
| [redis](apps/redis/compose/README.md) | Redis in-memory data store |

## Usage

Each app under `apps/` is a hardened base service definition. The recommended pattern is to create your own `compose.yml` that extends the base:

```yaml
# your/compose.yml
services:
  caddy:
    extends:
      file: path/to/secure-deployments/apps/caddy/compose/compose.yml
      service: caddy
    environment:
      LETSENCRYPT_EMAIL: hello@example.com
    ports:
      - 80:8080/tcp
      - 443:8443/tcp
      - 443:8443/udp
```

See each app's `README.md` for mount requirements, environment variables, and full examples.

## Hardening

All service definitions apply the following by default:

- Drop all Linux capabilities (`cap_drop: [ALL]`)
- Read-only root filesystem (`read_only: true`)
- No privilege escalation (`no-new-privileges:true`)
- AppArmor default profile
- Non-root container user

## Updates

Image tags are kept up to date automatically via Dependabot (weekly).
