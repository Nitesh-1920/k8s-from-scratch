# Docker Learnings: From Containers to Kubernetes

## Part 1: Today's Learnings

### The problem Docker solves

A developer builds an application that must run the same way on every machine. Docker packages the app together with its dependencies, libraries, and runtime versions into an **image**, so the app behaves **consistently** everywhere. This solves the classic "it works on my machine" problem.

Docker removes *environment differences*. It does not make an application bug-free.

### Core concepts

| Term | Meaning |
|---|---|
| **Dockerfile** | A set of instructions describing how to build an image |
| **Image** | A read-only template built from a Dockerfile |
| **Container** | A running, isolated instance of an image (one image can create many containers) |
| **Registry** | A service that stores and distributes images (Docker Hub, AWS ECR, Azure ACR, GHCR) |
| **Repository** | A collection of related images in a registry (usually versions of one app) |
| **Tag** | A label for a specific version of an image (`nginx:1.27`) |

Image name pattern: `<registry-host>/<repository>:<tag>`. If the registry host is omitted, Docker assumes Docker Hub.

### Docker architecture and workflow

- **Client**: where you type commands (`docker build`, `push`, `pull`, `run`). It sends requests to the daemon.
- **Docker daemon (dockerd)**: does the real work: builds images, runs containers, talks to registries.
- **Registry**: remote storage for images.

```
Dockerfile --build--> Image --push--> Registry --pull--> Image --run--> Container
```

`docker run` automatically pulls the image first if it is not available locally.

### Containers vs. virtual machines

Containers are lightweight because they share the host's OS kernel instead of running a full OS. They are isolated, but this is not a strong security boundary like a VM.

### Public vs. private registries

- **Public** (Docker Hub): open or shared images.
- **Private** (AWS ECR, Azure ACR, private Docker Hub repos): proprietary and production images.

### Commands learned

```bash
# Build an image and name (tag) it
docker build -t day02-todo .
docker build -t day02-todo:v1 .        # with an explicit version tag

# Run a container in the background with a port mapping
docker run -dp 3000:3000 username/new-reponame:tagname

# Open a shell inside a running container
docker exec -it <container-name-or-id> sh

# Useful inspection commands
docker ps -a                # list all containers, including stopped ones
docker images               # list local images
docker logs <container>     # view output and errors
docker inspect <container>  # config, env vars, exit code, limits
docker stats                # live CPU and memory usage
docker stop <container>     # stop a container
```

**What the flags mean**

- `-t name:tag`: **tag**. Names the image so you do not have to use a random ID. Without a tag, Docker uses `latest`.
- `.`: the **build context** (current directory).
- `-d`: **detached**. Runs the container in the background.
- `-p host:container`: **publish a port**. Maps a port on your machine to a port inside the container. The right-hand number must match the port the app actually listens on.
- `-dp` is simply `-d -p` bundled together.
- `-i`: keep input open. `-t` (in `docker exec`): allocate a terminal. Together, `-it` gives an interactive shell.
- `sh` is used instead of `bash` because it exists in almost every image, including minimal ones like Alpine.
- `docker exec` enters a container that is **already running**. `docker run -it image sh` starts a **new** one.
- Typing `exit` closes only your shell. The container keeps running.
- Manual changes made inside a container are lost when it is removed. Fix things in the Dockerfile and rebuild.

### Multi-stage builds

A multi-stage build uses more than one `FROM` in a single Dockerfile. You build in one stage and copy only the needed output into the final stage.

```dockerfile
FROM node:18-alpine AS installer
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:latest AS deployer
COPY --from=installer /app/build /usr/share/nginx/html
```

- **Stage 1 (`installer`)**: has Node, npm, `node_modules`, and source code. It builds the static files into `/app/build`.
- **Stage 2 (`deployer`)**: a fresh Nginx image. `COPY --from=installer` brings in only the built files.
- **Final image**: Nginx plus static files. Node, `node_modules`, and source are discarded.

**Why copy `package*.json` first?** Docker caches each instruction as a layer. If `package.json` has not changed, the `npm install` layer is reused even when the source code changed, which makes rebuilds much faster.

**Benefits:** smaller images, smaller attack surface, and a single Dockerfile.

**Running it:** Nginx listens on port 80 inside the container, so the mapping is `docker run -dp 8080:80 my-frontend`.

**Improvements to consider**

- Pin the Nginx version (e.g. `nginx:1.27-alpine`) instead of `latest` for reproducible builds.
- Add a `.dockerignore` listing `node_modules` and `.git`.
- Build a single stage for debugging with `docker build --target installer .`.

### Why containers can still fail

| Failure scenario | What happens | How to investigate |
|---|---|---|
| **Application bug** | The code crashes and the container exits | `docker logs <container>` |
| **Missing environment variables or secrets** | The app cannot find config, DB password, or API keys | `docker inspect <container>` |
| **Network or database connectivity** | The app cannot reach a database or another service | Check logs, test connectivity from inside the container |
| **CPU architecture mismatch** | An ARM-built image fails on an x86 server (`exec format error`) | `docker buildx build --platform linux/amd64` |
| **Insufficient memory or CPU limits** | The container is killed, often exit code `137` (SIGKILL, commonly the OOM killer) | `docker inspect`, `docker stats` |
| **Wrong or missing startup command** | The container starts and immediately exits | Review `CMD` / `ENTRYPOINT` |
| **Image pull failure** | Wrong tag, missing image, or expired registry credentials | Check the image name and tag, run `docker login` again |

**Takeaway:** Docker guarantees a **consistent environment**, not a **bug-free application**.

---

## Part 2: Challenges of Using Standalone Containers

Running containers directly with `docker run` works well for learning and small setups. In production, the limits show up quickly.

1. **Single point of failure.** All containers live on one host. If the host goes down, everything goes down.
2. **No self-healing.** If a container crashes, nothing restarts it automatically (apart from a basic `--restart` policy), and nothing replaces it if the host dies.
3. **Manual scaling.** To handle more traffic, you start more containers by hand, on the right machines, with the right ports. There is no automatic scaling based on load.
4. **No built-in load balancing.** Traffic is not automatically spread across multiple copies of a container. You must set up and maintain a load balancer yourself.
5. **Networking and service discovery are manual.** Containers get changing IPs. Connecting services across multiple hosts requires extra tooling and custom configuration.
6. **Risky deployments.** Updating means stopping the old container and starting a new one, which can cause downtime. Rolling back means redoing it by hand.
7. **Poor resource management.** Nothing decides which host has spare CPU and memory. One container can starve others unless limits are set manually on each one.
8. **Configuration and secrets handling.** Environment variables and secrets are passed per container through flags or files, which is hard to manage and audit at scale.
9. **Persistent storage.** Container filesystems are temporary. Keeping data safe, and attached to the right container on the right host, needs manual volume management.
10. **Operational sprawl.** With dozens of containers across several machines, tracking what runs where, collecting logs, and monitoring health becomes unmanageable.

---

## Part 3: How Kubernetes Solves These Challenges

Kubernetes (K8s) is a **container orchestration platform**. You describe the desired state of your application (for example, "run 3 copies of this image"), and Kubernetes continuously works to make reality match it.

| Challenge | How Kubernetes solves it |
|---|---|
| **Single point of failure** | Runs workloads across a **cluster of nodes**. If a node fails, its Pods are rescheduled onto healthy nodes. |
| **No self-healing** | **Controllers** (Deployment / ReplicaSet) keep the desired number of Pods running. Failed containers are restarted and failed Pods are replaced. **Liveness and readiness probes** detect unhealthy apps. |
| **Manual scaling** | **Horizontal Pod Autoscaler (HPA)** scales Pods based on CPU, memory, or custom metrics. Manual scaling is one command or one line in a manifest. |
| **No load balancing** | **Services** give a stable endpoint and distribute traffic across healthy Pods. **Ingress** handles external HTTP/HTTPS routing. |
| **Manual networking and discovery** | Every Pod gets an IP in a flat cluster network. **Cluster DNS** lets services find each other by name. |
| **Risky deployments** | **Rolling updates** replace Pods gradually with no downtime. **Rollbacks** return to a previous version with one command (`kubectl rollout undo`). |
| **Poor resource management** | The **scheduler** places Pods on nodes with enough capacity. **Requests and limits** and **namespaces and quotas** control resource usage. |
| **Config and secrets** | **ConfigMaps** and **Secrets** separate configuration from images and inject it into Pods. |
| **Persistent storage** | **PersistentVolumes and PersistentVolumeClaims** attach durable storage that outlives individual Pods. |
| **Operational sprawl** | **Declarative YAML manifests** (stored in Git) describe everything. One API (`kubectl`) manages the whole cluster, with labels for organizing and selecting resources. |

**Mental model:** Docker packages and runs a container. Kubernetes decides *where* containers run, *how many* run, and *keeps them healthy*.

```
Developer --> Image (Docker) --> Registry --> Kubernetes cluster
                                                |
                    +---------------------------+----------------------------+
                    |                           |                            |
                 Node 1                      Node 2                       Node 3
               [Pod] [Pod]                  [Pod] [Pod]                  [Pod]
```

---

## Part 4: When to Use Kubernetes (and When Not To)

### 5 use cases where you should use Kubernetes

1. **Microservices architectures.** Many independent services that need to discover each other, scale separately, and be deployed independently.
2. **High availability and zero-downtime requirements.** Customer-facing applications where outages are costly and rolling updates and self-healing are essential.
3. **Variable or unpredictable traffic.** Workloads with spikes (for example, food delivery or e-commerce at peak hours) that benefit from autoscaling up and down.
4. **Large-scale or multi-team platforms.** Dozens or hundreds of services and many teams that need standard deployment, namespaces, access control (RBAC), and resource quotas.
5. **Multi-cloud or hybrid deployments.** A consistent platform across AWS (EKS), Azure (AKS), GCP (GKE), or on-premises, which reduces lock-in to one provider's tooling.

### 5 use cases where you should not use Kubernetes

1. **Simple apps or small projects.** A single app, a static site, or a personal project. A single VM or a managed platform (PaaS) is far simpler.
2. **Small teams without Kubernetes expertise.** The learning curve and operational burden (upgrades, networking, security, troubleshooting) can outweigh the benefits.
3. **Low-traffic apps with stable load.** If you never need to scale and brief downtime is acceptable, orchestration adds cost and complexity for no gain.
4. **Short-lived prototypes, MVPs, and proof-of-concepts.** Speed matters more than resilience, so plain Docker or Docker Compose is faster to set up.
5. **Tightly coupled monoliths or stateful legacy systems.** Apps that are hard to containerize, depend on specific hardware or fixed servers, or rely heavily on local state may be a poor fit. Simpler alternatives include Docker Compose, AWS ECS/Fargate, or serverless options such as AWS Lambda.

**Rule of thumb:** use Kubernetes when the cost of managing many containers by hand is higher than the cost of running Kubernetes. Until then, simpler tools are usually the better choice.

