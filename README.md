# PoC Project for learning deployment to k8s cluster (AKS) using Helm and ArgoCD

This repo contains ArgoCD applications and Kustomizations. ArgoCD is used to deploy and manage itself (projects, ingress) plus other infrastructure (e.g. ingress-nginx) from this Git repo.

---

## Initial install: bootstrap ArgoCD on a new cluster

These steps are done **once** per cluster (e.g. Rancher Desktop locally or AKS).

### 1. Install ArgoCD

**Option A – official manifest**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Option B – Helm**

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd
```

Wait until ArgoCD is ready:

```bash
kubectl -n argocd wait --for=condition=Available deployment --all --timeout=300s
```

### 2. Get the initial admin password

```bash
# PowerShell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

# Linux/macOS
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Login user: `admin`.

### 3. Create the AppProjects (required for the app of apps)

The applications in this repo use the `infrastructure` project. Create the projects once:

```bash
kubectl apply -f kustomizations/argocd/base/projects.yaml
```

### 4. Point ArgoCD at this repo (app of apps)

From the repo root:

```bash
kubectl apply -f argocd-apps/infrastructure-aoa.yaml
```

This creates the **infrastructure-aoa** Application, which syncs `argocd-apps/infrastructure` and in turn creates the **argocd** and **ingress-nginx** apps (and any others in that folder).

### 5. (Optional) Connect a private GitHub repo

If the repo is private, add credentials so ArgoCD can clone it:

- **UI:** Settings → Repositories → Connect Repo (HTTPS + GitHub Personal Access Token).
- **CLI/Secret:**
  ```bash
  kubectl create secret generic repo-github --namespace argocd \
    --from-literal=url=https://github.com/YOUR_ORG/poc-aks-helm-argocd.git \
    --from-literal=password=YOUR_GITHUB_PAT
  kubectl label secret repo-github -n argocd argocd.argoproj.io/secret-type=repository
  ```

---

## Patching for local use (e.g. Rancher Desktop)

On a local cluster there is no cloud load balancer, and ArgoCD defaults to HTTPS. These one-time patches make the UI reachable via the ingress (NodePort + HTTP).

### 1. Use NodePort for ingress-nginx (instead of LoadBalancer)

So the ingress controller gets a stable port instead of staying `<pending>`:

```bash
kubectl -n ingress-nginx patch svc ingress-nginx-controller --type merge -p '{\"spec\":{\"type\":\"NodePort\"}}'
```

(Alternatively, the repo is already set to `type: NodePort` in `argocd-apps/infrastructure/ingress-nginx-app.yaml`; after a sync the service will match.)

### 2. Run ArgoCD server in insecure (HTTP) mode

So the ingress can proxy HTTP to the server on port 80:

```bash
kubectl -n argocd patch configmap argocd-cmd-params-cm --type merge -p '{\"data\":{\"server.insecure\":\"true\"}}'
kubectl -n argocd rollout restart deployment argocd-server
```

### 3. Access the UI via ingress

- Add to hosts (e.g. `C:\Windows\System32\drivers\etc\hosts`):  
  `127.0.0.1 argo.sixthnode.xyz`
- Get the HTTP NodePort:  
  `kubectl -n ingress-nginx get svc ingress-nginx-controller`
- Open in browser: **http://argo.sixthnode.xyz:&lt;NodePort&gt;** (e.g. `http://argo.sixthnode.xyz:31798`).

Or use port-forward instead of ingress:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Then open https://localhost:8080 (accept TLS warning), login admin / &lt;password from step 2 above&gt;
```

---

## For AKS (production)

- Use an overlay or separate values for **ingress-nginx** with `type: LoadBalancer` and `loadBalancerIP` (e.g. your AKS static IP).
- Do **not** set `server.insecure: "true"` on AKS; use TLS at the ingress and keep ArgoCD on HTTPS.
