# Documentation for

#### Rootless Docker + SBOM + Cosign: Build, Sign, Verify (2025)

[YouTube video](https://youtu.be/zyLUA8kOqzg)

---

#### References
- **Docker Rootless mode** - https://docs.docker.com/engine/security/rootless/
- **Buildx `build` (SBOM & provenance)** - https://docs.docker.com/reference/cli/docker/buildx/build/
- **SBOM attestations** - https://docs.docker.com/build/metadata/attestations/sbom/
- **Provenance attestations** - https://docs.docker.com/build/metadata/attestations/slsa-provenance/
- **`imagetools inspect`** - https://docs.docker.com/reference/cli/docker/buildx/imagetools/inspect/
- **Install Docker Scout** - https://docs.docker.com/scout/install/
- **`docker scout sbom`** - https://docs.docker.com/reference/cli/docker/scout/sbom/
- **Cosign install** - https://docs.sigstore.dev/cosign/system_config/installation/
- **Cosign verify** - https://docs.sigstore.dev/cosign/verifying/verify/
- **ttl.sh ephemeral registry** - https://ttl.sh/

---

## Prerequisites

- Docker installed (20.10+)

> ### 🖥️ Example Environment: Debian 12

## Safety First
- Use a **non‑privileged** user for rootless Docker.
- Don’t put secrets into **build args** (provenance may record them at `mode=max`).

---

## Pre‑Install

- Install runtime bits for rootless

```bash
sudo apt update
sudo apt install -y uidmap dbus-user-session slirp4netns jq
```

- Stop the rootful daemon for a clean context

```bash
sudo systemctl disable --now docker.service docker.socket
```

## Enable Rootless Docker

- **Run as your normal user (not root)**

```bash
dockerd-rootless-setuptool.sh install
systemctl --user enable --now docker
sudo loginctl enable-linger "$USER"  # auto‑start after reboot
```

- Point the CLI to the user socket & test

```bash
echo 'export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock' >> ~/.bashrc
. ~/.bashrc
docker info | grep -i rootless || true
docker run --rm hello-world
```

> Low ports need extra steps in rootless. See the Rootless docs for `ip_unprivileged_port_start` and RootlessKit notes.

---

## Create a Demo Dockerfile

```bash
cat > Dockerfile <<'EOF'
FROM alpine:latest
RUN apk add --no-cache ca-certificates
CMD ["sh","-c","echo Hello from rootless SBOM demo && sleep 5"]
EOF
```

---

## Build with SBOM + Provenance (attestations)

> `--provenance=mode=max` adds richer details. Default `true` uses `mode=min`.

- Create/Select a containerized Buildx builder (attestations need this)

```bash
# create and use
docker buildx create --name hsebuilder --driver docker-container --use
# or just use if already created
docker buildx use hsebuilder
```

**Push to registry**

Attaches an **SPDX SBOM** and **SLSA provenance** to the image in the **registry**.

```bash
# Use an ephemeral registry for demos (expires; choose 1h–24h)
docker buildx build -t ttl.sh/hse-sbom-demo:1h \
  --sbom=true \
  --provenance=mode=max \
  --push .
```

**(Alternative) Locally - no push**

```bash
docker buildx build --sbom=true --provenance=mode=max --output type=local,dest=out .
```

---

## View the Attested SBOM

- From registry

```bash
docker buildx imagetools inspect ttl.sh/hse-sbom-demo:1h \
  --format "{{ json .SBOM.SPDX }}" | jq . | head -n 50
```

- Locally

```bash
ls -1 out | grep sbom && head -n 50 out/sbom.spdx.json
```

## (Alternative) Use Docker Scout

Generates an SBOM on demand (even if you didn’t build with `--sbom`).

- Install Docker Scout (CLI plugin)

```bash
# Linux (official script)
curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
sh install-scout.sh
# verify
docker scout version
```

- View SBOM

```bash
docker scout sbom --format spdx ttl.sh/hse-sbom-demo:1h | jq . | head -n 50
```

### What’s the difference between these methods?

- **Build & push with attestations** (`--push --sbom --provenance=mode=max`): BuildKit attaches an SBOM and provenance **to the image in the registry** as OCI attestations. Retrieve the *attested* SBOM with `imagetools inspect`.
- **Local exporter** (`--output type=local,dest=out`): Writes artifacts to `./out/` (e.g., `sbom.spdx.json`) for pre‑publish review. No local image is created and nothing is pushed.
- **Docker Scout SBOM** (`docker scout sbom`): Generates an SBOM by analyzing the image **on demand**. This is not the signed/attested SBOM attached at build time.
- **Provenance detail**: `--provenance=true` = `mode=min`; use `--provenance=mode=max` for richer details.

---

## Sign the Image with Cosign

- Install Cosign

```bash
ARCH=$(uname -m); case "$ARCH" in x86_64) ARCH=amd64;; aarch64) ARCH=arm64;; esac
VER=$(curl -s https://api.github.com/repos/sigstore/cosign/releases/latest | grep -oP '"tag_name":\s*"\Kv[0-9.]+' )
curl -LO "https://github.com/sigstore/cosign/releases/download/${VER}/cosign_${VER#v}_${ARCH}.deb"
sudo dpkg -i "cosign_${VER#v}_${ARCH}.deb"
cosign version
```

**Sign image**

- A. Keyless (OIDC - will open a browser or device login)

```bash
cosign sign ttl.sh/hse-sbom-demo:1h
```

- B. Local key pair

```bash
cosign generate-key-pair
cosign sign --key cosign.key ttl.sh/hse-sbom-demo:1h
```

**Verify Signatures & Attestations**

- Keyless - verify expected identity & issuer

```bash
cosign verify ttl.sh/hse-sbom-demo:1h \
  --certificate-identity "<you@example.com>" \
  --certificate-oidc-issuer "https://github.com/login/oauth"
```

- Local key verify

```bash
cosign verify --key cosign.pub ttl.sh/hse-sbom-demo:1h
```

- Verify build attestations (incl. provenance/SBOM)

```bash
cosign verify-attestation ttl.sh/hse-sbom-demo:1h
```

> **Workaround** for ephemeral registry ttl.sh
> ```bash
> docker buildx imagetools inspect ttl.sh/hse-sbom-demo:1h \
>   --format '{{ json .Provenance.SLSA }}' > provenance.json
> cosign attest \
>   --type https://spdx.dev/Document \
>   --predicate out/sbom.spdx.json \
>   --key cosign.key \
>   ttl.sh/hse-sbom-demo:1h
> cosign attest \
>   --type slsaprovenance \
>   --predicate provenance.json \
>   --key cosign.key \
>   ttl.sh/hse-sbom-demo:1h
> cosign verify-attestation --key cosign.pub \
>   --type https://spdx.dev/Document \
>   ttl.sh/hse-sbom-demo:1h
> cosign verify-attestation --key cosign.pub \
>   --type slsaprovenance \
>   ttl.sh/hse-sbom-demo:1h
> ```

---

## Notes
- Rootless mode has feature differences (low ports, some networking). See the Rootless docs for details.
- Keep build contexts small (`.dockerignore`) to speed up builds and reduce noise in SBOMs.
- Prefer secret mounts over build args for sensitive data.
- Attestations are discoverable when the image is pushed to a registry.
