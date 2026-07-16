# CI/CD Runbook
## 1. Repository and pipeline layout
Each repo has its own Jenkinsfile at root. Pipelines are independent — a frontend change only triggers the frontend pipeline.

| Repo | Jenkinsfile | What it builds |
|------|------------|----------------|
| `frontend` | `Jenkinsfile` | React.js Docker image |
| `backend` | `Jenkinsfile` | ASP.NET Core API + Worker Docker image |
<!-- | `infra` | `Jenkinsfile` | Applies K8s manifests (no image build) | -->

## 2. Pipeline stages
 
### 2.1 Frontend pipeline (`frontend/Jenkinsfile`)
 
```
Git checkout → Install deps → Lint → Build → Docker build → Docker push → Deploy to servers
```
 
| Stage | Command | Failure behaviour |
|-------|---------|-------------------|
| Install deps | `npm ci` | Fail pipeline |
| Lint | `npm run lint` | Fail pipeline |
| Build | `npm run build` | Fail pipeline |
| Docker build | `docker build -t registry.yourdomain.com/frontend:$GIT_SHA .` | Fail pipeline |
| Docker push | `docker push ...` | Fail pipeline |
| Deploy | `docker compose down && docker compose up --build` | Fail pipeline, trigger alert |
<!-- | Deploy | `kubectl set image deployment/frontend-deployment frontend=...` | Fail pipeline, trigger alert | -->

## 2.2 Backend pipeline (`backend/Jenkinsfile`)
 
```
Git checkout → Restore → Build → Test → Docker build → Docker push → Deploy to K8s
```

| Stage | Command | Failure behaviour |
|-------|---------|-------------------|
| Restore | `dotnet restore` | Fail pipeline |
| Build | `dotnet build --no-restore -c Release` | Fail pipeline |
| Test | `dotnet test --no-build` | Fail pipeline, PR blocked |
| Docker build | `docker build -t registry.yourdomain.com/backend:$GIT_SHA .` | Fail pipeline |
| Docker push | `docker push ...` | Fail pipeline |
| Deploy API | `kubectl set image deployment/backend-deployment ...` | Fail pipeline, trigger alert |
| Deploy Worker | `kubectl set image deployment/worker-deployment ...` | Fail pipeline, trigger alert |

---

## 3. Branch and environment mapping
 
| Branch | Environment | Auto-deploy? |
|--------|------------|--------------|
| `main` | Production | Yes but require deploy confirm, on merge|
| `staging` | Staging | Yes, on merge |
| `develop` | Dev | Yes, on merge |
| `feature/*` | — | CI only (no deploy) |
 
Feature branches run all stages up to Docker push but skip the deploy stage. This lets you confirm the image builds before merging.
 
---

## 4. Image tagging strategy
Images are tagged with the Git commit SHA:

```
registry.yourdomain.com/frontend:a3f9b2c
registry.yourdomain.com/backend:a3f9b2c
```

Additionally, the branch name is applied as a floating tag:
 
```
registry.yourdomain.com/frontend:main
registry.yourdomain.com/frontend:staging
```

K8s deployments always reference the SHA tag, never a floating tag. The SHA is recorded in the Jenkins build log for every deploy.

---

## 5. Kubernetes namespace layout
 
```
cluster
├── namespace: production
│   ├── frontend-deployment     (image: frontend:$SHA, replicas: 2)
│   ├── backend-deployment      (image: backend:$SHA, replicas: 2)
│   ├── worker-deployment       (image: backend:$SHA, replicas: 1)
│   └── ingress                 (Nginx → frontend + backend services)
├── namespace: staging
│   ├── frontend-deployment     (replicas: 1)
│   ├── backend-deployment      (replicas: 1)
│   └── worker-deployment       (replicas: 1)
├── namespace: dev
│   ├── frontend-deployment     (replicas: 1)
│   ├── backend-deployment      (replicas: 1)
│   └── worker-deployment       (replicas: 1)
└── namespace: infra            (shared across all envs)
    ├── postgresql-statefulset
    ├── redis-statefulset
    └── minio-statefulset
```

Each environment has its own set of `ConfigMaps` and `Secrets`. Never share secrets between namespaces.
 
---
 
## 6. Triggering a deploy manually
 
Sometimes you need to deploy without a code change (e.g. applying a config change or recovering after an incident).
 
**Option A — Re-run the Jenkins build**
 
Go to Jenkins → the relevant pipeline → click "Build Now". The pipeline will re-tag and re-apply the current `main` image.
 
**Option B — kubectl directly**
 
```bash
# Force a rolling restart (picks up new ConfigMap values)
kubectl rollout restart deployment/backend-deployment -n production
 
# Deploy a specific image tag
kubectl set image deployment/backend-deployment \
  backend=registry.yourdomain.com/backend:a3f9b2c \
  -n production
 
# Watch the rollout
kubectl rollout status deployment/backend-deployment -n production
```
 
---
 
## 7. Rollback steps
 
K8s keeps the previous ReplicaSet, so rollback is fast.
 
```bash
# Roll back to the previous revision
kubectl rollout undo deployment/backend-deployment -n production
 
# Roll back to a specific revision
kubectl rollout history deployment/backend-deployment -n production
kubectl rollout undo deployment/backend-deployment --to-revision=3 -n production
 
# Confirm rollback completed
kubectl rollout status deployment/backend-deployment -n production
```
 
After rolling back, identify the broken image SHA from the Jenkins build log and create a hotfix branch. Do not leave the deployment running on a rolled-back revision longer than necessary.
 
---
 
## 8. Health checks
 
All deployments have `readinessProbe` and `livenessProbe` configured in the K8s manifests (`infra/k8s/`).
 
- **Backend readiness:** `GET /health/ready` — returns 200 when the DB connection and Redis connection are alive.
- **Backend liveness:** `GET /health/live` — returns 200 always; if the process is deadlocked this fails.
- **Frontend readiness:** `GET /api/health` — Next.js built-in.
Check health manually:
 
```bash
kubectl get pods -n production
kubectl describe pod <pod-name> -n production
kubectl logs <pod-name> -n production --tail=100
```
 
---
 
## 9. Secrets management
 
Secrets are stored as Kubernetes Secrets and mounted as environment variables. They are not committed to any repo.
 
To update a secret:
 
```bash
kubectl create secret generic backend-secrets \
  --from-literal=Jwt__SecretKey=new-value \
  --namespace production \
  --dry-run=client -o yaml | kubectl apply -f -
 
# Restart the deployment to pick up the new secret
kubectl rollout restart deployment/backend-deployment -n production
```
 
Keep a copy of all secret values in a shared password manager (e.g. Bitwarden) — not in any file, not in Slack.
 
---
 
## 10. Adding a new environment variable
 
1. Add to `.env.example` in the relevant repo.
2. Add to the K8s `ConfigMap` or `Secret` in `infra/k8s/<env>/`.
3. Apply the ConfigMap: `kubectl apply -f infra/k8s/production/configmap.yaml`.
4. Restart the relevant deployment.
5. Update `local-dev-setup.md` with the new variable.