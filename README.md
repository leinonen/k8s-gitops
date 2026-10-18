# k8s-gitops

Everything running on the single-node k3s cluster from
[leinonen/infrastructure `stacks/k8s`](https://github.com/leinonen/infrastructure/tree/main/stacks/k8s).
The box's cloud-init installs ArgoCD and the `root` Application, which syncs `bootstrap/`.

| Path | What |
|---|---|
| `bootstrap/` | One Application per platform component (ArgoCD itself, cert-manager, external-dns, external-secrets), `platform`, and the `apps` ApplicationSet |
| `platform/` | Cluster-wide config: Let's Encrypt ClusterIssuers, `ssm` ClusterSecretStore |
| `apps/<name>/` | One app per directory, deployed to namespace `<name>` |

## Adding an app

1. Create `apps/<name>/` with plain manifests (or a `kustomization.yaml`).
2. Ingress: `ingressClassName: traefik`, annotation `cert-manager.io/cluster-issuer: letsencrypt-prod`
   (or `letsencrypt-staging` while testing), and a `tls` block. See `apps/hello`.
3. Push to `main`. ArgoCD picks it up within ~3 minutes.

Apps built with [leinonen/app-factory](https://github.com/leinonen/app-factory) are written
here by their own CI from the app repo's `deploy/`: edit them there, not here.

## Hostnames

- `<anything>.k8s.leinonen.ninja` already resolves to the box (wildcard in the dns stack). Just use it.
- Any other host in leinonen.ninja or leinonen.se: put it in the Ingress `rules`; external-dns
  creates the A record (plus a TXT ownership record). It never deletes records: clean up in Route 53
  by hand. Never use a hostname that also exists in `stacks/dns/records_data.tf`.

## Secrets

Store them in SSM under `/k8s/<app>/...` (the instance role can read nothing else), then:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db
spec:
  secretStoreRef: { kind: ClusterSecretStore, name: ssm }
  target: { name: db }
  data:
    - secretKey: password
      remoteRef: { key: /k8s/myapp/db-password }
```

## Databases

Apps can use `shared-postgres` (infrastructure `stacks/postgres`; the `k8s` security group is allowed).
Create a role and database per app (infrastructure `docs/db-migration.md` step 1a), put the
password in SSM under `/k8s/<app>/`, and build `DATABASE_URL` in an ExternalSecret template.
Connect with `sslmode=verify-full&sslrootcert=/etc/rds/ca.pem`: copy `apps/go-hello/rds-ca.yaml`
(RDS eu-north-1 root CAs) into the app and mount it at `/etc/rds`. The Secret only reaches the pod
at startup, so after changing it run `kubectl rollout restart`. See `apps/go-hello`.

## Access

kubectl goes through an SSM tunnel; see the infrastructure README (Kubernetes section).
ArgoCD UI: https://argocd.k8s.leinonen.ninja, user `admin`, password from
`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`.
