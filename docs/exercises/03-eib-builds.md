# Exercise 3 — Build four EIB images — one per node

**Time:** 50 min (builds run in background — you keep moving)  
**Previous:** [Exercise 2 — Configure Elemental](02-elemental-setup.md)  
**Next:** [Exercise 4 — Seed and boot nodes](04-boot-nodes.md)

---

Each node gets its own image with its own static IP baked in. Four builds, four definition files, exactly one NMState config in `network/` per run.

| Image | Node | Base | Output | IP | Kubernetes |
|---|---|---|---|---|---|
| `elemental-edge1.iso` | edge1 | SL Micro 6.2 SelfInstall ISO | ISO | 192.168.122.31 | None (Elemental) |
| `elemental-edge2.iso` | edge2 | SL Micro 6.2 SelfInstall ISO | ISO | 192.168.122.32 | None (Elemental) |
| `rke2-edge3.raw` | edge3 | SL Micro 6.2 Default RAW | RAW | 192.168.122.33 | RKE2 v1.35.3+rke2r3 |
| `k3s-edge4.raw` | edge4 | SL Micro 6.2 Default RAW | RAW | 192.168.122.34 | K3s v1.35.5+k3s1 |

**The build pattern for each node:**
1. Clear the `network/` dir and drop only that node's NMState file there
2. Start EIB in the background
3. Move on to the next node

**How the workspace is structured:**

The definition files, scripts, and network configs come from the `eib-config` Gitea repo you cloned in Exercise 2 to `/home/eib-workspace/`. The SL Micro base images (ISO and RAW) were pre-staged by the lab deploy at `/home/eib-config/base-images/` — they come from the Hauler file server and are ready to use.

EIB runs with two volume mounts:
- `/home/eib-workspace` → definitions, scripts, elemental config, network config (from Gitea)
- `/home/eib-config/base-images` → read-only base OS images (from Hauler)

SSH to the eib VM and stay there for this entire exercise:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20
```

Verify the workspace and base images are in place:

```bash
# Definition files and scripts from Gitea
ls /home/eib-workspace/

# SL Micro base images pre-staged from Hauler
ls -lh /home/eib-config/base-images/

# Create the active network/ dir (git-ignored — managed at build time)
mkdir -p /home/eib-workspace/network
```

---

## 3.1 Elemental image for edge1

This image boots, installs SL Micro to disk, and on first boot `elemental-register` phones home to the management cluster. Static IP 192.168.122.31 is baked in via NMState.

```bash
# Load edge1's network config — only one file in network/ at a time
rm -f /home/eib-workspace/network/*.yaml
cp /home/eib-workspace/network-configs/edge1.yaml /home/eib-workspace/network/
```

The `files:` section in the definition embeds the Elemental registration config you downloaded in Exercise 2 into `/oem/elemental.yaml` on the OS. When the node boots and `elemental-register` runs, it reads that file and knows where to call home. The NMState config in `network/` gives the node its static IP.

Start the build in the background:

```bash
podman run --rm --privileged \
  -v /home/eib-workspace:/eib:z \
  -v /home/eib-config/base-images:/eib/base-images:ro \
  registry.suse.com/edge/3.6/edge-image-builder:1.3.3.1 \
  build --definition-file elemental-edge1-definition.yaml \
  > /tmp/eib-edge1.log 2>&1 &

echo "edge1 build PID: $!"
tail -5 /tmp/eib-edge1.log
```

## 3.2 Elemental image for edge2

Same Elemental path, different IP (192.168.122.32):

```bash
tail -10 /tmp/eib-edge1.log

rm -f /home/eib-workspace/network/*.yaml
cp /home/eib-workspace/network-configs/edge2.yaml /home/eib-workspace/network/

podman run --rm --privileged \
  -v /home/eib-workspace:/eib:z \
  -v /home/eib-config/base-images:/eib/base-images:ro \
  registry.suse.com/edge/3.6/edge-image-builder:1.3.3.1 \
  build --definition-file elemental-edge2-definition.yaml \
  > /tmp/eib-edge2.log 2>&1 &

echo "edge2 build PID: $!"
```

## 3.3 Standalone RKE2 image for edge3

This image boots as a running single-node RKE2 cluster. No phone-home, no registration. The full Kubernetes stack is embedded in the disk. Static IP 192.168.122.33, hostname `edge3` set at first boot via combustion script.

```bash
rm -f /home/eib-workspace/network/*.yaml
cp /home/eib-workspace/network-configs/edge3.yaml /home/eib-workspace/network/
```

The `kubernetes:` section in the definition is the key difference from the Elemental builds. EIB downloads the RKE2 binary, all required container images from the Hauler OCI registry, and configures CRI-O. Everything lands in the disk image. The node does not need internet access or a running management plane — it starts Kubernetes on its own at boot.

```bash
tail -10 /tmp/eib-edge2.log

podman run --rm --privileged \
  -v /home/eib-workspace:/eib:z \
  -v /home/eib-config/base-images:/eib/base-images:ro \
  registry.suse.com/edge/3.6/edge-image-builder:1.3.3.1 \
  build --definition-file rke2-edge3-definition.yaml \
  > /tmp/eib-rke2.log 2>&1 &

echo "edge3 RKE2 build PID: $!"
```

## 3.4 Standalone K3s image for edge4

Same standalone pattern as edge3 but K3s instead of RKE2 — lighter footprint, faster first-boot cluster start:

```bash
rm -f /home/eib-workspace/network/*.yaml
cp /home/eib-workspace/network-configs/edge4.yaml /home/eib-workspace/network/

podman run --rm --privileged \
  -v /home/eib-workspace:/eib:z \
  -v /home/eib-config/base-images:/eib/base-images:ro \
  registry.suse.com/edge/3.6/edge-image-builder:1.3.3.1 \
  build --definition-file k3s-edge4-definition.yaml \
  > /tmp/eib-k3s.log 2>&1 &

echo "edge4 K3s build PID: $!"
```

## 3.5 Watch all four builds

```bash
watch -n5 'echo "=== edge1 (Elemental ISO) ===" && tail -3 /tmp/eib-edge1.log && \
           echo "=== edge2 (Elemental ISO) ===" && tail -3 /tmp/eib-edge2.log && \
           echo "=== edge3 (RKE2 RAW)     ===" && tail -3 /tmp/eib-rke2.log && \
           echo "=== edge4 (K3s RAW)      ===" && tail -3 /tmp/eib-k3s.log'
```

A successful build ends with:

```
Build complete, the image can be found at: <outputImageName>
```

**Why builds take different amounts of time:** The Elemental ISOs are fast — EIB wraps an existing ISO with a config file and the NMState network config. The RKE2 and K3s RAW builds are slower because EIB downloads and embeds all required container images from the Hauler OCI registry. The `99-k3s-registries.sh` script tells EIB to pull from `192.168.122.20:5000` instead of the internet.

When all four builds are done, verify the output files:

```bash
ls -lh /home/eib-workspace/elemental-edge1.iso \
        /home/eib-workspace/elemental-edge2.iso \
        /home/eib-workspace/rke2-edge3.raw \
        /home/eib-workspace/k3s-edge4.raw
```

Exit back to the KVM host:

```bash
exit
```

---

**Next:** [Exercise 4 — Seed and boot nodes](04-boot-nodes.md)
