# Demo app auth sync (Traefik / File Provider)

Component: PVC + one CronJob + script + RBAC. One job run produces both `users-stage.htpasswd` and `users-prod.htpasswd` on the PVC (and reference ConfigMaps). Traefik File Provider uses them for `basicAuth` (no 1MB Secret limit). One Traefik instance, separate users per environment via middlewares `my-basic-auth-stage@file` and `my-basic-auth-prod@file`.

Used by **traefik** app (namespace `traefik`): `kustomizations/traefik-auth` includes this component; deployed with the Traefik Helm chart via `argocd-apps/infrastructure/traefik.yaml`.

- **PVC `traefik-auth-htpasswd-pvc`**: The CronJob writes both files here. Traefik mounts it at `/etc/traefik/auth`.
  - `users-stage.htpasswd` → middleware `my-basic-auth-stage@file` (stage Ingresses)
  - `users-prod.htpasswd` → middleware `my-basic-auth-prod@file` (prod Ingresses)
- **ConfigMaps** in `traefik`: `demo-app-generated-auth-stage`, `demo-app-generated-auth-prod` (reference credentials).

## Bootstrap (traefik namespace)

One job run creates both stage and prod files + ConfigMaps:

```bash
kubectl create job -n traefik auth-sync-once --from=cronjob/demo-app-auth-sync-traefik
kubectl logs job/auth-sync-once -n traefik -f
```

## Retrieve credentials

```powershell
# Stage users
(kubectl -n traefik get configmap demo-app-generated-auth-stage -o jsonpath='{.data.credentials}') -split "`n" | ForEach-Object {
  $u, $b64 = $_ -split ':', 2
  if ($b64) { $pass = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($b64.Trim())); "$u : $pass" }
}
# Prod users (same pattern with demo-app-generated-auth-prod)
```
