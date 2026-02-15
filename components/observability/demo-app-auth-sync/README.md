# Demo app auth sync (prod / local)

Generates ~10 random basic-auth users and writes them to:

- **Secret `bw-basic-auth`** (key `auth-file`): htpasswd file used by the **external** ingress (nginx basic auth).
- **ConfigMap `demo-app-generated-auth`**: reference copy with `htpasswd` and `credentials` (user:base64pass) for debugging.

No external secret provider; credentials are generated in-cluster with `openssl rand` and `openssl passwd -1`.

## Bootstrap (first run)

The CronJob runs daily at midnight. To create the Secret/ConfigMap immediately after deploy:

```bash
kubectl create job --from=cronjob/demo-app-auth-sync demo-app-auth-sync-bootstrap -n monitoring
kubectl logs job/demo-app-auth-sync-bootstrap -n monitoring -f
```

## Usernames

Generated users are `demouser01` … `demouser10`. To see credentials (base64-encoded passwords):

```bash
kubectl get configmap demo-app-generated-auth -n monitoring -o jsonpath='{.data.credentials}'
```
