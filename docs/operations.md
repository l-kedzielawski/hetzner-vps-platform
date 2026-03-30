# Operations Runbook

This document is a compact runbook for the documented stack. It is written for a small production VPS where fast recovery and clear procedures matter more than ceremony.

## Working Directories

The live host currently runs the stacks from paths like these:

- Traefik: `/srv/traefik`
- automation stack: `/srv/automation`
- ecommerce stack: `/srv/ecommerce`
- CRM stack: `/srv/crm`

The repository version in GitHub is a cleaned public template, so paths in this repo are relative and portable.

## Deploy or Update the Stack

Bring the documented stack up:

```bash
docker compose up -d
```

Rebuild services that use a local Dockerfile:

```bash
docker compose up -d --build
```

Pull newer image versions before restarting:

```bash
docker compose pull
docker compose up -d
```

## Check Running Services

Show container status:

```bash
docker ps
```

Show compose service status:

```bash
docker compose ps
```

Show Docker networks:

```bash
docker network ls
```

## Restart a Single Service

Restart Traefik:

```bash
docker compose restart traefik
```

Restart n8n only:

```bash
docker compose restart n8n
```

Restart PostgreSQL only:

```bash
docker compose restart postgres
```

## Inspect Logs

Traefik logs:

```bash
docker logs traefik
```

n8n logs:

```bash
docker logs n8n-app
```

Tail logs for a compose service:

```bash
docker compose logs -f n8n
```

## Validate Routing and TLS

Confirm Traefik is serving routers from labels:

```bash
docker logs traefik
```

Check an HTTPS endpoint:

```bash
curl -I https://automation.example.com
```

Inspect certificate details from the client side:

```bash
openssl s_client -connect automation.example.com:443 -servername automation.example.com
```

## Check ACME Storage

The live Traefik container stores ACME state in a mounted Let's Encrypt directory.

Expected path in the template:

```text
./traefik/letsencrypt/acme.json
```

On the live host the mounted directory followed the same pattern:

```text
/srv/traefik/letsencrypt
```

Make sure `acme.json` exists and is readable by Traefik only.

## Database Health Checks

The stack uses a PostgreSQL health check to avoid starting n8n before the database is ready.

Manual check inside the running container:

```bash
docker exec -it n8n-postgres pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"
```

## Backup Points

For this setup, the main backup targets are:

- PostgreSQL data volume
- n8n data volume
- Traefik ACME storage
- application source directories for locally built services

At minimum, preserve:

- workflow and credential state from `n8n_data`
- database contents from `postgres_data`
- certificate state from `acme.json`

## Rollback Approach

For a small VPS deployment, rollback should stay simple.

1. keep the previous working image or build artifact available
2. restore the last known-good compose file or env values
3. restart only the affected service first
4. restore volume-backed data only if the failure is data-related rather than application-related

If the issue is routing-related, check Traefik labels and network membership before rolling back application code.

## Common Failure Modes

### 1. Traefik Cannot Reach a Service

Check:

- service has `traefik.enable=true`
- service joins `traefik-public`
- `traefik.docker.network` matches the shared edge network
- router rule matches the intended hostname

### 2. n8n Starts but Behaves Incorrectly Behind the Proxy

The inspected live logs showed forwarded-header warnings. If that reappears, confirm proxy trust settings and public URL variables:

- `N8N_HOST`
- `N8N_PROTOCOL=https`
- `WEBHOOK_URL`
- `N8N_EDITOR_BASE_URL`
- `N8N_PROXY_HOPS=1`

### 3. ACME Certificate Issues

Check:

- Traefik is listening on `80/443`
- the hostname resolves to the VPS through Cloudflare
- the certificate resolver name matches the labels
- ACME storage is writable inside the container

## Operational Improvement Backlog

The live host is functional, but the next improvements are clear:

- migrate ACME to Cloudflare DNS challenge if fully proxied issuance becomes a requirement
- install and tune `fail2ban`
- add automated backups and restore testing
- add external monitoring and alerting
- pin image versions more aggressively where stability matters more than freshness
