# arsa - Arma Reforger Server Admin

## Known Restrictions
- No Experimental Branch Support
- Configs are not automatically adjusted if an Reforger update requires changes to the config syntax
- Mods on the server are downloaded for each server config separately in it's profile folder; this could lead to increasing storage usage

## Frontend
https://github.com/gruppe-adler/arsa-frontend

## Backend
https://github.com/gruppe-adler/arsa-backend

## Installation Linux

### Docker Host

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

// or if deno is used
deno task docker
```

### Frontend
- clone frontend repository from https://github.com/gruppe-adler/arsa-frontend.git
```bash
gh repo clone gruppe-adler/arsa-frontend
```
- install node/npm from your distro or from https://nodejs.org/en/download/package-manager
- change the environment variable to reflect the dns name or ip of your backend including the port if it differs from port 80 e.g. api.host.com:3000
```
# .env.production
VITE_API_URL=arsa-api.gruppe-adler.de
```
- build frontend container
```bash
cd ~/arsa-frontend (or wherever you cloned the repo)
npm run build
npm run docker
```

### Start
- clone this repository from https://github.com/gruppe-adler/arsa.git
```bash
gh repo clone gruppe-adler/arsa
```
- start backend and frontend with:
```bash
cd ~/arsa (or wherever you cloned the repo)
docker compose up -d
```
- allow the following ports on your firewall 80, 3000 and for the default ars 2001, 17777, 19999
- now you can access arsa on port 80 of the docker host

### Stop
- stop backend and frontend with:
```bash
cd ~/arsa (or wherever you cloned the repo)
docker compose down
```
