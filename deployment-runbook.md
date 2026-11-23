# Deployment Runbook

One droplet (proxy + apps + shared DB). Pi for hobby apps.

## 0. Architecture
- `central-proxy` (Nginx) is the only container that publishes ports (80/443) and handles SSL.
- Every app and `shared-mariadb` joins the external Docker network `web`. No app uses `ports:`.
- Containers reach each other by `container_name` (e.g. `shared-mariadb:3306`). Same internal port on several apps is fine.
- Build on Mac/CI, push to GHCR, only pull on the droplet. Never build there.
- Folders: `~/proxy`, `~/shared-db`, `~/<app>`. Each has its own compose file and `.env`.

## A. One-time: droplet setup
## 1. Create droplet 
- Select Ubuntu 24.04
- Add SSH key 
- Note the IP
## 2. Non-root user (as root):
```bash
adduser deploy && usermod -aG sudo deploy
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy
```
## 3. Lock down SSH (test `ssh deploy@IP` in a second terminal first):
```bash
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/; s/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```
## 4. Firewall (never open 3306):
```bash
sudo ufw allow OpenSSH && sudo ufw allow 80/tcp && sudo ufw allow 443/tcp && sudo ufw enable
```
## 5. Swap (512MB box):
```bash
sudo fallocate -l 1G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
## 6. Docker + shared network (then log out and back in):
```bash
curl -fsSL [https://get.docker.com](https://get.docker.com) | sudo sh
sudo usermod -aG docker deploy
sudo systemctl enable docker
docker network create web
```
## 7. Monit (restarts Docker, alerts on disk/memory/site down):
```bash
sudo apt install -y monit
sudo tee /etc/monit/conf.d/server <<'EOF'
set daemon 60
check system $HOST
  if memoryusage > 85% for 3 cycles then alert
  if swap usage > 50% then alert
check filesystem rootfs with path /
  if space usage > 80% then alert
check process docker with pidfile /run/docker.pid
  start program = "/bin/systemctl start docker"
  stop program = "/bin/systemctl stop docker"
check host tsm with address tsm.laopioneer.com
  if failed port 443 protocol https for 2 cycles then alert
EOF
sudo monit -t && sudo systemctl enable --now monit && sudo monit reload
```
Add one `check host` block per new subdomain. Status: `sudo monit status`. Without a mail setup, alerts only go to `/var/log/monit.log`. For email add `set mailserver smtp.gmail.com port 587 username "u" password "p" using tlsv12` and `set alert you@email.com`.

## 8. Shared DB (copy `docker-compose.yml`; create `.env` with `DB_ROOT_PASSWORD`):
```bash
mkdir ~/shared-db && cd ~/shared-db
echo "DB_ROOT_PASSWORD=$(openssl rand -base64 24)" > .env && chmod 600 .env
docker compose up -d
```
## 9. Proxy (copy the whole folder, `conf.d` is mounted). From Mac:
```bash
scp -r proxy deploy@IP:~/proxy
```
On droplet:
```bash
cd ~/proxy && docker compose up -d
```
## 10. Cert auto-renewal (`crontab -e`):
```cron
0 3 * * * certbot renew --quiet --pre-hook "docker compose -f /home/deploy/proxy/docker-compose.yml stop nginx" --post-hook "docker compose -f /home/deploy/proxy/docker-compose.yml start nginx"
```

## B. One-time: GitHub and Mac
1. Create a classic PAT (fine-grained tokens don't support packages): scopes `write:packages`, `read:packages`. Note the expiry date.
2. Mac login:
```bash
echo $PAT | docker login ghcr.io -u <github-user> --password-stdin
```
3. Droplet login (only needed if packages are private): same command.
4. CI deploy key (dedicated, not your personal key):
```bash
ssh-keygen -t ed25519 -f deploy_key -C github-deploy
ssh-copy-id -i deploy_key.pub deploy@IP
```
5. Repo secrets: `DROPLET_HOST`, `DROPLET_USER`, `DROPLET_SSH_KEY` (contents of `deploy_key`).

## C. Add a new app
1. DNS (using porkbun service): A record `<sub>` pointing to the droplet IP. Check:
```bash
dig <sub>.laopioneer.com +short
```
2. Database (skip if no DB):
```bash
docker exec -it shared-mariadb mariadb -u root -p
```
```sql
CREATE DATABASE app_db;
CREATE USER 'app_user'@'%' IDENTIFIED BY '<openssl rand -base64 24>';
GRANT ALL PRIVILEGES ON app_db.* TO 'app_user'@'%';
FLUSH PRIVILEGES;
```
3. Build and push (from Mac, project folder):
```bash
docker buildx build --platform linux/amd64 \
  -t ghcr.io/<user>/<app>-backend:latest \
  -t ghcr.io/<user>/<app>-backend:$(git rev-parse --short HEAD) --push .
```
Use `linux/arm64` for the Pi. `VITE_*` values are baked in at build time, so changing them means rebuilding.

4. GitHub package settings: set visibility to Public (or link to repo).
5. Copy compose + `.env` (apps need only these two files):
```bash
ssh deploy@IP mkdir -p ~/<app>
scp docker-compose.yml .env deploy@IP:~/<app>/
ssh deploy@IP chmod 600 ~/<app>/.env
```
`.env` uses `shared-mariadb` as the DB host.

6. Certificate:
```bash
cd ~/proxy && docker compose stop nginx
sudo certbot certonly --standalone -d <sub>.laopioneer.com
docker compose start nginx
```
7. Proxy entry: add `~/proxy/conf.d/<app>.conf`:
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
8. Start app, then reload proxy:
```bash
cd ~/<app> && docker compose pull && docker compose up -d
docker exec central-proxy nginx -t && docker exec central-proxy nginx -s reload
```
9. First run only: `docker compose exec backend node prisma/seed.js`. Migrations run automatically via the compose command.
10. Add a `check host` block to Monit.

## D. Redeploy existing app
```bash
# Mac: build and push (see C3)
# Droplet:
df -h /                      # need > 1GB free
cd ~/<app> && docker compose pull && docker compose up -d
docker image prune -af --filter "until=72h"
```

## E. Verify (in this order)
```bash
docker compose ps                              # all Up
docker compose logs --tail=50 backend          # no errors
docker exec central-proxy nginx -t             # config ok
curl -I https://<sub>.laopioneer.com           # 200
curl -I https://<sub>[.laopioneer.com/api/health](https://.laopioneer.com/api/health)
```
Then test login in the browser.

## F. Rollback
Pin the previous SHA tag in the app's compose file, then:
```bash
docker compose up -d
```

## G. Gotchas
- `depends_on` does not work across compose projects (retry comes from `restart: unless-stopped`).
- `proxy_pass http://x:4000/;` (trailing slash) strips `/api`. Without the slash it keeps it.
- `MARIADB_*` env vars only apply on first init of an empty volume. Later changes need `ALTER USER`.
- Prisma 7: CLI reads `DATABASE_URL` (via `prisma.config.ts`), runtime adapter reads `DATABASE_HOST`/`USER`/`PASSWORD`/`NAME`.
- `env_file` in `backend/` is not read by Docker. Only the `.env` next to the compose file counts.
- Cert files must exist before Nginx loads a `listen 443 ssl` block, or the whole proxy fails to start.
- Pulls fail silently when the disk is full. Check `df -h` first.