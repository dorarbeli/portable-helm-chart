# STATE — portable-helm-chart (live session handoff)

**Last updated:** 2026-06-14. **Read `PROMPT.md` + `GUIDE.md` in this folder first** — source of truth for who/why/rules and the phase-by-phase plan. This file = what's done and where to continue.

## Snapshot

- **Phase:** 3 complete. **Next: Phase 4** (write `values-aws.yaml` side-by-side diff — already done) → **Phase 5** (production-grade polish).
- **Machine:** Windows, PowerShell. VS Code + integrated terminal.
- **Paths:** project = `C:\Users\dorar\Desktop\Projects\devops-projects\portable-helm-chart\`; chart = `.\url-shortener\`; source app = `..\url-shortener\`.

## Done

### Phase 0 — cluster
- Tools installed via Chocolatey: Docker 29.5.2, Helm v4.1.4, k3d v5.9.0, kubectl v1.34.1.
- Default k3s image (v1.35.5) crashes with Docker 29. **Fix: pin `--image rancher/k3s:v1.31.11-k3s1`**.
- Cluster create command (run this every session start if cluster is stopped):
  ```powershell
  k3d cluster start devops
  # if that fails or cluster is gone:
  k3d cluster create devops -p "8080:80@loadbalancer" --agents 1 --image rancher/k3s:v1.31.11-k3s1 --timeout 180s
  ```
- **Known quirk:** Docker Desktop restart stops k3d containers → run `k3d cluster start devops` at the start of each session, then `kubectl get nodes` to confirm both Ready.

### Phase 1 — app raw on k3d
- Built `url-shortener:dev` image, imported into k3d, ran throwaway MySQL pod, applied temp manifests, confirmed `curl /api/shorten` returns a short code.
- **Lesson learned:** app pod races MySQL at startup → `ensure_schema()` fails silently → "Table doesn't exist". Fixed in Phase 3 with an initContainer.
- Temp objects cleaned up. `temp/shortener.yaml` still exists in project root (harmless, can delete).

### Phase 2 — chart scaffold
Chart at `url-shortener/` with these files:
- `Chart.yaml` — name: url-shortener, version: 0.1.0
- `values.yaml` — clean defaults (image: url-shortener:dev, pullPolicy: Never, mysql.internal: true)
- `values-local.yaml` — enables Traefik ingress, no TLS
- `values-aws.yaml` — ECR image, RDS endpoint, nginx ingress, TLS (for Phase 8)
- `templates/_helpers.tpl` — fullname / labels helpers
- `templates/deployment.yaml` — includes initContainer that waits for MySQL before app starts
- `templates/service.yaml` — ClusterIP 80→5000
- `templates/ingress.yaml` — guarded by `ingress.enabled`
- `templates/secret.yaml` — DB credentials as k8s Secret
- `templates/mysql.yaml` — inline MySQL StatefulSet + Service, guarded by `mysql.internal`
- `templates/NOTES.txt` — post-install instructions
- `helm lint .` passed clean.

### Phase 3 — helm install working
```powershell
cd url-shortener
helm install short . -f values-local.yaml
```
- MySQL StatefulSet comes up first; initContainer on app pod waits for port 3306; app starts cleanly.
- `curl.exe -X POST http://localhost:5000/api/shorten -d "url=https://anthropic.com"` → `{"short_code":"oDmPgu","short_url":"http://localhost:5000/oDmPgu"}` ✓
- **Important:** after every Docker Desktop restart, re-import the image before helm install/upgrade:
  ```powershell
  k3d image import url-shortener:dev -c devops
  ```

## Next steps — Phase 4 → 5

**Phase 4 (15 min, 🤖 heavy):** `values-aws.yaml` already written. Next step is the side-by-side diff:
```powershell
helm template short . -f values-local.yaml > /tmp/local.yaml
helm template short . -f values-aws.yaml  > /tmp/aws.yaml
# diff them — only env-specific lines change
```
This is the visual proof that the chart is portable.

**Phase 5 (polish):** Add `values.schema.json` (bad values fail with a clear error), finalize `NOTES.txt`, bump chart version on each change, run `helm lint .` until clean. Then Phase 6 = GitHub Actions CI.

## Working style (carry over)

🤖 assistant does typing/file edits; 👤 Dor runs anything needing his machine, Docker, or accounts. Explain the **why**. Be **concise**. **Never introduce Ansible.** **Never ask Dor for cloud creds.** Local k3d = **$0**; AWS only in optional Phase 8, torn down same day.
