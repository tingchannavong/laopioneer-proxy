# Deployment Runbook: New/Update App

## A. Add a new app
### 1. DNS (using porkbun service)
A record `<sub>` pointing to the droplet IP. Check:
```bash
dig <sub>.laopioneer.com +short
```
### 2. Database (skip if no DB):
```bash
docker exec -it shared-mariadb mariadb -u root -p
```
```sql
CREATE DATABASE app_db;
CREATE USER 'app_user'@'%' IDENTIFIED BY '<openssl rand -base64 24>';
GRANT ALL PRIVILEGES ON app_db.* TO 'app_user'@'%';
FLUSH PRIVILEGES;
```
### 3. Build and push (from Mac, project folder):
```bash
docker buildx build --platform linux/amd64 \
  -t ghcr.io/<user>/<app>-backend:latest \
  -t ghcr.io/<user>/<app>-backend:$(git rev-parse --short HEAD) --push .
```
Use `linux/arm64` for the Pi. `VITE_*` values are baked in at build time, so changing them means rebuilding.

### 4. GitHub package settings
Set visibility to Public (or link package to repo).

### 5. Copy compose + `.env`:
Apps need only these two files.
```bash
ssh deploy@IP mkdir -p ~/<app>
scp docker-compose.yml .env deploy@IP:~/<app>/
ssh deploy@IP chmod 600 ~/<app>/.env
```
`.env` uses `shared-mariadb` as the DB host.

### 6. Certificate:
```bash
cd ~/proxy && docker compose stop nginx
sudo certbot certonly --standalone -d <sub>.laopioneer.com
docker compose start nginx
```
### 7. Proxy entry: add `~/proxy/conf.d/<app>.conf`:
```nginx
server { listen 80; server_name <sub>.laopioneer.com; return 301 https://$host$request_uri; }
server {
  listen 443 ssl;
  server_name <sub>.laopioneer.com;
  ssl_certificate /etc/letsencrypt/live/<sub>[.laopioneer.com/fullchain.pem](https://.laopioneer.com/fullchain.pem);
  ssl_certificate_key /etc/letsencrypt/live/<sub>[.laopioneer.com/privkey.pem](https://.laopioneer.com/privkey.pem);
  location /api/ { proxy_pass http://<app>-backend:4000; proxy_set_header Host $host; proxy_set_header X-Forwarded-Proto $scheme; }
  location / { proxy_pass http://<app>-frontend:80; proxy_set_header Host $host; }
}
```
### 8. Start app, then reload proxy:
```bash
cd ~/<app> && docker compose pull && docker compose up -d
docker exec central-proxy nginx -t && docker exec central-proxy nginx -s reload
```
### 9. First run only: `docker compose exec backend node prisma/seed.js`. 
Migrations run automatically via the compose command.

### 10. Add a `check host` block to Monit.

## B. Redeploy existing app
```bash
# Mac: build and push (see A3)
# Droplet:
df -h /                      # need > 1GB free
cd ~/<app> && docker compose pull && docker compose up -d
docker image prune -af --filter "until=72h"
```

## C. Verify in this order
```bash
docker compose ps                              # all Up
docker compose logs --tail=50 backend          # no errors
docker exec central-proxy nginx -t             # config ok
curl -I https://<sub>.laopioneer.com           # 200
curl -I https://<sub>[.laopioneer.com/api/health](https://.laopioneer.com/api/health)
```
Then test login in the browser.

## D. Rollback
Pin the previous SHA tag in the app's compose file, then:
```bash
docker compose up -d
```

## E. Gotchas
- `depends_on` does not work across compose projects (retry comes from `restart: unless-stopped`).
- `proxy_pass http://x:4000/;` (trailing slash) strips `/api`. Without the slash it keeps it.
- `MARIADB_*` env vars only apply on first init of an empty volume. Later changes need `ALTER USER`.
- Prisma 7: CLI reads `DATABASE_URL` (via `prisma.config.ts`), runtime adapter reads `DATABASE_HOST`/`USER`/`PASSWORD`/`NAME`.
- `env_file` in `backend/` is not read by Docker. Only the `.env` next to the compose file counts.
- Cert files must exist before Nginx loads a `listen 443 ssl` block, or the whole proxy fails to start.
- Pulls fail silently when the disk is full. Check `df -h` first.