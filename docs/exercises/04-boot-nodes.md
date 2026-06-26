# Exercise 4 — Seed and boot the four nodes

**Time:** 25 min  
**Previous:** [Exercise 3 — Build EIB images](03-eib-builds.md)  
**Next:** [Exercise 5 — Provision a cluster](05-provision-cluster.md)

---

You have four nodes, two image formats, and two different boot workflows.

## 4.1 Elemental nodes (edge1 and edge2) — ISO workflow

edge1 and edge2 each boot from their own ISO. The ISO contains a self-installer: it boots, writes SL Micro to the virtual disk with the static IP already configured, powers off, and the node reboots into the installed OS where `elemental-register` runs.

Pull each ISO from the eib VM and attach it as a virtual CDROM:

```bash
# edge1 gets its specific ISO (192.168.122.31 baked in)
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-workspace/elemental-edge1.iso \
  --local-from-eib \
  --nodes edge1 \
  --yes

# edge2 gets its specific ISO (192.168.122.32 baked in)
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-workspace/elemental-edge2.iso \
  --local-from-eib \
  --nodes edge2 \
  --yes
```

Each command SCPs the ISO, attaches it as a CDROM (sda, boot-order-1) to that node's libvirt domain, and creates a blank `vda.qcow2` for the OS to install into.

Start both nodes:

```bash
virsh start edge1
virsh start edge2
```

Watch the boot on edge1's serial console (Ctrl+] to exit):

```bash
virsh console edge1
```

You will see OVMF, then the SL Micro installer, then a text progress bar as the OS writes to disk. When you see `System is shutting down` the install is done. The node powers off.

Wait for both to shut down:

```bash
watch virsh list --all | grep edge
```

When both show `shut off`, eject the ISOs and restore disk-first boot:

```bash
rodeo eject-iso --nodes edge1,edge2 --yes
```

Start them again — this time they boot from the installed disk:

```bash
virsh start edge1
virsh start edge2
```

From this point, the nodes are running SL Micro and `elemental-register` is starting. Watch for DHCP leases:

```bash
watch virsh net-dhcp-leases default | grep -E "edge|0e:62:a"
```

## 4.2 Standalone nodes (edge3 and edge4) — RAW workflow

edge3 and edge4 use pre-built RAW images. No installer, no reboot cycle. The node boots from a ready disk image with Kubernetes already configured to start.

Pull and thin-clone each image from the eib VM:

```bash
# edge3 gets the RKE2 image
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-workspace/rke2-edge3.raw \
  --local-from-eib \
  --nodes edge3 \
  --yes

# edge4 gets the K3s image
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-workspace/k3s-edge4.raw \
  --local-from-eib \
  --nodes edge4 \
  --yes
```

Check the disk layout:

```bash
ls -lh /var/lib/libvirt/images/rke2-edge3-base.qcow2 \
        /var/lib/libvirt/images/edge3-vda.qcow2 \
        /var/lib/libvirt/images/k3s-edge4-base.qcow2 \
        /var/lib/libvirt/images/edge4-vda.qcow2
```

The base images hold the full content. The per-node `vda.qcow2` files are thin clones — a few megabytes each — that only store writes that diverge from the base. If you had ten edge3-class nodes, you would clone the base ten times at near-zero disk cost.

Start edge3 and edge4:

```bash
virsh start edge3
virsh start edge4
```

These nodes take 3-5 minutes to come up fully. RKE2 and K3s do their first-run initialization: generating TLS certificates, starting the control plane, and marking the node Ready.

Check from edge3:

```bash
# Get edge3 IP
virsh net-dhcp-leases default | grep "0e:62:a3"

# SSH in and check Kubernetes
ssh -i /root/.ssh/id_ed25519 root@192.168.122.33 \
  "kubectl get nodes"
```

You should see one node in Ready state with the RKE2 version. Do the same for edge4 at 192.168.122.34 and confirm K3s is running there.

---

**Next:** [Exercise 5 — Provision a cluster](05-provision-cluster.md)
