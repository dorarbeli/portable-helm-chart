# PROMPT — Portable Helm Chart (Dor, DevOps practice project #2)

> Paste this whole file as the FIRST message to a fresh Opus session. It is built to **replace re-reading Dor's repos** — everything a new session needs is here, so don't go re-scanning all his files (saves tokens). Step-by-step build plan lives in `GUIDE.md` right next to this file.

## Who / goal
- **Dor Arbeli**, dorarbeli17@gmail.com. **NOC Operator** now (Octally, since Aug 2025); DevOps bootcamp grad (Ort Singalovski) + **RHCSA**.
- **Long-run goal:** junior **DevOps / Cloud / SRE**. **Short-run:** open to technical-support / NOC as a stepping-stone.
- Learns by doing. Wants the **why** behind each step and an explicit **🤖 (assistant) / 👤 (Dor)** split. Prefers concise over verbose. Non-native English speaker — keep language plain.

## Proven stack (don't re-teach these)
Docker · Kubernetes · Terraform (VPC/EKS/RDS) · AWS (ECR/EKS/VPC/RDS/Route53) · GitLab CI/CD (4-stage) · ArgoCD/GitOps · Prometheus · Python/Flask · MySQL · Bash/Linux. Two public repos prove it: **`feeder`** and **`url-shortener`** (both: AWS EKS + Terraform + GitLab CI, ~$5/day to run).

## Why THIS project (the gap)
Both existing repos are the **same AWS-only stack** and cost money to run. The untouched, market-relevant gaps are: **Helm** (authoring a real chart, not just a `helm_release`), **GitHub Actions** (he only has GitLab CI), and **portability** (everything is welded to AWS). One project that hits all three = maximum standout for minimum cost.
- **Hard rule: never introduce Ansible** — Dor's explicit exclusion.

## The project — `portable-helm-chart`
Take the existing **url-shortener** Flask app and convert its hand-written k8s manifests into a **production-grade, portable Helm chart** that deploys **unchanged** to (a) a **free local cluster now** and (b) **AWS EKS later** — the only difference between environments is **which values file you pass**. Then **publish** the chart as a Helm repo and drive **lint / test / package / publish with GitHub Actions**.

**App facts (from source — so the chart is accurate):**
- Flask, listens on **:5000**, has **/healthz** (liveness/readiness), routes: `/`, `POST /api/shorten`, `GET /<code>`.
- Config is **env-only**: `DB_HOST, DB_USER, DB_PASSWORD, DB_NAME, PUBLIC_BASE_URL`.
- Needs **MySQL**. Current manifests: `deployment` (replicas 2, ECR image hardcoded, probes, resource req/limits), `service` (ClusterIP 80→5000), `ingress` (host + TLS), `cert-issuer` (Let's Encrypt staging + prod ClusterIssuers), `db-secret.yaml.example`.

**Parameterize into `values.yaml`:** image repo+tag, replicas, resources, env block, `PUBLIC_BASE_URL`, ingress (enabled/host/className), TLS issuer (on/off), and **DB mode (in-cluster vs external)**.

**The portability trick (the core lesson):**
- **`values-local.yaml`** → image = locally built + `k3d image import`ed; DB = **in-cluster MySQL** (Bitnami subchart, or a tiny inline manifest); ingress via k3d's built-in Traefik (or NodePort); **no TLS**. Cost **$0**.
- **`values-aws.yaml`** → image = **ECR**; DB = external **RDS** endpoint via Secret (no subchart); ingress-nginx + cert-manager TLS. Reuses url-shortener's existing Terraform if/when he wants a live cloud proof.

## Environment & cost rules
- **"Local now, cloud-ready":** build + run on **k3d** (k3s-in-Docker) = **$0**. No AWS spend is required to finish or learn this project.
- **Optional free public demo:** **Cloudflare Tunnel** → a clickable URL for recruiters, no domain or load-balancer cost.
- **AWS is optional, last, and temporary:** only `helm install -f values-aws.yaml` on EKS to *prove* portability, screenshot it, then `terraform destroy`. Same ~$5/day caveat as his other projects — never leave it running.

## Publishing the chart (verified current, 2026)
Offer both; **default to gh-pages for visibility**:
- **gh-pages via `helm/chart-releaser-action`** → browsable public repo + `index.yaml`; recruiters can `helm repo add`. Most demoable for a portfolio.
- **OCI via GHCR** (`helm push … oci://ghcr.io/dorarbeli`) → native since Helm 3.8, no `index.yaml`, works on private repos. The modern default — mention as "what teams do now."
- **CI shape (GitHub Actions):** `helm lint` + `helm template` on every PR → **chart-testing (`ct`) install test in an ephemeral `kind` cluster** → on tag: package + publish. All on **free** GitHub-hosted runners.

## Working files / paths
- **New project dir:** `C:\Users\dorar\Desktop\Projects\devops-projects\portable-helm-chart\` (bash: `/sessions/<id>/mnt/Projects/devops-projects/portable-helm-chart/`).
- **Source app to package:** `…\devops-projects\url-shortener\` (`app.py`, `Dockerfile`, `k8s/*`).
- This brief + **`GUIDE.md`** (phase-by-phase source of truth) live in the new dir.
- If folders aren't mounted, request with `mcp__cowork__request_cowork_directory`.

## Landmines
- **Never** ask Dor to paste cloud creds into chat. Local k3d needs none; the AWS path uses his own CLI / GitHub repo secrets.
- Run **`helm dependency update`** before install when using the MySQL subchart (it pulls the dep into `charts/`).
- Bitnami's distribution/registry terms shifted in 2025 — **if the MySQL subchart pull fails, fall back to the small inline MySQL manifest** (both are covered in `GUIDE.md`).
- In-cluster MySQL needs a few hundred MB RAM + a PVC — fine on k3d; keep an eye on laptop RAM (~8 GB free recommended).
- Keep the **app minimal** — the point is the Helm / portability skill, not new app features.
- gitignore any real values (`values-*.secret.yaml`); ship `*.example` only — his established pattern.

## Supersedes
This replaces project #2 ("Notes API → DynamoDB + IRSA") from the old `devops-projects/HANDOFF.md` 7-project plan. Dor chose **Helm + portability** instead on 2026-06-07.

## First message to send Dor
"Ready to start **Phase 0** (install k3d + Helm, spin up a free local cluster), or do you want me to re-explain how a Helm chart makes the same app run anywhere first?"
