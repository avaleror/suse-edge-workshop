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
| `rke2-edge3.raw` | edge3 | SL Micro 6.2 Cloud RAW | RAW | 192.168.122.33 | RKE2 v1.35.3+rke2r3 |
| `k3s-edge4.raw` | edge4 | SL Micro 6.2 Cloud RAW | RAW | 192.168.122.34 | K3s v1.35.3+k3s1 |

**The build pattern for each node:**
1. Clear the `network/` dir and drop only that node's NMState file there
2. Write the node-specific definition yaml
3. Start EIB in the background
4. Move on to the next node

SSH to the eib VM and stay there for this entire exercise:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20
```

Download the base images from Hauler:

```bash
curl -s http://localhost:8080/ | grep -E "iso|raw|qcow"

mkdir -p /home/eib-config/base-images /home/eib-config/network /home/eib-config/scripts

# SL Micro SelfInstall ISO for Elemental nodes
curl -fsSL "http://localhost:8080/SL-Micro.x86_64-6.2-Base-SelfInstall-GM.install.iso" \
  -o /home/eib-config/base-images/SL-Micro.x86_64-6.2-Base-SelfInstall-GM.install.iso

# SL Micro Cloud RAW for standalone cluster nodes
curl -fsSL "http://localhost:8080/SL-Micro.x86_64-6.2-Default.raw" \
  -o /home/eib-config/base-images/SL-Micro.x86_64-6.2-Default.raw

ls -lh /home/eib-config/base-images/
```

---

## 3.1 Elemental image for edge1

This image boots, installs SL Micro to disk, and on first boot `elemental-register` phones home to the management cluster. Static IP 192.168.122.31 is baked in via NMState.

```bash
# Load edge1's network config — only one file in network/ at a time
rm -f /home/eib-config/network/*.yaml
cp /home/eib-config/network-configs/edge1.yaml /home/eib-config/network/

cat > /home/eib-config/elemental-edge1-definition.yaml << 'EOF'
apiVersion: 1.0

image:
  imageType: iso
  arch: x86_64
  baseImage: SL-Micro.x86_64-6.2-Base-SelfInstall-GM.install.iso
  outputImageName: elemental-edge1.iso

operatingSystem:
  kernelArgs:
    - net.ifnames=0
  files:
    - sourcePath: elemental/elemental_config.yaml
      destinationPath: /oem/elemental.yaml

embeddedArtifacts:
  registries:
    urls:
      - 192.168.122.20:5000
EOF
```

The `files:` section embeds the Elemental registration config you downloaded in Exercise 2 into `/oem/elemental.yaml` on the OS. When the node boots and `elemental-register` runs, it reads that file and knows where to call home. The NMState config in `network/` gives the node its static IP.

Start the build in the background:

```bash
podman run --rm --privileged \
  -v /home/eib-config:/eib:z \
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

rm -f /home/eib-config/network/*.yaml
cp /home/eib-config/network-configs/edge2.yaml /home/eib-config/network/

cat > /home/eib-config/elemental-edge2-definition.yaml << 'EOF'
apiVersion: 1.0

image:
  imageType: iso
  arch: x86_64
  baseImage: SL-Micro.x86_64-6.2-Base-SelfInstall-GM.install.iso
  outputImageName: elemental-edge2.iso

operatingSystem:
  kernelArgs:
    - net.ifnames=0
  files:
    - sourcePath: elemental/elemental_config.yaml
      destinationPath: /oem/elemental.yaml

embeddedArtifacts:
  registries:
    urls:
      - 192.168.122.20:5000
EOF

podman run --rm --privileged \
  -v /home/eib-config:/eib:z \
  registry.suse.com/edge/3.6/edge-image-builder:1.3.3.1 \
  build --definition-file elemental-edge2-definition.yaml \
  > /tmp/eib-edge2.log 2>&1 &

echo "edge2 build PID: $!"
```

## 3.3 Standalone RKE2 image for edge3

This image boots as a running single-node RKE2 cluster. No phone-home, no registration. The full Kubernetes stack is embedded in the disk. Static IP 192.168.122.33, hostname `edge3` set at first boot.

```bash
rm -f /home/eib-config/network/*.yaml
cp /home/eib-config/network-configs/edge3.yaml /home/eib-config/network/

# Hostname script — runs at first boot via combustion
cat > /home/eib-config/scripts/10-hostname-edge3.sh << 'EOF'
#!/bin/bash
hostnamectl set-hostname edge3
EOF
chmod +x /home/eib-config/scripts/10-hostname-edge3.sh

cat > /home/eib-config/rke2-edge3-definition.yaml << 'EOF'
apiVersion: 1.0

image:
  imageType: raw
  arch: x86_64
  baseImage: SL-Micro.x86_64-6.2-Default.raw
  outputImageName: rke2-edge3.raw

operatingSystem:
  kernelArgs:
    - net.ifnames=0
  scripts:
    - 10-hostname-edge3.sh
    - 99-k3s-registries.sh

kubernetes:
  version: v1.35.3+rke2r3

embeddedArtifacts:
  registries:
    urls:
      - 192.168.122.20:5000
EOF
```

The `kubernetes:` section is the key difference from the Elemental builds. EIB downloads the RKE2 binary, all required container images, and configures CRI-O. Everything lands in the disk image. The node does not need internet access or a running management plane — it starts Kubernetes on its own at boot.

```bash
tail -10 /tmp/eib-edge2.log

podman run --rm --privileged \
  -v /home/eib-config:/eib:z \
  registry.suse.com/edge/3.6/edge-image-builder:1.3.3.1 \
  build --definition-file rke2-edge3-definition.yaml \
  > /tmp/eib-rke2.log 2>&1 &

echo "edge3 RKE2 build PID: $!"
```

## 3.4 Standalone K3s image for edge4

Same standalone pattern as edge3 but K3s instead of RKE2 — lighter footprint, faster first-boot cluster start:

```bash
rm -f /home/eib-config/network/*.yaml
cp /home/eib-config/network-configs/edge4.yaml /home/eib-config/network/

cat > /home/eib-config/scripts/10-hostname-edge4.sh << 'EOF'
#!/bin/bash
hostnamectl set-hostname edge4
EOF
chmod +x /home/eib-config/scripts/10-hostname-edge4.sh

cat > /home/eib-config/k3s-edge4-definition.yaml << 'EOF'
apiVersion: 1.0

image:
  imageType: raw
  arch: x86_64
  baseImage: SL-Micro.x86_64-6.2-Default.raw
  outputImageName: k3s-edge4.raw

operatingSystem:
  kernelArgs:
    - net.ifnames=0
  scripts:
    - 10-hostname-edge4.sh
    - 99-k3s-registries.sh

kubernetes:
  version: v1.35.3+k3s1

embeddedArtifacts:
  registries:
    urls:
      - 192.168.122.20:5000
EOF

podman run --rm --privileged \
  -v /home/eib-config:/eib:z \
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
ls -lh /home/eib-config/elemental-edge1.iso \
        /home/eib-config/elemental-edge2.iso \
        /home/eib-config/rke2-edge3.raw \
        /home/eib-config/k3s-edge4.raw
```

Exit back to the KVM host:

```bash
exit
```

---

**Next:** [Exercise 4 — Seed and boot nodes](04-boot-nodes.md)
