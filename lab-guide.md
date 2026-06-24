# SUSE Edge 3.6 Rodeo — Lab Guide

**Version:** 2.0 | **Date:** June 2026 | **Duration:** ~2.5 hours  
**Author:** Andres Valero, Principal Technology Advocate, SUSE

---

## Before we start

This is a hands-on lab. You will build OS images, boot edge nodes, register them against a central management plane, and deploy workloads across a mixed fleet — all from one terminal. The goal is not to click through slides. It is to leave knowing how SUSE Edge actually works at the system level.

Your environment is already partially running. A bare metal host is running five KVM virtual machines: a management cluster, an image-builder VM, and four edge nodes that are currently off. You will turn them on one group at a time, after building the right OS image for each.

**Access credentials** are in `~/.rodeo/secrets.yaml` on the host.

---

## The scenario

AeroGrid is a regional airport operator. They run self-service kiosks, baggage tracking terminals, and cargo management dashboards across 47 airport locations. Each location runs between three and five Linux nodes. IT manages them centrally from HQ.

The problem: each site has its own config history. Some nodes have not been updated in years. Security audit is next month. The CTO's mandate is that every edge node should have the same OS, installed from the same image, managed from a single control plane. New nodes at any site should onboard automatically with no hands-on config at the remote end.

AeroGrid also has two types of sites: larger hubs that need Kubernetes clusters capable of running containerized applications, and smaller branch terminals that just need a managed Linux node with lightweight workloads. SUSE Edge handles both from the same management plane.

---

## Lab topology

```
KVM host (bare metal)
│
├── rancher   192.168.122.9    Rancher Prime 2.14.1 + Elemental Operator 1.9.0 + Fleet
├── eib       192.168.122.20   EIB 1.3.3.1 + Hauler 1.2.2 (local artifact registry)
│
├── edge1     192.168.122.31   [OFF] vTPM 2.0, UEFI  →  Elemental onboarding  →  K3s cluster
├── edge2     192.168.122.32   [OFF] vTPM 2.0, UEFI  →  Elemental onboarding  →  standalone
├── edge3     192.168.122.33   [OFF] UEFI             →  EIB standalone        →  RKE2 cluster
└── edge4     192.168.122.34   [OFF] UEFI             →  EIB standalone        →  K3s cluster
```

**Two provisioning paths run in parallel in this lab:**

- **Elemental path (edge1, edge2):** Phone-home onboarding via TPM. The node boots, the Elemental agent contacts the registration endpoint, and the management cluster takes control. Cluster provisioning happens from Rancher — not from the node.
- **EIB standalone path (edge3, edge4):** The image is self-contained. Boot it, and you have a running Kubernetes cluster. No phone-home. No registration step. Import into Rancher afterwards if you want central management.

Understanding when to use each path is one of the core takeaways from this lab.

---

## Lab exercises

| # | Exercise | Time |
|---|---|---|
| 1 | Tour the environment | 15 min |
| 2 | Configure Elemental, the registration endpoint, and the node network plan | 30 min |
| 3 | Build four EIB images — one per node, each with its own network config | 50 min |
| 4 | Seed and boot the four nodes | 25 min |
| 5 | Provision a K3s cluster on an Elemental node | 20 min |
| 6 | Deploy workloads via Fleet | 20 min |

Total run time assumes EIB builds overlap with other steps. Each build runs in the background while you move on. Start each build, check it is progressing cleanly, then move to the next.

---

## Exercise 1 — Tour the environment

Get your bearings before touching anything. From the KVM host:

```bash
# What VMs are running, what is off
virsh list --all

# Check the management cluster
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get nodes && kubectl get pods -A | grep -E 'rancher|elemental|cert-manager|fleet'"
```

You should see one K3s node (the rancher VM itself) with pods for Rancher Prime, cert-manager, the Elemental Operator, and Fleet. This is your management cluster. It manages everything else.

Now check the eib VM — your image factory:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "podman images | grep edge-image-builder && \
   systemctl is-active hauler-registry hauler-fileserver && \
   ls /home/eib-config/"
```

Hauler is running two services: an OCI registry on port 5000 (container images) and an HTTP fileserver on port 8080 (binary artifacts and OS base images). Everything used in this lab comes from one of these two endpoints. The eib VM does not need internet access once Hauler is populated.

Check what is in the Hauler store:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "hauler store info --store /var/lib/hauler 2>/dev/null | head -40"
```

You should see the EIB container image, the Elemental register agent, the Alien-Geeko app image, and the SL Micro base image. These were pre-staged by the instructor before the lab started.

---

## Exercise 2 — Configure Elemental and the registration endpoint

This exercise runs before you build any images. The reason: the Elemental path needs a registration URL embedded in the OS image. You cannot build the image before you have the endpoint configured and know its URL.

### 2.1 What Elemental does

Elemental is the component that handles autonomous node onboarding. An edge node boots, the `elemental-register` agent starts, and it contacts a registration URL that you define on the management cluster. The Elemental Operator validates the node's identity (via TPM in this lab), creates a `MachineInventory` record, and from that point the management cluster controls the node's lifecycle.

You do not SSH into the node. You do not run scripts on it. The node phones home and identifies itself.

### 2.2 Inspect the MachineRegistration

A `MachineRegistration` defines who is allowed to register and what labels the node gets when it does. One has already been created for this lab. Open it:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration -n fleet-default -o yaml"
```

Look at the spec. Key fields:

```yaml
spec:
  machineName: "${System Information/Manufacturer}-${System Information/UUID}"
  config:
    elemental:
      registration:
        auth: tpm                   # TPM-based identity — hardware-bound
      install:
        powerOff: true              # Power off after install, before first-run reboot
  machineInventoryLabels:
    manufacturer: "${System Information/Manufacturer}"
    productName: "${System Information/Product Name}"
    locationID: ""                  # You will fill this in per node
```

`auth: tpm` means the node's registration token is derived from its TPM. A cloned disk on a different machine will fail to register because the TPM identity will not match. For edge security — remote sites where you cannot guarantee physical security — this matters.

### 2.3 Add labels for cluster assignment

You are going to use labels to tell the management cluster which nodes should form clusters. Add the labels the MachineInventorySelectorTemplate will match later:

```bash
# Check the current registration name
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration -n fleet-default"
```

Note the registration name (something like `suse-edge-reg-1`). Now look at the registration URL — you need this for the EIB image build in Exercise 3:

```bash
REGURL=$(ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration suse-edge-reg-1 \
   -n fleet-default \
   -o jsonpath='{.status.registrationURL}'")

echo "Registration URL: $REGURL"
```

**Copy this URL.** You will embed it in the Elemental EIB image definition in the next exercise.

### 2.4 Download the registration config

The registration URL does more than identify the endpoint. Fetching it returns a full YAML config blob that `elemental-register` uses at boot. Download it now:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 "
  mkdir -p /home/eib-config/elemental
  curl -k \"$REGURL\" -o /home/eib-config/elemental/elemental_config.yaml
  cat /home/eib-config/elemental/elemental_config.yaml
"
```

This file contains the registration URL, the CA certificate for the management cluster's TLS, and the config that `elemental-register` needs to authenticate via TPM. EIB will embed it into the OS image so it is present at first boot — no network config required at the remote site.

### 2.5 Verify Elemental Operator is healthy

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get pods -n cattle-elemental-system"
```

Both `elemental-operator` and `elemental-operator-webhook` should be `Running`. If either is not, stop and flag it before building images — nodes cannot register against a broken operator.

### 2.6 Node network plan

Each node gets a static IP baked into its disk image by EIB. No DHCP dependency at the edge, and addresses you can put in monitoring config before the node ever boots.

| Node | IP | Prefix | Gateway | DNS | MAC |
|---|---|---|---|---|---|
| edge1 | 192.168.122.31 | /24 | 192.168.122.1 | 192.168.122.1 | 02:00:00:0E:62:A1 |
| edge2 | 192.168.122.32 | /24 | 192.168.122.1 | 192.168.122.1 | 02:00:00:0E:62:A2 |
| edge3 | 192.168.122.33 | /24 | 192.168.122.1 | 192.168.122.1 | 02:00:00:0E:62:A3 |
| edge4 | 192.168.122.34 | /24 | 192.168.122.1 | 192.168.122.1 | 02:00:00:0E:62:A4 |

EIB picks up any YAML files in the `network/` subdirectory of its config dir and processes them as NMState configs. Each build will use exactly one file — one node, one IP. Create all four files now so they are ready when you start the builds:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20

mkdir -p /home/eib-config/network-configs

cat > /home/eib-config/network-configs/edge1.yaml << 'EOF'
interfaces:
  - name: eth0
    type: ethernet
    state: up
    ipv4:
      address:
        - ip: 192.168.122.31
          prefix-length: 24
      dhcp: false
      enabled: true
routes:
  config:
    - destination: 0.0.0.0/0
      next-hop-address: 192.168.122.1
      next-hop-interface: eth0
dns-resolver:
  config:
    servers:
      - 192.168.122.1
EOF

cat > /home/eib-config/network-configs/edge2.yaml << 'EOF'
interfaces:
  - name: eth0
    type: ethernet
    state: up
    ipv4:
      address:
        - ip: 192.168.122.32
          prefix-length: 24
      dhcp: false
      enabled: true
routes:
  config:
    - destination: 0.0.0.0/0
      next-hop-address: 192.168.122.1
      next-hop-interface: eth0
dns-resolver:
  config:
    servers:
      - 192.168.122.1
EOF

cat > /home/eib-config/network-configs/edge3.yaml << 'EOF'
interfaces:
  - name: eth0
    type: ethernet
    state: up
    ipv4:
      address:
        - ip: 192.168.122.33
          prefix-length: 24
      dhcp: false
      enabled: true
routes:
  config:
    - destination: 0.0.0.0/0
      next-hop-address: 192.168.122.1
      next-hop-interface: eth0
dns-resolver:
  config:
    servers:
      - 192.168.122.1
EOF

cat > /home/eib-config/network-configs/edge4.yaml << 'EOF'
interfaces:
  - name: eth0
    type: ethernet
    state: up
    ipv4:
      address:
        - ip: 192.168.122.34
          prefix-length: 24
      dhcp: false
      enabled: true
routes:
  config:
    - destination: 0.0.0.0/0
      next-hop-address: 192.168.122.1
      next-hop-interface: eth0
dns-resolver:
  config:
    servers:
      - 192.168.122.1
EOF

ls -la /home/eib-config/network-configs/
exit
```

The interface name `eth0` comes from `net.ifnames=0` in the EIB definition's `kernelArgs`. Without that kernel arg, SL Micro would name the first NIC something like `ens3` depending on PCI bus order. With the arg set, the old naming convention applies consistently across builds.

---

## Exercise 3 — Build four EIB images — one per node

Each node gets its own image with its own static IP baked in. That means four builds, four definition files, and exactly one NMState config in `network/` for each run.

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

Check the base images available from Hauler and download them:

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

### 3.1 Elemental image for edge1

This image boots, installs SL Micro to disk, and on first boot `elemental-register` phones home to the management cluster. The static IP 192.168.122.31 is baked in via NMState so the node is reachable at that address the moment it finishes installing.

Set the network config and create the definition:

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

The `files:` section embeds the Elemental registration config you downloaded in Exercise 2 into `/oem/elemental.yaml` on the OS. When the node boots and `elemental-register` runs, it reads that file and knows where to call home. The NMState config in `network/` is what gives the node its static IP.

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

Move on to the next definition while this runs. ISO builds are fast — usually a few minutes.

### 3.2 Elemental image for edge2

Same Elemental path as edge1 but a different IP (192.168.122.32). Two separate ISOs so each node has its own identity from the moment it boots.

```bash
# Check edge1 build is progressing before starting edge2
tail -10 /tmp/eib-edge1.log

# Swap in edge2's network config
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

### 3.3 Standalone RKE2 image for edge3

This image boots and comes up as a running single-node RKE2 cluster — no registration, no phone-home. The full Kubernetes stack is embedded in the disk. Static IP 192.168.122.33, hostname `edge3` set at first boot.

```bash
# Swap in edge3's network config
rm -f /home/eib-config/network/*.yaml
cp /home/eib-config/network-configs/edge3.yaml /home/eib-config/network/

# Hostname script — EIB embeds this as a combustion script, runs at first boot
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
# Check edge2 build is moving before adding another job
tail -10 /tmp/eib-edge2.log

podman run --rm --privileged \
  -v /home/eib-config:/eib:z \
  registry.suse.com/edge/3.6/edge-image-builder:1.3.3.1 \
  build --definition-file rke2-edge3-definition.yaml \
  > /tmp/eib-rke2.log 2>&1 &

echo "edge3 RKE2 build PID: $!"
```

### 3.4 Standalone K3s image for edge4

Same standalone pattern as edge3 but K3s instead of RKE2. Lighter footprint, faster first-boot cluster start.

```bash
# Swap in edge4's network config
rm -f /home/eib-config/network/*.yaml
cp /home/eib-config/network-configs/edge4.yaml /home/eib-config/network/

# Hostname script for edge4
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

### 3.5 Watch all four builds

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

## Exercise 4 — Seed and boot the four nodes

You have four nodes, two image formats, and two different boot workflows. This exercise walks through both.

### 4.1 Elemental nodes (edge1 and edge2) — ISO workflow

edge1 and edge2 each boot from their own ISO. The ISO contains a self-installer: it boots, writes SL Micro to the virtual disk with the static IP already configured, powers off, and the node reboots into the installed OS where `elemental-register` runs.

Pull each ISO from the eib VM and attach it as a virtual CDROM:

```bash
# edge1 gets its specific ISO (192.168.122.31 baked in)
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-config/elemental-edge1.iso \
  --local-from-eib \
  --nodes edge1 \
  --yes

# edge2 gets its specific ISO (192.168.122.32 baked in)
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-config/elemental-edge2.iso \
  --local-from-eib \
  --nodes edge2 \
  --yes
```

Each command SCPs the ISO, attaches it as a CDROM (sda, boot-order-1) to that node's libvirt domain, and creates a blank `vda.qcow2` for the OS to install into.

Start both nodes:

```bash
virsh start edge1
virsh start edge2
```

Watch the boot on edge1's serial console (Ctrl+] to exit):

```bash
virsh console edge1
```

You will see OVMF, then the SL Micro installer, then a text progress bar as the OS writes to disk. When you see `System is shutting down` the install is done. The node powers off.

Wait for both to shut down:

```bash
watch virsh list --all | grep edge
```

When both show `shut off`, eject the ISOs and restore disk-first boot:

```bash
rodeo eject-iso --nodes edge1,edge2 --yes
```

Now start them again — this time they boot from the installed disk:

```bash
virsh start edge1
virsh start edge2
```

From this point, the nodes are running SL Micro and `elemental-register` is starting up. Watch for DHCP leases:

```bash
watch virsh net-dhcp-leases default | grep -E "edge|0e:62:a"
```

### 4.2 Standalone nodes (edge3 and edge4) — RAW workflow

edge3 and edge4 use pre-built RAW images. No installer, no reboot cycle. The node boots from a ready disk image with Kubernetes already configured to start.

Pull and thin-clone both images from the eib VM:

```bash
# edge3 gets the RKE2 image
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-config/rke2-edge3.raw \
  --local-from-eib \
  --nodes edge3 \
  --yes

# edge4 gets the K3s image
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-config/k3s-edge4.raw \
  --local-from-eib \
  --nodes edge4 \
  --yes
```

Check the disk layout:

```bash
ls -lh /var/lib/libvirt/images/rke2-edge3-base.qcow2 \
        /var/lib/libvirt/images/edge3-vda.qcow2 \
        /var/lib/libvirt/images/k3s-edge4-base.qcow2 \
        /var/lib/libvirt/images/edge4-vda.qcow2
```

The base images hold the full content. The per-node `vda.qcow2` files are thin clones — a few megabytes each — that only store writes that diverge from the base. If you had ten edge3-class nodes, you would clone the base ten times at near-zero disk cost.

Start edge3 and edge4:

```bash
virsh start edge3
virsh start edge4
```

These nodes take 3-5 minutes to come up fully. RKE2 and K3s do their first-run initialization: generating TLS certificates, starting the control plane, and marking the node Ready.

Check from edge3 (once it gets an IP from DHCP):

```bash
# Get edge3 IP
virsh net-dhcp-leases default | grep "0e:62:a3"

# SSH in and check Kubernetes
ssh -i /root/.ssh/id_ed25519 root@192.168.122.33 \
  "kubectl get nodes"
```

You should see one node in Ready state with the RKE2 version. Do the same for edge4 at 192.168.122.34 and confirm K3s is running there.

---

## Exercise 5 — Provision a K3s cluster on an Elemental node

edge1 and edge2 are now running SL Micro. The `elemental-register` agent has phoned home and registered them with the Elemental Operator. But they are not yet Kubernetes nodes — that happens in this exercise.

### 5.1 Check MachineInventory

Watch for the registered nodes on the management cluster:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineinventory -n fleet-default -w"
```

You should see one or two entries appear — one per node that has completed registration. Each row is a node that has proven its TPM identity and is now waiting to be told what to do.

In the Rancher UI: go to **OS Management > MachineInventory**. The same records appear there with more detail — hardware info, labels inherited from the MachineRegistration, and status.

### 5.2 Create a MachineInventorySelectorTemplate

This resource links a label selector to a machine pool. Any `MachineInventory` that matches the selector becomes a candidate node for a cluster.

Apply this on the management cluster:

```bash
cat << 'EOF' | ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 "kubectl apply -f -"
apiVersion: elemental.cattle.io/v1beta1
kind: MachineInventorySelectorTemplate
metadata:
  name: aerogrid-hub-selector
  namespace: fleet-default
spec:
  template:
    spec:
      selector:
        matchLabels:
          site-role: hub
EOF
```

### 5.3 Label edge1 for cluster assignment

Right now neither node matches `site-role: hub`. Label edge1 to trigger the match:

```bash
# Get edge1's MachineInventory name
EDGE1=$(ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineinventory -n fleet-default \
   -o jsonpath='{.items[0].metadata.name}'")

echo "edge1 MachineInventory: $EDGE1"

# Label it
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl label machineinventory $EDGE1 \
   -n fleet-default \
   site-role=hub \
   demo=true \
   edge-type=x86-cluster"
```

### 5.4 Create the cluster

Now create a `Cluster` resource that uses the selector template as its machine pool:

```bash
cat << 'EOF' | ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 "kubectl apply -f -"
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: aerogrid-hub-01
  namespace: fleet-default
spec:
  kubernetesVersion: v1.35.3+k3s1
  rkeConfig:
    machinePools:
      - name: hub-nodes
        quantity: 1
        etcdRole: true
        controlPlaneRole: true
        workerRole: true
        machineConfigRef:
          kind: MachineInventorySelectorTemplate
          name: aerogrid-hub-selector
          apiVersion: elemental.cattle.io/v1beta1
EOF
```

### 5.5 Watch the cluster provision

In Rancher UI: go to **Cluster Management**. You will see `aerogrid-hub-01` appear with status `Provisioning`. The management cluster is now remotely installing K3s on edge1 via the Elemental system agent.

From the terminal:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get cluster -n fleet-default -w"
```

Provisioning takes 5-10 minutes. When the status changes to `Active`, the cluster is ready.

**What just happened:** you provisioned a Kubernetes cluster on a remote node without touching the node. The node registered itself, you applied a label, and the management cluster did the rest. This is the Elemental model for large-scale edge deployments.

---

## Exercise 6 — Deploy workloads via Fleet

You now have four running nodes:
- **aerogrid-hub-01** on edge1 (Elemental-provisioned K3s)
- **edge2** registered but not yet assigned to a cluster
- **edge3** running RKE2 standalone (not yet in Rancher)
- **edge4** running K3s standalone (not yet in Rancher)

Fleet is already watching for clusters with the right labels. A `GitRepo` called `alien-geeko` is pre-configured to target any cluster labeled `demo=true` and `edge-type=x86-cluster`.

### 6.1 Trigger Fleet deployment on aerogrid-hub-01

edge1 already has the `demo=true` and `edge-type=x86-cluster` labels from the MachineInventorySelectorTemplate labeling step. Once the cluster is active and imported into Rancher's fleet-default workspace, Fleet should pick it up automatically.

Check in the Rancher UI: **Continuous Delivery > Git Repos > alien-geeko**. Look at the target clusters section. When `aerogrid-hub-01` appears in the list and its status moves to `Active`, Fleet is deploying the app.

If it does not appear automatically:

```bash
# Make sure the cluster has the right labels
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl label cluster aerogrid-hub-01 -n fleet-default \
   demo=true edge-type=x86-cluster --overwrite"
```

### 6.2 Import edge3 and edge4 into Rancher (optional)

edge3 and edge4 are running standalone clusters that Rancher does not know about yet. Import them:

In Rancher UI: **Cluster Management > Import Existing**. Give each cluster a name (`aerogrid-branch-rke2`, `aerogrid-branch-k3s`). Rancher generates a `kubectl apply` command with a registration manifest. Run it on each node:

```bash
# On edge3 (RKE2)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.33 \
  "kubectl apply -f <paste-the-registration-manifest-url>"

# On edge4 (K3s)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.34 \
  "kubectl apply -f <paste-the-registration-manifest-url>"
```

Once imported, label them for Fleet:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 "
  kubectl label cluster aerogrid-branch-rke2 -n fleet-default \
    demo=true edge-type=x86-cluster --overwrite
  kubectl label cluster aerogrid-branch-k3s -n fleet-default \
    demo=true edge-type=x86-cluster --overwrite
"
```

Fleet will deploy Alien-Geeko to each cluster the moment the labels match. Go to **Continuous Delivery > Git Repos > alien-geeko** and watch the bundle status update for each cluster.

### 6.3 Verify deployment

Alien-Geeko is a Node.js web terminal that shows live Kubernetes cluster vitals. Once Fleet deploys it, find the NodePort and open it in a browser.

For each cluster, run:

```bash
# On the management cluster, check fleet bundle status
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get bundle -n fleet-default | grep alien"

# On edge1 (via management cluster kubeconfig), find the NodePort
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get svc -A | grep alien"
```

Open `http://192.168.122.31:<nodeport>` for the edge1 deployment.

---

## What you just built

Four nodes, three image types, two provisioning paths, one management plane. Let me walk back through the architecture so the pieces connect.

**The image-first model:** every node in this lab booted from a purpose-built disk image. No manual SSH config, no `apt install`, no Ansible playbook applied to a running system. The image IS the configuration. If a node breaks, you re-image it. If a new site opens, you ship the image. The management cluster tells nodes what cluster to join — it does not configure the OS.

**Why two paths exist:** Elemental (edge1/edge2) is for sites where you do not know the node's final role at image-build time. You build one generic SL Micro image, ship it everywhere, and the management cluster decides what each node becomes after it registers. EIB standalone (edge3/edge4) is for sites where the role is known upfront — you bake K3s or RKE2 into the image and the node is a cluster the moment it boots.

**Why TPM matters:** without TPM, a cloned disk can register as any node in your fleet. With `auth: tpm`, the registration token is derived from hardware. You can revoke a specific node's registration by deleting its `MachineInventory`. Physical theft of the hardware does not compromise other nodes.

**Why Hauler is there:** in a real deployment, remote sites have unreliable internet. Hauler is a portable artifact store. You populate it once at HQ, copy it to a USB drive, and the site's nodes pull everything locally. Build K3s into the image from Hauler, the image never touches the internet. In this lab, Hauler was pre-populated by the instructor — in production, you run `hauler store sync` to update it and redistribute.

---

## Two-minute comparison

| | Elemental (edge1/edge2) | EIB Standalone (edge3/edge4) |
|---|---|---|
| Node role decided | At provisioning time (after registration) | At image-build time |
| Cluster control | Rancher provisions and manages K8s | Node runs its own K8s; optionally imported |
| Onboarding | Automatic via TPM phone-home | Manual import OR pre-registered via EIB |
| Good for | Mixed-role sites, large fleets, dynamic scaling | Fixed-function nodes, strict airgap, predictable workloads |
| Recovery | Re-register = re-provision from management | Re-image = re-flash disk |

---

## Bonus exercises

### Try a second Elemental cluster with RKE2

Label edge2 with `site-role: hub` and create a second `Cluster` resource pointing at the same `aerogrid-hub-selector` template but with `quantity: 1` targeting edge2. Rancher will provision RKE2 on edge2.

### See what a failed registration looks like

Power off edge1, clear its TPM state, and boot it again:

```bash
virsh destroy edge1
# Clearing a vTPM: delete the swtpm state directory
rm -rf /var/lib/libvirt/swtpm/<edge1-uuid>
virsh start edge1
```

Watch the Elemental logs on the management cluster:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl logs -n cattle-elemental-system -l app=elemental-operator -f"
```

The registration attempt will be rejected because the TPM identity changed. The `MachineInventory` record will not update. This is the expected failure mode for hardware replacement at a remote site — you control the replacement from the management cluster, not from the site.

---

## Lab reference

### Access

| Resource | Value |
|---|---|
| KVM host IP | 192.168.1.241 (or lab-assigned) |
| Rancher UI | `https://<host-ip>:30002` |
| Rancher user | `admin` |
| Rancher password | `cat ~/.rodeo/secrets.yaml` |
| Management cluster SSH | `ssh -i /root/.ssh/id_ed25519 root@192.168.122.9` |
| EIB VM SSH | `ssh -i /root/.ssh/id_ed25519 root@192.168.122.20` |
| Hauler OCI registry | `http://192.168.122.20:5000` |
| Hauler fileserver | `http://192.168.122.20:8080` |

### Node reference

| Node | IP | MAC | Type | Image |
|---|---|---|---|---|
| edge1 | 192.168.122.31 | 02:00:00:0E:62:A1 | Elemental | elemental-edge1.iso |
| edge2 | 192.168.122.32 | 02:00:00:0E:62:A2 | Elemental | elemental-edge2.iso |
| edge3 | 192.168.122.33 | 02:00:00:0E:62:A3 | EIB standalone | rke2-edge3.raw |
| edge4 | 192.168.122.34 | 02:00:00:0E:62:A4 | EIB standalone | k3s-edge4.raw |

### Key commands

```bash
# EIB build (run on eib VM, /home/eib-config is the working dir)
podman run --rm --privileged \
  -v /home/eib-config:/eib:z \
  registry.suse.com/edge/3.6/edge-image-builder:1.3.3.1 \
  build --definition-file <definition-file.yaml>

# Seed edge nodes from eib VM (ISO or RAW auto-detected)
rodeo pull-edge-image --config-dir /root/rodeo-lab --nodes edge1,edge2 --yes

# ISO workflow: eject after install, then start from disk
rodeo eject-iso --nodes edge1,edge2 --yes

# Watch Elemental registration
kubectl get machineinventory -n fleet-default -w

# Get registration URL
kubectl get machineregistration suse-edge-reg-1 \
  -n fleet-default \
  -o jsonpath='{.status.registrationURL}'

# Watch cluster provisioning
kubectl get cluster -n fleet-default -w

# Watch Fleet bundle status
kubectl get bundle -n fleet-default
```

### Component versions

| Component | Version |
|---|---|
| Rancher Prime | 2.14.1 |
| K3s (management cluster) | v1.35.3+k3s1 |
| cert-manager | v1.20.1 |
| Elemental Operator | 1.9.0 |
| Edge Image Builder | 1.3.3.1 |
| Hauler | 1.2.2 |
| SUSE Linux Micro | 6.2 |

---

## Instructor notes

**Before the lab starts:**

1. Run `rodeo up --profile suse-edge` on the bare metal host to deploy the management stack and the four edge VM shells.
2. Pre-stage SL Micro images in Hauler on the eib VM:
   ```bash
   ssh root@192.168.122.20
   hauler store add file \
     "https://download.suse.com/SL-Micro/6.2/SL-Micro.x86_64-6.2-Base-SelfInstall-GM.install.iso" \
     --store /var/lib/hauler
   hauler store add file \
     "https://download.suse.com/SL-Micro/6.2/SL-Micro.x86_64-6.2-Default.raw" \
     --store /var/lib/hauler
   systemctl restart hauler-fileserver
   ```
3. Verify the Elemental Operator is running and the MachineRegistration exists.
4. Share access credentials and host IP with students.

**Timing:** EIB builds for RAW images with embedded Kubernetes take 15-30 minutes each. Start the Elemental ISO build first (Exercise 3.1), then run through Exercise 2 details while it builds. Start the RKE2 and K3s builds at the end of Exercise 3. Exercises 4, 5, and 6 can begin once the Elemental ISO build completes.

---

*Lab built with rodeo-cli v0.10.x. Source: https://github.com/avaleror/rodeo-cli*
