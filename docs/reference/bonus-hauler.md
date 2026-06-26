# Bonus lab — Hauler: build and serve an air-gap bundle

**Time:** 45 min  
**Requires:** Lab deployed and EIB VM reachable

---

## What Hauler is (and what it isn't)

Hauler is an open source project from the Rancher Government team. It solves one specific problem: getting OCI images, Helm charts, and arbitrary files from an internet-connected machine into a fully disconnected environment, in a portable, reproducible way.

It is **not** a SUSE Edge product. It is not in the SUSE Edge support matrix. You will not find it in the SUSE Edge Helm chart repos or get SUSE support for it. It is MIT-licensed, community-maintained, and happens to be very useful for the kind of air-gap work that comes with edge deployments.

This lab uses Hauler to pre-stage content for the exercises. Production teams use it alongside or instead of Harbor, Quay, or plain Docker Distribution — depending on the use case. Hauler is particularly good when you need to **transport** content (USB stick, bastion transfer, S3 sync) rather than run a permanent registry.

If you want Hauler's GitHub repo: [https://github.com/hauler-dev/hauler](https://github.com/hauler-dev/hauler)  
Docs: [https://docs.hauler.dev](https://docs.hauler.dev)

---

## The Hauler mental model

Hauler works in three stages:

```
[connected]   hauler store add ...        →  local store (content-addressable)
              hauler store save           →  portable archive (.tar.zst)

[transfer]    USB / SCP / S3 / bastion   →  disconnected side

[disconnected] hauler store load         →  local store (same format)
               hauler store serve        →  OCI registry (port 5000)
                                            + file server (port 8080)
```

The "store" is a local directory (default `/var/lib/hauler`). Content is stored as OCI artifacts — images, charts, and files all use the same format. The archive is just a compressed snapshot of the store.

---

## Set up

SSH to the EIB VM. Hauler is already installed here from the lab deploy.

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20
hauler version
```

Create a scratch store for this exercise so you don't touch the lab's live store at `/var/lib/hauler`:

```bash
export SCRATCH=/var/lib/hauler-scratch
mkdir -p $SCRATCH
```

---

## Part 1 — Add content to the store

### 1.1 Add container images

```bash
# Add a single image
hauler store add image alpine:3.21 --store $SCRATCH

# Add a specific platform variant
hauler store add image nginx:1.27 --platform linux/amd64 --store $SCRATCH

# Check what's in the store
hauler store info --store $SCRATCH
```

`store info` shows each artifact as a line: type, name, and size. Images are stored as OCI manifests — no Docker daemon involved.

### 1.2 Add a Helm chart

```bash
# Add from a Helm repo
hauler store add chart cert-manager \
  --repo https://charts.jetstack.io \
  --version v1.20.1 \
  --store $SCRATCH

hauler store info --store $SCRATCH
```

Charts are stored as OCI artifacts using the `application/vnd.cncf.helm.config.v1+json` media type. You can install them directly via `helm install oci://...` from the Hauler registry later.

### 1.3 Add files

```bash
# Add any file by URL — Hauler downloads and stores it
hauler store add file \
  "https://github.com/k3s-io/k3s/releases/download/v1.35.5%2Bk3s1/k3s" \
  --name k3s-binary \
  --store $SCRATCH

hauler store info --store $SCRATCH
```

Files are also OCI artifacts internally, served as blobs over HTTP on port 8080.

---

## Part 2 — Declarative approach (Hauler manifest)

Imperative `store add` is fine for quick tests. For repeatable, versioned bundles, use a **Hauler manifest** — a YAML file listing everything you want.

```bash
cat > /tmp/my-bundle.yaml << 'EOF'
apiVersion: content.hauler.cattle.io/v1alpha1
kind: Images
metadata:
  name: demo-images
spec:
  images:
    - name: alpine:3.21
    - name: nginx:1.27
    - name: docker.io/avaleror/alien-geeko:latest
---
apiVersion: content.hauler.cattle.io/v1alpha1
kind: Charts
metadata:
  name: demo-charts
spec:
  charts:
    - name: cert-manager
      repoURL: https://charts.jetstack.io
      version: v1.20.1
---
apiVersion: content.hauler.cattle.io/v1alpha1
kind: Files
metadata:
  name: demo-files
spec:
  files:
    - path: https://get.k3s.io
      name: k3s-install.sh
EOF
```

Sync the manifest into the store:

```bash
hauler store sync --filename /tmp/my-bundle.yaml --store $SCRATCH
hauler store info --store $SCRATCH
```

`store sync` is idempotent — run it again and Hauler skips content that is already present (by digest).

---

## Part 3 — Save and load (simulating the transport)

This is the key step for air-gap workflows. On the internet-connected side, you save the store to a portable archive. On the disconnected side, you load it.

### Save

```bash
hauler store save \
  --filename /tmp/my-bundle.tar.zst \
  --store $SCRATCH

ls -lh /tmp/my-bundle.tar.zst
```

The archive is a zstd-compressed tarball. Copy it to a USB stick, push it to an S3 bucket with a bastion, or SCP it directly — Hauler doesn't care.

### Load

Simulate arriving at the disconnected side by loading into a different store directory:

```bash
export AIRGAP=/var/lib/hauler-airgap
mkdir -p $AIRGAP

hauler store load /tmp/my-bundle.tar.zst --store $AIRGAP

hauler store info --store $AIRGAP
```

The content and the digest list should match what was in `$SCRATCH`.

---

## Part 4 — Serve and consume

Start the Hauler OCI registry and file server from the loaded store:

```bash
# In one terminal (or use &)
hauler store serve registry --port 5100 --store $AIRGAP &
hauler store serve fileserver --port 8100 --store $AIRGAP &
```

Note: using ports 5100 and 8100 here to avoid colliding with the live lab Hauler on 5000/8080.

### Pull an image from Hauler

```bash
# Check what's in the OCI catalog
curl -s http://localhost:5100/v2/_catalog | python3 -m json.tool

# Pull with podman (uses the local Hauler registry)
podman pull localhost:5100/library/alpine:3.21

# Confirm
podman image inspect localhost:5100/library/alpine:3.21 | python3 -m json.tool | grep -A2 '"Id"'
```

### Download a file from Hauler

```bash
# List what the file server has
curl -s http://localhost:8100/

# Download the K3s install script
curl -fsSL http://localhost:8100/k3s-install.sh -o /tmp/k3s-install.sh
head -5 /tmp/k3s-install.sh
```

### Install a Helm chart from Hauler

```bash
# Charts are served at the registry endpoint
helm pull oci://localhost:5100/hauler/cert-manager --version v1.20.1 --destination /tmp/
ls -lh /tmp/cert-manager-v1.20.1.tgz
```

---

## Part 5 — Configuring K3s to use Hauler as a mirror

In production, you want K3s nodes to pull all container images from Hauler instead of the internet. The mechanism is `registries.yaml`.

```bash
# See the registries.yaml that was baked into the edge nodes in this lab
cat /home/eib-config/scripts/99-k3s-registries.sh
```

The structure for your own K3s deployment:

```yaml
# /etc/rancher/k3s/registries.yaml
mirrors:
  "docker.io":
    endpoint:
      - "http://<HAULER-HOST>:5000"
  "registry.suse.com":
    endpoint:
      - "http://<HAULER-HOST>:5000"
  "ghcr.io":
    endpoint:
      - "http://<HAULER-HOST>:5000"
  "registry.k8s.io":
    endpoint:
      - "http://<HAULER-HOST>:5000"
```

K3s reads this at startup. If the mirror responds with the requested image, containerd uses it. If not, it falls back to the original source. Set `tls.insecure_skip_verify: true` under `configs:` if your Hauler instance does not have a TLS cert.

For a wildcard catch-all (redirect everything to Hauler):

```yaml
mirrors:
  "*":
    endpoint:
      - "http://<HAULER-HOST>:5000"
```

After editing, restart K3s: `systemctl restart k3s`

---

## Clean up

Stop the scratch Hauler servers and remove the test directories:

```bash
kill %1 %2 2>/dev/null
rm -rf $SCRATCH $AIRGAP /tmp/my-bundle.tar.zst /tmp/my-bundle.yaml
```

---

## When to use Hauler vs other tools

| Tool | Good for | Watch out for |
|---|---|---|
| **Hauler** | Transport: USB sticks, one-time migrations, airgap bundles | Not a production registry — no auth, no replication, no HA |
| **Harbor** | Production private registry with RBAC, scanning, replication | Complex to operate; needs persistent storage and a team |
| **Docker Distribution** | Minimal registry, easy to run as a container | No UI, no auth out of the box |
| **Zot** | Production OCI registry, CNCF project, lightweight | Less ecosystem tooling than Harbor |
| **Hauler + Harbor** | Hauler for transport, Harbor as the destination | Copy from Hauler into Harbor with `hauler store copy registry://harbor.local` |

A common pattern for edge deployments: use Hauler to collect and transport content from the internet side, then push into Harbor or Distribution on the disconnected side with `hauler store copy`. Use K3s `registries.yaml` to mirror pulls through that registry.

---

## Inspect what this lab uses

The live Hauler instance for the lab runs at `192.168.122.20:5000` (registry) and `:8080` (fileserver). You can inspect it without affecting the lab:

```bash
# What images are stored
curl -s http://localhost:5000/v2/_catalog | python3 -m json.tool

# Tags for a specific image
curl -s http://localhost:5000/v2/edge/3.6/edge-image-builder/tags/list | python3 -m json.tool

# What files are available
curl -s http://localhost:8080/

# Hauler store summary
hauler store info --store /var/lib/hauler
```
