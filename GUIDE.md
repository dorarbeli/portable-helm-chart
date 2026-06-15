# Step-by-step Build Guide — Portable Helm Chart

This walks you from "hand-written YAML welded to AWS" to "one Helm chart that runs free on your laptop **and** on EKS unchanged, published as a real Helm repo, tested by GitHub Actions." Same teaching style as your `url-shortener/GUIDE.md`.

> **Legend**
> 🤖 = I do it (or you can ask me to — pure typing/file edits)
> 👤 = You do it (needs your machine, accounts, money, or judgment)

**The one big idea:** a Helm chart is your manifests with the environment-specific bits pulled out into `values.yaml`. Same chart + `values-local.yaml` → runs on k3d for $0. Same chart + `values-aws.yaml` → runs on EKS. *That* is portability, and it's the skill this project sells.

**Total cash cost to finish: $0.** AWS only appears in optional Phase 8.

---

## Phase 0 — Tools + a free local cluster (15 min, $0)

**Goal:** a real Kubernetes cluster running on your laptop, costing nothing.

1. 👤 **Install:** Docker Desktop, `kubectl`, `helm`, and **`k3d`** (k3s-in-Docker — lightest local k8s). *Why you: I can't install software on your machine.*
2. 👤 **Create the cluster** (maps localhost:8080 to the cluster's web entry point so we can reach apps without a cloud load balancer):
   ```sh
   k3d cluster create devops -p "8080:80@loadbalancer" --agents 1
   kubectl get nodes        # 2 nodes Ready (1 server, 1 agent)
   ```
   *Why you: runs on your hardware.*
3. 🤖 Ask me to explain what k3d/k3s actually is vs EKS if you want the mental model first.

**What you learn:** local k8s, why k3d ≈ EKS for practice but free.

---

## Phase 1 — Run the app raw on k3d first (20 min, $0)

**Goal:** prove the **existing** url-shortener works locally *before* Helm-ifying it, so you know any later breakage is the chart, not the app.

1. 👤 **Build the image and load it into k3d** (no registry needed):
   ```sh
   cd ..\url-shortener
   docker build -t url-shortener:dev .
   k3d image import url-shortener:dev -c devops
   ```
   *Why you: builds on your Docker.*
2. 👤 **Run a throwaway in-cluster MySQL** (one command, no cloud DB):
   ```sh
   kubectl run mysql --image=mysql:8 --env="MYSQL_ROOT_PASSWORD=devpass" --env="MYSQL_DATABASE=shortener" --port=3306
   kubectl expose pod mysql --port=3306
   ```
3. 🤖 I'll hand you a temporary `deployment`/`service` pointed at `url-shortener:dev` and that MySQL, with `PUBLIC_BASE_URL` empty.
4. 👤 **Apply + smoke test:**
   ```sh
   kubectl apply -f <the temp manifests>
   kubectl port-forward svc/shortener 5000:80
   curl -X POST localhost:5000/api/shorten -H 'Content-Type: application/json' -d '{"url":"https://anthropic.com"}'
   ```
   A short URL back = the app is healthy on k3d. Delete these temp objects after — Helm takes over next.

**What you learn:** the manual baseline you're about to automate away.

---

## Phase 2 — Scaffold the chart (30 min, $0) — 🤖 heavy

**Goal:** a chart skeleton with your manifests moved into templates and the AWS-specific bits ripped out into values.

1. 👤 **Generate the skeleton:** `helm create url-shortener` inside `portable-helm-chart/`, then delete the sample `templates/*` Helm gives you.
2. 🤖 I move your real manifests into `templates/` (`deployment.yaml`, `service.yaml`, `ingress.yaml`, `cert-issuer.yaml`, `secret.yaml`) and replace hardcoded values with `{{ .Values.* }}`:
   - image → `{{ .Values.image.repository }}:{{ .Values.image.tag }}`
   - `replicas`, `resources`, every `env` value, `PUBLIC_BASE_URL`
   - ingress host/class + a `{{- if .Values.ingress.enabled }}` guard
   - TLS block + cert-issuer behind `{{- if .Values.tls.enabled }}`
3. 🤖 I add `_helpers.tpl` (standard name/label helpers) so naming is consistent.
4. 👤 **Sanity render** (no cluster needed): `helm template . | less` — read the YAML it produces.

**What you learn:** the anatomy of a chart — `Chart.yaml`, `values.yaml`, `templates/`, `_helpers.tpl`, and how templating replaces copy-paste YAML.

---

## Phase 3 — `values-local.yaml` + in-cluster DB → install for $0 (30 min)

**Goal:** `helm install` the chart locally with **one** values file and hit it in a browser.

1. 🤖 I write **`values-local.yaml`:** local image, `ingress.enabled: true` (k3d Traefik), `tls.enabled: false`, and **DB mode = in-cluster**.
2. 🤖 **DB choice (I'll set up whichever you pick):**
   - **A) Bitnami MySQL subchart** — add a `dependencies:` entry in `Chart.yaml`, then 👤 run `helm dependency update` (pulls it into `charts/`). Real-world skill: depending on community charts.
   - **B) Inline MySQL** — a small `mysql.yaml` template guarded by `{{- if .Values.mysql.internal }}`. More in your control; teaches subchart-vs-inline tradeoffs.
   > If Bitnami's pull fails (their terms changed in 2025), we use B. Both end the same place.
3. 👤 **Install + test:**
   ```sh
   helm install short . -f values-local.yaml
   helm status short
   # open http://localhost:8080/ (Host header may need a hosts-file entry — I'll tell you)
   ```
4. 👤 **Prove it's repeatable:** `helm upgrade short . -f values-local.yaml --set replicas=3`, watch pods scale, then `helm rollback short 1`.

**What you learn:** install/upgrade/rollback, release lifecycle, subchart dependencies, values files. **This is the milestone** — the app runs anywhere with no AWS.

---

## Phase 4 — `values-aws.yaml`: portability on paper (15 min, $0) — 🤖

**Goal:** show the *same chart* deploys to EKS, without spending yet.

1. 🤖 I write **`values-aws.yaml`:** ECR image repo, `ingress.className: nginx`, `tls.enabled: true` (cert-manager issuer), **DB mode = external** (RDS endpoint via Secret, no subchart).
2. 🤖 I produce a side-by-side: `helm template . -f values-local.yaml` vs `-f values-aws.yaml` so you can *see* that only the env-specific lines differ.
3. 👤 Be ready to say in an interview: *"Same chart, two values files — laptop and EKS. The chart doesn't know or care which cloud it's on."*

**What you learn:** the actual definition of portability, and how to talk about it.

---

## Phase 5 — Make the chart "production-grade" (30 min, $0) — 🤖/👤

**Goal:** the polish that separates a toy chart from one a team would accept.

1. 🤖 `NOTES.txt` (post-install "here's your URL / next steps" output).
2. 🤖 `values.schema.json` so bad values fail fast with a clear error.
3. 🤖 Sensible `values.yaml` defaults + comments on every field.
4. 👤 **Version + lint discipline:** bump `version:` in `Chart.yaml` per change (SemVer), run `helm lint .` until clean.
5. 🤖 A `README.md` for the chart: what it is, `helm install` quickstart, the values table.

**What you learn:** chart UX, validation, SemVer, lint — the "I've seen real charts" signals.

---

## Phase 6 — GitHub Actions CI (40 min, $0) — 🤖/👤

**Goal:** fill the GitHub Actions gap; every push gets linted and **actually installed into a throwaway cluster** to prove it works.

1. 👤 **Create a GitHub repo** `portable-helm-chart`, push the folder. *Why you: your account.*
2. 🤖 I write `.github/workflows/ci.yml`:
   - on PR: `helm lint` + `helm template` + **`helm/chart-testing` (`ct lint`/`ct install`) inside a `helm/kind-action` cluster** — a real install test on free runners.
   - this is the GitHub-Actions equivalent of your GitLab lint/test stages, but k8s-aware.
3. 👤 Open a PR, watch the checks go green. Break a value on purpose, watch CI catch it.

**What you learn:** GitHub Actions syntax (vs GitLab CI you already know), and CI that tests a *deployment*, not just code.

---

## Phase 7 — Publish the chart + demo (30 min, $0) — 👤/🤖

**Goal:** a public chart anyone can `helm repo add` — a clickable portfolio artifact.

1. 🤖 **Pick a path (I'll wire whichever):**
   - **gh-pages via `helm/chart-releaser-action`** (recommended for visibility): on tag, it packages the chart, makes a GitHub Release, and updates `index.yaml` on the `gh-pages` branch. Recruiters then:
     ```sh
     helm repo add dorarbeli https://dorarbeli.github.io/portable-helm-chart
     helm install short dorarbeli/url-shortener
     ```
   - **OCI via GHCR** (modern default): `helm push url-shortener-x.y.z.tgz oci://ghcr.io/dorarbeli`. No `index.yaml`; works private too.
2. 👤 Tag a release (`git tag v0.1.0 && git push --tags`), confirm the chart installs from the public repo on a fresh `k3d cluster create`.
3. 👤 **Optional free live URL:** run **Cloudflare Tunnel** to expose your local Traefik — a real `https://…` link for your CV, no domain or LB bill.
4. 🤖 I'll draft a short LinkedIn post (Hebrew, like your others) showing the `helm repo add` one-liner.

**What you learn:** chart distribution, releases, OCI registries — and you get a thing people can actually run.

---

## Phase 8 — OPTIONAL cloud proof on EKS (later, ~$5/day, tear down same day) — 👤

**Goal:** screenshot the *same chart* live on AWS, then stop paying.

1. 👤 Reuse `url-shortener/terraform` to bring up EKS + RDS (you've done this).
2. 👤 `helm install short . -f values-aws.yaml`, get the real HTTPS URL, screenshot it next to the local one.
3. 👤 **`terraform destroy` the same day.** Don't leave it running.

**What you learn:** nothing new technically — it's the *evidence* that the portability claim is real.

---

## Phase 9 — Teardown ($0)

- 👤 `helm uninstall short` and `k3d cluster delete devops` when you're done for the day (local costs nothing, but keep it tidy).
- 👤 Confirm AWS is destroyed if you did Phase 8.

---

## What this proves on your CV / in interviews
- **Helm (real):** authored a parameterized chart with subchart deps, helpers, schema validation, NOTES, SemVer — not just `helm install` of someone else's chart.
- **Portability:** one chart, multiple environments via values files; can explain the local↔cloud split fluently.
- **GitHub Actions:** a CI pipeline that lint-and-*installs* into an ephemeral kind cluster — fills your GitLab-only gap.
- **Distribution:** a published, `helm repo add`-able chart (gh-pages or OCI/GHCR) — a clickable artifact.
- **Cost-awareness:** "I built and proved it for $0 locally; I only touch cloud to demo, then tear it down" — exactly the instinct hiring managers want in a junior.

## Interview one-liner
*"I took an app that was hand-wired to AWS and packaged it into a portable Helm chart. The same chart runs on my laptop via k3d for free and on EKS unchanged — only the values file differs. GitHub Actions lints it and installs it into a throwaway kind cluster on every PR, and it's published as a Helm repo you can add and install in two commands."*
