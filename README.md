# ARSA – Deployment & Setup Guide

ARSA is a self-hosted admin stack made up of two services — a **frontend** SPA and a **backend** API — that lets authenticated users provision and manage `ars` (dedicated game server) container instances through Docker. Login and role-based access control are delegated to an external **Keycloak** realm via OpenID Connect (OIDC).

This guide documents the `docker-compose.yml` / `.env` pair for a local (`localhost`) deployment and what needs to be configured on the Keycloak side for it to work.

---

## 1. Architecture

```
                     ┌───────────────────────────────┐
   Browser  ───────▶ │        arsa-frontend           │
                     │ (static SPA, env baked in)     │
                     └───────────────┬─────────────────┘
                                     │ (arsa network)
                     ┌───────────────▼─────────────────┐
                     │         arsa-backend             │
                     │  - REST/API server               │
                     │  - OIDC login/callback handling   │
                     │  - orchestrates `ars` containers  │
                     └───┬──────────┬──────────┬────────┘
                         │          │          │
              /app/db    │          │          │  /var/run/docker.sock
          (arsa-db-vol)  │          │          │  (host Docker engine)
                         │          │          │
              /ars/servers          │  /ars/repo
        (arsa-servers-vol,   (arsa-repo-volume)
         or host bind path
         if ARSA_USE_VOLUME=false)

   External:  Keycloak realm (ARSA_OIDC_ISSUER) — not part of this stack
```

- **frontend** — serves the UI; its configuration (API URL, etc.) is compiled into the build, so config changes to the frontend image typically require a rebuild rather than just an env change.
- **backend** — the only service with credentials, DB access, and (critically) a mount of the host's `/var/run/docker.sock`, which it uses to start/stop/manage the individual `ars` server containers defined by `ARSA_ARS_CONTAINER`.
- **Keycloak** — an existing realm you provide; ARSA does not run its own identity provider.

---

## 2. Prerequisites

- Docker Engine + Docker Compose v2
- A reachable Keycloak instance with a realm already created (here: `grad` at `kc.gruppe-adler.de`)
- Network access from the **backend container** to the Keycloak issuer URL
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
    hostname: "arsa-frontend"
    image: ghcr.io/gruppe-adler/arsa-frontend:main
    restart: unless-stopped
    depends_on:
      backend:
        condition: service_started
    networks:
      - arsa

  backend:
    hostname: "arsa-backend"
    image: docker.io/library/arsa-backend:latest
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - arsa-db-volume:/app/db
      - arsa-servers-volume:/ars/servers
      - arsa-repo-volume:/ars/repo
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - arsa

volumes:
  arsa-db-volume:
  arsa-servers-volume:
  arsa-repo-volume:

networks:
  arsa:
    internal: true
```

> **⚠️ Ports:** neither service publishes a port, and the `arsa` network is marked `internal: true`. Docker does **not** publish container ports onto internal networks, so as written the stack is unreachable from the host or a browser. Since the callback/frontend URIs point at `localhost:3000` and `localhost:5173`, you need one of:
>
> - Add explicit `ports:` mappings to `frontend` and `backend` **and** remove `internal: true` from the `arsa` network, or
> - Put a reverse proxy (nginx/Traefik/Caddy) in front, attached to both the `arsa` network and a second, non-internal network that is itself port-published.
>
> Keep `internal: true` only if you add a proxy this way — it's what stops the frontend/backend from reaching the wider internet directly, which is good hardening, but it means something else has to bridge the gap to your browser.

### `.env`

```dotenv
ARSA_ALLOWED_ORIGINS=http://localhost:5173
ARSA_ACCESS_ROLE=adler
ARSA_ADMIN_ROLE=arsa-admin
ARSA_BIND_ADDRESS=0.0.0.0:3000
ARSA_USE_VOLUME=false
ARSA_SERVER_VOLUME=arsa-servers-volume
ARSA_REPO_VOLUME=arsa-repo-volume
ARSA_OIDC_CLIENT_ID=arsa-dev
ARSA_OIDC_CLIENT_SECRET=<your-keycloak-client-secret>
ARSA_OIDC_ISSUER=https://kc.gruppe-adler.de/realms/grad
ARSA_OIDC_CALLBACK_URI=http://localhost:3000/api/v2/auth/callback
ARSA_OIDC_FRONTEND_URI=http://localhost:5173/
ARSA_ARS_CONTAINER=ghcr.io/gruppe-adler/ars-image
ARSA_IP_ADDR=127.0.0.1
ARSA_BASE_PATH=./ars
```

> Never commit a real `ARSA_OIDC_CLIENT_SECRET` to version control. Treat `.env` as a secret file (add it to `.gitignore`) and rotate the secret if it's ever exposed.

---

## 4. Environment variable reference

| Variable                  | Purpose                                                                                                                                                                                                                |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ARSA_ALLOWED_ORIGINS`    | CORS allow-list for the backend API. Comma-separate multiple origins if you serve the frontend from more than one URL.                                                                                                 |
| `ARSA_ACCESS_ROLE`        | Keycloak role required (present in the ID token) for a user to access ARSA at all.                                                                                                                                     |
| `ARSA_ADMIN_ROLE`         | Keycloak role required for admin-level actions inside ARSA.                                                                                                                                                            |
| `ARSA_BIND_ADDRESS`       | Host:port the backend process listens on **inside its container**.                                                                                                                                                     |
| `ARSA_USE_VOLUME`         | `true` → server files live in the named Docker volume `ARSA_SERVER_VOLUME`. `false` → server files live at the host path `ARSA_BASE_PATH` instead (bind mount, managed directly by the backend via the Docker socket). |
| `ARSA_SERVER_VOLUME`      | Name of the Docker volume used for `ars` server data, when `ARSA_USE_VOLUME=true`.                                                                                                                                     |
| `ARSA_REPO_VOLUME`        | Name of the Docker volume used for the mod/asset repository shared with `ars` containers.                                                                                                                              |
| `ARSA_OIDC_CLIENT_ID`     | Client ID of the confidential Keycloak client ARSA authenticates as.                                                                                                                                                   |
| `ARSA_OIDC_CLIENT_SECRET` | Secret for that client. Keep private.                                                                                                                                                                                  |
| `ARSA_OIDC_ISSUER`        | Full issuer URL of the Keycloak realm, e.g. `https://<host>/realms/<realm>`. Used for OIDC discovery.                                                                                                                  |
| `ARSA_OIDC_CALLBACK_URI`  | The backend URL Keycloak redirects to after login. Must exactly match a "Valid redirect URI" on the client.                                                                                                            |
| `ARSA_OIDC_FRONTEND_URI`  | Where the backend sends the browser back to once login/session handling is done.                                                                                                                                       |
| `ARSA_ARS_CONTAINER`      | Image reference the backend uses when it spins up individual dedicated-server containers.                                                                                                                              |
| `ARSA_IP_ADDR`            | Host/bind IP the backend uses when publishing ports for the `ars` server containers it creates.                                                                                                                        |
| `ARSA_BASE_PATH`          | Host directory used for `ars` server files when `ARSA_USE_VOLUME=false`.                                                                                                                                               |

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
# 2. Fill in ARSA_OIDC_CLIENT_SECRET and any other environment specific values
# 3. Start the stack
docker compose up -d

# Check status / logs
docker compose ps
docker compose logs -f backend
```

Once the networking note in section 3 is addressed, the frontend should be reachable at `ARSA_OIDC_FRONTEND_URI` (`http://localhost:5173/`), and logging in will route through Keycloak and back via `ARSA_OIDC_CALLBACK_URI`.

---

## 7. Data & volumes

| Volume / Path                                                                           | Contents                                                                                                                   |
| --------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `arsa-db-volume` → `/app/db`                                                            | Backend's persistent application database.                                                                                 |
| `arsa-servers-volume` → `/ars/servers` (or `ARSA_BASE_PATH` if `ARSA_USE_VOLUME=false`) | Files for individual managed `ars` server instances.                                                                       |
| `arsa-repo-volume` → `/ars/repo`                                                        | Shared repository of server assets/mods used across instances.                                                             |
| `/var/run/docker.sock`                                                                  | **Not user data** — gives the backend direct control of the host's Docker engine so it can create/manage `ars` containers. |

---

## 8. Security notes

- Mounting `/var/run/docker.sock` into the backend is equivalent to giving that container root-level control over the Docker host. Only run trusted `arsa-backend` images, and restrict who can reach the backend's admin functionality (`ARSA_ADMIN_ROLE`).
- `.env` contains a live OIDC client secret — do not check it into a public (or even private, non-restricted) repository. Consider a secrets manager for production deployments.
- The `arsa` network being `internal: true` is good practice for limiting the frontend/backend's own outbound access; just make sure whatever fronts it (reverse proxy) is the only path in from outside.

---

## 9. Troubleshooting

- **Redirect/login loop or "invalid redirect_uri" from Keycloak** — the URI Keycloak sees must exactly match (scheme, host, port, path) one of the client's "Valid redirect URIs". Double-check `ARSA_OIDC_CALLBACK_URI`.
- **Logged in, but access denied inside ARSA** — check that the user actually has the `ARSA_ACCESS_ROLE` / `ARSA_ADMIN_ROLE` role assigned, and that the ID-token mapper (step 8 in section 5) is present and enabled so the role shows up in the token claims.
- **Frontend/backend not reachable from the browser** — see the ports/`internal: true` note in section 3.
- **Backend can't create `ars` containers** — verify `/var/run/docker.sock` is mounted and the backend's user has permission to talk to the Docker daemon, and that `ARSA_ARS_CONTAINER` is pullable from wherever the backend runs.
