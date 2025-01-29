# arsa - Arma Reforger Server Admin

## Known Restrictions/Issues
- No Experimental Branch Support
- Configs are not automatically adjusted if an Reforger update requires changes to the config syntax
- Mods on the server are downloaded for each server config separately in it's profile folder; this could lead to increasing storage usage
- Logfile timestamps from different timezone (UTC?)

## Frontend
https://github.com/gruppe-adler/arsa-frontend

## Backend
https://github.com/gruppe-adler/arsa-backend

## Installation Linux (Dev Environment)

- install docker https://docs.docker.com/engine/install/
- allow non-root user https://docs.docker.com/engine/install/linux-postinstall/
- install GitHub CLI https://github.com/cli/cli/blob/trunk/docs/install_linux.md
- install Deno https://docs.deno.com/runtime/getting_started/installation/
- install NVM/NPM/Node https://nodejs.org/en/download/package-manager

### Backend
- clone backend repository from https://github.com/gruppe-adler/arsa-backend.git
```bash
gh repo clone gruppe-adler/arsa-backend
```
- check the current group id (GID) of docker
```bash
getent group | grep docker
```
- start backend dev environment using the above mentioned GID
```bash
cd ~/arsa-backend (or wherever you cloned the repo)

deno task dev
```

### Frontend
- clone frontend repository from https://github.com/gruppe-adler/arsa-frontend.git
```bash
gh repo clone gruppe-adler/arsa-frontend
```
- install node/npm from your distro or from https://nodejs.org/en/download/package-manager
- change the environment variable to reflect the dns name or ip of your backend including the port if it differs from port 80 e.g. api.host.com:3000, as well as the used protocolls
```
# .env.development
VITE_API_URL=localhost:3000
VITE_API_PROTOCOL=http
VITE_API_WEBSOCKET_PROTOCOL=ws
```
- start frontend dev environment
```bash
cd ~/arsa-frontend (or wherever you cloned the repo)
npm ci
npm run dev
```

### Installation Linux (Production Environment on Docker Host)

- install docker https://docs.docker.com/engine/install/
- allow non-root user https://docs.docker.com/engine/install/linux-postinstall/
- install GitHub CLI https://github.com/cli/cli/blob/trunk/docs/install_linux.md
- install Deno https://docs.deno.com/runtime/getting_started/installation/
- install NVM/NPM/Node https://nodejs.org/en/download/package-manager

### Backend
- clone backend repository from https://github.com/gruppe-adler/arsa-backend.git
```bash
gh repo clone gruppe-adler/arsa-backend
```
- check the current group id (GID) of docker
```bash
getent group | grep docker
```
- build backend and ars container using the above mentioned GID
```bash
cd ~/arsa-backend (or wherever you cloned the repo)
docker build -t arsa-backend .
// use --build-arg DOCKER_GID=988 or similar to adjust GID
// use --build-arg PORT=80 or similar to adjust port of the backend
// also edit the port in .env.production

// or if deno is used
deno task docker
```

### Frontend
- clone frontend repository from https://github.com/gruppe-adler/arsa-frontend.git
```bash
gh repo clone gruppe-adler/arsa-frontend
```
- install node/npm from your distro or from https://nodejs.org/en/download/package-manager
- change the environment variable to reflect the dns name or ip of your backend including the port if it differs from port 80 e.g. api.host.com:3000, as well as the used protocolls
```
# .env.production
VITE_API_URL=arsa.gruppe-adler.de
VITE_API_PROTOCOL=https
VITE_API_WEBSOCKET_PROTOCOL=wss
```
- build frontend container
```bash
cd ~/arsa-frontend (or wherever you cloned the repo)
npm ci
npm run build
npm run docker
```

### Incomplete
This manual doesn't tell you anything about the two containers or host binds that you need to add to the backend. Every created ars instance also needs access to these volumes or host binds. This is configured in `.env.production` file in the backend repository.
