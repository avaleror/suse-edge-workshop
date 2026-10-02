# SUSE Edge 3.7 Rodeo: Lab Guide

**Version:** 3.7 | **Date:** October 2026 | **Duration:** ~2.5 hours  
**Author:** Andres Valero, Principal Technology Advocate, SUSE

---

## Before we start

This is a hands-on lab. You will build OS images, boot edge nodes, register them against a central management plane, and deploy workloads across a mixed fleet, all from one terminal. The goal is not to click through slides. It is to leave knowing how SUSE Edge actually works at the system level.

Your environment is already partially running. A bare metal host is running five KVM virtual machines: a management cluster, an image-builder VM, and four edge nodes that are currently off. You will turn them on one group at a time, after building the right OS image for each.

**Access credentials** are in `~/.rodeo/secrets.yaml` on the host.

---

## The scenario

Vertex Trust Bank operates ATMs, branch teller terminals, and regional processing hubs across 47 branch locations. Each location runs between three and five Linux nodes. IT manages them centrally from HQ.

The problem: each branch has its own config history. Some nodes have not been updated in years. A regulatory audit is next month. The CTO's mandate is that every edge node should have the same OS, installed from the same image, managed from a single control plane. New nodes at any branch should onboard automatically with no hands-on config at the remote end.

Vertex Trust Bank also has two types of sites: larger regional hubs that need Kubernetes clusters capable of running containerized applications, and smaller branch terminals that just need a managed Linux node with lightweight workloads. SUSE Edge handles both from the same management plane.

---

## Lab topology

```
KVM host (bare metal)
│
├── rancher   192.168.122.9    Rancher Prime 2.15.1 + Elemental Operator 1.9.2 + Fleet
├── eib       192.168.122.20   EIB 1.3.4 + Hauler 1.2.2 (local artifact registry)
│
├── edge1     192.168.122.31   [OFF] vTPM 2.0, UEFI  →  Elemental onboarding  →  K3s cluster
├── edge2     192.168.122.32   [OFF] vTPM 2.0, UEFI  →  Elemental onboarding  →  standalone
├── edge3     192.168.122.33   [OFF] UEFI             →  EIB standalone        →  RKE2 cluster
└── edge4     192.168.122.34   [OFF] UEFI             →  EIB standalone        →  K3s cluster
```

**Two provisioning paths run in parallel in this lab:**

- **Elemental path (edge1, edge2):** Phone-home onboarding via TPM. The node boots, the Elemental agent contacts the registration endpoint, and the management cluster takes control. Cluster provisioning happens from Rancher, not from the node.
- **EIB standalone path (edge3, edge4):** The image is self-contained. Boot it, and you have a running Kubernetes cluster. No phone-home. No registration step. Import into Rancher afterwards if you want central management.

Understanding when to use each path is one of the core takeaways from this lab.

---

## Lab exercises

| # | Exercise | Time |
|---|---|---|
| 1 | Tour the environment | 15 min |
| 2 | Configure Elemental, the registration endpoint, and the node network plan | 30 min |
| 3 | Build four EIB images, one per node, each with its own network config | 50 min |
| 4 | Seed and boot the four nodes | 25 min |
| 5 | Provision a K3s cluster on an Elemental node | 20 min |
| 6 | Deploy workloads via Fleet | 20 min |

Total run time assumes EIB builds overlap with other steps. Each build runs in the background while you move on. Start each build, check it is progressing cleanly, then move to the next.

---

## Exercise 1: Tour the environment



Get your bearings before touching anything. From the KVM host:

```bash
# What VMs are running, what is off
virsh list --all

# Check the management cluster
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get nodes && kubectl get pods -A | grep -E 'rancher|elemental|cert-manager|fleet'"
```

You should see one K3s node (the rancher VM itself) with pods for Rancher Prime, cert-manager, the Elemental Operator, and Fleet. This is your management cluster. It manages everything else.

Now check the eib VM, your image factory:

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

You should see the EIB container image, the Elemental register agent, the vertex-bank-app image, and the openSUSE Leap Micro base image. These were pre-staged by the instructor before the lab started.

---

## Exercise 2: Configure Elemental and the node network plan



This exercise runs before you build any images. The Elemental path needs a registration URL embedded in the OS image. You cannot build the image before you have the endpoint configured and know its URL.

### 2.1 What Elemental does

Elemental handles autonomous node onboarding. An edge node boots, the `elemental-register` agent starts, and it contacts a registration URL that you define on the management cluster. The Elemental Operator validates the node's identity (via TPM in this lab), creates a `MachineInventory` record, and from that point the management cluster controls the node's lifecycle.

You do not SSH into the node. You do not run scripts on it. The node phones home and identifies itself.

### 2.2 Inspect the MachineRegistration

A `MachineRegistration` defines who is allowed to register and what labels the node gets when it does. One has already been created for this lab:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration -n fleet-default -o yaml"
```

Key fields in the spec:

```yaml
spec:
  machineName: "${System Information/Manufacturer}-${System Information/UUID}"
  config:
    elemental:
      registration:
        auth: tpm                   # TPM-based identity, hardware-bound
      install:
        device: /dev/vda            # Install target for Elemental's own installer
        poweroff: true              # Power off after an Elemental-driven install
  machineInventoryLabels:
    manufacturer: "${System Information/Manufacturer}"
    productName: "${System Information/Product Name}"
    registration: "<this-registration's-name>"
```

`auth: tpm` means the node's registration token is derived from its TPM. A cloned disk on a different machine will fail to register because the TPM identity will not match. This matters for edge deployments: remote sites where you cannot guarantee physical security.

The `install` block configures Elemental's own installer: which disk to use and whether to power off afterwards. It does not reach the openSUSE Leap Micro SelfInstall ISO you build in this lab, which has its own installer. That one asks for two keypresses before it writes the disk, and Exercise 4 walks you through them.

### 2.3 Get the registration URL

The registration name is derived from your lab's plan name, so it varies by deployment. Discover it first, then use it to fetch the URL you will embed in the EIB image:

```bash
# Discover the registration name
REGNAME=$(ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration -n fleet-default -o jsonpath='{.items[0].metadata.name}'")

echo "Registration name: $REGNAME"

# Capture the registration URL
REGURL=$(ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration $REGNAME \
   -n fleet-default \
   -o jsonpath='{.status.registrationURL}'")

echo "Registration URL: $REGURL"
```

**Keep this shell session open.** You need `$REGURL` in the next step.

### 2.4 Clone the EIB workspace and configure it

The EIB image definitions, NMState network configs, and combustion scripts live in the `eib-config` Gitea repo on the EIB VM. Fetch it to get a ready-made workspace, then replace the Elemental config placeholder with the live registration URL.

SSH to the EIB VM:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20
```

The EIB VM does not have `git` installed, because it stays a minimal build host. Fetch the workspace as an archive from Gitea instead, which gives you the exact same file layout:

```bash
mkdir -p /home/eib-workspace
curl -sL http://192.168.122.20:3000/gitea/eib-config/archive/main.tar.gz \
  | tar -xz --strip-components=1 -C /home/eib-workspace
ls /home/eib-workspace/
```

You should see four definition files and the `elemental/`, `network-configs/` and `scripts-available/` directories.

Now download the live registration config from the Elemental Operator and overwrite the placeholder:

```bash
# If REGNAME/REGURL are not in the current shell, re-capture them
REGNAME=$(ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration -n fleet-default -o jsonpath='{.items[0].metadata.name}'")
REGURL=$(ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration $REGNAME \
   -n fleet-default \
   -o jsonpath='{.status.registrationURL}'")

curl -k "$REGURL" -o /home/eib-workspace/elemental/elemental_config.yaml

cat /home/eib-workspace/elemental/elemental_config.yaml
```

This file contains the registration URL, the CA certificate for the management cluster's TLS, and the config that `elemental-register` needs to authenticate via TPM. The `elemental/` directory is EIB's own auto-discovered location for this, and it is what makes EIB bundle the `elemental-register`/`elemental-system-agent` packages into the image and wire up registration on first boot. No network config required at the remote site.

Exit back to the KVM host:

```bash
exit
```

### 2.5 Verify the Elemental Operator is healthy

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get pods -n cattle-elemental-system"
```

`elemental-operator` should be `Running`. Some Elemental Operator releases also run a separate `elemental-operator-webhook` pod; its absence alone is not a problem, but if `elemental-operator` itself is not `Running`, stop and flag it before building images. Nodes cannot register against a broken operator.

### 2.6 Node network plan

Each node gets a static IP baked into its disk image by EIB. The network configs are pre-populated in the workspace you just cloned from Gitea:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20
ls /home/eib-workspace/network-configs/
cat /home/eib-workspace/network-configs/edge1.yaml
exit
```

| Node | IP | Prefix | Gateway | DNS | MAC |
|---|---|---|---|---|---|
| edge1 | 192.168.122.31 | /24 | 192.168.122.1 | 192.168.122.1 | 02:00:00:0E:62:A1 |
| edge2 | 192.168.122.32 | /24 | 192.168.122.1 | 192.168.122.1 | 02:00:00:0E:62:A2 |
| edge3 | 192.168.122.33 | /24 | 192.168.122.1 | 192.168.122.1 | 02:00:00:0E:62:A3 |
| edge4 | 192.168.122.34 | /24 | 192.168.122.1 | 192.168.122.1 | 02:00:00:0E:62:A4 |

EIB picks up any YAML file in the `network/` subdirectory of its config dir. Each build uses exactly one file, one node, one IP. In Exercise 3 you will copy the right file into `network/` before starting each build.

The configs call the interface `eth0`, but that is only a logical name. EIB's network tool matches each config to the real NIC by its MAC address at first boot, so the node keeps its predictable kernel name (`enp1s0` on these VMs) and still gets the right IP. Don't add `net.ifnames=0` to the definitions: the SelfInstall ISO boots into the installed system once without that kernel argument, so the NIC would change name on the next reboot and lose its static IP.

---

## Exercise 3: Build four EIB images, one per node



Each node gets its own image with its own static IP baked in. Four builds, four definition files, exactly one NMState config in `network/` per run.

| Image | Node | Base | Output | IP | Kubernetes |
|---|---|---|---|---|---|
| `elemental-edge1.iso` | edge1 | openSUSE Leap Micro 6.2 SelfInstall ISO | ISO | 192.168.122.31 | None (Elemental) |
| `elemental-edge2.iso` | edge2 | openSUSE Leap Micro 6.2 SelfInstall ISO | ISO | 192.168.122.32 | None (Elemental) |
| `rke2-edge3.raw` | edge3 | openSUSE Leap Micro 6.2 Default RAW | RAW | 192.168.122.33 | RKE2 v1.36.3+rke2r1 |
| `k3s-edge4.raw` | edge4 | openSUSE Leap Micro 6.2 Default RAW | RAW | 192.168.122.34 | K3s v1.36.3+k3s1 |

**The build pattern for each node:**
1. Clear the `network/` dir and drop only that node's NMState file there
2. Clear `custom/scripts/` and drop in only the scripts that node actually needs (EIB auto-discovers and runs everything under `custom/scripts/`, with no way to select a subset, so leftover scripts from another node's build would get baked in and run too)
3. Start EIB in the background
4. Move on to the next node

**How the workspace is structured:**

The definition files, staged scripts, and network configs come from the `eib-config` Gitea repo you cloned in Exercise 2. The openSUSE Leap Micro base images (ISO and RAW) were pre-staged by the lab deploy at `/home/eib-config/base-images/`. They come from the Hauler file server and are ready to use.

You use two copies of that repo:
- `/home/eib-workspace` (from Exercise 2) holds `elemental/elemental_config.yaml`, so it builds the Elemental images for edge1 and edge2.
- `/home/eib-standalone` (you create it in 3.3) is the same repo without `elemental/`, for the standalone RKE2 and K3s images.

Why two? EIB treats any build whose config directory contains an `elemental/` directory as an Elemental build, and then refuses a standalone RAW build that doesn't side-load the Elemental RPMs. You can't just delete `elemental/` from the shared workspace either, because the edge1 and edge2 builds read it several minutes into their run.

EIB runs with these volume mounts:
- the workspace (`/home/eib-workspace` or `/home/eib-standalone`) → definitions, `custom/scripts/`, `network/`, and `elemental/` for the Elemental builds
- `/home/eib-config/base-images` → read-only base OS images (from Hauler)
- `/home/eib-config/rpms` → read-only elemental-register/elemental-system-agent RPMs (edge1/edge2 only, from Hauler)

`custom/scripts/` starts empty. The actual combustion scripts live in `scripts-available/` (same idea as `network-configs/` for NMState files), and each build below copies in only the scripts it needs.

The Elemental builds (edge1/edge2) side-load the `elemental-register`/`elemental-system-agent` RPMs already staged at `/home/eib-config/rpms/`, so no SUSE Customer Center registration code is needed. The standalone `edge3`/`edge4` builds don't use Elemental at all and don't need this mount.

SSH to the eib VM and stay there for this entire exercise:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20
```

Verify the workspace and base images are in place:

```bash
# Definition files and scripts from Gitea
ls /home/eib-workspace/

# openSUSE Leap Micro base images pre-staged from Hauler
ls -lh /home/eib-config/base-images/

# Elemental RPMs pre-staged from Hauler (edge1/edge2 side-load these)
ls -lh /home/eib-config/rpms/

# Create the active network/ dir (git-ignored, managed at build time)
mkdir -p /home/eib-workspace/network
```

---

### 3.1 Elemental image for edge1

This image boots, installs openSUSE Leap Micro to disk, and on first boot `elemental-register` phones home to the management cluster. Static IP 192.168.122.31 is baked in via NMState.

```bash
# Load edge1's network config, only one file in network/ at a time
rm -f /home/eib-workspace/network/*.yaml
cp /home/eib-workspace/network-configs/edge1.yaml /home/eib-workspace/network/

# Elemental builds need no combustion scripts. EIB errors out if
# custom/scripts/ exists but is empty, so remove the directory entirely
# rather than just clearing it.
rm -rf /home/eib-workspace/custom/scripts
```

The Elemental registration config you downloaded in Exercise 2 lives at `elemental/elemental_config.yaml` in the workspace. That is EIB's own auto-discovered directory for Elemental: it makes EIB bundle the `elemental-register`/`elemental-system-agent` packages into the image and embed the config for first boot. A plain `os-files/` drop-in does not trigger this. When the node boots and `elemental-register` runs, it reads that file and knows where to call home. The NMState config in `network/` gives the node its static IP.

Start the build in the background:

```bash
podman run --rm --privileged \
  -v /home/eib-workspace:/eib:z \
  -v /home/eib-config/base-images:/eib/base-images:ro \
  -v /home/eib-config/rpms:/eib/rpms:ro \
  registry.suse.com/edge/3.7/edge-image-builder:1.3.4 \
  build --definition-file elemental-edge1-definition.yaml \
  > /tmp/eib-edge1.log 2>&1 &

echo "edge1 build PID: $!"
tail -5 /tmp/eib-edge1.log
```

### 3.2 Elemental image for edge2

Same Elemental path, different IP (192.168.122.32):

```bash
tail -10 /tmp/eib-edge1.log

rm -f /home/eib-workspace/network/*.yaml
cp /home/eib-workspace/network-configs/edge2.yaml /home/eib-workspace/network/
rm -rf /home/eib-workspace/custom/scripts

podman run --rm --privileged \
  -v /home/eib-workspace:/eib:z \
  -v /home/eib-config/base-images:/eib/base-images:ro \
  -v /home/eib-config/rpms:/eib/rpms:ro \
  registry.suse.com/edge/3.7/edge-image-builder:1.3.4 \
  build --definition-file elemental-edge2-definition.yaml \
  > /tmp/eib-edge2.log 2>&1 &

echo "edge2 build PID: $!"
```

### 3.3 Standalone RKE2 image for edge3

This image boots as a running single-node RKE2 cluster. No phone-home, no registration. The full Kubernetes stack is embedded in the disk. Static IP 192.168.122.33, hostname `edge3` set at first boot via combustion script.

Create the standalone workspace first: the same Gitea repo, without `elemental/`:

```bash
mkdir -p /home/eib-standalone
curl -sL http://192.168.122.20:3000/gitea/eib-config/archive/main.tar.gz \
  | tar -xz --strip-components=1 -C /home/eib-standalone
rm -rf /home/eib-standalone/elemental
mkdir -p /home/eib-standalone/network
```

Then load edge3's network config and scripts:

```bash
rm -f /home/eib-standalone/network/*.yaml
cp /home/eib-standalone/network-configs/edge3.yaml /home/eib-standalone/network/

# Only edge3's own combustion scripts, not edge4's
rm -rf /home/eib-standalone/custom/scripts
mkdir -p /home/eib-standalone/custom/scripts
cp /home/eib-standalone/scripts-available/60-hostname-edge3.sh \
   /home/eib-standalone/scripts-available/99-k3s-registries.sh \
   /home/eib-standalone/custom/scripts/
```

The `kubernetes:` section in the definition is the key difference from the Elemental builds. EIB downloads the RKE2 binary, all required container images from the Hauler OCI registry, and configures CRI-O. Everything lands in the disk image. The node does not need internet access or a running management plane. It starts Kubernetes on its own at boot.

```bash
tail -10 /tmp/eib-edge2.log

podman run --rm --privileged \
  -v /home/eib-standalone:/eib:z \
  -v /home/eib-config/base-images:/eib/base-images:ro \
  registry.suse.com/edge/3.7/edge-image-builder:1.3.4 \
  build --definition-file rke2-edge3-definition.yaml \
  > /tmp/eib-rke2.log 2>&1 &

echo "edge3 RKE2 build PID: $!"
```

### 3.4 Standalone K3s image for edge4

Same standalone pattern as edge3 but K3s instead of RKE2, lighter footprint, faster first-boot cluster start:

```bash
rm -f /home/eib-standalone/network/*.yaml
cp /home/eib-standalone/network-configs/edge4.yaml /home/eib-standalone/network/

rm -rf /home/eib-standalone/custom/scripts
mkdir -p /home/eib-standalone/custom/scripts
cp /home/eib-standalone/scripts-available/60-hostname-edge4.sh \
   /home/eib-standalone/scripts-available/99-k3s-registries.sh \
   /home/eib-standalone/custom/scripts/

podman run --rm --privileged \
  -v /home/eib-standalone:/eib:z \
  -v /home/eib-config/base-images:/eib/base-images:ro \
  registry.suse.com/edge/3.7/edge-image-builder:1.3.4 \
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

**Why builds take different amounts of time:** The Elemental ISOs are fast. EIB wraps an existing ISO with a config file and the NMState network config. The RKE2 and K3s RAW builds are slower because EIB downloads and embeds all required container images from the Hauler OCI registry. The `99-k3s-registries.sh` script tells EIB to pull from `192.168.122.20:5000` instead of the internet.

When all four builds are done, verify the output files:

```bash
ls -lh /home/eib-workspace/elemental-edge1.iso \
        /home/eib-workspace/elemental-edge2.iso \
        /home/eib-standalone/rke2-edge3.raw \
        /home/eib-standalone/k3s-edge4.raw
```

Exit back to the KVM host:

```bash
exit
```

---

## Exercise 4: Seed and boot the four nodes



You have four nodes, two image formats, and two different boot workflows.

### 4.1 Elemental nodes (edge1 and edge2): ISO workflow

edge1 and edge2 each boot from their own ISO. The ISO contains a self-installer: it boots, writes openSUSE Leap Micro to the virtual disk with the static IP already configured, powers off, and the node reboots into the installed OS where `elemental-register` runs.

Pull each ISO from the eib VM and attach it as a virtual CDROM:

```bash
# edge1 gets its specific ISO (192.168.122.31 baked in)
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-workspace/elemental-edge1.iso \
  --nodes edge1 \
  --yes

# edge2 gets its specific ISO (192.168.122.32 baked in)
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-workspace/elemental-edge2.iso \
  --nodes edge2 \
  --yes
```

Each command SCPs the ISO, attaches it as a CDROM (sda, boot-order-1) to that node's libvirt domain, and creates a blank `vda.qcow2` for the OS to install into.

Start both nodes:

```bash
virsh start edge1
virsh start edge2
```

The installer needs two keypresses per node before it runs unattended. The GRUB boot menu waits indefinitely instead of auto-selecting (by design, so install media never silently wipes a disk), and the partitioner asks you to confirm before it writes to `/dev/vda`. Watch each node's console in turn (Ctrl+] to exit) and press Enter at both points:

```bash
virsh console edge1
# Press Enter to select "Install openSUSE Leap Micro"
# Press Enter again to confirm "Destroying ALL data on /dev/vda, continue?"
```

If the console shows nothing, send the keys straight from the KVM host instead. Wait about 30 seconds after the first one, until the confirmation dialog is on screen:

```bash
virsh send-key edge1 KEY_ENTER     # GRUB: "Install openSUSE Leap Micro"
virsh send-key edge1 KEY_ENTER     # "Destroying ALL data on /dev/vda, continue?" -> Yes
```

Do the same for edge2. Once both are past the confirmation, the rest runs on its own. You see a progress bar while the OS writes to disk, then the node reboots straight into the installed system and `elemental-register` phones home. After a few minutes the console shows a login prompt with the node's static IP (`enp1s0: 192.168.122.31` for edge1).

Check that both nodes registered with the management cluster:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineinventory -n fleet-default"
```

When both appear, shut the nodes down cleanly, eject the ISOs and restore disk-first boot. The ISO is still attached at this point, so without this step the next reboot would land on the installer's GRUB menu again:

```bash
virsh shutdown edge1
virsh shutdown edge2
watch "virsh list --all | grep edge"     # wait until both show "shut off"

rodeo eject-iso --nodes edge1,edge2 --yes
virsh start edge1
virsh start edge2
```

They now boot from the installed disk and keep their static IPs:

```bash
ping -c3 192.168.122.31
ping -c3 192.168.122.32
```

### 4.2 Standalone nodes (edge3 and edge4): RAW workflow

edge3 and edge4 use pre-built RAW images. No installer, no reboot cycle. The node boots from a ready disk image with Kubernetes already configured to start.

Pull and thin-clone each image from the eib VM:

```bash
# edge3 gets the RKE2 image
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-standalone/rke2-edge3.raw \
  --nodes edge3 \
  --yes

# edge4 gets the K3s image
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-standalone/k3s-edge4.raw \
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

The base images hold the full content. The per-node `vda.qcow2` files are thin clones, a few megabytes each, that only store writes that diverge from the base. If you had ten edge3-class nodes, you would clone the base ten times at near-zero disk cost.

Start edge3 and edge4:

```bash
virsh start edge3
virsh start edge4
```

These nodes take 3-5 minutes to come up fully. RKE2 and K3s do their first-run initialization: generating TLS certificates, starting the control plane, and marking the node Ready.

Both nodes have static IPs, so there are no DHCP leases to look for. SSH in and check Kubernetes. The RAW images authorize the KVM host's key for root, and RKE2 keeps its own `kubectl` and kubeconfig under its install paths:

```bash
# edge3 (RKE2)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.33 \
  "/var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes"

# edge4 (K3s)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.34 \
  "kubectl get nodes"
```

Each shows one node in `Ready` state: `v1.36.3+rke2r1` on edge3 and `v1.36.3+k3s1` on edge4.

---

## Exercise 5: Provision a K3s cluster on an Elemental node



edge1 and edge2 are now running openSUSE Leap Micro. The `elemental-register` agent has phoned home and registered them with the Elemental Operator. But they are not yet Kubernetes nodes. That happens here.

### 5.1 Check MachineInventory

Watch for the registered nodes on the management cluster:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineinventory -n fleet-default -w"
```

You should see one or two entries appear, one per node that has completed registration. Each row is a node that has proven its TPM identity and is now waiting to be told what to do.

In the Rancher UI: go to **OS Management > MachineInventory**. The same records appear there with hardware info, labels inherited from the MachineRegistration, and status.

### 5.2 Create a MachineInventorySelectorTemplate

This resource links a label selector to a machine pool. Any `MachineInventory` that matches the selector becomes a candidate node for a cluster:

```bash
cat << 'EOF' | ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 "kubectl apply -f -"
apiVersion: elemental.cattle.io/v1beta1
kind: MachineInventorySelectorTemplate
metadata:
  name: vertex-hub-selector
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

Create a `Cluster` resource that uses the selector template as its machine pool:

```bash
cat << 'EOF' | ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 "kubectl apply -f -"
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: vertex-hub-01
  namespace: fleet-default
spec:
  kubernetesVersion: v1.36.3+k3s1
  rkeConfig:
    machinePools:
      - name: hub-nodes
        quantity: 1
        etcdRole: true
        controlPlaneRole: true
        workerRole: true
        machineConfigRef:
          kind: MachineInventorySelectorTemplate
          name: vertex-hub-selector
          apiVersion: elemental.cattle.io/v1beta1
EOF
```

### 5.5 Watch the cluster provision

In Rancher UI: go to **Cluster Management**. You will see `vertex-hub-01` appear with status `Provisioning`. The management cluster is now remotely installing K3s on edge1 via the Elemental system agent.

From the terminal:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get cluster -n fleet-default -w"
```

Provisioning takes 5-10 minutes. When status changes to `Active`, the cluster is ready.

**What just happened:** you provisioned a Kubernetes cluster on a remote node without touching the node. The node registered itself, you applied a label, and the management cluster did the rest. This is the Elemental model for large-scale edge deployments.

---

## Exercise 6: Deploy workloads via Fleet



You now have four running nodes:

- **vertex-hub-01** on edge1 (Elemental-provisioned K3s)
- **edge2** registered but not yet assigned to a cluster
- **edge3** running RKE2 standalone (not yet in Rancher)
- **edge4** running K3s standalone (not yet in Rancher)

Fleet is already watching for clusters with the right labels. A `GitRepo` called `vertex-bank-app` is pre-configured to target any cluster labeled `demo=true` and `edge-type=x86-cluster`.

The `vertex-bank-app` GitRepo points at the local Gitea instance on the EIB VM, not GitHub. Fleet syncs from `http://192.168.122.20:3000/gitea/vertex-bank-app.git` every 15 seconds. No internet access is needed.

```bash
# Confirm the GitRepo source
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl --kubeconfig=/etc/rancher/k3s/k3s.yaml \
   get gitrepo vertex-bank-app -n fleet-default \
   -o jsonpath='{.spec.repo}'"
```

### 6.1 Trigger Fleet deployment on vertex-hub-01

edge1's `MachineInventory` has the `demo=true` and `edge-type=x86-cluster` labels from Exercise 5, but those do not propagate to the cluster Fleet actually watches. Label it explicitly:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl label clusters.fleet.cattle.io vertex-hub-01 -n fleet-default \
   demo=true edge-type=x86-cluster --overwrite"
```

Use the full `clusters.fleet.cattle.io` name. This Rancher/Fleet setup has at least three separate CRDs all called `Cluster`, and a bare `kubectl label cluster ...` resolves to a different one. It succeeds with no error, but Fleet keeps showing 0/0 targeted clusters because the label never lands where Fleet is looking.

Check in Rancher UI: **Continuous Delivery > Git Repos > vertex-bank-app**. When `vertex-hub-01` appears in the target clusters section, Fleet is deploying the app. It picks up the label within its normal 15-second poll cycle.

### 6.2 Import edge3 and edge4 into Rancher

edge3 and edge4 are running standalone clusters that Rancher does not know about yet. Import them:

In Rancher UI: **Cluster Management > Import Existing > Generic**. Give each cluster a name (`vertex-branch-rke2`, `vertex-branch-k3s`) and create it. Rancher then shows registration commands. This lab's Rancher uses a self-signed certificate, so copy the **second** one, the `curl --insecure ... | kubectl apply -f -` variant. The plain `kubectl apply -f <url>` command fails with `x509: certificate signed by unknown authority`.

Run it on each node:

```bash
# On edge3 (RKE2 keeps its kubectl under /var/lib/rancher/rke2/bin)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.33 \
  "curl --insecure -sfL <registration-manifest-url> | \
   /var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml apply -f -"

# On edge4 (K3s)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.34 \
  "curl --insecure -sfL <registration-manifest-url> | kubectl apply -f -"
```

Each cluster turns `Active` in **Cluster Management** within a couple of minutes. Fleet registers its own copy of each cluster a little after that, so check that both show up before you label them (labelling too early fails with `NotFound`):

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get clusters.fleet.cattle.io -n fleet-default"
```

When `vertex-branch-rke2` and `vertex-branch-k3s` are listed, label them for Fleet:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 "
  kubectl label clusters.fleet.cattle.io vertex-branch-rke2 -n fleet-default \
    demo=true edge-type=x86-cluster --overwrite
  kubectl label clusters.fleet.cattle.io vertex-branch-k3s -n fleet-default \
    demo=true edge-type=x86-cluster --overwrite
"
```

Fleet deploys vertex-bank-app to each cluster the moment the labels match. Go to **Continuous Delivery > Git Repos > vertex-bank-app** and watch the bundle status update per cluster.

### 6.3 Verify deployment

vertex-bank-app is a web app that shows live Kubernetes cluster vitals. Check the bundle status from the management cluster:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get bundle -n fleet-default | grep vertex-bank"
```

The app itself runs on the edge clusters, not on the management cluster, so query vertex-hub-01 through the kubeconfig Rancher stores for it:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 "
  kubectl get secret vertex-hub-01-kubeconfig -n fleet-default \
    -o jsonpath='{.data.value}' | base64 -d > /tmp/vertex-hub-01.yaml
  kubectl --kubeconfig /tmp/vertex-hub-01.yaml get pods,svc -n vertex-bank
"
```

The service publishes NodePort `30080`, and that is what you use. On an older copy of vertex-bank-app the service may show up as a `LoadBalancer` with `EXTERNAL-IP` stuck at `<pending>`, and the bundle as `NotReady` in Rancher, because nothing in this lab hands out LoadBalancer IPs. The app still answers on the NodePort. Open it on each node:

```bash
curl -s http://192.168.122.31:30080/ | grep -o "<title>.*</title>"   # vertex-hub-01 (edge1)
curl -s http://192.168.122.33:30080/ | grep -o "<title>.*</title>"   # vertex-branch-rke2 (edge3)
curl -s http://192.168.122.34:30080/ | grep -o "<title>.*</title>"   # vertex-branch-k3s (edge4)
```

Or open `http://192.168.122.31:30080` in a browser.

---

### What you just built

Four nodes, four images, two provisioning paths, one management plane.

**The image-first model:** every node in this lab booted from a purpose-built disk image. No manual SSH config, no package installs, no configuration management applied to a running system. The image IS the configuration. If a node breaks, you re-image it. If a new site opens, you ship the image.

**Why two paths exist:** Elemental (edge1/edge2) is for sites where you do not know the node's final role at image-build time. You build one generic openSUSE Leap Micro image, ship it everywhere, and the management cluster decides what each node becomes after it registers. EIB standalone (edge3/edge4) is for sites where the role is known upfront: you bake K3s or RKE2 into the image and the node is a cluster the moment it boots.

**Why TPM matters:** without TPM, a cloned disk can register as any node in your fleet. With `auth: tpm`, the registration token is derived from hardware. You can revoke a specific node's registration by deleting its `MachineInventory`. Physical theft of the hardware does not compromise other nodes.

**Why Hauler is there:** in a real deployment, remote sites have unreliable internet. Hauler is a portable artifact store. Populate it once at HQ, copy it to a USB drive, and the site's nodes pull everything locally. In this lab, Hauler was pre-populated by the instructor. In production, you run `hauler store sync` to update and redistribute.

---

### Two-minute comparison

| | Elemental (edge1/edge2) | EIB standalone (edge3/edge4) |
|---|---|---|
| Node role decided | At provisioning time (after registration) | At image-build time |
| Cluster control | Rancher provisions and manages K8s | Node runs its own K8s; optionally imported |
| Onboarding | Automatic via TPM phone-home | Manual import or pre-registered via EIB |
| Good for | Mixed-role sites, large fleets, dynamic scaling | Fixed-function nodes, strict airgap, predictable workloads |
| Recovery | Re-register = re-provision from management | Re-image = re-flash disk |

---

## Bonus exercises

### Try a second Elemental cluster with RKE2

Label edge2 with `site-role: hub` and create a second `Cluster` resource pointing at the same `vertex-hub-selector` template but with `quantity: 1` targeting edge2. Rancher will provision RKE2 on edge2.

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

The registration attempt will be rejected because the TPM identity changed. The `MachineInventory` record will not update. This is the expected failure mode for hardware replacement at a remote site: you control the replacement from the management cluster, not from the site.

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
# EIB build (run on the eib VM). Elemental images build from /home/eib-workspace,
# standalone RAW images from /home/eib-standalone (same repo, no elemental/ dir)
podman run --rm --privileged \
  -v /home/eib-workspace:/eib:z \
  -v /home/eib-config/base-images:/eib/base-images:ro \
  -v /home/eib-config/rpms:/eib/rpms:ro \
  registry.suse.com/edge/3.7/edge-image-builder:1.3.4 \
  build --definition-file <definition-file.yaml>

# Seed an edge node from an image on the eib VM (ISO or RAW auto-detected)
rodeo pull-edge-image --config-dir /root/rodeo-lab \
  --image /home/eib-workspace/elemental-edge1.iso --nodes edge1 --yes

# ISO workflow: after install and registration, shut down, eject, start from disk
rodeo eject-iso --nodes edge1,edge2 --yes

# Watch Elemental registration
kubectl get machineinventory -n fleet-default -w

# Get registration URL (the registration name varies by plan, so discover it first)
REGNAME=$(kubectl get machineregistration -n fleet-default -o jsonpath='{.items[0].metadata.name}')
kubectl get machineregistration "$REGNAME" \
  -n fleet-default \
  -o jsonpath='{.status.registrationURL}'

# Watch cluster provisioning
kubectl get clusters.provisioning.cattle.io -n fleet-default -w

# Watch Fleet bundle status
kubectl get bundle -n fleet-default
```

### Component versions

| Component | Version |
|---|---|
| Rancher Prime | 2.15.1 |
| K3s (management cluster) | v1.36.3+k3s1 |
| cert-manager | v1.20.1 |
| Elemental Operator | 1.9.2 |
| Edge Image Builder | 1.3.4 |
| Hauler | 1.2.2 |
| openSUSE Leap Micro | 6.2 |

---

## Instructor notes

**Before the lab starts:**

1. From this repo's directory (it already ships its own `rodeo-plan.yaml`), run `rodeo up` on the bare metal host. It self-escalates with sudo, generates `~/.rodeo/secrets.yaml`, and runs the full pipeline including Hauler population, the Elemental Operator install, and staging the elemental-register/elemental-system-agent RPMs students side-load in Exercise 3. No SUSE Customer Center entitlement or manual pre-staging is needed.
2. Verify the Elemental Operator is running and the MachineRegistration exists.
3. Share access credentials and host IP with students.

**Timing:** on a validation run (AWS m8id.4xlarge, October 2026) the two RAW builds took about 4 minutes and the two Elemental ISO builds about 10 minutes, all four running in parallel. Slower hosts take longer, so start the builds in order and keep explaining while they run. Elemental registration takes a few minutes after install, and cluster provisioning in Exercise 5 about 2 minutes.

---

*Lab built with [rodeo-cli](https://github.com/avaleror/rodeo-cli).*
