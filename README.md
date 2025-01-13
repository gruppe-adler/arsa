# arsa - Arma Reforger Server Admin

## Frontend
https://github.com/y0014984/arsa-frontend

## Backend
https://github.com/y0014984/arsa-backend

## Installation Linux

### Docker Host

- install docker https://docs.docker.com/engine/install/
- allow non-root user https://docs.docker.com/engine/install/linux-postinstall/
- install GitHub CLI https://github.com/cli/cli/blob/trunk/docs/install_linux.md
- install Deno https://docs.deno.com/runtime/getting_started/installation/
- install NVM/NPM/Node https://nodejs.org/en/download/package-manager

### Backend
- clone backend repository from https://github.com/y0014984/arsa-backend.git
```bash
gh repo clone y0014984/arsa-backend
```
- check the current group id (GID) of docker
```bash
getent group | grep docker
```
- build backend and ars container using the above mentioned GID
```bash
cd ~/arsa-backend (or wherever you cloned the repo)
docker build -t arsa-backend .
docker build -t ars ./ars
// use --build-arg DOCKER_GID=988 or similar to adjust GID

// or if deno is used
deno task docker
deno task docker-ars
```

### Frontend
- clone frontend repository from https://github.com/y0014984/arsa-frontend.git
```bash
gh repo clone y0014984/arsa-frontend
```
- install node/npm from your distro or from https://nodejs.org/en/download/package-manager
- change the environment variable to reflect the dns name or ip of your docker host
```
# .env.production
VITE_API_URL=arsa.y0014984.org
```
- build frontend container
```bash
cd ~/arsa-frontend (or wherever you cloned the repo)
npm run build
npm run docker
```

### Start
- clone this repository from https://github.com/y0014984/arsa.git
```bash
gh repo clone y0014984/arsa
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
