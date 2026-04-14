# Docker Security

**Hardening Containers from Build to Runtime**

Tags: `Security` `Hardening` `Scanning` `DevSecOps`

---

## 01 -- Why Docker Security Matters

Containers share the host kernel -- a misconfigured container can compromise the entire host. Security must be addressed at every layer: image, build, runtime, network, and orchestration.

- **Container Escapes** -- Privileged containers or kernel exploits can break isolation boundaries
- **Supply Chain Attacks** -- Malicious base images or compromised dependencies injected at build time
- **Data Exposure** -- Secrets, credentials, and sensitive data leaked through images or logs

According to Sysdig's 2025 Container Security Report, **87%** of container images contain high or critical vulnerabilities.

---

## 02 -- Container Attack Surface Overview

```
Image --> Build --> Registry --> [Runtime] --> Network --> Orchestration
```

### Host-Level Vectors
- Shared kernel vulnerabilities (CVEs)
- Docker daemon running as root
- Mounted host filesystems (`/var/run/docker.sock`)
- Excessive Linux capabilities

### Container-Level Vectors
- Vulnerable packages in base images
- Hardcoded secrets in layers
- Running processes as root (UID 0)
- Unrestricted network access between containers

---

## 03 -- Image Security & Vulnerability Scanning

Start with **minimal base images** and scan every layer for known CVEs before deployment.

### Base Image Selection
- Use `alpine`, `distroless`, or `chainguard` images
- Pin image digests, not just tags
- Avoid `:latest` -- use explicit versions
- Rebuild images regularly to pick up patches

### Scanning Tools
- **Trivy** -- fast, comprehensive, OSS
- **Snyk** -- developer-focused, CI integration
- **Grype** -- Anchore's OSS scanner
- **Docker Scout** -- built into Docker Desktop

```bash
# Scan an image with Trivy
trivy image --severity HIGH,CRITICAL myapp:latest

# Scan with Snyk
snyk container test myapp:latest --severity-threshold=high
```

---

## 04 -- Dockerfile Security Best Practices

### Do
- Use multi-stage builds to exclude build tools
- Pin package versions explicitly
- Remove caches: `rm -rf /var/cache/apt/*`
- Use `COPY` instead of `ADD`
- Add a `.dockerignore` file
- Use `HEALTHCHECK` instructions

### Don't
- Store secrets in `ENV` or `ARG`
- Run `apt-get upgrade` (non-reproducible)
- Use `:latest` tags for base images
- Install unnecessary packages
- Leave package managers in final image
- Use `chmod 777` anywhere

```dockerfile
# Multi-stage build example
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM gcr.io/distroless/nodejs20-debian12
COPY --from=builder /app /app
USER nonroot
CMD ["app/server.js"]
```

---

## 05 -- Running as Non-Root Users

By default, containers run as **root (UID 0)**. If an attacker escapes the container, they have root on the host. Always run as a non-root user.

### In the Dockerfile

```dockerfile
# Create a dedicated user
RUN addgroup -S appgroup && \
    adduser -S appuser -G appgroup

# Set ownership and switch
COPY --chown=appuser:appgroup . /app
USER appuser
```

### At Runtime

```bash
# Override user at run time
docker run --user 1000:1000 myapp

# Verify running user
docker exec mycontainer whoami
# Output: appuser
```

**Tip:** Use `--user` flag even if the Dockerfile sets `USER` -- defence in depth.

---

## 06 -- Read-Only Filesystems

Prevent attackers from writing malicious files, scripts, or binaries into the container filesystem at runtime.

```bash
# Run with read-only root filesystem
docker run --read-only myapp

# Allow specific writable paths with tmpfs
docker run --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --tmpfs /var/run:rw,noexec,nosuid \
  myapp
```

### Benefits
- Prevents malware installation
- Blocks reverse shells writing to disk
- Enforces immutable infrastructure
- Catches apps that write unexpectedly

### Considerations
- Apps may need writable `/tmp` or log dirs
- Use `tmpfs` mounts with `noexec` flag
- Test thoroughly before production
- Combine with `--no-new-privileges`

---

## 07 -- Docker Secrets Management

Never bake secrets into images. Use runtime secret injection mechanisms to provide credentials securely.

### Never Do This

```dockerfile
ENV DB_PASS=s3cret
# Visible in image history!
# docker history myimg
```

### Docker Swarm Secrets

```bash
echo "s3cret" | docker secret create db_pass -
# Mounted at /run/secrets/ -- in-memory tmpfs, never on disk
```

### BuildKit Secrets

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret
```

**Production:** Use external vaults -- HashiCorp Vault, AWS Secrets Manager, Azure Key Vault -- and inject at runtime via environment or mounted files.

---

## 08 -- Linux Capabilities & Seccomp Profiles

Docker grants containers a subset of Linux capabilities by default. Reduce the attack surface by dropping all and adding only what is needed.

### Capabilities

```bash
# Drop all capabilities, add only needed
docker run --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=CHOWN \
  myapp

# NEVER use --privileged in production
# It grants ALL 40+ capabilities
```

### Seccomp Profiles

```bash
# Use Docker's default seccomp profile (blocks ~44 of 300+ syscalls)

# Apply a custom profile
docker run --security-opt seccomp=custom-profile.json myapp
```

| Capability | Purpose | Risk if Granted |
|---|---|---|
| `SYS_ADMIN` | Mount filesystems, namespaces | **Critical -- near root** |
| `NET_RAW` | Raw sockets, ping | Network spoofing |
| `SYS_PTRACE` | Debug processes | Process injection |
| `NET_BIND_SERVICE` | Bind ports <1024 | Low risk |

---

## 09 -- AppArmor & SELinux

Mandatory Access Control (MAC) systems add a kernel-level enforcement layer beyond standard Unix permissions.

### AppArmor (Ubuntu/Debian)
- Docker loads `docker-default` profile automatically
- Restricts mount, ptrace, and signal operations
- Custom profiles for fine-grained control

```bash
docker run --security-opt apparmor=my-custom-profile myapp
```

### SELinux (RHEL/Fedora)
- Labels containers with `container_t` type
- Prevents containers accessing host files
- MCS (Multi-Category Security) isolates containers

```bash
docker run --security-opt label=type:container_t myapp
```

---

## 10 -- Network Security & Isolation

By default, all containers on the same bridge network can communicate. Use network segmentation to enforce least-privilege communication.

### Network Segmentation

```bash
# Create isolated networks
docker network create --internal backend
docker network create frontend

# Frontend can reach internet
docker run --network frontend web

# Backend is internal-only
docker run --network backend db
```

### Additional Controls
- Disable inter-container communication: `--icc=false` in daemon config
- Use `--network=none` for offline containers
- Never expose Docker API on `0.0.0.0:2375`
- Use TLS for Docker daemon remote access
- Restrict published ports: `-p 127.0.0.1:8080:80`

```yaml
# docker-compose.yml -- network isolation
services:
  web:
    networks: [frontend, backend]
  api:
    networks: [backend]
  db:
    networks: [backend]
networks:
  frontend:
  backend:
    internal: true
```

---

## 11 -- Content Trust & Image Signing

Verify that images have not been tampered with between build and deployment using cryptographic signatures.

### Docker Content Trust (DCT)

```bash
# Enable content trust globally
export DOCKER_CONTENT_TRUST=1

# Push a signed image
docker push myregistry/myapp:v1.2

# Pull only signed images (fails if unsigned)
docker pull myregistry/myapp:v1.2
```

### Cosign (Sigstore)

```bash
# Generate a key pair
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key myregistry/myapp@sha256:abc...

# Verify before deployment
cosign verify --key cosign.pub myregistry/myapp@sha256:abc...
```

**Best Practice:** Enforce signature verification in your CI/CD pipeline and admission controllers (e.g., Kyverno, OPA Gatekeeper).

---

## 12 -- Runtime Security Monitoring

Detect anomalous behaviour in running containers -- unexpected processes, file access, network connections, and syscalls.

### Falco
- CNCF project, eBPF-based
- Detects shell in container
- Alerts on sensitive file reads
- Custom rules engine

### Sysdig Secure
- Commercial runtime protection
- Drift detection
- Forensic capture
- Compliance reporting

### Tracee
- Aqua Security OSS tool
- eBPF-based tracing
- Detects container escapes
- Signature-based detection

```yaml
# Falco rule example -- detect shell spawned in container
- rule: Terminal Shell in Container
  desc: Detect a shell spawned in a container
  condition: >
    spawned_process and container and proc.name in (bash, sh, zsh)
  output: "Shell spawned in container (user=%user.name container=%container.name)"
  priority: WARNING
```

---

## 13 -- Docker Bench for Security

An open-source script that checks your Docker host and containers against the CIS Docker Benchmark -- dozens of automated checks.

```bash
docker run --rm --net host --pid host --userns host --cap-add audit_control \
  -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
  -v /var/lib:/var/lib:ro -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro -v /etc:/etc:ro \
  docker/docker-bench-security
```

### Categories Checked
- Host configuration
- Docker daemon configuration
- Docker daemon config files
- Container images and build files
- Container runtime
- Docker Security Operations
- Docker Swarm configuration

---

## 14 -- Registry Security

Your container registry is a critical supply chain component. Secure it like any production system.

### Access Control
- Enforce RBAC -- least-privilege pull/push
- Require authentication for all operations
- Use short-lived tokens, not long-lived passwords
- Enable audit logging on all registry access
- Use TLS everywhere -- never plain HTTP

### Image Policies
- Enable automatic vulnerability scanning on push
- Block deployment of images with critical CVEs
- Use immutable tags to prevent overwrites
- Set retention policies to remove old images
- Allowlist approved base images only

| Registry | Scanning | Signing | RBAC |
|---|---|---|---|
| Docker Hub | Docker Scout | DCT / Cosign | Teams & Orgs |
| GitHub GHCR | Dependabot | Cosign | Org-level |
| AWS ECR | Inspector | Cosign / Notation | IAM Policies |
| Harbor (OSS) | Trivy built-in | Cosign / Notation | Fine-grained |

---

## 15 -- Supply Chain Security

Secure every link in the chain from source code to running container using SLSA and SBOM frameworks.

```
[Source] --> [Build] --> [Attest] --> [Sign] --> [Verify] --> [Deploy]
```

### SBOM (Software Bill of Materials)
- Generate with `syft`, `trivy`, or `docker sbom`
- Formats: SPDX, CycloneDX
- Attach to images as attestations
- Required by US Executive Order 14028

### SLSA Framework (Levels 1-4)
- **L1:** Build process documented
- **L2:** Signed provenance from hosted build
- **L3:** Hardened, isolated build platform
- **L4:** Hermetic, reproducible builds

```bash
# Generate SBOM with Syft
syft myapp:latest -o spdx-json > sbom.spdx.json

# Attach SBOM attestation with Cosign
cosign attest --predicate sbom.spdx.json --type spdxjson \
  --key cosign.key myregistry/myapp@sha256:abc...
```

---

## 16 -- Rootless Docker

Run the entire Docker daemon and containers without root privileges -- eliminating the most dangerous attack vector.

### How It Works
- Docker daemon runs as a regular user
- Uses `user namespaces` for UID remapping
- Root inside container maps to unprivileged UID on host
- Uses `slirp4netns` or `pasta` for networking

### Limitations
- Cannot bind to ports <1024 without CAP
- Some storage drivers not supported
- AppArmor/SELinux profiles may need adjustment
- Slightly higher networking overhead

```bash
# Install rootless Docker
dockerd-rootless-setuptool.sh install

# Verify rootless mode
docker info --format '{{.SecurityOptions}}'
# Output includes "rootless"

# Set environment for rootless
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```

---

## 17 -- CIS Docker Benchmark Highlights

The CIS Docker Benchmark provides a comprehensive set of recommendations for securing Docker in production.

| Section | Key Recommendations | Priority |
|---|---|---|
| 1. Host Config | Keep host OS and Docker up to date, audit Docker files | **Critical** |
| 2. Daemon Config | Enable user namespace support, restrict network traffic | **Critical** |
| 3. Config Files | Set ownership/permissions on daemon files to root:root | High |
| 4. Images | Use trusted base images, scan for vulnerabilities, no secrets | **Critical** |
| 5. Runtime | No privileged containers, drop capabilities, read-only FS | **Critical** |
| 6. Operations | Avoid image sprawl, use centralised logging | High |
| 7. Swarm | Encrypt overlay networks, rotate join tokens | Medium |

**Automation:** Use `docker-bench-security` and integrate CIS checks into CI/CD pipelines.

---

## 18 -- Summary & Further Reading

### Key Takeaways
- Start with **minimal, scanned base images**
- Run as **non-root** with **read-only filesystems**
- Drop **all capabilities**, add only what is needed
- Never bake **secrets** into images
- Sign images and verify in CI/CD
- Monitor runtime with Falco or equivalent
- Audit regularly with CIS benchmarks
- Consider **rootless Docker** for defence in depth

### Resources
- [Docker Security Documentation](https://docs.docker.com/engine/security/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [Falco Runtime Security](https://falco.org/)
- [Trivy Vulnerability Scanner](https://aquasecurity.github.io/trivy/)
- [Sigstore / Cosign](https://www.sigstore.dev/)
- [SLSA Framework](https://slsa.dev/)
- [Docker Bench for Security](https://github.com/docker/docker-bench-security)
- [OWASP Docker Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
