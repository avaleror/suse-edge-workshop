# Exercise 2: Configure Elemental and the node network plan

**Time:** 20 min  
**Previous:** [Exercise 1: Tour the environment](01-environment-tour.md)  
**Next:** [Exercise 3: Build EIB images](03-eib-builds.md)

---

This exercise runs before you build any images. The Elemental path needs a registration URL embedded in the OS image. You cannot build the image before you have the endpoint configured and know its URL.

## 2.1 What Elemental does

Elemental handles autonomous node onboarding. An edge node boots, the `elemental-register` agent starts, and it contacts a registration URL that you define on the management cluster. The Elemental Operator validates the node's identity (via TPM in this lab), creates a `MachineInventory` record, and from that point the management cluster controls the node's lifecycle.

You do not SSH into the node. You do not run scripts on it. The node phones home and identifies itself.

## 2.2 Inspect the MachineRegistration

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

## 2.3 Get the registration URL

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

## 2.4 Clone the EIB workspace and configure it

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

## 2.5 Verify the Elemental Operator is healthy

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get pods -n cattle-elemental-system"
```

`elemental-operator` should be `Running`. Some Elemental Operator releases also run a separate `elemental-operator-webhook` pod; its absence alone is not a problem, but if `elemental-operator` itself is not `Running`, stop and flag it before building images. Nodes cannot register against a broken operator.

## 2.6 Node network plan

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

**Next:** [Exercise 3: Build EIB images](03-eib-builds.md)
