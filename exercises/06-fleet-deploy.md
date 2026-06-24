# Exercise 6 — Deploy workloads via Fleet

**Time:** 20 min  
**Previous:** [Exercise 5 — Provision a cluster](05-provision-cluster.md)

---

You now have four running nodes:

- **aerogrid-hub-01** on edge1 (Elemental-provisioned K3s)
- **edge2** registered but not yet assigned to a cluster
- **edge3** running RKE2 standalone (not yet in Rancher)
- **edge4** running K3s standalone (not yet in Rancher)

Fleet is already watching for clusters with the right labels. A `GitRepo` called `alien-geeko` is pre-configured to target any cluster labeled `demo=true` and `edge-type=x86-cluster`.

## 6.1 Trigger Fleet deployment on aerogrid-hub-01

edge1 already has the `demo=true` and `edge-type=x86-cluster` labels from the labeling step in Exercise 5. Once the cluster is active and imported into Rancher's fleet-default workspace, Fleet picks it up automatically.

Check in Rancher UI: **Continuous Delivery > Git Repos > alien-geeko**. When `aerogrid-hub-01` appears in the target clusters section and status moves to `Active`, Fleet is deploying the app.

If it does not appear automatically:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl label cluster aerogrid-hub-01 -n fleet-default \
   demo=true edge-type=x86-cluster --overwrite"
```

## 6.2 Import edge3 and edge4 into Rancher

edge3 and edge4 are running standalone clusters that Rancher does not know about yet. Import them:

In Rancher UI: **Cluster Management > Import Existing**. Give each cluster a name (`aerogrid-branch-rke2`, `aerogrid-branch-k3s`). Rancher generates a `kubectl apply` command with a registration manifest. Run it on each node:

```bash
# On edge3 (RKE2)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.33 \
  "kubectl apply -f <paste-the-registration-manifest-url>"

# On edge4 (K3s)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.34 \
  "kubectl apply -f <paste-the-registration-manifest-url>"
```

Once imported, label them for Fleet:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 "
  kubectl label cluster aerogrid-branch-rke2 -n fleet-default \
    demo=true edge-type=x86-cluster --overwrite
  kubectl label cluster aerogrid-branch-k3s -n fleet-default \
    demo=true edge-type=x86-cluster --overwrite
"
```

Fleet deploys Alien-Geeko to each cluster the moment the labels match. Go to **Continuous Delivery > Git Repos > alien-geeko** and watch the bundle status update per cluster.

## 6.3 Verify deployment

Alien-Geeko is a Node.js web terminal that shows live Kubernetes cluster vitals. Once Fleet deploys it, find the NodePort and open it in a browser:

```bash
# Check fleet bundle status
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get bundle -n fleet-default | grep alien"

# Find the NodePort on edge1
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get svc -A | grep alien"
```

Open `http://192.168.122.31:<nodeport>` for the edge1 deployment.

---

## What you just built

Four nodes, four images, two provisioning paths, one management plane.

**The image-first model:** every node in this lab booted from a purpose-built disk image. No manual SSH config, no package installs, no configuration management applied to a running system. The image IS the configuration. If a node breaks, you re-image it. If a new site opens, you ship the image.

**Why two paths exist:** Elemental (edge1/edge2) is for sites where you do not know the node's final role at image-build time. You build one generic SL Micro image, ship it everywhere, and the management cluster decides what each node becomes after it registers. EIB standalone (edge3/edge4) is for sites where the role is known upfront — you bake K3s or RKE2 into the image and the node is a cluster the moment it boots.

**Why TPM matters:** without TPM, a cloned disk can register as any node in your fleet. With `auth: tpm`, the registration token is derived from hardware. You can revoke a specific node's registration by deleting its `MachineInventory`. Physical theft of the hardware does not compromise other nodes.

**Why Hauler is there:** in a real deployment, remote sites have unreliable internet. Hauler is a portable artifact store. Populate it once at HQ, copy it to a USB drive, and the site's nodes pull everything locally. In this lab, Hauler was pre-populated by the instructor — in production, you run `hauler store sync` to update and redistribute.

---

## Two-minute comparison

| | Elemental (edge1/edge2) | EIB standalone (edge3/edge4) |
|---|---|---|
| Node role decided | At provisioning time (after registration) | At image-build time |
| Cluster control | Rancher provisions and manages K8s | Node runs its own K8s; optionally imported |
| Onboarding | Automatic via TPM phone-home | Manual import or pre-registered via EIB |
| Good for | Mixed-role sites, large fleets, dynamic scaling | Fixed-function nodes, strict airgap, predictable workloads |
| Recovery | Re-register = re-provision from management | Re-image = re-flash disk |
