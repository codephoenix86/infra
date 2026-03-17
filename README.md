# infra

Shared infrastructure configuration used across projects. Each subdirectory is a self-contained service that can be composed into any project's deployment.

---

## Structure

```
infra/
├── compose.yml
├── nginx/
│   └── nginx.conf        # Nginx configuration (reverse proxy, HTTPS, WebSocket support)
└── certbot/
    ├── certs/            # Let's Encrypt certificates (managed by Certbot)
    └── www/              # ACME challenge webroot
```

---

## Services

### `nginx`

A lightweight reverse proxy based on `nginx:alpine`. It handles HTTP-to-HTTPS redirection, TLS termination, and proxies traffic to an upstream service.

**Key behaviours:**

- Listens on port `80` — redirects all traffic to HTTPS and serves ACME challenge files for certificate renewal
- Listens on port `443 ssl` — terminates TLS and proxies requests to the upstream service
- Proxies all traffic to the upstream at `http://fastchat:3000`
- Forwards standard headers (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`)
- Supports WebSocket upgrades via the `Upgrade` / `Connection` header map
- Long-lived connections supported — read/send timeouts set to 24 hours

**Volumes:**

| Host Path | Container Path | Purpose |
|---|---|---|
| `./nginx/nginx.conf` | `/etc/nginx/nginx.conf` | Nginx config (read-only) |
| `./certbot/certs` | `/etc/letsencrypt` | TLS certificates managed by Certbot |
| `./certbot/www` | `/var/www/certbot` | ACME HTTP-01 challenge webroot |

---

### `certbot`

A one-shot container based on `certbot/certbot` used to obtain and renew TLS certificates from Let's Encrypt.

**Volumes:**

| Host Path | Container Path | Purpose |
|---|---|---|
| `./certbot/certs` | `/etc/letsencrypt` | Persists issued certificates |
| `./certbot/www` | `/var/www/certbot` | Shared webroot for ACME challenges |

> Certbot and Nginx share the same volume mounts so that Nginx can serve the ACME challenge and Certbot can write the resulting certificates in one flow.

---

## Networks

Both services attach to two external Docker networks:

| Network | Purpose |
|---|---|
| `shared-gateway` | Common ingress network shared across projects |
| `fastchat` | Project-specific backend network for upstream services |

Both networks must exist before starting the containers (created externally via `docker network create`).

---

## Getting Started

### 1. Create required networks

```bash
docker network create shared-gateway
docker network create fastchat
```

### 2. Obtain a TLS certificate (first time only)

Ensure your domain's DNS is pointing to this host, then run:

```bash
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path /var/www/certbot \
  -d fastchat.duckdns.org \
  --email your@email.com \
  --agree-tos \
  --no-eff-email
```

> Nginx must be running to serve the ACME challenge. Start it first with `docker compose up -d nginx`.

### 3. Start all services

```bash
docker compose up -d
```

### 4. Stop all services

```bash
docker compose down
```

---

## Certificate Renewal

Certbot certificates expire after 90 days. To renew:

```bash
docker compose run --rm certbot renew
docker compose exec nginx nginx -s reload
```

Consider scheduling this in a cron job:

```cron
0 3 * * * cd /path/to/infra && docker compose run --rm certbot renew && docker compose exec nginx nginx -s reload
```

---

## Adapting for a New Project

1. Update `server_name` in `nginx/nginx.conf` to your domain.
2. Update the `proxy_pass` URL to point to your upstream service and port.
3. Update the SSL certificate paths in the `443 ssl` server block to match your domain.
4. Update the network names in `compose.yml` to match your project's Docker networks.
5. Re-issue a certificate for your new domain via the Certbot command above.

---

## Prerequisites

- Docker Engine with Compose plugin (`docker compose` v2+)
- External networks created before starting services
- Domain DNS record pointing to this host (required for Let's Encrypt certificate issuance)