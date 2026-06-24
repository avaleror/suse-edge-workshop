# Pre-lab checklist

Run through this before handing the lab to students. The deploy (`rodeo deploy`) takes 30-60 minutes; leave time for this checklist on top.

## After deploy completes

```bash
# 1. Management cluster is healthy
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get nodes && kubectl get pods -n cattle-elemental-system"

# 2. Elemental Operator is running — both pods
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get pods -n cattle-elemental-system"

# 3. MachineRegistration exists
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineregistration -n fleet-default"

# 4. EIB VM is reachable and EIB container is pulled
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "podman images | grep edge-image-builder"

# 5. Hauler is running
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "systemctl is-active hauler-registry hauler-fileserver"

# 6. Edge VMs are defined but off
virsh list --all | grep edge
```

## Pre-stage SL Micro images in Hauler

This step is NOT automated — do it manually after deploy:

```bash
ssh root@192.168.122.20

# SL Micro 6.2 SelfInstall ISO (for Elemental nodes edge1/edge2)
hauler store add file \
  "https://download.suse.com/SL-Micro/6.2/SL-Micro.x86_64-6.2-Base-SelfInstall-GM.install.iso" \
  --store /var/lib/hauler

# SL Micro 6.2 Cloud RAW (for standalone cluster nodes edge3/edge4)
hauler store add file \
  "https://download.suse.com/SL-Micro/6.2/SL-Micro.x86_64-6.2-Default.raw" \
  --store /var/lib/hauler

systemctl restart hauler-fileserver

# Verify both are served
curl -s http://localhost:8080/ | grep -E "iso|raw"
```

## Timing notes

- EIB builds for RAW images with embedded Kubernetes take 15-30 minutes each.
- Start the Elemental ISO builds first (Exercise 3.1 and 3.2) — they are fast and let students proceed to Exercises 4 and 5 early.
- The RKE2 and K3s builds can run while students do Exercise 4 (Elemental install boot cycle).
- Exercises 5 and 6 require the Elemental nodes to have completed registration — typically 10-15 minutes after the ISO boot cycle.

## Student credentials

| Item | Where to find |
|---|---|
| Rancher admin password | `cat ~/.rodeo/secrets.yaml` on the KVM host |
| KVM host IP | Output of `rodeo deploy` success screen, or `ip addr show` |
| SSH key for VMs | `/root/.ssh/id_ed25519` on the KVM host |
