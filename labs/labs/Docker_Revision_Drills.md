# Docker Revision Drills
```
Core concepts → Dockerfile mastery → Networking & volumes → Compose → Production patterns
```
---

## Module 1 — Core Concepts

> Images are blueprints. Containers are running instances. Think: Terraform plan vs actual EC2.

---

**Drill 1: Image vs container**

```bash
docker images
docker ps -a
```

`docker images` lists all local images. `docker ps -a` shows all containers — running and stopped.

---

**Drill 2: Run your first container**

```bash
docker run -it ubuntu:22.04 bash
```

`-it` gives you an interactive terminal inside the container. Type `exit` to leave.

---

**Drill 3: Run detached + name it**

```bash
docker run -d --name mynginx -p 8080:80 nginx
```

`-d` = detached (like PM2 daemon). `-p host:container` maps ports. Visit `http://localhost:8080`.

---

**Drill 4: Exec into a running container**

```bash
docker exec -it mynginx bash
```

Same as SSH-ing into a server. Critical for debugging live containers.

---

**Drill 5: Container lifecycle**

```bash
docker stop mynginx      # graceful SIGTERM
docker start mynginx     # restart it
docker rm mynginx        # delete it (must be stopped first)
docker rm -f mynginx     # force remove even if running
```

`stop` sends SIGTERM (graceful). `kill` sends SIGKILL (immediate). Always prefer `stop`.

---

**Drill 6: Inspect container details**

```bash
docker inspect mynginx
```

Returns JSON with IP address, mounts, env vars, network config — indispensable for debugging.

---

## Module 2 — Images & Dockerfile

> Layer caching and multi-stage builds are the two most important concepts here.

---

**Drill 7: Write a minimal Dockerfile**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

Alpine base = smaller image. Copy `package.json` first so `npm ci` is cached unless dependencies change.

---

**Drill 8: Build with a tag**

```bash
docker build -t myapp:1.0 .
docker build -t myapp:latest .
```

Tag format: `name:version`. Same tagging convention you use when pushing to ECR.

---

**Drill 9: Multi-stage build**

```dockerfile
# Stage 1: build
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

# Stage 2: lean production image
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

Builder stage compiles. Final stage is lean. Smaller image = smaller Trivy attack surface — directly relevant to your CI pipeline.

---

**Drill 10: Inspect layer cache**

```bash
docker history myapp:1.0
```

Shows each layer's size and command. Expensive layers (`npm install`, `pip install`) should come early and change rarely.

---

**Drill 11: Build args vs env vars**

```dockerfile
ARG NODE_ENV=production      # build-time only
ENV PORT=3000                # runtime
```

```bash
docker build --build-arg NODE_ENV=production -t myapp:prod .
```

Never bake secrets into `ARG` — they show up in `docker history`. Use secrets management (AWS SSM / Secrets Manager) instead.

---

**Drill 12: Scan your image with Trivy**

```bash
trivy image myapp:1.0
```

You already run Trivy in CI. Run it locally before pushing to catch issues early.

---

## Module 3 — Networking & Volumes

> Bridge networks = K8s service DNS. Named volumes = persistent storage. These map directly to concepts you use on EKS.

---

**Drill 13: List and create networks**

```bash
docker network ls
docker network create mynet
docker network inspect mynet
```

Bridge = default isolated network. Containers on the same bridge can reach each other by container name.

---

**Drill 14: Connect two containers on a network**

```bash
docker run -d --name db --network mynet postgres:15
docker run -d --name app --network mynet myapp:1.0
```

The `app` container can reach the `db` container simply via hostname `db`. Same concept as Kubernetes service DNS (`db.namespace.svc.cluster.local`).

---

**Drill 15: Named volumes for persistent data**

```bash
docker volume create pgdata
docker volume ls
docker run -d -v pgdata:/var/lib/postgresql/data postgres:15
```

Named volumes survive container restarts and removal. Data lives at `/var/lib/docker/volumes/pgdata/`.

---

**Drill 16: Bind mount for live development**

```bash
docker run -v $(pwd):/app -p 3000:3000 myapp:dev
```

Code changes on your host reflect immediately inside the container. Never use bind mounts in production.

---

**Drill 17: EXPOSE vs -p**

```dockerfile
EXPOSE 3000   # documentation only — does NOT open any port
```

```bash
docker run -p 3000:3000 myapp   # this actually publishes the port
docker run -P myapp             # publishes all EXPOSEd ports to random host ports
```

`EXPOSE` is metadata. `-p` is what actually opens the port to the host.

---

## Module 4 — Docker Compose

> Think of `docker-compose.yml` as a local Kubernetes manifest. Same mental model as Helm / Kustomize.

---

**Drill 18: Write a docker-compose.yml**

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      DB_HOST: db
      NODE_ENV: production
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

`depends_on` controls startup order. `restart: unless-stopped` = automatic recovery like PM2.

---

**Drill 19: Up, down, logs**

```bash
docker compose up -d            # start all services detached
docker compose logs -f app      # tail logs of the app service
docker compose ps               # status of all services
docker compose down             # stop + remove containers & networks
docker compose down -v          # also removes volumes (destructive!)
```

---

**Drill 20: Scale a service**

```bash
docker compose up -d --scale app=3
```

Spins up 3 replicas of `app`. Great for quick local load-testing. In production you'd use EKS HPA — same concept.

---

**Drill 21: Compose overrides for dev vs prod**

```bash
# Base config
docker-compose.yml

# Dev override (auto-merged when you run `docker compose up`)
docker-compose.override.yml

# Explicit prod override
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

This mirrors the Kustomize overlay pattern you use for staging vs prod on EKS.

---

## Module 5 — Production Patterns

> Security, healthchecks, resource limits, and ECR push — all directly from your resume stack.

---

**Drill 22: Run as non-root user**

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

Security baseline. Same principle as least-privilege IAM (IRSA) — containers should never run as root in production.

---

**Drill 23: Healthcheck in Dockerfile**

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

Docker marks the container `unhealthy` after 3 failures. Kubernetes liveness probe does the same at a higher level.

---

**Drill 24: Resource limits**

```bash
docker run --memory=512m --cpus=0.5 myapp:1.0
```

In K8s you set `requests` and `limits` in the pod spec. Same concept — prevents noisy-neighbor problems on shared nodes.

---

**Drill 25: Reduce layers — combine RUN commands**

```dockerfile
# Bad: 3 layers, apt cache bloat
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# Good: 1 layer, no bloat
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

Each `RUN` = a new layer. Clean up in the same `RUN` or the deleted files still bloat the image.

---

**Drill 26: Push to ECR (your workflow, manually)**

```bash
# Authenticate
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin \
  <account_id>.dkr.ecr.ap-south-1.amazonaws.com

# Tag
docker tag myapp:1.0 <account_id>.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0

# Push
docker push <account_id>.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

This is exactly what your GitHub Actions CI pipeline does under the hood. Understanding it manually first makes you a better engineer.

---

**Drill 27: Prune dangling resources**

```bash
docker system prune           # stopped containers, dangling images, unused networks
docker system prune -a        # also removes unused images (not just dangling)
docker system prune -a --volumes  # nuclear option — also removes volumes
docker system df              # see disk usage before pruning
```

Run `docker system df` first to assess. Never run `prune --volumes` on a production host.

---

## Quick Reference Cheatsheet

| Command | What it does |
|---|---|
| `docker ps -a` | List all containers |
| `docker images` | List all images |
| `docker run -d -p 8080:80 nginx` | Run detached with port mapping |
| `docker exec -it <name> bash` | Shell into running container |
| `docker logs -f <name>` | Tail container logs |
| `docker stop / rm <name>` | Stop and delete container |
| `docker build -t name:tag .` | Build image from Dockerfile |
| `docker system prune` | Clean up unused resources |
| `docker inspect <name>` | Full container metadata |
| `docker compose up -d` | Start all Compose services |
| `docker compose down -v` | Stop and delete everything |
| `trivy image name:tag` | Scan image for vulnerabilities |

---


