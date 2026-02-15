# Demo app auth sync (Traefik / stage)

Same as `demo-app-auth-sync` (nginx) but creates the Secret with key **`users`** for Traefik’s BasicAuth Middleware. Used by the **stage** overlay with Traefik ingress.

- **Secret `bw-basic-auth`** (key `users`): htpasswd file (APR1) for Traefik Middleware `basicAuth.secret`.
- **ConfigMap `demo-app-generated-auth`**: reference copy (same as nginx component).

## Bootstrap (stage, namespace `monitoring-stage`)

```bash
kubectl create job --from=cronjob/demo-app-auth-sync-traefik demo-app-auth-sync-traefik-bootstrap -n monitoring-stage
kubectl logs job/demo-app-auth-sync-traefik-bootstrap -n monitoring-stage -f
```

## Retrieve credentials

Same as nginx component; use namespace **monitoring-stage**:

```powershell
(kubectl -n monitoring-stage get configmap demo-app-generated-auth -o jsonpath='{.data.credentials}') -split "`n" | ForEach-Object {
  $u, $b64 = $_ -split ':', 2
  if ($b64) { $pass = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($b64.Trim())); "$u : $pass" }
}
```
