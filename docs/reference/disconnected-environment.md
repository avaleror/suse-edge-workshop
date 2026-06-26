# Disconnected environment reference

This lab is designed to run offline after the initial deploy. This page explains what that means in practice: what was pre-staged, what stays local during exercises, and what would break if internet access were cut.

---

## The two phases

| Phase | Internet needed? | Who runs it |
|---|---|---|
| `rodeo deploy` | Yes — pulls several GB | Instructor, once |
| Lab exercises (01 – 06) | No | Students |

Once `rodeo deploy` completes, the lab network at `192.168.122.0/24` is self-sufficient. Every image, binary, and artifact needed for the exercises is already on the EIB VM.

---

## What Hauler stores

Hauler runs on the EIB VM (`192.168.122.20`) and serves two endpoints:

**OCI registry — port 5000**

| Image | Source | Used in |
|---|---|---|
| `registry.suse.com/edge/3.6/edge-image-builder:1.3.3.1` | SUSE registry | Exercise 3 — all EIB builds |
| `registry.suse.com/rancher/elemental-register:1.9.0` | SUSE registry | Exercise 3 — Elemental ISO builds |
| `docker.io/avaleror/alien-geeko:latest` | Docker Hub | Exercise 6 — Fleet deploy to edge clusters |

**File server — port 8080**

| File | Used in |
|---|---|
| `SL-Micro.x86_64-6.2-Base-SelfInstall-GM.install.iso` | Exercise 3.1, 3.2 — EIB Elemental ISO base |
| `SL-Micro.x86_64-6.2-Default.raw` | Exercise 3.3, 3.4 — EIB standalone RAW base |

Verify everything is in place:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20

# Registry contents
curl -s http://localhost:5000/v2/_catalog | python3 -m json.tool

# File server contents
curl -s http://localhost:8080/ | grep -E "iso|raw|SL-Micro"

# Files staged for EIB use
ls -lh /home/eib-config/base-images/
```

---

## How EIB builds stay offline

EIB runs inside Podman on the EIB VM. Each build pulls the EIB container from Hauler (not from `registry.suse.com`). The `embeddedArtifacts.registries` section in every definition file tells EIB where to resolve image references:

```yaml
embeddedArtifacts:
  registries:
    urls:
      - 192.168.122.20:5000
```

EIB queries Hauler's OCI registry for each image it needs to embed into the edge node OS. As long as those images are in Hauler, the build never touches the internet.

The base OS images come from the file server:

```bash
# Elemental path (ISO output)
curl -fsSL http://localhost:8080/SL-Micro.x86_64-6.2-Base-SelfInstall-GM.install.iso \
  -o /home/eib-config/base-images/SL-Micro.x86_64-6.2-Base-SelfInstall-GM.install.iso

# Standalone path (RAW output)
curl -fsSL http://localhost:8080/SL-Micro.x86_64-6.2-Default.raw \
  -o /home/eib-config/base-images/SL-Micro.x86_64-6.2-Default.raw
```

Exercise 3 walks through this download step before starting the builds.

---

## How edge nodes stay offline

EIB bakes a `registries.yaml` into every edge node image via the `99-k3s-registries.sh` script. When K3s starts on the node, it reads this file and routes all container pulls through the Hauler OCI registry:

```yaml
# /etc/rancher/k3s/registries.yaml (on every edge node)
mirrors:
  "docker.io":
    endpoint:
      - "http://192.168.122.20:5000"
  "registry.suse.com":
    endpoint:
      - "http://192.168.122.20:5000"
  "ghcr.io":
    endpoint:
      - "http://192.168.122.20:5000"
```

K3s checks this on startup. If a mirror responds with the requested image, containerd uses it. If not, containerd falls back to the original registry. In this lab, all images that edge nodes need are already in Hauler, so every pull stays local.

### Verifying on a running edge node

```bash
# From the KVM host
ssh -i /root/.ssh/id_ed25519 root@192.168.122.31   # edge1

# Check the mirror config is in place
cat /etc/rancher/k3s/registries.yaml

# Check containerd is using the Hauler mirror
journalctl -u k3s | grep "pulling image" | head -10
```

---

## How Rancher and Elemental stay offline

Rancher was installed with `useBundledSystemChart=true`, which tells Rancher to use the system charts bundled inside its container image instead of fetching them from GitHub at runtime. Without this flag, Rancher would try to reach `github.com/rancher/system-charts` every time it starts catalog apps (monitoring, logging, alerting).

Verify the setting:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9

kubectl --kubeconfig=/etc/rancher/k3s/k3s.yaml \
  get deployment rancher -n cattle-system \
  -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n' | grep bundled
```

Elemental Operator images are pulled from `registry.suse.com` during the `elemental` phase install. After that, the running operator pods need no external access. MachineRegistrations are already created at deploy time, so students do not need to create them from scratch.

One thing to be aware of: the default **Elemental OS channel** image contains absolute `registry.suse.com` URLs. This lab does not use the OS channel for provisioning — edge nodes boot from EIB-built images that already contain the Elemental agent. Students never trigger a channel-based OS pull in these exercises.

---

## How Fleet stays offline

Fleet uses a GitRepo resource that points to `github.com/SUSE-Technical-Marketing/Alien-Geeko`. This is created automatically at deploy time.

Fleet on the **management cluster** does need to reach GitHub to clone the GitRepo. This is the one intentional internet dependency that remains during the exercises — it is a management-plane pull, not an edge-node pull.

Edge **clusters** do not access GitHub directly. Fleet pushes the workload bundle from the management cluster to each downstream cluster agent. The agent applies manifests locally. The only internet-facing action at the edge is the container image pull, which goes through Hauler.

If you need to run this lab with no management-plane internet access at all (fully isolated), set up a local Git server (Gitea or Forgejo) and point the GitRepo resource at it:

```bash
kubectl --kubeconfig=/etc/rancher/k3s/k3s.yaml \
  patch gitrepo alien-geeko -n fleet-default \
  --type=merge \
  -p '{"spec":{"repo":"http://192.168.122.X/your-org/Alien-Geeko.git"}}'
```

---

## Troubleshooting: what to check when something can't pull

If an EIB build fails or an edge node can't start a pod, check in this order:

**1. Is Hauler running?**

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20
systemctl is-active hauler-registry hauler-fileserver
```

Restart if needed:

```bash
systemctl restart hauler-registry hauler-fileserver
```

**2. Is the image in Hauler?**

```bash
curl -s http://localhost:5000/v2/_catalog | python3 -m json.tool
```

If the image is missing, add it:

```bash
hauler store add image <image:tag> --store /var/lib/hauler
systemctl restart hauler-registry
```

**3. Is the edge node's registries.yaml correct?**

```bash
ssh root@192.168.122.31   # replace with edge node IP
cat /etc/rancher/k3s/registries.yaml
```

If it is missing or pointing to the wrong host, recreate the EIB image with the correct `99-k3s-registries.sh` script and re-seed the node.

**4. Can the edge node reach Hauler?**

```bash
# From edge node
curl -s http://192.168.122.20:5000/v2/_catalog
curl -s http://192.168.122.20:8080/
```

If these fail, the lab network (virbr0 at `192.168.122.1`) may be down. Check on the KVM host:

```bash
ip link show virbr0
virsh net-list
```

---

## Component-level summary

| Component | Offline after deploy? | Notes |
|---|---|---|
| K3s management cluster | Yes | Installed; no runtime image pulls |
| Rancher Prime | Yes | `useBundledSystemChart=true`; system charts bundled |
| cert-manager | Yes | Installed; no runtime image pulls |
| Elemental Operator | Yes | Installed; no OS channel pull in these exercises |
| EIB (on eib VM) | Yes | Container image in Hauler; base images in Hauler |
| Hauler registry + fileserver | Yes | Self-contained on eib VM |
| Edge node K3s | Yes | Images via Hauler mirror; `registries.yaml` baked in by EIB |
| Alien-Geeko app (Fleet) | Yes | Image in Hauler; pulled via K3s mirror on edge nodes |
| Fleet GitRepo | **Partial** | Management cluster needs GitHub to clone; edge agents do not |

---

## Further reading

- [SUSE Edge 3.6 — Air-gapped deployments with EIB](https://documentation.suse.com/suse-edge/3.5/html/edge/id-air-gapped-deployments-with-edge-image-builder.html)
- [K3s — Air-gap install](https://docs.k3s.io/installation/airgap)
- [K3s — Private registry configuration](https://docs.k3s.io/installation/private-registry)
- [Rancher Prime 2.14 — Air-gap HA install](https://documentation.suse.com/cloudnative/rancher-manager/v2.14/en/installation-and-upgrade/other-installation-methods/air-gapped/install-rancher-ha.html)
- [Elemental — Air-gap install](https://documentation.suse.com/cloudnative/os-manager/1.6/en/airgap.html)
- [Hauler documentation](https://docs.hauler.dev/docs/intro)
