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
# EIB build (run on eib VM — /home/eib-config is the working dir)
podman run --rm --privileged \
  -v /home/eib-config:/eib:z \
  registry.suse.com/edge/3.6/edge-image-builder:1.3.3.1 \
  build --definition-file <definition-file.yaml>

# Seed an ISO to a specific edge node
rodeo pull-edge-image --config-dir /root/rodeo-lab \
  --image /home/eib-config/<image.iso> \
  --local-from-eib --nodes <node> --yes

# Seed a RAW image to a specific edge node
rodeo pull-edge-image --config-dir /root/rodeo-lab \
  --image /home/eib-config/<image.raw> \
  --local-from-eib --nodes <node> --yes

# Eject ISO after Elemental install, restore disk-first boot
rodeo eject-iso --nodes edge1,edge2 --yes

# Watch Elemental node registration
kubectl get machineinventory -n fleet-default -w

# Get Elemental registration URL
kubectl get machineregistration suse-edge-reg-1 \
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
| Rancher Prime | 2.14.1 |
| K3s (management cluster) | v1.35.3+k3s1 |
| cert-manager | v1.20.1 |
| Elemental Operator | 1.9.0 |
| Edge Image Builder | 1.3.3.1 |
| Hauler | 1.2.2 |
| SUSE Linux Micro | 6.2 |
