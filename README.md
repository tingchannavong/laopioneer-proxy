One droplet (proxy + apps + shared DB).

## Architecture
- `central-proxy` (Nginx) is the only container that publishes ports (80/443) and handles SSL.
- Every app and `shared-mariadb` joins the external Docker network `web`. No app uses `ports:`.
- Containers reach each other by `container_name` (e.g. `shared-mariadb:3306`). Same internal port on several apps is fine.
- Build on Mac/CI, push to GHCR, only pull on the droplet. Never build there.
- Folders: `~/proxy`, `~/shared-db`, `~/<app>`. Each has its own compose file and `.env`.
