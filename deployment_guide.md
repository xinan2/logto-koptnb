## Deployment guide (forked Logto with numeric username support)

This guide shows how to run your forked Logto build (with numeric usernames allowed) in local development and in production with HTTPS.

### Prerequisites

- Docker (Docker Desktop on Windows/macOS; Docker Engine on Linux)
- For Windows: WSL2 integration enabled for Docker Desktop
- Optional for production: domain names (e.g., `auth.example.com`, `admin.example.com`) pointing to your server’s public IP (A records), and ports 80/443 open

### Repository notes

- This fork already includes the username rule change and a Dockerfile fix to normalize `*.sh` line endings (avoids CRLF build errors on Windows).

---

## 1) Local development (uses your local forked code)

From your repository root:

```bash
docker compose -f docker-compose.dev.yml up -d --build
```

Access:

- App (Experience): http://localhost:3001
- Admin Console: http://localhost:3002

Stop:

```bash
docker compose -f docker-compose.dev.yml down
```

---

## 2) Production with automatic HTTPS (Caddy in Docker)

Recommended if you want a single Docker stack that handles TLS automatically via Let’s Encrypt.

### 2.1 Create environment file

Create `.env.production` in your repo root:

```env
# Database
POSTGRES_USER=logto
POSTGRES_PASSWORD=change_this_strong_password
POSTGRES_DB=logto
DB_URL=postgres://logto:change_this_strong_password@postgres:5432/logto

# Application
PORT=3001
ADMIN_PORT=3002
NODE_ENV=production
TRUST_PROXY_HEADER=1

# Domains
ENDPOINT=https://auth.example.com
ADMIN_ENDPOINT=https://admin.example.com

# Optional: cookie keys (comma-separated random strings)
OIDC_COOKIE_KEYS=your_random_key_1,your_random_key_2
```

### 2.2 Create Caddy reverse proxy config

Add a `Caddyfile` in the repo root:

```
auth.example.com {
  reverse_proxy app:3001
  header -Server
}

admin.example.com {
  reverse_proxy app:3002
  header -Server
}
```

### 2.3 Create production compose for Caddy

Create `docker-compose.caddy.yml`:

```yaml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-p0stgr3s}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      TRUST_PROXY_HEADER: 1
      DB_URL: ${DB_URL}
      ENDPOINT: ${ENDPOINT}
      ADMIN_ENDPOINT: ${ADMIN_ENDPOINT}
    depends_on:
      postgres:
        condition: service_healthy
    command: ["sh", "-c", "npm run cli db seed -- --swe && npm start"]

  caddy:
    image: caddy:2
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - app

volumes:
  postgres_data:
  caddy_data:
  caddy_config:
```

### 2.4 Deploy

```bash
docker compose -f docker-compose.caddy.yml --env-file .env.production up -d --build
```

Once up, Caddy obtains and renews TLS certificates automatically. Visit your domains over HTTPS.

---

## 3) Production with Nginx + Certbot (host-managed HTTPS)

Use this if you prefer to manage TLS on the host with Nginx.

### 3.1 Create a production compose (no public TLS in the stack)

Create `docker-compose.prod.yml`:

```yaml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      DB_URL: ${DB_URL}
      ENDPOINT: ${ENDPOINT}
      ADMIN_ENDPOINT: ${ADMIN_ENDPOINT}
      PORT: ${PORT}
      ADMIN_PORT: ${ADMIN_PORT}
      NODE_ENV: ${NODE_ENV}
      TRUST_PROXY_HEADER: ${TRUST_PROXY_HEADER}
    depends_on:
      postgres:
        condition: service_healthy
    command: ["sh", "-c", "npm run cli db seed -- --swe && npm start"]

volumes:
  postgres_data:
```

Run:

```bash
docker compose -f docker-compose.prod.yml --env-file .env.production up -d --build
```

### 3.2 Configure Nginx on the host

Create two server blocks:

- `auth.example.com` → proxy to `http://127.0.0.1:3001`
- `admin.example.com` → proxy to `http://127.0.0.1:3002`

Issue certificates with Certbot:

```bash
sudo certbot --nginx -d auth.example.com -d admin.example.com
```

---

## 4) Verifying numeric usernames

1) Open Admin Console and create a user with username `123456` → should succeed.
2) Test sign-up with a numeric username in the user-facing app.

---

## 5) Updates & maintenance

Pull code and rebuild:

```bash
git pull
docker compose -f <your-compose>.yml build
docker compose -f <your-compose>.yml up -d
```

View logs:

```bash
docker compose -f <your-compose>.yml logs -f
docker logs <container-name> -f
```

Backup database (example, Postgres in Docker):

```bash
docker exec <postgres-container> pg_dump -U <db_user> <db_name> > backup_$(date +%Y%m%d).sql
```

---

## 6) Troubleshooting

- Force a clean rebuild:

```bash
docker compose -f <compose>.yml down
docker system prune -a
docker compose -f <compose>.yml up -d --build --no-cache
```

- If builds fail with shell script syntax errors on Windows, ensure the Dockerfile step normalizes CRLF → LF for `*.sh` (already included in this repo’s Dockerfile).

---

## 7) References

- Upstream project (for general usage): `https://github.com/logto-io/logto`


