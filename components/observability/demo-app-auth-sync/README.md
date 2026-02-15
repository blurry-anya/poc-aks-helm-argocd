# Demo app auth sync (prod / local)

Generates ~10 random basic-auth users and writes them to:

- **Secret `bw-basic-auth`** (key `auth`): htpasswd file (APR1 format) used by the **external** ingress (nginx basic auth).
- **ConfigMap `demo-app-generated-auth`**: reference copy with `htpasswd` and `credentials` (user:base64pass) for looking up plain passwords.

No external secret provider; credentials are generated in-cluster.

---

## How passwords are generated

- **Usernames:** `demouser01` … `demouser10`.
- **Passwords:** 16 random alphanumeric characters from `openssl rand -base64 12 | tr -dc 'a-zA-Z0-9' | head -c 16`.
- **Secret (ingress):** each line is `user:hash` with **APR1** hashes from `htpasswd -nb` (nginx-compatible). Stored in Secret `bw-basic-auth` under key `auth`.
- **ConfigMap (reference):** same passwords stored as `user:base64(password)` in ConfigMap `demo-app-generated-auth` under key `credentials` (and the htpasswd file under key `htpasswd`).

---

## Retrieve all usernames and passwords

**PowerShell** (decode base64 and print `username : password`):

```powershell
(kubectl -n monitoring get configmap demo-app-generated-auth -o jsonpath='{.data.credentials}') -split "`n" | ForEach-Object {
  $u, $b64 = $_ -split ':', 2
  if ($b64) {
    $pass = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($b64.Trim()))
    "$u : $pass"
  }
}
```

**Bash** (Linux/macOS):

```bash
kubectl -n monitoring get configmap demo-app-generated-auth -o jsonpath='{.data.credentials}' | while IFS= read -r line; do
  user="${line%%:*}"
  b64="${line#*:}"
  [ -n "$b64" ] && printf '%s : %s\n' "$user" "$(echo "$b64" | base64 -d)"
done
```

**Raw credentials only** (base64-encoded passwords, one line per user):

```bash
kubectl -n monitoring get configmap demo-app-generated-auth -o jsonpath='{.data.credentials}'
```

---

## Bootstrap (first run)

The CronJob runs daily at midnight. To create the Secret/ConfigMap immediately after deploy:

```bash
kubectl create job --from=cronjob/demo-app-auth-sync demo-app-auth-sync-bootstrap -n monitoring
kubectl logs job/demo-app-auth-sync-bootstrap -n monitoring -f
```

To regenerate credentials (e.g. after changing the script):

```bash
kubectl create job --from=cronjob/demo-app-auth-sync demo-app-auth-sync-regenerate -n monitoring
kubectl logs job/demo-app-auth-sync-regenerate -n monitoring -f
```

Then use the commands above to retrieve the new passwords.
