# Docker Interview Questions & Answers

---

**Q: What happens internally when you run `docker run`?**

**A:**
1. Docker CLI sends a REST API request to the Docker daemon (`dockerd`).
2. The daemon checks if the image exists locally. If not, it pulls it from the registry layer by layer.
3. The daemon calls `containerd` to create a container from the image.
4. `containerd` uses `runc` (OCI runtime) to set up Linux namespaces (PID, NET, MNT, UTS, IPC, USER) and cgroups for resource isolation.
5. The overlay filesystem is assembled: read-only image layers + a thin writable container layer on top.
6. Network namespace is configured (veth pair bridged to `docker0` by default).
7. The container's entrypoint/cmd process starts as PID 1 inside the namespace.

---

**Q: How does Docker layer caching work?**

**A:** Each Dockerfile instruction creates a read-only layer. Docker uses a content-addressable cache keyed on:
- The parent layer's ID.
- The instruction string.
- For `COPY`/`ADD`: the checksum of the copied files.

If a layer's cache key matches, Docker reuses the cached layer and skips rebuilding. Cache is invalidated for a layer and all layers below it when:
- The instruction or its inputs change.
- A `COPY`/`ADD` source file changes.
- `--no-cache` flag is passed.

**Best practice**: Order instructions from least-to-most frequently changing. Copy dependency manifests (`package.json`, `requirements.txt`) and install dependencies before copying application source code so dependency layers are cached even when source changes.

---

**Q: What happens when systemd hits a failing unit in a containerized node? How would you auto-recover?**

**A:** When a systemd unit fails, systemd marks it as `failed` and stops restart attempts after `StartLimitBurst` retries. In a containerized environment (e.g., EKS node):
- Container runtime units (containerd, kubelet) failing can cause node NotReady.
- Auto-recovery: configure `Restart=always` and `RestartSec=5s` in the unit file; set `StartLimitIntervalSec=0` to remove the restart rate limit for critical units.
- At the cluster level, node problem detector (NPD) reports systemic failures as node conditions, and cluster-autoscaler or a node health controller can cordon and drain the node, then replace it.

---

**Q: What is the difference between an image and a container?**

**A:**
- **Image**: A read-only, immutable template consisting of stacked filesystem layers plus metadata (entrypoint, env vars, exposed ports). It exists on disk.
- **Container**: A running (or stopped) instance of an image. Docker adds a thin writable layer on top of the image layers. Multiple containers can share the same image with their own isolated writable layers.

Analogy: Image = class definition; Container = object/instance.

---

**Q: How to persist data across container restarts?**

**A:**
- **Named volumes** (`docker volume create`): managed by Docker, stored outside the container filesystem. Survive container deletion.
- **Bind mounts**: map a host path directly into the container (`-v /host/path:/container/path`). Good for development.
- **tmpfs mounts**: in-memory, not persistent — useful for sensitive temporary data.

For production, use named volumes or external storage (EFS, EBS CSI in Kubernetes) rather than bind mounts to avoid host path coupling.

---

**Q: What is the use of docker-compose?**

**A:** `docker-compose` defines and runs multi-container applications using a YAML file (`docker-compose.yml`). It handles:
- Service definition (image, build context, ports, volumes, env vars).
- Automatic network creation so services can reach each other by service name.
- Dependency ordering (`depends_on`).
- Single-command lifecycle management (`up`, `down`, `logs`, `exec`).

Primarily used for local development and CI; for production, Kubernetes or Docker Swarm is preferred.

---

**Q: How do you check logs of a specific container?**

**A:**
```bash
docker logs <container_name_or_id>          # all logs
docker logs -f <container_name_or_id>       # follow/tail
docker logs --tail 100 <container_id>       # last 100 lines
docker logs --since 1h <container_id>       # logs from last hour
```
For containers managed by Kubernetes: `kubectl logs <pod> -c <container> --previous` (for the previous crashed instance).

---

**Q: How to expose a container to the outside world?**

**A:**
- **Port mapping**: `docker run -p 8080:80 nginx` maps host port 8080 to container port 80.
- **Expose in Dockerfile**: `EXPOSE 80` documents the port but does not publish it; publishing requires `-p` at runtime.
- **Host network**: `--network host` removes network isolation and binds directly to the host's network stack (Linux only, not recommended for production).
- In Kubernetes: use a `Service` of type `LoadBalancer` or `NodePort`, or an `Ingress` controller.

---

**Q: What are the stages in a Docker image build? Why do we use ENTRYPOINT and CMD instructions?**

**A:** Build stages follow the Dockerfile top-to-bottom, each instruction creating a new layer. Multi-stage builds add an explicit `FROM ... AS <stage>` to isolate build tools from the runtime image.

- **ENTRYPOINT**: defines the executable that always runs. Cannot be easily overridden at `docker run` (only with `--entrypoint`). Use for the main process.
- **CMD**: provides default arguments to ENTRYPOINT, or the default command if no ENTRYPOINT is set. Easily overridden by arguments passed to `docker run`.

Combined pattern:
```dockerfile
ENTRYPOINT ["node"]
CMD ["server.js"]
```
Running `docker run myimage debug.js` overrides CMD but keeps `node` as the entrypoint.

---

**Q: Which container registry do you use for storing Docker images?**

**A:** Commonly used registries:
- **AWS ECR**: tight IAM integration, lifecycle policies for image cleanup, image scanning with Inspector.
- **GCR / Artifact Registry**: for GCP workloads.
- **Docker Hub**: for public images or small teams.
- **Harbor**: self-hosted, with RBAC, replication, and vulnerability scanning.
- **Artifactory JFrog**: enterprise-grade, supports multiple artifact types.

Choice depends on cloud provider, compliance requirements, and cost.

---

**Q: How do you pass environment variables during Docker build commands? What services do you use for storing Docker images?**

**A:**
```bash
# Build-time args (not available at runtime unless set as ENV)
docker build --build-arg API_URL=https://api.example.com -t myapp .

# Runtime environment variables
docker run -e DB_PASSWORD=secret myapp
docker run --env-file .env myapp
```

In Dockerfile:
```dockerfile
ARG API_URL
ENV API_URL=${API_URL}
```

**Important**: Do not use `--build-arg` for secrets — they are visible in image history. Use Docker BuildKit secrets (`--secret`) instead.

For registries: ECR on AWS, Artifact Registry on GCP, Harbor for on-prem.

---

**Q: Write a Dockerfile for a Node.js application with multi-stage builds.**

**A:**
```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Runtime
FROM node:20-alpine AS runtime
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
USER appuser
EXPOSE 3000
ENTRYPOINT ["node"]
CMD ["server.js"]
```

The builder stage installs dependencies; only the artifacts (node_modules, source) are copied to the slim runtime image. This keeps the final image small and free of build tools.

---

**Q: How do you debug a container that has exited?**

**A:**
```bash
# View exit code and last state
docker ps -a
docker inspect <container_id> | jq '.[0].State'

# Read logs of the exited container
docker logs <container_id>

# Start with a different entrypoint to get a shell
docker run -it --entrypoint /bin/sh <image>

# Copy files out of stopped container
docker cp <container_id>:/app/log.txt ./log.txt
```
In Kubernetes: `kubectl logs <pod> --previous` for the previous crash; `kubectl describe pod` for events like OOMKilled.

---

**Q: What's the difference between COPY and ADD commands in Dockerfile?**

**A:**

| Feature | COPY | ADD |
|---|---|---|
| Local files | Yes | Yes |
| Remote URLs | No | Yes (downloads) |
| Auto-extract tar | No | Yes |
| Recommended | Yes (explicit) | Only when tar extraction needed |

`COPY` is preferred for its predictability. `ADD` with a URL downloads at build time, which can break reproducibility and caching. Use `RUN curl` + `COPY` for better control.

---

**Q: How would you handle secrets in a Docker container for a PHP application connecting to MySQL?**

**A:**
1. **Never bake secrets into the image** — no hardcoded credentials in Dockerfile or source code.
2. **At runtime**, inject via environment variables from a secrets manager:
   - AWS Secrets Manager: use `aws secretsmanager get-secret-value` in an init script or use the Secrets Manager sidecar.
   - HashiCorp Vault: use the Vault agent sidecar to write secrets to a shared volume.
3. **Docker Swarm / Kubernetes Secrets**: mount secrets as files (`/run/secrets/db_password`) rather than env vars to avoid exposure in `/proc/<pid>/environ`.
4. **BuildKit secrets** for build-time needs (e.g., private Composer packages):
   ```dockerfile
   RUN --mount=type=secret,id=composer_token composer install
   ```
5. Rotate secrets without rebuilding images by referencing the secrets manager at startup.

- You want to enforce that all images used in the cluster must come from a trusted internal registry. How do you implement this at the policy level?


