# Deployment Runbook: One-time Setup Things

## A. One-time: droplet setup
### 1. Create droplet 
- Select Ubuntu 24.04
- Add SSH key 
- Note the IP
### 2. Non-root user (as root):
```bash
adduser deploy && usermod -aG sudo deploy
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy
```
### 3. Lock down SSH (test `ssh deploy@IP` in a second terminal first):
```bash
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/; s/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```
### 4. Firewall (never open 3306):
```bash
sudo ufw allow OpenSSH && sudo ufw allow 80/tcp && sudo ufw allow 443/tcp && sudo ufw enable
```
### 5. Swap (512MB box):
```bash
sudo fallocate -l 1G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
### 6. Docker + shared network (then log out and back in):
```bash
curl -fsSL [https://get.docker.com](https://get.docker.com) | sudo sh
sudo usermod -aG docker deploy
sudo systemctl enable docker
docker network create web
```
### 7. Monit (restarts Docker, alerts on disk/memory/site down):
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

### 8. Shared DB (copy `docker-compose.yml`; create `.env` with `DB_ROOT_PASSWORD`):
```bash
mkdir ~/shared-db && cd ~/shared-db
echo "DB_ROOT_PASSWORD=$(openssl rand -base64 24)" > .env && chmod 600 .env
docker compose up -d
```
### 9. Proxy (copy the whole folder, `conf.d` is mounted). From Mac:
```bash
scp -r proxy deploy@IP:~/proxy
```
On droplet:
```bash
cd ~/proxy && docker compose up -d
```
### 10. Cert auto-renewal (`crontab -e`):
```cron
0 3 * * * certbot renew --quiet --pre-hook "docker compose -f /home/deploy/proxy/docker-compose.yml stop nginx" --post-hook "docker compose -f /home/deploy/proxy/docker-compose.yml start nginx"
```

## B. One-time: GitHub and Mac
### 1. Create a classic PAT 
With scopes `write:packages`, `read:packages`. Note the expiry date.
### 2. Mac login:
```bash
echo $PAT | docker login ghcr.io -u <github-user> --password-stdin
```
### 3. Droplet login (only needed if packages are private): same command.
### 4. CI deploy key (dedicated, not your personal key):
```bash
ssh-keygen -t ed25519 -f deploy_key -C github-deploy
ssh-copy-id -i deploy_key.pub deploy@IP
```
### 5. Repo secrets: `DROPLET_HOST`, `DROPLET_USER`, `DROPLET_SSH_KEY` (contents of `deploy_key`).