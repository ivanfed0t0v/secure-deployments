# Caddy compose

This directory contains a hardened Docker Compose service definition for running **Caddy** as a reverse proxy / web server.

## Layout

This setup expects the following on the host (mounted into the container):

- `./conf` → `/etc/caddy` (must contain your `Caddyfile`)
- `./data` → `/data` (Caddy state: certificates, etc.)

## Usage

The recommended pattern is to keep this repository as a “library” and create your own `compose.yml` that
`extends` the provided `caddy` service. You typically only customize:

- `environment` (at least `LETSENCRYPT_EMAIL`, optionally `TZ`)
- networking (`ports:` mappings or `network_mode: host`)
- optional custom build (when you need extra Caddy modules via `xcaddy`)
- your `Caddyfile` in `./conf/Caddyfile`

The service expects you to provide a health endpoint in the Caddyfile at `http://localhost:8081/health`
(the examples below include this). If you change that path or port, also update the `healthcheck` in your
own compose override.

### Prepare storage

Create the host directories that will be mounted into the container. They must be writable by the
container user (`10000:10001`) because Caddy stores certificates and state in `/data`.

```sh
cd custom/path
mkdir -p conf data
chown -R 10000:10001 conf data
```

### Basic usage example

This example runs Caddy on the default Docker bridge network and publishes HTTP/HTTPS to the host with
`ports:` mappings. It also sets explicit internal ports (`8080`/`8443`) in the Caddyfile so the container
can safely run as non-root while still serving standard ports on the host.

1. Create `custom/path/conf/Caddyfile`:

```yaml
# custom/path/conf/Caddyfile
{
  admin off
  http_port 8080
  https_port 8443
  email {env.LETSENCRYPT_EMAIL}
}

http://localhost:8081 {
  respond /health "OK" 200
}

example.com {
  respond "Hello, world!"
}
```

2. Create `custom/path/compose.yml` extending the secure base service:

```yaml
# custom/path/compose.yml
services:
  caddy:
    extends:
      file: path/to/secure-deployments/apps/caddy/compose/compose.yml
      service: caddy
    environment:
      TZ: Europe/Amsterdam
      LETSENCRYPT_EMAIL: hello@example.com
    ports:
      - 80:8080/tcp
      - 443:8443/tcp
      - 443:8443/udp # HTTP/3 support
```

### Advanced usage example with custom Dockerfile

This example shows how to build a custom Caddy image that includes extra modules (here: the Cloudflare DNS
provider) using `xcaddy`. This is useful for DNS-01 challenges (e.g. wildcard certificates) and other
third-party integrations.

The container is run with `network_mode: host` in this example. When using host networking and binding to
privileged ports (80/443), add `NET_BIND_SERVICE` so a non-root Caddy process can bind to them.

1. Create `custom/path/conf/Caddyfile` (includes a Cloudflare DNS challenge):

```yaml
# custom/path/conf/Caddyfile
{
  admin off
  email {env.LETSENCRYPT_EMAIL}
}

http://localhost:8081 {
  respond /health "OK" 200
}

# Request wildcard cert with DNS challenge via Cloudflare module
*.example.com {
  tls {
    dns cloudflare {env.CF_TOKEN}
  }
}

site.example.com {
  respond "Hello, world!"
}
```

2. Create `custom/path/compose.yml` with an inline Dockerfile build:

```yaml
# custom/path/compose.yml
services:
  caddy:
    extends:
      file: path/to/secure-deployments/apps/caddy/compose/compose.yml
      service: caddy
    build:
      # Install cloudflare module for automatic DNS challenge
      dockerfile_inline: |
        FROM docker.io/library/caddy:2.11.1-builder-alpine AS builder
        RUN xcaddy build \
            --with github.com/caddy-dns/cloudflare

        FROM docker.io/library/caddy:2.11.1-alpine
        COPY --from=builder /usr/bin/caddy /usr/bin/caddy
    # This capability is required to use privileged ports with host networking
    cap_add:
      - NET_BIND_SERVICE
    environment:
      TZ: Europe/Amsterdam
      LETSENCRYPT_EMAIL: hello@example.com
      CF_TOKEN: supersecret
    image: mycaddy:latest
    network_mode: host
```
