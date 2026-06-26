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

# 4. EIB VM is reachable and EIB container is in Hauler
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "hauler store info --store /var/lib/hauler 2>/dev/null | grep edge-image-builder"

# 5. Hauler is running and both SL Micro base images are served
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "systemctl is-active hauler-registry hauler-fileserver && \
   curl -s http://localhost:8080/ | grep -c 'SL-Micro'"

# 6. SL Micro files staged in eib-config/base-images (used by EIB exercise)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "ls -lh /home/eib-config/base-images/"

# 7. Edge VMs are defined but off
virsh list --all | grep edge
```

The SL Micro SelfInstall ISO and Default RAW are now downloaded automatically by `rodeo deploy` during the `elemental` phase. No manual Hauler steps needed.

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
