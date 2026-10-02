# Exercise 6: Deploy workloads via Fleet

**Time:** 20 min  
**Previous:** [Exercise 5: Provision a cluster](05-provision-cluster.md)

---

You now have four running nodes:

- **vertex-hub-01** on edge1 (Elemental-provisioned K3s)
- **edge2** registered but not yet assigned to a cluster
- **edge3** running RKE2 standalone (not yet in Rancher)
- **edge4** running K3s standalone (not yet in Rancher)

Fleet is already watching for clusters with the right labels. A `GitRepo` called `vertex-bank-app` is pre-configured to target any cluster labeled `demo=true` and `edge-type=x86-cluster`.

The `vertex-bank-app` GitRepo points at the local Gitea instance on the EIB VM, not GitHub. Fleet syncs from `http://192.168.122.20:3000/gitea/vertex-bank-app.git` every 15 seconds. No internet access is needed.

```bash
# Confirm the GitRepo source
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl --kubeconfig=/etc/rancher/k3s/k3s.yaml \
   get gitrepo vertex-bank-app -n fleet-default \
   -o jsonpath='{.spec.repo}'"
```

## 6.1 Trigger Fleet deployment on vertex-hub-01

edge1's `MachineInventory` has the `demo=true` and `edge-type=x86-cluster` labels from Exercise 5, but those do not propagate to the cluster Fleet actually watches. Label it explicitly:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl label clusters.fleet.cattle.io vertex-hub-01 -n fleet-default \
   demo=true edge-type=x86-cluster --overwrite"
```

Use the full `clusters.fleet.cattle.io` name. This Rancher/Fleet setup has at least three separate CRDs all called `Cluster`, and a bare `kubectl label cluster ...` resolves to a different one. It succeeds with no error, but Fleet keeps showing 0/0 targeted clusters because the label never lands where Fleet is looking.

Check in Rancher UI: **Continuous Delivery > Git Repos > vertex-bank-app**. When `vertex-hub-01` appears in the target clusters section, Fleet is deploying the app. It picks up the label within its normal 15-second poll cycle.

## 6.2 Import edge3 and edge4 into Rancher

edge3 and edge4 are running standalone clusters that Rancher does not know about yet. Import them:

In Rancher UI: **Cluster Management > Import Existing > Generic**. Give each cluster a name (`vertex-branch-rke2`, `vertex-branch-k3s`) and create it. Rancher then shows registration commands. This lab's Rancher uses a self-signed certificate, so copy the **second** one, the `curl --insecure ... | kubectl apply -f -` variant. The plain `kubectl apply -f <url>` command fails with `x509: certificate signed by unknown authority`.

Run it on each node:

```bash
# On edge3 (RKE2 keeps its kubectl under /var/lib/rancher/rke2/bin)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.33 \
  "curl --insecure -sfL <registration-manifest-url> | \
   /var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml apply -f -"

# On edge4 (K3s)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.34 \
  "curl --insecure -sfL <registration-manifest-url> | kubectl apply -f -"
```

Each cluster turns `Active` in **Cluster Management** within a couple of minutes.

Once imported, label them for Fleet:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 "
  kubectl label clusters.fleet.cattle.io vertex-branch-rke2 -n fleet-default \
    demo=true edge-type=x86-cluster --overwrite
  kubectl label clusters.fleet.cattle.io vertex-branch-k3s -n fleet-default \
    demo=true edge-type=x86-cluster --overwrite
"
```

Fleet deploys vertex-bank-app to each cluster the moment the labels match. Go to **Continuous Delivery > Git Repos > vertex-bank-app** and watch the bundle status update per cluster.

## 6.3 Verify deployment

vertex-bank-app is a web app that shows live Kubernetes cluster vitals. Check the bundle status from the management cluster:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get bundle -n fleet-default | grep vertex-bank"
```

The app itself runs on the edge clusters, not on the management cluster, so query vertex-hub-01 through the kubeconfig Rancher stores for it:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 "
  kubectl get secret vertex-hub-01-kubeconfig -n fleet-default \
    -o jsonpath='{.data.value}' | base64 -d > /tmp/vertex-hub-01.yaml
  kubectl --kubeconfig /tmp/vertex-hub-01.yaml get pods,svc -n vertex-bank
"
```

The service publishes NodePort `30080`. Its `EXTERNAL-IP` stays `<pending>` because nothing in this lab hands out LoadBalancer IPs, so the bundle can also show `NotReady` in Rancher. That is expected; the NodePort is what you use. Open the app on each node:

```bash
curl -s http://192.168.122.31:30080/ | grep -o "<title>.*</title>"   # vertex-hub-01 (edge1)
curl -s http://192.168.122.33:30080/ | grep -o "<title>.*</title>"   # vertex-branch-rke2 (edge3)
curl -s http://192.168.122.34:30080/ | grep -o "<title>.*</title>"   # vertex-branch-k3s (edge4)
```

Or open `http://192.168.122.31:30080` in a browser.

---

## What you just built

Four nodes, four images, two provisioning paths, one management plane.

**The image-first model:** every node in this lab booted from a purpose-built disk image. No manual SSH config, no package installs, no configuration management applied to a running system. The image IS the configuration. If a node breaks, you re-image it. If a new site opens, you ship the image.

**Why two paths exist:** Elemental (edge1/edge2) is for sites where you do not know the node's final role at image-build time. You build one generic openSUSE Leap Micro image, ship it everywhere, and the management cluster decides what each node becomes after it registers. EIB standalone (edge3/edge4) is for sites where the role is known upfront: you bake K3s or RKE2 into the image and the node is a cluster the moment it boots.

**Why TPM matters:** without TPM, a cloned disk can register as any node in your fleet. With `auth: tpm`, the registration token is derived from hardware. You can revoke a specific node's registration by deleting its `MachineInventory`. Physical theft of the hardware does not compromise other nodes.

**Why Hauler is there:** in a real deployment, remote sites have unreliable internet. Hauler is a portable artifact store. Populate it once at HQ, copy it to a USB drive, and the site's nodes pull everything locally. In this lab, Hauler was pre-populated by the instructor. In production, you run `hauler store sync` to update and redistribute.

---

## Two-minute comparison

| | Elemental (edge1/edge2) | EIB standalone (edge3/edge4) |
|---|---|---|
| Node role decided | At provisioning time (after registration) | At image-build time |
| Cluster control | Rancher provisions and manages K8s | Node runs its own K8s; optionally imported |
| Onboarding | Automatic via TPM phone-home | Manual import or pre-registered via EIB |
| Good for | Mixed-role sites, large fleets, dynamic scaling | Fixed-function nodes, strict airgap, predictable workloads |
| Recovery | Re-register = re-provision from management | Re-image = re-flash disk |
