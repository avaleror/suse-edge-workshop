# Host Setup — SUSE Edge 3.6 Workshop

This workshop deploys on a bare metal Linux host with KVM. These steps cover host prep and initial deploy.

## Requirements

| Resource | Minimum |
|---|---|
| OS | SLES 15 SP6 / SLES 16 / openSUSE Leap 15.6 |
| RAM | 40 GiB |
| vCPU | 16 |
| Disk | 250 GiB (free in `/var/lib/libvirt/images`) |
| Network | Internet access during deploy only — lab exercises run fully offline |

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

## Lab runs offline after deploy

Once `rodeo deploy` completes, the lab is fully self-contained. No internet access is needed during the exercises.

During the `elemental` phase, rodeo-cli automatically populates the Hauler store on the EIB VM with everything students need:

| Artifact | Served at | Used in |
|---|---|---|
| EIB container image (`edge-image-builder:1.3.3.1`) | Hauler OCI registry `:5000` | Exercise 3 — EIB builds |
| `elemental-register` image | Hauler OCI registry `:5000` | Exercise 3 — Elemental ISO builds |
| Alien-Geeko app image | Hauler OCI registry `:5000` | Exercise 6 — Fleet deploy |
| SL Micro 6.2 SelfInstall ISO | Hauler fileserver `:8080` | Exercise 3 — EIB Elemental ISO base |
| SL Micro 6.2 Default RAW | Hauler fileserver `:8080` | Exercise 3 — EIB standalone RAW base |
| `gitea/alien-geeko` repo | Gitea `:3000` | Exercise 6 — Fleet GitRepo source |
| `gitea/eib-config` repo | Gitea `:3000` | Exercise 2 — students clone this for EIB definitions + scripts |

Edge nodes boot with `registries.yaml` baked in by EIB, pointing all container pulls (`docker.io`, `registry.suse.com`, `ghcr.io`) to the local Hauler registry at `192.168.122.20:5000`. Fleet syncs from local Gitea at `192.168.122.20:3000` — no GitHub access needed. EIB pulls image definitions and scripts from the `eib-config` Gitea repo.

See the [Disconnected environment reference](../reference/disconnected-environment.md) for full details on what runs offline and why.

## Verify before handing to students

```bash
# Management cluster healthy
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get nodes && kubectl get pods -n cattle-elemental-system"

# EIB VM + Hauler running, both SL Micro files in the store
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "systemctl is-active hauler-registry hauler-fileserver && \
   curl -s http://localhost:8080/ | grep -E 'SL-Micro.*iso|SL-Micro.*raw'"

# SL Micro base images staged for EIB
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "ls -lh /home/eib-config/base-images/"

# Gitea running and both repos present
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "podman ps --filter name=gitea --format '{{.Status}}' && \
   curl -s http://localhost:3000/api/v1/repos/gitea/alien-geeko \
   | python3 -c \"import sys,json; r=json.load(sys.stdin); print('alien-geeko:', r['full_name'])\" && \
   curl -s http://localhost:3000/api/v1/repos/gitea/eib-config \
   | python3 -c \"import sys,json; r=json.load(sys.stdin); print('eib-config:', r['full_name'])\""

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
