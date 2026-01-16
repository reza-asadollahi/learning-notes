# Quick notes of the docer concepts and useful commands

## Base concepts

**Image**: Images are *read‑only templates* that contain everything needed to run an application (code, runtime, libraries, environment). They are the blueprint for creating containers.

**Containers**: Containers are *running instances of images*. They include a writable layer on top of the image and run with isolated processes, configs, and resources. They can be started, stopped, removed, or recreated easily.

**Volumes**: Containers lose their internal data when stopped or removed. Volumes provide *persistent storage* by mapping container paths to the host or to named/anonymous volumes. They are the recommended way to store databases, logs, or any data that must survive container restarts.

**Network**: Docker networks are *isolated virtual networks* that containers can join. Containers on the same network can communicate using their container name as a hostname (e.g., `myfrontend:3000`). Networks help isolate services and control connectivity.

**Dockerfile**: A Dockerfile is a *set of instructions* used to build custom images. It defines the base image, dependencies, environment variables, exposed ports, commands, and entrypoints.

**Docker Compose** and **docker-compose.yml**: Docker Compose is a tool for *defining and running multi‑container applications*. The `docker-compose.yml` file describes services, networks, volumes, and configurations so you can start everything with a single command (`docker compose up`).



----------------------
----------------------
----------------------
## useful docker commands

### common commands

1. **Pull image**  
Pulls an image from `docker.io/library` into the local Docker registry.
```bash
docker pull <image-name:tag>
```

You can also pull from a custom registry:

```bash
docker login <registry-url>
docker pull <your-custom-registry>/<image-name:tag>
```

2. **List local images**

```bash
docker images
```

3. **Remove image from local registry**

```bash
docker rmi <image-name:tag>
```

4. **Create & start a container from an image**
   Pulls the image automatically if it does not exist locally.

```bash
docker run <image-name:tag>
```

**docker run options:**

* `-d` → detached mode (run in background)
* `-p <host-port>:<container-port>` → port mapping
* `--name <container-name>` → assign a readable name
* `-e KEY=value` → set environment variables
* `--net <network-name>` → attach container to a specific network
* `-v <host-path>:<container-path>` → bind mount (absolute path)
* `-v <volume-name>:<container-path>` → named volume
* `-v <container-path>` → anonymous volume
* `--rm` → automatically remove container when it stops (**important**)

5. **Stop a running container**

```bash
docker stop <container-name-or-id>
```

6. **Start an existing (stopped) container**

```bash
docker start <container-name-or-id>
```

7. **List containers**

```bash
docker ps
```

Options:

* `-a` → show all containers (including stopped ones)

8. **View container logs**

```bash
docker logs <container-name-or-id>
```

Options:

* `-f` → follow logs (live)
* `--tail <n>` → show last `n` log lines

9. **Access running container terminal**

```bash
docker exec -it <container-name-or-id> sh
# or
docker exec -it <container-name-or-id> bash
```

10. **Remove container**

```bash
docker rm <container-name-or-id>
```

(force remove running container)

```bash
docker rm -f <container-name-or-id>
```

11. **Inspect container / image (debugging)**

```bash
docker inspect <container-name-or-id>
```

---

### network-specific

12. **List Docker networks**

```bash
docker network ls
```

13. **Create a Docker network**

```bash
docker network create <network-name>
```

14. **Remove a Docker network**

```bash
docker network rm <network-name>
```

---

### volumes

15. **List volumes**

```bash
docker volume ls
```

16. **Remove a volume**

```bash
docker volume rm <volume-name>
```

---

### docker compose commands

> Docker Compose is installed by default with Docker Desktop
> (new syntax: `docker compose`, old syntax: `docker-compose`)

17. **Create & start containers**

```bash
docker compose -f <path-to-compose.yaml> up
```

Options:

* `-d` → detached mode
* `--build` → rebuild images

18. **Stop & remove containers, networks**

```bash
docker compose -f <path-to-compose.yaml> down
```

Options:

* `-v` → also remove volumes

---

### custom images

19. **Build image from Dockerfile**

```bash
docker build -t <image-name:tag> <path-to-dockerfile-folder>
```

20. **Tag image for custom registry**

```bash
docker tag <image-name:tag> <registry-url>/<image-name:tag>
```

21. **Push image to custom registry**

```bash
docker push <registry-url>/<image-name:tag>
```

### important notes

* Containers are **ephemeral**; data must be stored in **volumes**
* Prefer **Docker networks** for container-to-container communication
* Use `.dockerignore` to speed up builds and reduce image size
* One container = **one main process** (best practice)

--------------------
--------------------
--------------------

## Docker Compose (docker-compose.yml)

Docker Compose allows you to define and run **multiple containers** using a single YAML file.  
It replaces many `docker run` commands with **declarative configuration**.

---

### basic structure
```yaml
version: "3.9"

services:
  container-name:
    image: image-name:tag
    container_name: custom-name
    ports:
      - "host:container"
    environment:
      KEY: value
    volumes:
      - host-path:container-path
    networks:
      - network-name

networks:
  network-name:

volumes:
  volume-name:
```

---

### main sections explained

#### `version`

* Compose file format version
* New Docker versions ignore it, but still commonly used

---

#### `services`

Defines **containers** (each service = one container)

Example:

```yaml
services:
  app:
    image: node:20
```

---

#### `image`

* Image to run (like `docker run image`)

```yaml
image: postgres:16
```

---

#### `build`

* Build image from a `Dockerfile` instead of pulling

```yaml
build:
  context: .
  dockerfile: Dockerfile
```

---

#### `container_name`

* Explicit container name

```yaml
container_name: my-app
```

---

#### `ports`

* Port mapping (`host:container`)

```yaml
ports:
  - "3000:3000"
```

---

#### `environment`

* Environment variables inside container

```yaml
environment:
  NODE_ENV: production
  DB_HOST: db
```

---

#### `env_file`

* Load env vars from file

```yaml
env_file:
  - .env
```

---

#### `volumes`

* Persistent data or bind mounts

```yaml
volumes:
  - ./src:/app
  - db-data:/var/lib/postgresql/data
```

---

#### `depends_on`

* Controls startup order (**not readiness**)

```yaml
depends_on:
  - db
```

---

#### `command`

* Override default container command

```yaml
command: npm run dev
```

---

#### `restart`

* Restart policy

```yaml
restart: always
# no | always | on-failure | unless-stopped
```

---

#### `networks`

* Attach service to custom networks

```yaml
networks:
  - backend
```

---

### defining multiple containers (real example)

```yaml
version: "3.9"

services:
  app:
    build: .
    container_name: web-app
    ports:
      - "3000:3000"
    environment:
      DB_HOST: db
    depends_on:
      - db
    volumes:
      - .:/app
    networks:
      - backend

  db:
    image: postgres:16
    container_name: postgres-db
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: secret
    volumes:
      - pg-data:/var/lib/postgresql/data
    networks:
      - backend

networks:
  backend:

volumes:
  pg-data:
```

---

### important compose concepts

* Services can **communicate via service name** (DNS)

  * `app` → `db:5432`
* No need to expose DB ports unless host access is needed
* Volumes persist data even if containers are deleted
* One Compose file = **one isolated environment**

---

### compose vs docker run (mapping)

| docker run | docker compose   |
| ---------- | ---------------- |
| `-p`       | `ports`          |
| `-e`       | `environment`    |
| `-v`       | `volumes`        |
| `--name`   | `container_name` |
| `--net`    | `networks`       |
| `--rm`     | `down`           |

---

### best practices

* Use `.env` for secrets
* One service = one responsibility
* Use named volumes for databases
* Do NOT hardcode passwords in Git
* Use `depends_on` + healthchecks for production




--------------------
--------------------
--------------------
--------------------
## Dockerfile structure & explanation

A `Dockerfile` defines **how an image is built**.  
Each instruction creates a **layer**, and layers are cached for faster rebuilds.

---

### basic Dockerfile example
```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
````

---

### instructions explained

#### `FROM`

* Base image
* Must be the **first instruction**

```dockerfile
FROM python:3.12-slim
```


---

#### `WORKDIR`

* Sets working directory inside container
* Creates it if it does not exist

```dockerfile
WORKDIR /usr/src/app
```

---

#### `COPY`

* Copies files from host → container

```dockerfile
COPY . .
```

Best practice (cache-friendly):

```dockerfile
COPY package.json package-lock.json ./
```

---

#### `ADD` (use with care)

* Like `COPY` but supports URLs & auto-extract
* Prefer `COPY` unless needed

```dockerfile
ADD archive.tar.gz /app
```

---

#### `RUN`

* Executes command during **build time**

```dockerfile
RUN npm install
```

Used for:

* Installing dependencies
* Compiling source code

---

#### `EXPOSE`

* Documents which port the container listens on
* Does **not** publish the port

```dockerfile
EXPOSE 8080
```

---

#### `CMD`

* Default command when container starts
* Can be overridden in `docker run`

```dockerfile
CMD ["node", "server.js"]
```

---

#### `ENTRYPOINT`

* Fixed executable (harder to override)

```dockerfile
ENTRYPOINT ["python", "app.py"]
```

**CMD vs ENTRYPOINT**

* `ENTRYPOINT` → always runs
* `CMD` → default args

---

#### `ENV`

* Set environment variables

```dockerfile
ENV NODE_ENV=production
```

---

#### `ARG`

* Build-time variables (not available at runtime)

```dockerfile
ARG VERSION
```

---

#### `USER`

* Run container as non-root (**security**)

```dockerfile
USER node
```

---

#### `VOLUME`

* Declares a mount point

```dockerfile
VOLUME ["/data"]
```

---

### multi-stage builds (important)

Used to reduce final image size.

```dockerfile
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

Benefits:

* Smaller images
* No dev dependencies in production

---

### docker ignore (must-have)

`.dockerignore`

```gitignore
node_modules
.env
.git
dist
```

Prevents:

* Large images
* Leaking secrets
* Slow builds

---

### build & run

```bash
docker build -t my-app .
docker run -p 3000:3000 my-app
```

---

### best practices

* Use **small base images** (`alpine`, `slim`)
* Order instructions for caching
* One process per container
* Avoid `latest` tag in production
* Run as non-root user
* Use multi-stage builds

---

### common mistakes

* Copying everything before `npm install`
* Storing secrets in Dockerfile
* Large images due to missing `.dockerignore`
* Using root user in production
