# arsa

## Installation Linux

### Docker Host

- install docker https://docs.docker.com/engine/install/
- allow non-root user https://docs.docker.com/engine/install/linux-postinstall/
- install git from your distro or from https://git-scm.com/downloads/linux

### Backend
- clone backend repository from https://github.com/y0014984/arsa-backend.git
```bash
git clone https://github.com/y0014984/arsa-backend
```
- check the current group id (GID) of docker
```bash
getent group | grep docker
```
- build backend and ars container using the above mentioned GID
```bash
cd ~/arsa-backend (or whereever you cloned the repo)
docker build -t arsa-backend --build-arg DOCKER_GID=988 .
docker build -t ars ./ars
```

### Frontend
- clone frontend repository from https://github.com/y0014984/arsa-frontend.git
```bash
git clone https://github.com/y0014984/arsa-frontend
```
- install node/npm from your distro or from https://nodejs.org/en/download/package-manager
- build fromntend container
```bash
cd ~/arsa-frontend (or whereever you cloned the repo)
npm run build
npm run docker
```

### Start
- clone this repository from https://github.com/y0014984/arsa.git
```bash
git clone https://github.com/y0014984/arsa
```
- start backend and frontend with:
```bash
cd ~/arsa (or whereever you cloned the repo)
docker compose up -d
```
- allow the following ports on yopur firewall 80, 3000 and for the default ars 2001, 17777, 19999
- modify arsa-backend to allow access to docker
```bash


# enter running container
docker exec -it arsa-backend /bin/sh

# create group with correct gid and assign user
addgroup --gid 989 docker && adduser deno docker
```