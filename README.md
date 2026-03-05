# Platform Technical Assessment

## Part 1: Kubernetes Troubleshooting

### Issues Found and Fixes Applied

#### Deployment (`manifests/deployment.yaml`)

| # | Issue | Fix |
|---|-------|-----|
| 1 | `spec.selector.matchLabels.app` uses `web-app` but pod template label key is `application` — selector won't match pods | Changed pod template label to `app: web-app` to match the selector |
| 2 | `prometheus.io/scrape: true` — annotation value must be a string | Changed to `"true"` |
| 3 | Container port set to `443/UDP` but nginx serves HTTP | Changed to `containerPort: 80`, `protocol: TCP` |
| 4 | `requests.memory: 64Mi` but `limits.memory: 32Mi` — limit cannot be less than request | Increased `limits.memory` to `128Mi` |
| 5 | `DATABASE_PASSWORD` hardcoded as a plain env value | Moved to a `secretKeyRef` pointing to a new `Secret` manifest |
| 6 | `volumeMount` references `data` but only `cache` is defined in `volumes` | Changed volumeMount name to `cache` |

#### Secret (`manifests/secret.yaml`)

- Created a new `Secret` resource for `DATABASE_PASSWORD`
- Uses `${DATABASE_PASSWORD}` placeholder to be substituted by CI/CD at deploy time

#### Service (`manifests/service.yaml`)

| # | Issue | Fix |
|---|-------|-----|
| 1 | Service namespace is `staging` but all other resources are in `production` — ingress can't route to it | Changed namespace to `production` |
| 2 | `spec.selector.app` doesn't match deployment pod labels | Changed selector to `app: web-app` |

#### Ingress (`manifests/ingress.yaml`)

| # | Issue | Fix |
|---|-------|-----|
| 1 | Missing required `pathType` field | Added `pathType: Prefix` |
| 2 | Backend points to non-existent service `webapp-svc` | Changed to `web-app-service` |

#### NetworkPolicy (`manifests/networkpolicy.yaml`)

| # | Issue | Fix |
|---|-------|-----|
| 1 | Ingress rule allows port `443` but app runs on HTTP port `80` | Changed to port `80` |

#### PodDisruptionBudget (`manifests/pdb.yaml`)

| # | Issue | Fix |
|---|-------|-----|
| 1 | Both `minAvailable: 3` and `maxUnavailable: 1` are set — these contradict each other (3 min available with 3 replicas means 0 can go down, but maxUnavailable says 1 can) | Removed `minAvailable`, kept only `maxUnavailable: 1` |

#### HorizontalPodAutoscaler (`manifests/hpa.yaml`)

| # | Issue | Fix |
|---|-------|-----|
| 1 | `scaleTargetRef.name: webapp` but Deployment is named `web-app` | Changed to `web-app` |
| 2 | `minReplicas` and `maxReplicas` both set to `3` — HPA can't scale | Changed `maxReplicas` to `10` |

### Project Structure

```
Manifests/
├── deployment.yaml      # Nginx deployment with 3 replicas
├── secret.yaml          # Database credentials (CI/CD substituted)
├── service.yaml         # ClusterIP service on port 80
├── ingress.yaml         # Ingress routing for webapp.example.com
├── networkpolicy.yaml   # Ingress/egress traffic rules
├── pdb.yaml             # Pod disruption budget
└── hpa.yaml             # Horizontal pod autoscaler
└── configmap.yaml       # ConfigMap
```

### How to Apply

```bash
kubectl apply -f manifests/
```

---

## Part 2: CI/CD Pipeline

See [azure-pipelines.yml](azure-pipelines.yml) for the build and deployment pipeline targeting AKS.
