# Techbleat Banking — Argo CD GitOps (Kind lab)

This repository stores **application Kubernetes desired state** only. It does not install or manage PostgreSQL, Kafka, Redis, Keycloak, PVCs, the existing Istio Gateway or Ubuntu Nginx.

## Directory structure

- `apps/banking/lab/`: five Deployments, five Services, two frontend ConfigMaps, Istio VirtualService and Kustomize resource index.
- `argocd/banking-app.yaml`: manually synchronised Argo CD Application.

## Before first sync — REQUIRED

1. Back up manifests of the running system (example):
   ```bash
   kubectl get deployments,services,configmaps,virtualservices -n banking -o yaml > ~/banking-before-gitops.yaml
   ```
2. **Compare these starter manifests against your ACTUAL running cluster.** This bundle is reconstructed from the agreed conversation configuration and has not been validated against your live Kind cluster. In particular, verify transaction-service and finance-agent GHCR image refs, all image versions, probe paths, all env variable names, CORS proxy headers and the existing `frontend-route` VirtualService name. Replace starter values with the live working versions before any sync.
   ```bash
   kubectl get deployments -n banking -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image
   kubectl get virtualservices -A
   kubectl get deployment transaction-service -n banking -o yaml
   ```
3. Check existing Secrets in `banking` (names only):
   ```bash
   kubectl get secret -n banking banking-db banking-user-config banking-keycloak ghcr-auth finance-agent-openai
   ```
   `banking-db` must have the **working** `bankuser` password; earlier it was stale. Don't add Secret manifests containing credentials to this repo.
4. Validate Kustomize:
   ```bash
   kubectl kustomize apps/banking/lab >/tmp/techbleat-banking-rendered.yaml
   kubectl apply --dry-run=server -f /tmp/techbleat-banking-rendered.yaml
   ```
   Dry-run checks schema/admission but does not guarantee runtime correctness.
5. Verify there are no conflicting Istio VirtualServices routing the same host and that the existing `demo/nginx-gateway` permits `bank.techbleat-training.com`.
6. The frontend Nginx proxy is intentionally production-host-specific for the Transaction Service to preserve the observed working CORS headers. Review this if changing the public domain or testing through HTTP localhost.
7. Bank APIs discussed in the lab are **not uniformly protected by backend JWT checks**. Do not expose untrusted users or real customer data; restrict site access until enforcement is implemented.

## Create repository and register Argo CD

Create `https://github.com/techbleat/techbleat-gitops` (or edit repoURL in `argocd/banking-app.yaml`). Commit and push this directory to `main`.

```bash
kubectl apply -f argocd/banking-app.yaml
kubectl get applications -n argocd
```

In Argo CD, click **Refresh**, inspect **Diff** carefully and only then click **Sync** with **Prune unchecked**. There is intentionally no `automated` sync policy. Do **not** blindly enable prune/self-heal until initial adoption is complete.

Note: the Argo CD Application object itself is bootstrapped separately via `kubectl apply`. It is not part of the banking Kustomization to avoid installing it into the wrong namespace.

## Secret strategy

Current Secrets remain managed *outside Git* in Kubernetes:

- `banking/ghcr-auth` — GHCR read token
- `banking/banking-db` — PostgreSQL user password
- `banking/banking-user-config` — SQLAlchemy DATABASE_URL
- `banking/banking-keycloak` — confidential service account secret
- `banking/finance-agent-openai` — OpenAI API key
- `keycloak/keycloak-db` — Keycloak PostgreSQL password

Do not commit plaintext, base64-encoded, or decoded secrets. For a later stage, use Sealed Secrets (back up sealing keys) or External Secrets Operator backed by a secret manager. Keep the credentials provider separate from Argo CD source code.

## Updating frontend Nginx ConfigMap

The frontend mounts `default.conf` and `config.js` through `subPath`. After applying config changes, perform a rollout restart; mounted `subPath` files do not update automatically:

```bash
kubectl rollout restart deployment/frontend -n banking
kubectl rollout status deployment/frontend -n banking
```

For GitOps, plan a controlled restart mechanism (e.g. ConfigMap checksum annotation) rather than an ongoing manual workaround.

## Suggested next evolution

Pin all image versions to verified commit SHA tags (or digests), then have CI open a PR updating this repository after building images. Argo CD is a deployment reconciler, not a credential store.
