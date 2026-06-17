# Postiz on Dokploy — Deployment Guide

> Deploy Postiz with automatic HTTPS via Traefik + Let's Encrypt on Dokploy.

---

## Prerequisites

- A **Dokploy** instance running with Traefik configured
- A **domain name** with DNS A records pointing to your Dokploy server:
  - `postiz.yourdomain.com` → your server IP
  - `temporal.yourdomain.com` → your server IP (optional, for Temporal UI)
- **2 GB RAM** minimum (4 GB recommended with Temporal stack)

---

## Step 1 — Create the Compose App

1. In Dokploy, navigate to your project → **Create App** → **Compose**
2. Choose **Docker Compose** as the compose type
3. Paste the contents of [`docker-compose.dokploy.yml`](docker-compose.dokploy.yml) into the editor
4. Click **Save**

![Compose editor](https://docs.dokploy.com/assets/images/compose/overview.png)

---

## Step 2 — Configure Environment Variables

Go to the **Environment** tab and add at minimum these two variables:

```env
DOMAIN=postiz.yourdomain.com
JWT_SECRET=<your-generated-secret>
```

To generate a secure `JWT_SECRET`:

```bash
openssl rand -hex 32
```

### Optional variables

| Variable | Description |
|----------|-------------|
| `TEMPORAL_UI_DOMAIN` | Subdomain for Temporal UI (defaults to `temporal.${DOMAIN}`) |
| `OPENAI_API_KEY` | OpenAI key for AI features |
| `X_API_KEY` / `X_API_SECRET` | X (Twitter) integration |
| `FACEBOOK_APP_ID` / `FACEBOOK_APP_SECRET` | Facebook / Instagram |
| `LINKEDIN_CLIENT_ID` / `LINKEDIN_CLIENT_SECRET` | LinkedIn |
| `STRIPE_SECRET_KEY` | Stripe payments |

See [`.env.dokploy.example`](.env.dokploy.example) for the full list.

> **Note**: Dokploy writes these variables to a `.env` file. The compose file
> references them with `${VAR}` syntax. No `env_file` directive is needed.

---

## Step 3 — Deploy (no manual seed needed)

The `temporal-dynamicconfig-init` service runs once at startup and automatically
creates the required Temporal configuration file in the named volume. No manual
intervention is needed.

> **How it works**: An Alpine init container checks if `/data/development-sql.yaml`
> exists. If not, it writes the file. The `temporal` service waits for this init
> container to complete successfully before starting. The config persists in the
> named volume across restarts.

---

## Step 4 — Deploy

1. Click **Deploy**
2. Wait for the build to complete (2–4 minutes on first run)
3. Dokploy will show the deployment progress in the **Deployments** tab

### What happens during deployment

1. Dokploy pulls all images (Postiz, PostgreSQL, Redis, Elasticsearch, Temporal)
2. Containers start in dependency order (DBs → Temporal → Postiz)
3. Traefik detects the labels and requests Let's Encrypt certificates
4. Postiz runs its database migrations automatically

---

## Step 5 — Verify

### Check service health

All 8 services should show **healthy**:

| Service | Role | Internal Port |
|---------|------|---------------|
| `postiz` | Web application | 5000 |
| `postiz-postgres` | App database | 5432 |
| `postiz-redis` | Queue / cache | 6379 |
| `temporal` | Workflow engine | 7233 |
| `temporal-postgresql` | Temporal database | 5432 |
| `temporal-elasticsearch` | Temporal visibility | 9200 |
| `temporal-ui` | Temporal dashboard | 8080 |
| `temporal-admin-tools` | CLI utilities | — |

### Access the app

- **Postiz**: `https://postiz.yourdomain.com`
- **Temporal UI**: `https://temporal.yourdomain.com`

> Certificates may take 30–60 seconds after the first deploy. If you see a
> certificate error, wait a minute and reload.

---

## Step 6 — Create Your Account

1. Open `https://postiz.yourdomain.com` — you'll see the signup page
2. Fill in your email, password, and company name
3. Click **Create Account**
4. You'll land on the Calendar dashboard

> Registration is enabled by default. To lock it down after creating your
> account, set `DISABLE_REGISTRATION=true` in the Environment tab and redeploy.

---

## Post-Deployment

### Enable Volume Backups (recommended)

Go to **Volume Backups** to configure automated S3 backups for your named volumes:

- `postgres-volume` — Postiz database
- `postiz-uploads` — Uploaded media
- `temporal-postgres-data` — Temporal database

### Configure Social Providers

Add the relevant API keys in the Environment tab and redeploy. See the
[Postiz provider documentation](https://docs.postiz.com/providers/overview) for
setup instructions per platform.

### Disable Registration (production)

Once your account is created, set this in the Environment tab:

```env
DISABLE_REGISTRATION=true
```

This prevents new signups while keeping your existing account active.

---

## Troubleshooting

### Temporal container exits or stays unhealthy

**Symptom**: `temporal` container shows `Exited (1)` or `unhealthy`.

**Cause**: Missing dynamic config file.

**Fix**: Run the seed command from [Step 3](#step-3--seed-the-dynamic-config-volume-one-time), then redeploy.

---

### Postiz container stays "starting"

**Symptom**: `postiz` never reaches `healthy`, healthcheck fails.

**Fix**: Check the logs in Dokploy's **Logs** tab. Common causes:

- **Database connection**: Verify `postiz-postgres` is healthy first
- **Temporal connection**: Verify `temporal` is healthy first
- **Missing JWT_SECRET**: Ensure `JWT_SECRET` is set in Environment

---

### 502 Bad Gateway on first visit

**Symptom**: Browser shows 502.

**Fix**: The frontend SSR takes ~60s to compile on first start. Wait and reload.
If it persists, check the `postiz` container logs for errors.

---

### Certificate not issued

**Symptom**: Browser shows "Connection not secure" or certificate error.

**Fix**:
1. Verify DNS A records point to your Dokploy server
2. Check that Traefik can reach Let's Encrypt (port 80 must be open)
3. Wait 60 seconds and try again — first issuance can be slow

---

### "ChunkLoadError" after signup

**Symptom**: After creating an account, the page shows a chunk load error.

**Fix**: This is a race condition during SSR compilation. Reload the page
(`F5`). The error does not recur on subsequent loads.

---

## Architecture

```
                         ┌──────────────────────────────────┐
                         │         Dokploy Server           │
                         │                                  │
  HTTPS (443) ──────────▶│  Traefik (dokploy-network)       │
                         │  ├─ postiz.yourdomain.com ──┐    │
                         │  └─ temporal.yourdomain.com─┐    │
                         │                             │    │
                         │  ┌──────────────────────────┼────┤
                         │  │  postiz (port 5000)       │    │
                         │  │  ├─ postgres (5432)       │    │
                         │  │  ├─ redis (6379)          │    │
                         │  │  └─ temporal (7233) ──────┤    │
                         │  │     ├─ postgresql          │    │
                         │  │     ├─ elasticsearch       │    │
                         │  │     ├─ admin-tools         │    │
                         │  │     └─ temporal-ui :8080 ◀─┘    │
                         │  └─────────────────────────────────┤
                         └────────────────────────────────────┘
```

---

## Reference

- [Postiz Documentation](https://docs.postiz.com)
- [Postiz Docker Compose repo](https://github.com/gitroomhq/postiz-docker-compose)
- [Dokploy Documentation](https://docs.dokploy.com)
- [Dokploy Docker Compose guide](https://docs.dokploy.com/docs/core/docker-compose)
- [Dokploy Domains guide](https://docs.dokploy.com/docs/core/domains)
