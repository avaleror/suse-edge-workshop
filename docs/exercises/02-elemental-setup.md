# Exercise 2 — Configure Elemental and the node network plan

**Time:** 20 min  
**Previous:** [Exercise 1 — Tour the environment](01-environment-tour.md)  
**Next:** [Exercise 3 — Build EIB images](03-eib-builds.md)

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
        auth: tpm                   # TPM-based identity — hardware-bound
      install:
        powerOff: true              # Power off after install, before first-run reboot
  machineInventoryLabels:
    manufacturer: "${System Information/Manufacturer}"
    productName: "${System Information/Product Name}"
    locationID: ""                  # You will fill this in per node
```

`auth: tpm` means the node's registration token is derived from its TPM. A cloned disk on a different machine will fail to register because the TPM identity will not match. For edge deployments — remote sites where you cannot guarantee physical security — this matters.

## 2.3 Get the registration URL

Note the registration name, then get the URL you will embed in the EIB image:

```bash
# Check the registration name
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration -n fleet-default"

# Capture the registration URL
REGURL=$(ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration suse-edge-reg-1 \
   -n fleet-default \
   -o jsonpath='{.status.registrationURL}'")

echo "Registration URL: $REGURL"
```

**Keep this shell session open.** You need `$REGURL` in the next step.

## 2.4 Clone the EIB workspace and configure it

The EIB image definitions, NMState network configs, and combustion scripts live in the `eib-config` Gitea repo on the EIB VM. Clone it to get a ready-made workspace, then replace the Elemental config placeholder with the live registration URL.

SSH to the EIB VM:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20
```

Clone the workspace:

```bash
git clone http://192.168.122.20:3000/gitea/eib-config /home/eib-workspace
ls /home/eib-workspace/
```

You should see four definition files, `network-configs/`, `scripts/`, and `elemental/` directories.

Now download the live registration config from the Elemental Operator and overwrite the placeholder:

```bash
# If REGURL is not in the current shell, re-capture it
REGURL=$(ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration suse-edge-reg-1 \
   -n fleet-default \
   -o jsonpath='{.status.registrationURL}'")

curl -k "$REGURL" -o /home/eib-workspace/elemental/elemental_config.yaml

cat /home/eib-workspace/elemental/elemental_config.yaml
```

This file contains the registration URL, the CA certificate for the management cluster's TLS, and the config that `elemental-register` needs to authenticate via TPM. EIB will embed it at `/oem/elemental.yaml` in the OS image — no network config required at the remote site.

Exit back to the KVM host:

```bash
exit
```

## 2.5 Verify the Elemental Operator is healthy

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get pods -n cattle-elemental-system"
```

Both `elemental-operator` and `elemental-operator-webhook` should be `Running`. If either is not, stop and flag it before building images — nodes cannot register against a broken operator.

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

EIB picks up any YAML file in the `network/` subdirectory of its config dir. Each build uses exactly one file — one node, one IP. In Exercise 3 you will copy the right file into `network/` before starting each build.

The interface name `eth0` comes from `net.ifnames=0` in the EIB definition's `kernelArgs`. Without that kernel arg, SL Micro would name the first NIC something like `ens3` depending on PCI bus order. With the arg set, the old naming convention applies consistently across all four builds.

---

**Next:** [Exercise 3 — Build EIB images](03-eib-builds.md)
