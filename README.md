# SUSE Edge 3.6 Rodeo — Workshop

A hands-on workshop covering the full SUSE Edge 3.6 lifecycle: image building with EIB, TPM-based edge node onboarding via Elemental, cluster provisioning from Rancher, and GitOps workload delivery via Fleet.

**Duration:** ~2.5 hours  
**Audience:** Platform engineers, SREs, DevOps teams evaluating SUSE Edge for their edge fleet

---

## The scenario

AeroGrid is a regional airport operator managing kiosks, baggage terminals, and cargo dashboards across 47 sites. Every edge node must boot from the same OS image, onboard automatically, and be manageable from a single control plane. New nodes at any site ship with zero pre-configuration at the remote end.

This workshop deploys the AeroGrid management stack and walks you through building images for four edge nodes across two different provisioning paths.

---

## Lab topology

```
KVM host (bare metal)
│
├── rancher   192.168.122.9    Rancher Prime 2.14.1 + Elemental Operator 1.9.0 + Fleet
├── eib       192.168.122.20   EIB 1.3.3.1 + Hauler 1.2.2 (local artifact registry)
│
├── edge1     192.168.122.31   vTPM 2.0, UEFI  →  Elemental onboarding  →  K3s cluster
├── edge2     192.168.122.32   vTPM 2.0, UEFI  →  Elemental onboarding  →  standalone
├── edge3     192.168.122.33   UEFI             →  EIB standalone        →  RKE2 cluster
└── edge4     192.168.122.34   UEFI             →  EIB standalone        →  K3s cluster
```

Two provisioning paths run in parallel:

- **Elemental path (edge1, edge2):** Phone-home onboarding via TPM. The node boots, the Elemental agent contacts the registration endpoint, and the management cluster takes control.
- **EIB standalone path (edge3, edge4):** The full Kubernetes stack is baked into the disk image. Boot it, you have a running cluster. No phone-home. No registration.

---

## Exercises

| # | Exercise | Time |
|---|---|---|
| [1](exercises/01-environment-tour.md) | Tour the environment | 15 min |
| [2](exercises/02-elemental-setup.md) | Configure Elemental, the registration endpoint, and the node network plan | 30 min |
| [3](exercises/03-eib-builds.md) | Build four EIB images — one per node, each with its own network config | 50 min |
| [4](exercises/04-boot-nodes.md) | Seed and boot the four nodes | 25 min |
| [5](exercises/05-provision-cluster.md) | Provision a K3s cluster on an Elemental node | 20 min |
| [6](exercises/06-fleet-deploy.md) | Deploy workloads via Fleet | 20 min |

---

## Quick access

| Resource | Value |
|---|---|
| Rancher UI | `https://<host-ip>:30002` |
| Rancher user | `admin` |
| Rancher password | `cat ~/.rodeo/secrets.yaml` |
| Management cluster SSH | `ssh -i /root/.ssh/id_ed25519 root@192.168.122.9` |
| EIB VM SSH | `ssh -i /root/.ssh/id_ed25519 root@192.168.122.20` |
| Hauler fileserver | `http://192.168.122.20:8080` |

Full reference: [reference/quick-reference.md](reference/quick-reference.md)

---

## For instructors

- [Host setup and deployment](instructor/host-setup.md)
- [Pre-lab checklist](instructor/pre-lab-checklist.md)

---

## Component versions

| Component | Version |
|---|---|
| Rancher Prime | 2.14.1 |
| Elemental Operator | 1.9.0 |
| Edge Image Builder | 1.3.3.1 |
| Hauler | 1.2.2 |
| SUSE Linux Micro | 6.2 |
| K3s (management) | v1.35.3+k3s1 |

---

*Deployed with [rodeo-cli](https://github.com/avaleror/rodeo-cli) v0.10.x.*
