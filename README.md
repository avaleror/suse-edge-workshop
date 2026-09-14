# SUSE Edge 3.6 Rodeo: Workshop

Hands-on workshop covering the full SUSE Edge 3.6 lifecycle: image building with EIB, TPM-based edge node onboarding via Elemental, cluster provisioning from Rancher, and GitOps workload delivery via Fleet.

**Workshop site:** https://avaleror.github.io/suse-edge-workshop/

---

## Repo structure

```
docs/
  index.md                  # Landing page (GitHub Pages home)
  exercises/                # Six step-by-step exercises
  reference/                # Quick reference and bonus exercises
  instructor/               # Host setup and pre-lab checklist
  lab-guide.md              # Full single-file version (printable)
rodeo-plan.yaml             # Deployment plan for rodeo-cli
mkdocs.yml                  # MkDocs Material config
.github/workflows/
  deploy.yml                # Auto-deploy to GitHub Pages on push to main
```

## Run the workshop

Deploy the lab environment on a bare metal KVM host:

```bash
curl -fsSL https://raw.githubusercontent.com/avaleror/rodeo-cli/main/install.sh | bash
git clone https://github.com/avaleror/suse-edge-workshop.git
cd suse-edge-workshop
rodeo doctor
rodeo up
```

`rodeo up` self-escalates with sudo, generates `~/.rodeo/secrets.yaml`, wraps the deploy in tmux, and prints login URLs when finished — no separate `rodeo init`/`sudo rodeo deploy` steps needed, since this repo already ships its own `rodeo-plan.yaml`.

Then follow the lab guide at https://avaleror.github.io/suse-edge-workshop/

## Local docs preview

```bash
pip install mkdocs-material
mkdocs serve
```

Open http://127.0.0.1:8000

---

*Deployed with [rodeo-cli](https://github.com/avaleror/rodeo-cli).*
