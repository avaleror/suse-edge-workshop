# Bonus exercises

If you finish early or want to go deeper.

Want to understand how Hauler works under the hood? There is a dedicated bonus lab: [Hauler — build and serve an air-gap bundle](bonus-hauler.md).

## Try a second Elemental cluster with RKE2

Label edge2 with `site-role: hub` and create a second `Cluster` resource pointing at the same `aerogrid-hub-selector` template with `quantity: 1` targeting edge2. Rancher will provision RKE2 on edge2.

## See what a failed registration looks like

Power off edge1, clear its TPM state, and boot it again:

```bash
virsh destroy edge1
# Clear the vTPM state directory
rm -rf /var/lib/libvirt/swtpm/<edge1-uuid>
virsh start edge1
```

Watch the Elemental logs on the management cluster:

```bash
ssh -i /root/.ssh/id_ed25519 root@192.168.122.9 \
  "kubectl logs -n cattle-elemental-system -l app=elemental-operator -f"
```

The registration attempt will be rejected because the TPM identity changed. The `MachineInventory` record will not update. This is the expected failure mode for hardware replacement at a remote site — you control the replacement from the management cluster, not from the site.

## Build a custom EIB image with additional packages

Add a `packages:` block to any of the definition yamls and rebuild:

```yaml
operatingSystem:
  packages:
    packageList:
      - htop
      - curl
      - vim
  packageManager:
    additionalRepos:
      - url: https://download.opensuse.org/distribution/leap-micro/6.2/product/repo/Leap-Micro-6.2-x86_64-Media.repo
```

This downloads and embeds the packages at build time so the node never needs to call a package repo after boot.
