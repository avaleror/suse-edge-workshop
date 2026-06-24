# Host Setup — SUSE Edge 3.6 Workshop

This workshop deploys on a bare metal Linux host with KVM. These steps cover host prep and initial deploy.

## Requirements

| Resource | Minimum |
|---|---|
| OS | SLES 15 SP6 / SLES 16 / openSUSE Leap 15.6 |
| RAM | 40 GiB |
| vCPU | 16 |
| Disk | 250 GiB (free in `/var/lib/libvirt/images`) |
| Network | Internet access at deploy time (Hauler pulls images) |

## Install rodeo-cli

```bash
pip install git+https://github.com/avaleror/rodeo-cli.git@main
```

Or clone and install in editable mode:

```bash
git clone https://github.com/avaleror/rodeo-cli.git
cd rodeo-cli
pip install -e .
```

## Generate secrets

```bash
rodeo init --profile suse-edge --dir /path/to/suse-edge-workshop
```

This creates `~/.rodeo/secrets.yaml` with generated passwords. The `rodeo-plan.yaml` in this repo references them via `??placeholder` notation.

## Deploy

From the workshop directory:

```bash
sudo rodeo deploy --config-dir .
```

The deploy runs in phases: `kvm_host → vms → boot → rancher → elemental → finalise`. Total time is 30-60 minutes depending on internet speed (Hauler pulls several GB of images).

When complete, the success screen shows:

- Rancher URL and admin password location
- EIB VM SSH command
- Edge node MAC/IP reference table
- Next steps for the lab guide

## Pre-stage SL Micro images in Hauler

The `elemental` phase pre-stages the EIB container and Elemental agent. You still need to add SL Micro images manually before running the lab:

```bash
ssh root@192.168.122.20

# SL Micro 6.2 SelfInstall ISO (for Elemental nodes)
hauler store add file \
  "https://download.suse.com/SL-Micro/6.2/SL-Micro.x86_64-6.2-Base-SelfInstall-GM.install.iso" \
  --store /var/lib/hauler

# SL Micro 6.2 Cloud RAW (for standalone cluster nodes)
hauler store add file \
  "https://download.suse.com/SL-Micro/6.2/SL-Micro.x86_64-6.2-Default.raw" \
  --store /var/lib/hauler

systemctl restart hauler-fileserver
```

Verify both are available:

```bash
curl -s http://localhost:8080/ | grep -E "iso|raw"
```

## Verify before handing to students

```bash
# Management cluster healthy
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get nodes && kubectl get pods -n cattle-elemental-system"

# EIB VM + Hauler running
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "systemctl is-active hauler-registry hauler-fileserver && \
   hauler store info --store /var/lib/hauler | head -20"

# MachineRegistration exists
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration -n fleet-default"

# Edge VMs are defined but off
virsh list --all | grep edge
```

Everything should be green before students start `lab-guide.md`.

## Tear down

```bash
sudo rodeo clean --config-dir .
```

This removes all VMs, disk images, and libvirt network config. The host is left clean.
