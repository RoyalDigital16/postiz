# Postiz - Local Deployment Credentials & Configuration

## Access

| Item | Value |
|------|-------|
| **URL** | http://localhost:4007 |
| **Temporal UI** | http://localhost:8088 |

## Account

| Field | Value |
|-------|-------|
| **Email** | `test@postiz-local.dev` |
| **Password** | `StrongP4ss!2024` |
| **Company** | `TestCorp` |

## Podman-Specific Fixes Applied

### 1. JWT_SECRET — replaced placeholder
Original was a plaintext string. Replaced with `openssl rand -hex 32` output.

### 2. SELinux volume mount — `:Z` flag
The `dynamicconfig` volume mount was blocked by SELinux (permission denied inside the container).
```yaml
# Before
- ./dynamicconfig:/etc/temporal/config/dynamicconfig
# After
- ./dynamicconfig:/etc/temporal/config/dynamicconfig:Z
```

### 3. Temporal admin-tools — removed tty/stdin_open
`tty: true` + `stdin_open: true` causes `podman-compose up -d` to hang.
Replaced with `command: ["sleep", "infinity"]`.

### 4. Temporal healthcheck — hostname gRPC resolution bug
The Temporal CLI cannot resolve its own container hostname via gRPC DNS (works with IP but not hostname inside the container).
```yaml
# Before
test: ["CMD", "temporal", "operator", "cluster", "health", "--address", "temporal:7233"]
# After
test: ["CMD-SHELL", "temporal operator cluster health --address $$(hostname -i):7233"]
```
Also added `BIND_ON_IP=0.0.0.0` to the Temporal environment.

### 5. Temporal UI port conflict
Port 8080 was already in use by `evolution-go`. Changed to 8088.
```yaml
# Before
- '8080:8080'
# After
- '8088:8080'
```

## Services

| Service | Status | Port |
|---------|--------|------|
| postiz (app) | healthy | 4007 |
| postiz-postgres | healthy | 5432 (internal) |
| postiz-redis | healthy | 6379 (internal) |
| temporal | healthy | 7233 |
| temporal-postgresql | healthy | 5432 (internal) |
| temporal-elasticsearch | healthy | 9200 (internal) |
| temporal-ui | healthy | 8088 |
| temporal-admin-tools | running | — |

## Commands

```bash
# Start the stack
cd /var/home/herold/docker/postiz
podman-compose up -d

# View logs
podman logs postiz
podman logs temporal

# Stop the stack
podman-compose down
```

## Notes

- Registration is enabled (`DISABLE_REGISTRATION=false`)
- Storage is local (`STORAGE_PROVIDER=local`)
- No social media providers configured (empty keys)
- No payment provider configured
- The ChunkLoadError on first load after signup is a race condition that resolves on page refresh

---

## Dokploy + Traefik (HTTPS) Deployment

A separate `docker-compose.dokploy.yml` is provided for Dokploy with automatic
HTTPS via Traefik + Let's Encrypt.

### Files

| File | Purpose |
|------|---------|
| `docker-compose.dokploy.yml` | Compose file for Dokploy (no `container_name`, uses `dokploy-network`, Traefik labels) |
| `.env.dokploy.example` | Template for the Dokploy Environment tab variables |

### Key differences from the Podman version

| Aspect | Podman (`docker-compose.yml`) | Dokploy (`docker-compose.dokploy.yml`) |
|--------|-------------------------------|----------------------------------------|
| Domain | `localhost:4007` | `https://${DOMAIN}` (configurable) |
| Container names | Present | Removed (Dokploy restriction) |
| Port exposure | `ports: 4007:5000` | `expose: 5000` (Traefik handles routing) |
| SSL/TLS | None | Let's Encrypt via Traefik `websecure` entrypoint |
| dynamicconfig | `./dynamicconfig:...:Z` (bind mount) | Named volume `temporal-dynamicconfig` |
| Temporal UI | Port 8088 on host | Exposed via Traefik on `temporal.${DOMAIN}` |
| Environment vars | Hardcoded in compose | `${VAR}` references, managed in Dokploy UI |

### Setup in Dokploy

1. **Create app**: New → Compose → Docker Compose
2. **Paste** `docker-compose.dokploy.yml` as the compose file
3. **Environment tab**: Set at minimum `DOMAIN` and `JWT_SECRET`
4. **Seed dynamicconfig** (one-time):
   ```bash
   docker run --rm -v postiz_temporal-dynamicconfig:/data alpine sh -c \
     "cat > /data/development-sql.yaml << 'EOF'
   limit.maxIDLength:
     - value: 255
       constraints: {}
   system.forceSearchAttributesCacheRefreshOnRead:
     - value: true
       constraints: {}
   EOF"
   ```
5. **DNS**: Create A records for `postiz.yourdomain.com` and `temporal.yourdomain.com`
6. **Deploy**

### Traefik labels (auto-applied)

```yaml
# Postiz app → https://postiz.example.com
- traefik.http.routers.postiz.rule=Host(`${DOMAIN}`)
- traefik.http.routers.postiz.entrypoints=websecure
- traefik.http.routers.postiz.tls.certResolver=letsencrypt
- traefik.http.services.postiz.loadbalancer.server.port=5000

# Temporal UI → https://temporal.postiz.example.com
- traefik.http.routers.postiz-temporal-ui.rule=Host(`${TEMPORAL_UI_DOMAIN}`)
- traefik.http.routers.postiz-temporal-ui.entrypoints=websecure
- traefik.http.routers.postiz-temporal-ui.tls.certResolver=letsencrypt
- traefik.http.services.postiz-temporal-ui.loadbalancer.server.port=8080
```
