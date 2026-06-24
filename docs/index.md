---
hide:
  - toc
---

<div class="hero">
  <div class="subtitle">SUSE Edge 3.6 — Hands-on Workshop</div>
  <h1>The Edge Rodeo</h1>
  <div class="tagline">Rancher Prime &bull; Elemental &bull; Edge Image Builder &bull; Fleet &bull; Hauler</div>

  <a href="exercises/01-environment-tour/" class="md-button md-button--primary">Start the lab &rarr;</a>
  &nbsp;
  <a href="reference/quick-reference/" class="md-button">Quick reference</a>
</div>

---

## The scenario

<div class="scenario">
AeroGrid is a regional airport operator managing self-service kiosks, baggage tracking terminals, and cargo dashboards across 47 sites. Every edge node must boot from the same OS image, onboard automatically with no hands-on config at the remote end, and be manageable from a single control plane.

This workshop deploys the AeroGrid management stack and walks you through building and booting four edge nodes across two different provisioning paths.
</div>

## Lab topology

<div class="topology">
<pre>
KVM host (bare metal)
│
├── rancher   192.168.122.9    Rancher Prime 2.14.1 + Elemental Operator 1.9.0 + Fleet
├── eib       192.168.122.20   EIB 1.3.3.1 + Hauler 1.2.2
│
├── edge1     192.168.122.31   vTPM 2.0  ──► Elemental onboarding  ──► K3s cluster
├── edge2     192.168.122.32   vTPM 2.0  ──► Elemental onboarding  ──► standalone
├── edge3     192.168.122.33   UEFI      ──► EIB standalone        ──► RKE2 cluster
└── edge4     192.168.122.34   UEFI      ──► EIB standalone        ──► K3s cluster
</pre>
</div>

Two provisioning paths run in parallel throughout this lab:

<div style="display:grid;grid-template-columns:1fr 1fr;gap:1rem;margin:1rem 0;">
<div style="background:#1e2e24;border:1px solid #30BA78;border-radius:0.4rem;padding:1rem;">
<span class="path-badge elemental">Elemental path</span><br>
<strong>edge1, edge2</strong><br>
Phone-home onboarding via TPM. The node boots, the Elemental agent contacts the registration endpoint, and the management cluster takes control. Cluster provisioning happens from Rancher.
</div>
<div style="background:#1a1e2e;border:1px solid #6495ed;border-radius:0.4rem;padding:1rem;">
<span class="path-badge eib">EIB standalone path</span><br>
<strong>edge3, edge4</strong><br>
The full Kubernetes stack is baked into the disk image. Boot it, you have a running cluster. No phone-home. No registration step.
</div>
</div>

---

## Exercises

<div class="exercise-grid">

<a href="exercises/01-environment-tour/" class="exercise-card">
  <div class="ex-number">Exercise 01</div>
  <div class="ex-title">Tour the environment</div>
  <div class="ex-time">⏱ 15 min</div>
  <div class="ex-desc">Check the management cluster, the EIB image factory, and the Hauler artifact store before touching anything.</div>
</a>

<a href="exercises/02-elemental-setup/" class="exercise-card">
  <div class="ex-number">Exercise 02</div>
  <div class="ex-title">Configure Elemental and the network plan</div>
  <div class="ex-time">⏱ 30 min</div>
  <div class="ex-desc">Create the MachineRegistration, get the registration URL, and define a static IP for each node via NMState configs.</div>
</a>

<a href="exercises/03-eib-builds/" class="exercise-card">
  <div class="ex-number">Exercise 03</div>
  <div class="ex-title">Build four EIB images</div>
  <div class="ex-time">⏱ 50 min</div>
  <div class="ex-desc">One image per node, each with its own static IP baked in. Two Elemental ISOs, one RKE2 RAW, one K3s RAW.</div>
</a>

<a href="exercises/04-boot-nodes/" class="exercise-card">
  <div class="ex-number">Exercise 04</div>
  <div class="ex-title">Seed and boot the four nodes</div>
  <div class="ex-time">⏱ 25 min</div>
  <div class="ex-desc">ISO workflow for Elemental nodes, thin-clone RAW workflow for standalone cluster nodes.</div>
</a>

<a href="exercises/05-provision-cluster/" class="exercise-card">
  <div class="ex-number">Exercise 05</div>
  <div class="ex-title">Provision a K3s cluster</div>
  <div class="ex-time">⏱ 20 min</div>
  <div class="ex-desc">Label a registered Elemental node and watch Rancher provision a full K3s cluster on it remotely.</div>
</a>

<a href="exercises/06-fleet-deploy/" class="exercise-card">
  <div class="ex-number">Exercise 06</div>
  <div class="ex-title">Deploy workloads via Fleet</div>
  <div class="ex-time">⏱ 20 min</div>
  <div class="ex-desc">Label clusters for Fleet, import standalone nodes into Rancher, and watch Alien-Geeko deploy across the fleet.</div>
</a>

</div>

---

## Resources

<div class="link-row">
  <a href="reference/quick-reference/">📋 Quick reference</a>
  <a href="reference/bonus-exercises/">🎯 Bonus exercises</a>
  <a href="lab-guide/">📄 Full lab guide (print)</a>
  <a href="instructor/host-setup/">🔧 Instructor: host setup</a>
  <a href="instructor/pre-lab-checklist/">✅ Instructor: pre-lab checklist</a>
  <a href="https://github.com/avaleror/rodeo-cli" target="_blank">⚙️ rodeo-cli</a>
</div>

---

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
