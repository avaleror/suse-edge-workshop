# Exercise 1 — Tour the environment

**Time:** 15 min  
**Next:** [Exercise 2 — Configure Elemental](02-elemental-setup.md)

---

Get your bearings before touching anything. From the KVM host:

```bash
# What VMs are running, what is off
virsh list --all

# Check the management cluster
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl get nodes && kubectl get pods -A | grep -E 'rancher|elemental|cert-manager|fleet'"
```

You should see one K3s node (the rancher VM itself) with pods for Rancher Prime, cert-manager, the Elemental Operator, and Fleet. This is your management cluster. It manages everything else.

Now check the eib VM — your image factory:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "podman images | grep edge-image-builder && \
   systemctl is-active hauler-registry hauler-fileserver && \
   ls /home/eib-config/"
```

Hauler is running two services: an OCI registry on port 5000 (container images) and an HTTP fileserver on port 8080 (binary artifacts and OS base images). Everything used in this lab comes from one of these two endpoints. The eib VM does not need internet access once Hauler is populated.

Check what is in the Hauler store:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.20 \
  "hauler store info --store /var/lib/hauler 2>/dev/null | head -40"
```

You should see the EIB container image, the Elemental register agent, the Alien-Geeko app image, and the SL Micro base image. These were pre-staged by the instructor before the lab started.

---

**Next:** [Exercise 2 — Configure Elemental](02-elemental-setup.md)
