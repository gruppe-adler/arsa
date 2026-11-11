# ARSA – Deployment & Setup Guide

ARSA is a self-hosted admin stack made up of two services — a **frontend** SPA and a **backend** API — that lets authenticated users provision and manage `ars` (Arma Reforger dedicated server) container instances through Docker. Login and role-based access control are delegated to an external **Keycloak** realm via OpenID Connect (OIDC).

This guide documents the `docker-compose.yml` / `.env` pair and what needs to be configured on the Keycloak side for it to work.

---

## 1. Architecture

```
                     ┌──────────────────────────────────┐
   Browser  ───────▶ │  reverse proxy (nginx-proxy net) │
                     └───────┬───────────────────┬──────┘
                             │                   │
             ┌───────────────▼──────┐   ┌────────▼─────────────────────┐
             │    arsa-frontend      │   │        arsa-backend           │
             │ static SPA, port 80   │   │  REST/WebSocket API, port 3000│
             │ user www-data         │   │  OIDC login/callback          │
             └──────────────────────┘   │  orchestrates `ars` containers│
                                        │  user 1000:1000               │
                                        └──┬──────────┬──────────┬──────┘
                                           │          │          │
                                 /app/db   │          │          │  /var/run/docker.sock
                             (arsa-db-volume)         │          │  (host Docker engine)
                                           │          │
                              /app/ars/servers     /app/ars/repo
                            (arsa-servers-volume) (arsa-repo-volume)
                                           │          │
                                           ▼          ▼   same volumes, mounted by the backend
                             ┌─────────────────────────────────────┐
                             │  ars containers (one per server)     │
                             │  /ars/arsa/servers, /ars/arsa/repo   │
                             │  user 1000:1000, no capabilities     │
                             └─────────────────────────────────────┘

   External:  Keycloak realm (ARSA_OIDC_ISSUER) — not part of this stack
```

- **frontend** — serves the UI; its configuration (API URL, etc.) is compiled into the build, so config changes to the frontend image typically require a rebuild rather than just an env change.
- **backend** — the only service with credentials, DB access, and (critically) a mount of the host's `/var/run/docker.sock`, which it uses to start/stop/manage the individual `ars` server containers defined by `ARSA_ARS_CONTAINER`. It mounts the servers and repo volumes into those containers by name (`ARSA_SERVER_VOLUME`, `ARSA_REPO_VOLUME`).
- **Keycloak** — an existing realm you provide; ARSA does not run its own identity provider.

All containers run as unprivileged users, see [section 8](#8-users--permissions).

---

## 2. Prerequisites

- Docker Engine + Docker Compose v2
- The external Docker network `nginx-proxy` with a reverse proxy attached to it that routes to `arsa-frontend:80` and `arsa-backend:3000` (create the network with `docker network create nginx-proxy` if it doesn't exist)
- A reachable Keycloak instance with a realm already created (here: `grad` at `auth.gruppe-adler.de`)
- Network access from the **backend container** to the Keycloak issuer URL and the Arma Reforger workshop
- Network access from the **user's browser** to both the frontend and backend origins

---

## 3. Repository layout

```
.
├── docker-compose.yml
└── .env
```

### `docker-compose.yml`

```yaml
name: arsa

services:
  frontend:
    hostname: 'arsa-frontend'
    image: ghcr.io/gruppe-adler/arsa-frontend:main
    restart: unless-stopped
    depends_on:
      backend:
        condition: service_started
    networks:
      - arsa
      - nginx

  backend:
    hostname: 'arsa-backend'
    image: docker.io/library/arsa-backend:latest
    restart: unless-stopped
    env_file:
      - .env
    # The backend runs as 1000:1000 and needs the group of the docker socket (DOCKER_GID in .env)
    group_add:
      - '${DOCKER_GID:?set DOCKER_GID in .env to the group id of /var/run/docker.sock}'
    networks:
      - arsa
      - nginx
    volumes:
      - arsa-db-volume:/app/db
      - arsa-servers-volume:/app/ars/servers
      - arsa-repo-volume:/app/ars/repo
      - /var/run/docker.sock:/var/run/docker.sock


volumes:
  arsa-db-volume:
    name: arsa-db-volume
  arsa-servers-volume:
    name: arsa-servers-volume
  arsa-repo-volume:
    name: arsa-repo-volume

networks:
  arsa:
    internal: true
  nginx:
    name: nginx-proxy
    external: true
```

> **Networking:** neither service publishes a port. They are reached through the reverse proxy on the external `nginx-proxy` network. The `arsa` network is `internal: true`, which blocks all outbound traffic on it, so the backend's outbound access to Keycloak and the workshop also goes through `nginx-proxy`; that network must therefore not be internal. The game server containers are not on either network: the backend starts them on the default `bridge` network and publishes their UDP ports on the host.

> **Volume paths:** the backend image declares `/app/db`, `/app/ars/servers` and `/app/ars/repo` as volumes, and with `ARSA_BASE_PATH` unset it reads and writes the server files under `/app/ars`. Mounting the volumes anywhere else leaves anonymous volumes behind.

### `.env`

Example values for a local test with the frontend on `localhost:5173` and the backend on `localhost:3000`. For a deployment behind the proxy, use the public URLs instead.

```dotenv
ARSA_ALLOWED_ORIGINS=http://localhost:5173
ARSA_ACCESS_ROLE=adler
ARSA_ADMIN_ROLE=arsa-admin
ARSA_BIND_ADDRESS=0.0.0.0:3000
ARSA_USE_VOLUME=true
ARSA_SERVER_VOLUME=arsa-servers-volume
ARSA_REPO_VOLUME=arsa-repo-volume
ARSA_OIDC_CLIENT_ID=arsa-dev
ARSA_OIDC_CLIENT_SECRET=<your-keycloak-client-secret>
ARSA_OIDC_ISSUER=https://auth.gruppe-adler.de/realms/grad
ARSA_OIDC_CALLBACK_URI=http://localhost:3000/api/v2/auth/callback
ARSA_OIDC_FRONTEND_URI=http://localhost:5173/
ARSA_ARS_CONTAINER=ghcr.io/gruppe-adler/ars-image
ARSA_IP_ADDR=127.0.0.1
# group id of /var/run/docker.sock on the host: stat -c %g /var/run/docker.sock
DOCKER_GID=<docker-socket-gid>
```

> Never commit a real `ARSA_OIDC_CLIENT_SECRET` to version control. Treat `.env` as a secret file (add it to `.gitignore`) and rotate the secret if it's ever exposed.

---

## 4. Environment variable reference

| Variable                  | Purpose                                                                                                                                                                                                                                                         |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ARSA_ALLOWED_ORIGINS`    | CORS allow-list for the backend API. Separate multiple origins with `;` if you serve the frontend from more than one URL.                                                                                                                                       |
| `ARSA_ACCESS_ROLE`        | Keycloak role required (present in the ID token) for a user to access ARSA at all.                                                                                                                                                                              |
| `ARSA_ADMIN_ROLE`         | Keycloak role required for admin-level actions inside ARSA.                                                                                                                                                                                                     |
| `ARSA_BIND_ADDRESS`       | Host:port the backend process listens on **inside its container**.                                                                                                                                                                                              |
| `ARSA_USE_VOLUME`         | `true` → the backend mounts the named Docker volumes `ARSA_SERVER_VOLUME` and `ARSA_REPO_VOLUME` into the `ars` containers. Use this with the compose file. `false` → it bind mounts its own `ARSA_BASE_PATH` directories instead, which only works when the backend runs directly on the host (local development). |
| `ARSA_SERVER_VOLUME`      | Name of the Docker volume used for `ars` server data, when `ARSA_USE_VOLUME=true`. Must match the volume mounted at `/app/ars/servers`.                                                                                                                          |
| `ARSA_REPO_VOLUME`        | Name of the Docker volume used for the mod/asset repository shared with `ars` containers, when `ARSA_USE_VOLUME=true`. Must match the volume mounted at `/app/ars/repo`.                                                                                       |
| `ARSA_OIDC_CLIENT_ID`     | Client ID of the confidential Keycloak client ARSA authenticates as.                                                                                                                                                                                            |
| `ARSA_OIDC_CLIENT_SECRET` | Secret for that client. Keep private.                                                                                                                                                                                                                           |
| `ARSA_OIDC_ISSUER`        | Full issuer URL of the Keycloak realm, e.g. `https://<host>/realms/<realm>`. Used for OIDC discovery.                                                                                                                                                           |
| `ARSA_OIDC_CALLBACK_URI`  | The backend URL Keycloak redirects to after login. Must exactly match a "Valid redirect URI" on the client.                                                                                                                                                     |
| `ARSA_OIDC_FRONTEND_URI`  | Where the backend sends the browser back to once login/session handling is done.                                                                                                                                                                                |
| `ARSA_ARS_CONTAINER`      | Image reference the backend uses when it spins up individual dedicated-server containers.                                                                                                                                                                       |
| `ARSA_ARS_USER`           | Optional, default `1000:1000`. `uid:gid` the `ars` containers run as. Must own the servers and repo volumes, i.e. be the same user as the backend.                                                                                                              |
| `ARSA_IP_ADDR`            | Public IP address of the host, shown to users so they can connect to the game servers. Defaults to the detected local IP. The server ports are always published on all host interfaces.                                                                        |
| `ARSA_BASE_PATH`          | Optional, default `./ars` (`/app/ars` in the container). Directory the backend keeps server files (`servers/`) and mods (`repo/`) in. Leave it unset with the compose file.                                                                                     |
| `DOCKER_GID`              | Read by `docker-compose.yml`, not the backend: the group id of `/var/run/docker.sock` on the host, added to the backend's groups so it can use the Docker API.                                                                                                  |

---

## 5. Keycloak client configuration

Create (or reuse) a client in the target realm with these settings:

1. **Client authentication:** ON (confidential client, since ARSA uses a client secret).
2. **Root URL / Home URL:** the frontend URL (e.g. `http://localhost:5173/`).
3. **Valid redirect URIs:** the backend URL(s), e.g. `http://localhost:3000/*` (must cover `ARSA_OIDC_CALLBACK_URI`).
4. **Valid post logout redirect URIs:** the backend URL, same as above.
5. **Web origins:** the backend URL (e.g. `http://localhost:3000`), so CORS preflight from the backend's own callback works.
6. **Credentials tab:** copy the generated client secret into `ARSA_OIDC_CLIENT_SECRET`.
7. **Roles:** create realm (or client) roles matching `ARSA_ACCESS_ROLE` (`adler`) and `ARSA_ADMIN_ROLE` (`arsa-admin`), and assign them to the appropriate users/groups.
8. **Client scopes → mappers:** add a mapper (e.g. "User Realm Role" or "User Client Role") with **Add to ID token** enabled, so the roles above actually appear as claims in the ID token ARSA receives — without this step, role checks on the backend will fail even if the roles are assigned to the user.

---

## 6. Deploying

```bash
# 1. Place docker-compose.yml and .env next to each other
# 2. Fill in ARSA_OIDC_CLIENT_SECRET, DOCKER_GID and any other environment specific values
# 3. Start the stack
docker compose up -d

# Check status / logs
docker compose ps
docker compose logs -f backend
```

Once the reverse proxy routes to both services, the frontend is reachable at `ARSA_OIDC_FRONTEND_URI`, and logging in will route through Keycloak and back via `ARSA_OIDC_CALLBACK_URI`.

When upgrading from a version where the containers ran as root, follow [section 8](#upgrading-from-root-containers) first.

---

## 7. Data & volumes

| Volume / Path                               | Contents                                                                                                                   |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `arsa-db-volume` → `/app/db`                | Backend's persistent application database.                                                                                 |
| `arsa-servers-volume` → `/app/ars/servers`  | Config, profile, logs and saves of the individual managed `ars` server instances. Mounted into them at `/ars/arsa/servers`. |
| `arsa-repo-volume` → `/app/ars/repo`        | Shared repository of mods downloaded by the instances. Mounted into them at `/ars/arsa/repo`.                              |
| `/var/run/docker.sock`                      | **Not user data** — gives the backend direct control of the host's Docker engine so it can create/manage `ars` containers. |

All files in the three volumes are owned by `1000:1000`.

---

## 8. Users & permissions

| Container      | User        | Notes                                                                                                                                   |
| -------------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| frontend       | `www-data`  | Listens on port 80; Docker allows unprivileged processes to bind low ports inside a container.                                         |
| backend        | `1000:1000` | Gets the docker socket's group via `group_add` (`DOCKER_GID`).                                                                          |
| `ars` servers  | `1000:1000` | Set by the backend (`ARSA_ARS_USER`), together with dropped capabilities and `no-new-privileges`, regardless of the image's own user. |

The backend creates the server configs and profile directories, and the servers write their logs, saves and mods into the same volumes, so both run as the same user.

### Upgrading from root containers

Volumes written by containers that ran as root contain files owned by root, which the new containers can't write. Before starting the new version:

1. Stop all game servers in ARSA. Running servers keep their old container and would keep writing files as root; stopped servers get a new container on their next start.
2. Stop the stack: `docker compose down`.
3. Check that `docker volume ls` shows `arsa-db-volume`, `arsa-servers-volume` and `arsa-repo-volume` with your data, and that `ARSA_SERVER_VOLUME` / `ARSA_REPO_VOLUME` in `.env` use those names.
4. Hand all files to `1000:1000`:
   ```bash
   docker run --rm -v arsa-db-volume:/a -v arsa-servers-volume:/b -v arsa-repo-volume:/c alpine chown -R 1000:1000 /a /b /c
   ```
5. Update `.env`: add `DOCKER_GID` (`stat -c %g /var/run/docker.sock`), set `ARSA_USE_VOLUME=true`, and remove `ARSA_BASE_PATH` if it points to the old mount paths (e.g. `/ars/arsa`).
6. `docker compose pull && docker compose up -d`, then start the game servers again.

The anonymous volume the old backend image created for `/app/ars` is no longer used. Find it with `docker volume ls -f dangling=true` and remove it after checking it's the right one.

---

## 9. Security notes

- Mounting `/var/run/docker.sock` into the backend is equivalent to giving that container root-level control over the Docker host, even though the backend itself runs as an unprivileged user. Only run trusted `arsa-backend` images, and restrict who can reach the backend's admin functionality (`ARSA_ADMIN_ROLE`).
- The `ars` containers run untrusted workshop mod scripts and accept connections from the internet. They run as `1000:1000` with all capabilities dropped and `no-new-privileges`.
- `.env` contains a live OIDC client secret — do not check it into a public (or even private, non-restricted) repository. Consider a secrets manager for production deployments.
- The `arsa` network is `internal: true`, so traffic between frontend and backend stays off the outside networks. The only path in from outside is the reverse proxy on `nginx-proxy`.

---

## 10. Troubleshooting

- **Redirect/login loop or "invalid redirect_uri" from Keycloak** — the URI Keycloak sees must exactly match (scheme, host, port, path) one of the client's "Valid redirect URIs". Double-check `ARSA_OIDC_CALLBACK_URI`.
- **Logged in, but access denied inside ARSA** — check that the user actually has the `ARSA_ACCESS_ROLE` / `ARSA_ADMIN_ROLE` role assigned, and that the ID-token mapper (step 8 in section 5) is present and enabled so the role shows up in the token claims.
- **Frontend/backend not reachable from the browser** — check that both containers are attached to `nginx-proxy` and that the proxy routes to `arsa-frontend:80` and `arsa-backend:3000`. See the networking note in section 3.
- **Backend can't create `ars` containers / "permission denied" on `/var/run/docker.sock`** — `DOCKER_GID` must be the group id of the socket on the host (`stat -c %g /var/run/docker.sock`). Also check that `ARSA_ARS_CONTAINER` is pullable from wherever the backend runs.
- **Backend exits with "attempt to write a readonly database"** or **`ars` servers can't write their logs or download mods** — the volumes contain files owned by root. Run the `chown` from [Upgrading from root containers](#upgrading-from-root-containers).
- **Backend exits with "Permission denied" on startup** — `ARSA_BASE_PATH` is still set to a path outside the mounted volumes (e.g. `/ars/arsa`). Remove it from `.env`.
- **Game servers start, but don't find their config** — the volume names in `docker volume ls` don't match `ARSA_SERVER_VOLUME` / `ARSA_REPO_VOLUME`.
