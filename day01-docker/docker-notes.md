## Understanding Docker: From Application to Running Container

A developer builds an application that needs to run the same way on every machine. Docker solves this by packaging the app together with its dependencies, libraries, and runtime versions into an **image**.

### The workflow

1. **Dockerfile**: a set of instructions telling Docker how to build the image (base image, dependencies, application code, startup command).
2. **Image**: built from the Dockerfile with `docker build`. It is a read-only template.
3. **Registry**: images are pushed to a registry so others can use them.
   - **Public** (e.g., Docker Hub) for open or shared images.
   - **Private** (e.g., AWS ECR, Azure ACR) for proprietary and production images.
4. **Container**: anyone with access pulls the image and runs it with `docker run`. A container is an isolated, running instance of an image. One image can create many containers.

```
Dockerfile --build--> Image --push--> Registry --pull--> Image --run--> Container
```

### Why this helps

Containers are lightweight because they share the host's OS kernel instead of running a full OS like a virtual machine. Since the image carries the same dependencies and versions everywhere, the application behaves **consistently** across developer laptops, test servers, and production. This solves the classic "it works on my machine" problem.

### Docker doesn't guarantee zero errors

Docker removes *environment differences*, not every possible problem. A container can still fail for reasons like these:

| Failure scenario | What happens | How to investigate |
|---|---|---|
| **Application bug** | The code itself crashes, and the container exits | `docker logs <container>` |
| **Missing environment variables or secrets** | The app starts but can't find its config, DB password, or API keys | `docker inspect <container>`, check the `-e` flags or env file |
| **Network or database connectivity** | The app can't reach a database or another service, due to wrong host, port, network, or firewall | Check logs, then test connectivity from inside the container |
| **CPU architecture mismatch** | An image built on an ARM machine (e.g., Apple Silicon Mac) fails on an x86 server with an `exec format error` | Build with `docker buildx --platform linux/amd64` |
| **Insufficient memory or CPU limits** | The container is killed with an OOM (out of memory) error, exit code `137` | `docker inspect <container>` and `docker stats` |
| **Wrong or missing startup command** | The container starts and immediately exits | Review `CMD` / `ENTRYPOINT` in the Dockerfile |
| **Image pull failure** | Wrong tag, missing image, or expired registry credentials | Check the image name and tag, then run `docker login` again |

### Quick debugging commands

```bash
docker ps -a                 # list all containers, including stopped ones
docker logs <container>      # view application output and errors
docker inspect <container>   # config, env vars, exit code, limits
docker stats                 # live CPU and memory usage
docker exec -it <container> sh   # open a shell inside a running container
```

### Key takeaway

Docker guarantees a **consistent environment**, not a **bug-free application**. Containerizing an app removes dependency and version surprises, but you still need good logging, correct configuration, proper resource limits, and monitoring.
