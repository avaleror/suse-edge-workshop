# Exercise 5 — Provision a K3s cluster on an Elemental node

**Time:** 20 min  
**Previous:** [Exercise 4 — Seed and boot nodes](04-boot-nodes.md)  
**Next:** [Exercise 6 — Deploy workloads via Fleet](06-fleet-deploy.md)

---

edge1 and edge2 are now running SL Micro. The `elemental-register` agent has phoned home and registered them with the Elemental Operator. But they are not yet Kubernetes nodes — that happens here.

## 5.1 Check MachineInventory

Watch for the registered nodes on the management cluster:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineinventory -n fleet-default -w"
```

You should see one or two entries appear — one per node that has completed registration. Each row is a node that has proven its TPM identity and is now waiting to be told what to do.

In the Rancher UI: go to **OS Management > MachineInventory**. The same records appear there with hardware info, labels inherited from the MachineRegistration, and status.

## 5.2 Create a MachineInventorySelectorTemplate

This resource links a label selector to a machine pool. Any `MachineInventory` that matches the selector becomes a candidate node for a cluster:

```bash
cat << 'EOF' | ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 "kubectl apply -f -"
apiVersion: elemental.cattle.io/v1beta1
kind: MachineInventorySelectorTemplate
metadata:
  name: aerogrid-hub-selector
  namespace: fleet-default
spec:
  template:
    spec:
      selector:
        matchLabels:
          site-role: hub
EOF
```

## 5.3 Label edge1 for cluster assignment

Right now neither node matches `site-role: hub`. Label edge1 to trigger the match:

```bash
# Get edge1's MachineInventory name
EDGE1=$(ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineinventory -n fleet-default \
   -o jsonpath='{.items[0].metadata.name}'")

echo "edge1 MachineInventory: $EDGE1"

# Label it
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl label machineinventory $EDGE1 \
   -n fleet-default \
   site-role=hub \
   demo=true \
   edge-type=x86-cluster"
```

## 5.4 Create the cluster

Create a `Cluster` resource that uses the selector template as its machine pool:

```bash
cat << 'EOF' | ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 "kubectl apply -f -"
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: aerogrid-hub-01
  namespace: fleet-default
spec:
  kubernetesVersion: v1.35.3+k3s1
  rkeConfig:
    machinePools:
      - name: hub-nodes
        quantity: 1
        etcdRole: true
        controlPlaneRole: true
        workerRole: true
        machineConfigRef:
          kind: MachineInventorySelectorTemplate
          name: aerogrid-hub-selector
          apiVersion: elemental.cattle.io/v1beta1
EOF
```

## 5.5 Watch the cluster provision

In Rancher UI: go to **Cluster Management**. You will see `aerogrid-hub-01` appear with status `Provisioning`. The management cluster is now remotely installing K3s on edge1 via the Elemental system agent.

From the terminal:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get cluster -n fleet-default -w"
```

Provisioning takes 5-10 minutes. When status changes to `Active`, the cluster is ready.

**What just happened:** you provisioned a Kubernetes cluster on a remote node without touching the node. The node registered itself, you applied a label, and the management cluster did the rest. This is the Elemental model for large-scale edge deployments.

---

**Next:** [Exercise 6 — Deploy workloads via Fleet](06-fleet-deploy.md)
