# Quick Reference

## Access

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

## Node reference

| Node | IP | MAC | Type | Image |
|---|---|---|---|---|
| edge1 | 192.168.122.31 | 02:00:00:0E:62:A1 | Elemental | elemental-edge1.iso |
| edge2 | 192.168.122.32 | 02:00:00:0E:62:A2 | Elemental | elemental-edge2.iso |
| edge3 | 192.168.122.33 | 02:00:00:0E:62:A3 | EIB standalone | rke2-edge3.raw |
| edge4 | 192.168.122.34 | 02:00:00:0E:62:A4 | EIB standalone | k3s-edge4.raw |

## Key commands

```bash
# EIB build (run on the eib VM). Elemental images build from /home/eib-workspace,
# standalone RAW images from /home/eib-standalone (same repo, no elemental/ dir)
podman run --rm --privileged \
  -v /home/eib-workspace:/eib:z \
  -v /home/eib-config/base-images:/eib/base-images:ro \
  -v /home/eib-config/rpms:/eib/rpms:ro \
  registry.suse.com/edge/3.7/edge-image-builder:1.3.4 \
  build --definition-file <definition-file.yaml>

# Seed an ISO to a specific edge node
rodeo pull-edge-image --config-dir /root/rodeo-lab \
  --image /home/eib-workspace/<image.iso> \
  --nodes <node> --yes

# Seed a RAW image to a specific edge node
rodeo pull-edge-image --config-dir /root/rodeo-lab \
  --image /home/eib-standalone/<image.raw> \
  --nodes <node> --yes

# After the Elemental install and registration: shut down, eject the ISO, boot from disk
virsh shutdown edge1; virsh shutdown edge2
rodeo eject-iso --nodes edge1,edge2 --yes
virsh start edge1; virsh start edge2

# kubectl on the standalone nodes (RKE2 keeps its own binary and kubeconfig)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.33 \
  "/var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes"
ssh -i /root/.ssh/id_ed25519 root@192.168.122.34 "kubectl get nodes"

# Watch Elemental node registration
kubectl get machineinventory -n fleet-default -w

# Get Elemental registration URL (the registration name varies by plan, so discover it first)
REGNAME=$(kubectl get machineregistration -n fleet-default -o jsonpath='{.items[0].metadata.name}')
kubectl get machineregistration "$REGNAME" \
  -n fleet-default \
  -o jsonpath='{.status.registrationURL}'

# Watch cluster provisioning
kubectl get cluster -n fleet-default -w

# Watch Fleet bundle status
kubectl get bundle -n fleet-default
```

## Component versions

| Component | Version |
|---|---|
| Rancher Prime | 2.15.1 |
| K3s (management cluster) | v1.36.3+k3s1 |
| cert-manager | v1.20.1 |
| Elemental Operator | 1.9.2 |
| Edge Image Builder | 1.3.4 |
| Hauler | 1.2.2 |
| openSUSE Leap Micro | 6.2 |
