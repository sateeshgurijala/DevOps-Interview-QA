# Docker — The Practical Deep‑Dive (Commands, Flags, Syntax, and Examples)

> A comprehensive, example‑first tutorial covering the **commands you actually use**, the **flags you’ll see**, **what they mean**, and **how to combine them**. Includes answers to common questions and copy‑paste recipes. Designed for Windows (PowerShell) and Linux/macOS; WSL notes where it matters.

---

## 0) How Docker Works (Mental Model)

- **Image**: Immutable template (layered filesystem + metadata). E.g., `python:3.12-alpine`.
- **Container**: Runtime instance of an image (has its own PID namespace, FS, network). Can be running or stopped.
- **Registry**: Remote store for images (Docker Hub, GHCR, ECR, ACR, GCR).
- **Volume**: Persistent data managed by Docker. Survives container deletion.
- **Bind mount**: Host path mounted into container path.
- **Network**: Virtual switch enabling container‑to‑container and external connectivity.
- **Dockerfile → build → image → run → container**.

**Key advantages**: repeatable builds, isolation, portability, fast startup, small footprint.

---

## 1) CLI Basics & Conventions

### Syntax & Conventions
- `docker <noun> <verb>` is older style (e.g., `docker image ls`).
- Shortcuts exist: `docker images` == `docker image ls`; `docker ps` == `docker container ls`.
- `docker compose` (v2) lives under the main CLI (not a separate `docker-compose`).
- **PowerShell vs Bash quoting**:
  - Bash: `-e KEY="VALUE WITH SPACES"`
  - PowerShell: `-e "KEY=VALUE WITH SPACES"` or use backticks `` ` `` to escape.

### Help
- `docker --help`, `docker <cmd> --help` → official flags & descriptions.

---

## 2) System‑Level Commands

### `docker version`
Shows client/server versions.

### `docker info`
Detailed environment (storage driver, logging driver, CPUs, memory, root dir, registries).

### `docker system df`
Disk usage for images, containers, volumes, build cache.

### `docker system prune`
Removes unused objects.
- `docker system prune` → prune stopped containers, networks not used by at least one container, build cache.
- `-a/--all` → **also** remove unused images (not just dangling).
- `--volumes` → **also** remove volumes (dangerous if you store data there).

**Example** (free space, keep volumes):
```bash
docker system prune -a
```

**FAQ**: *I’m out of disk space!*
- Check with `docker system df`.
- Prune cautiously; volumes often hold DB data.

---

## 3) Image Management

### `docker pull`
Download an image from a registry.
```bash
docker pull nginx:1.27
```
**Tip**: Pin versions; avoid unqualified `latest` in prod.

### `docker images` / `docker image ls`
List local images. Flags:
- `--digests` show content digests
- `--format` Go templates to format output

### `docker image inspect <name:tag|id>`
JSON metadata: layers, env, entrypoint, exposed ports.
```bash
docker image inspect nginx:1.27 | jq '.[0].Config'
```

### `docker rmi <id|name:tag>`
Remove images. If containers depend on them, remove those containers first (`docker rm -f ...`).

### `docker tag` & `docker push`
Retag and upload to a registry.
```bash
docker tag myapp:1.0 ghcr.io/user/myapp:1.0
docker push ghcr.io/user/myapp:1.0
```

**Q**: *How do I authenticate?* → `docker login <registry>` then username/password or token.

---

## 4) Building Images (`docker build`)

**Core syntax**:
```bash
docker build -t myapp:dev .
```
- `-t name:tag` set image name
- `-f path/Dockerfile` use custom Dockerfile
- `--build-arg KEY=VALUE` pass ARGs
- `--target` select a stage in multi‑stage builds
- `--no-cache` ignore cache (fresh build)
- `--pull` update base layers

**Examples**
```bash
# Multi-stage build (select prod stage)
docker build -t myapp:prod --target prod .

# Pass build arg (Dockerfile `ARG NODE_ENV`)
docker build -t myapp:prod --build-arg NODE_ENV=production .

# Use another Dockerfile & context
docker build -f docker/Dockerfile -t myapp:1.2.3 docker/
```

**Common error**: *COPY failed: file not found* → Build **context** is the directory passed last. Ensure files are inside that directory.

---

## 5) Dockerfile Reference (Every Instruction Explained)

### `FROM <image>[:tag|@digest] [AS name]`
Base image. `AS` names a stage for multi‑stage builds.

### `RUN <shell command>` or `RUN ["exec", "form"]`
Executes at build time, creates a new layer.
- Prefer fewer layers; chain commands with `&&`.
- Use exec form for predictable escaping.

### `COPY [--chown=user:group] src dest`
Copy from build context into image. Use `--chown` to set ownership.

### `ADD src dest`
Like COPY but also supports remote URLs and tar auto‑extract. **Prefer COPY** for clarity; use ADD when you need those extras.

### `WORKDIR /path`
Sets the working directory for subsequent instructions.

### `ENV KEY=value`
Set environment variables in the image.

### `ARG NAME[=default]`
Build‑time variable (available only during build unless persisted).

### `EXPOSE <port>[/protocol]`
Documentation hint for the port your app listens on; does not publish ports by itself.

### `USER <user>[:group]`
Switch to non‑root user for security.

### `VOLUME ["/data"]`
Declare intended mountpoint (creates anonymous volume at runtime if not supplied). Often omitted in app images; explicitly mount via `-v` instead.

### `ENTRYPOINT ["exec", "form"]`
Defines the executable to run by default. Usually an array (exec form). Combine with CMD for default args.

### `CMD ["arg1", "arg2"]` or `CMD "shell form"`
Default arguments to ENTRYPOINT **or** the default command if ENTRYPOINT not set.
- **Pattern**: `ENTRYPOINT ["mybinary"]` + `CMD ["--flag", "value"]`.

### `HEALTHCHECK`
Container self‑reporting health.
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 CMD curl -f http://localhost:8080/health || exit 1
```

### `SHELL ["/bin/sh", "-c"]`
Change default shell for RUN on Windows images or special shells.

### `LABEL key=value`
Metadata (maintainer, vcs-url, org.opencontainers.image.* annotations).

### `STOPSIGNAL SIGTERM`
Signal sent on `docker stop`.

### `ONBUILD <INSTRUCTION>`
Triggers when image is used as a base. Rare; for builder templates.

**Example Dockerfile (Node)**
```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules node_modules
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/dist dist
COPY package*.json .
RUN npm ci --omit=dev
EXPOSE 8080
USER node
ENTRYPOINT ["node", "dist/index.js"]
```

---

## 6) Running Containers (`docker run`) — All the Must‑Know Flags

**Base**:
```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

**High‑frequency options**
- `-d, --detach` run in background
- `--name <name>` readable container name
- `--rm` auto‑remove when process exits
- `-p hostPort:containerPort` publish a port
- `-P` publish **all** exposed ports to random hosts ports
- `-e, --env KEY=VAL` set environment variables
- `--env-file .env` load many envs
- `-v, --volume src:dest[:ro|:z]` mount (bind or named volume)
- `--mount type=bind|volume,src=...,dst=...,ro` (newer, explicit style)
- `-w, --workdir /path` working directory inside container
- `--network <net>` attach to a user network
- `--restart no|on-failure[:max]|always|unless-stopped` policy
- `-u, --user UID[:GID]` run as a specific user
- `-it` interactive + TTY (for shells)
- `--cap-add <cap>` / `--cap-drop <cap>` adjust Linux capabilities
- `--security-opt` (e.g., `no-new-privileges:true`)

**Examples**

1) Quick test & auto‑cleanup
```bash
docker run --rm -p 8080:80 nginx:1.27
```

2) Named service with env + restart
```bash
docker run -d --name web -p 8080:80 \
  -e NGINX_HOST=example.com --restart unless-stopped nginx:1.27
```

3) Dev shell with bind mount & working dir
```bash
docker run -it --name app -v "$PWD":/usr/src/app -w /usr/src/app node:20 bash
```

4) DB with named volume (persistent data)
```bash
docker volume create pgdata
docker run -d --name pg -p 5432:5432 -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data postgres:16
```

---

## 7) Container Lifecycle & Inspection

- List: `docker ps` (add `-a` for stopped)
- Stop: `docker stop <name|id>` (graceful)
- Kill: `docker kill <name|id>` (SIGKILL)
- Start: `docker start <name|id>`
- Restart: `docker restart <name|id>`
- Remove: `docker rm <name|id>` (add `-f` to stop+remove)
- Logs: `docker logs [-f] [--tail N] <name|id>`
- Processes: `docker top <name|id>`
- Inspect JSON: `docker inspect <name|id>`
- Port map: `docker port <name|id>`

**Recipes**
```bash
# Watch logs
docker logs -f api

# Remove everything related to a service
docker rm -f api && docker rmi myorg/api:broken || true
```

---

## 8) Exec Into Containers & File Transfer

### `docker exec`
Run a command inside a **running** container.
```bash
docker exec -it api sh              # Alpine/BusyBox
# or
docker exec -it api bash
```

### `docker cp`
Copy files to/from a container.
```bash
docker cp api:/etc/nginx/nginx.conf ./
docker cp ./config.json api:/app/config.json
```

---

## 9) Volumes, Bind Mounts, and Data

### Types
- **Named volume**: managed by Docker (`docker volume create`), portable, good default.
- **Bind mount**: host path → container path (live‑edit code, share artifacts).

### Commands
- `docker volume ls` / `inspect` / `rm`
- Use in `docker run`: `-v pgdata:/var/lib/postgresql/data`

**Best practices**
- DBs → **named volumes** (not bind mounts) for performance & safety.
- Mount read‑only when you can: `:ro`.
- On SELinux hosts (RHEL/Fedora), use `:z`/`:Z` for label fixes.

---

## 10) Networking

### Networks 101
- **bridge (default)**: isolated per host; containers can reach the internet via NAT.
- **user‑defined bridge**: built‑in DNS by *service/container name*. Recommended for multi‑container apps.
- **host** (Linux only): shares host network namespace (no port publishing needed).
- **none**: fully isolated.

### Commands
```bash
docker network ls
docker network create app-net
docker network inspect app-net
docker network connect app-net web
docker network disconnect app-net web
docker network rm app-net
```

**Test connectivity**
```bash
docker run --rm -it --network app-net nicolaka/netshoot
# inside netshoot
curl http://api:8080/health
```

---

## 11) Healthchecks & Restart Policies

- Add a `HEALTHCHECK` in your Dockerfile.
- Use `--restart unless-stopped` for long‑running services.
- Check health: `docker ps` shows `healthy`/`unhealthy`.

---

## 12) Logs, Metrics & Events

- `docker logs -f --tail=200 <name>` stream logs
- `docker stats` live CPU, mem, net, IO per container
- `docker events` low‑level events (start/stop, die, oom)

---

## 13) Docker Compose (v2) — Multi‑Container Made Easy

### Files
- `compose.yml` or `docker-compose.yml` in project root.

### Core commands
```bash
docker compose up -d           # create & start
docker compose up -d --build   # rebuild images
docker compose logs -f         # aggregate logs
docker compose ps              # list services
docker compose exec api sh     # exec into a service
docker compose run --rm job    # one-off task
docker compose down            # stop & remove (keeps volumes)
docker compose down -v         # also remove volumes (danger!)
```

### Minimal example
```yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    env_file: .env
    depends_on:
      - db
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: example
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

### Profiles & Override
- Use `profiles` to include/exclude services per environment.
- `compose.override.yml` merges on `docker compose up`.

---

## 14) Security & Production Hygiene

- Prefer small bases: `alpine`, `ubi-micro`, `distroless` (for final stage).
- Don’t run as root → `USER app`.
- Pin versions; avoid `latest`.
- Secrets: don’t bake into images; pass via env, Docker/Compose `secrets`, or orchestrator KMS.
- Drop capabilities you don’t need; add `--security-opt no-new-privileges:true`.
- Enable `HEALTHCHECK`; use proper `STOPSIGNAL`.

---

## 15) Windows/PowerShell & WSL Notes

- **Execution policy** error when activating venv:
  - In PowerShell (admin): `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`.
- Path separators: Windows uses `C:\path`; WSL/Linux `/path`.
- Bind mounts from Windows to Linux containers can have perf/perm quirks; WSL2 improves this. If slow, try placing project under your WSL filesystem (`/home/<user>/project`).

---

## 16) Copy‑Paste Recipes (Common Tasks)

**1) Build, tag, push**
```bash
docker build -t myapp:1.0 .
docker tag myapp:1.0 ghcr.io/user/myapp:1.0
docker push ghcr.io/user/myapp:1.0
```

**2) One‑off Python job using host files**
```bash
docker run --rm -v "$PWD":/work -w /work python:3.12 python script.py
```

**3) Hot‑reload Node dev**
```bash
docker run -it --rm -p 3000:3000 -v "$PWD":/app -w /app node:20 bash
npm ci && npm run dev
```

**4) Export & import an image (no registry)**
```bash
docker save myapp:1.0 | gzip > myapp_1.0.tar.gz
docker load -i myapp_1.0.tar.gz
```

**5) Debug a crashed container’s FS**
```bash
# container has exited; commit its state to a temporary image
docker commit <stopped-id> temp:debug
docker run -it --rm temp:debug bash
```

**6) Verify inter‑service DNS**
```bash
docker exec -it api sh -c "wget -qO- http://db:5432 || true"
```

---

## 17) Troubleshooting Q&A

**Q1: Container exits immediately. Why?**
- The main process ended (e.g., `node server.js` crashed). Check `docker logs <name>`.
- If you started a shell image without a long‑running command, it will exit.
- Add a foreground command or use `-it` to keep the shell open.

**Q2: Port already in use / not reachable.**
- Check host port conflicts: `docker ps` then `docker port <name>`.
- Change the **left** side of `-p host:container` to a free port.

**Q3: Permission denied on bind mounts (Linux/WSL).**
- Fix ownership (`chown`) or run container with matching UID via `--user $(id -u):$(id -g)`.
- On SELinux: append `:z` to the volume.

**Q4: `COPY` can’t find files during build.**
- Ensure the build context contains the files (path passed to `docker build`).
- `.dockerignore` might exclude them; check contents.

**Q5: Out of space.**
- `docker system df`; prune cautiously. Remove unused images/containers. Avoid `--volumes` unless sure.

**Q6: Environment vars not picked up.**
- Confirm with `docker exec <name> env`.
- If using Compose, check `env_file` path and `docker compose config` to preview merged config.

**Q7: How do I see container’s effective command?**
- `docker inspect <name> --format '{{.Path}} {{join .Args " "}}'`

**Q8: How do I tail logs since a time window?**
- `docker logs --since 10m -f <name>`

**Q9: How to run multiple replicas locally?**
- With Compose: `docker compose up -d --scale worker=3`.

**Q10: Make a container start on boot.**
- Use `--restart unless-stopped` (and ensure Docker service starts on boot).

---

## 18) Reference: Most Useful Flags (Quick Lookup)

**Global**: `--format`, `--no-trunc`, `-q/--quiet`.

**build**: `-t`, `-f`, `--build-arg`, `--target`, `--no-cache`, `--pull`.

**run**: `-d`, `--rm`, `--name`, `-p`, `-P`, `-v/--mount`, `-e/--env-file`, `--network`, `--restart`, `-it`, `-w`, `-u`, `--cap-add`, `--security-opt`.

**logs**: `-f`, `--tail N`, `--since <dur|ts>`.

**compose**: `up -d`, `up --build`, `logs -f`, `exec`, `run --rm`, `ps`, `down [-v]`, `config`.

---

## 19) Practice Lab (Do It End‑to‑End)

1) **Create a minimal app**
```bash
mkdir docker-lab && cd docker-lab
printf 'console.log("Hello from Docker!")\n' > index.js
printf '{"name":"lab","type":"module","scripts":{"start":"node index.js"}}' > package.json
```
2) **Dockerfile**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json .
RUN npm ci --ignore-scripts || true
COPY . .
CMD ["npm","run","start"]
```
3) **Build & run**
```bash
docker build -t docker-lab:1 .
docker run --rm docker-lab:1
```
4) **Compose it with a DB**
```yaml
# compose.yml
services:
  app:
    build: .
    depends_on:
      - db
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: example
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```
```bash
docker compose up -d && docker compose logs -f
```

---

## 20) Next Steps
- Add `HEALTHCHECK` to your images.
- Run with non‑root `USER`.
- Split dev/prod via multi‑stage builds & Compose profiles.
- Learn `--mount` syntax (explicit & CI‑friendly).
- For orchestration, graduate to Kubernetes; your Docker images carry over unchanged.

