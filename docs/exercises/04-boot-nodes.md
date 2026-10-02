# Exercise 4: Seed and boot the four nodes

**Time:** 25 min  
**Previous:** [Exercise 3: Build EIB images](03-eib-builds.md)  
**Next:** [Exercise 5: Provision a cluster](05-provision-cluster.md)

---

You have four nodes, two image formats, and two different boot workflows.

## 4.1 Elemental nodes (edge1 and edge2): ISO workflow

edge1 and edge2 each boot from their own ISO. The ISO contains a self-installer: it boots, writes openSUSE Leap Micro to the virtual disk with the static IP already configured, powers off, and the node reboots into the installed OS where `elemental-register` runs.

Pull each ISO from the eib VM and attach it as a virtual CDROM:

```bash
# edge1 gets its specific ISO (192.168.122.31 baked in)
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-workspace/elemental-edge1.iso \
  --nodes edge1 \
  --yes

# edge2 gets its specific ISO (192.168.122.32 baked in)
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-workspace/elemental-edge2.iso \
  --nodes edge2 \
  --yes
```

Each command SCPs the ISO, attaches it as a CDROM (sda, boot-order-1) to that node's libvirt domain, and creates a blank `vda.qcow2` for the OS to install into.

Start both nodes:

```bash
virsh start edge1
virsh start edge2
```

The installer needs two keypresses per node before it runs unattended. The GRUB boot menu waits indefinitely instead of auto-selecting (by design, so install media never silently wipes a disk), and the partitioner asks you to confirm before it writes to `/dev/vda`. Watch each node's console in turn (Ctrl+] to exit) and press Enter at both points:

```bash
virsh console edge1
# Press Enter to select "Install openSUSE Leap Micro"
# Press Enter again to confirm "Destroying ALL data on /dev/vda, continue?"
```

If the console shows nothing, send the keys straight from the KVM host instead. Wait about 30 seconds after the first one, until the confirmation dialog is on screen:

```bash
virsh send-key edge1 KEY_ENTER     # GRUB: "Install openSUSE Leap Micro"
virsh send-key edge1 KEY_ENTER     # "Destroying ALL data on /dev/vda, continue?" -> Yes
```

Do the same for edge2. Once both are past the confirmation, the rest runs on its own. You see a progress bar while the OS writes to disk, then the node reboots straight into the installed system and `elemental-register` phones home. After a few minutes the console shows a login prompt with the node's static IP (`enp1s0: 192.168.122.31` for edge1).

Check that both nodes registered with the management cluster:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get machineinventory -n fleet-default"
```

When both appear, shut the nodes down cleanly, eject the ISOs and restore disk-first boot. The ISO is still attached at this point, so without this step the next reboot would land on the installer's GRUB menu again:

```bash
virsh shutdown edge1
virsh shutdown edge2
watch "virsh list --all | grep edge"     # wait until both show "shut off"

rodeo eject-iso --nodes edge1,edge2 --yes
virsh start edge1
virsh start edge2
```

They now boot from the installed disk and keep their static IPs:

```bash
ping -c3 192.168.122.31
ping -c3 192.168.122.32
```

## 4.2 Standalone nodes (edge3 and edge4): RAW workflow

edge3 and edge4 use pre-built RAW images. No installer, no reboot cycle. The node boots from a ready disk image with Kubernetes already configured to start.

Pull and thin-clone each image from the eib VM:

```bash
# edge3 gets the RKE2 image
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-standalone/rke2-edge3.raw \
  --nodes edge3 \
  --yes

# edge4 gets the K3s image
rodeo pull-edge-image \
  --config-dir /root/rodeo-lab \
  --image /home/eib-standalone/k3s-edge4.raw \
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

The base images hold the full content. The per-node `vda.qcow2` files are thin clones, a few megabytes each, that only store writes that diverge from the base. If you had ten edge3-class nodes, you would clone the base ten times at near-zero disk cost.

Start edge3 and edge4:

```bash
virsh start edge3
virsh start edge4
```

These nodes take 3-5 minutes to come up fully. RKE2 and K3s do their first-run initialization: generating TLS certificates, starting the control plane, and marking the node Ready.

Both nodes have static IPs, so there are no DHCP leases to look for. SSH in and check Kubernetes. The RAW images authorize the KVM host's key for root, and RKE2 keeps its own `kubectl` and kubeconfig under its install paths:

```bash
# edge3 (RKE2)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.33 \
  "/var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes"

# edge4 (K3s)
ssh -i /root/.ssh/id_ed25519 root@192.168.122.34 \
  "kubectl get nodes"
```

Each shows one node in `Ready` state: `v1.36.3+rke2r1` on edge3 and `v1.36.3+k3s1` on edge4.

---

**Next:** [Exercise 5: Provision a cluster](05-provision-cluster.md)
