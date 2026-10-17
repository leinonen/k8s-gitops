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

## Access

kubectl goes through an SSM tunnel; see the infrastructure README (Kubernetes section).
ArgoCD UI: https://argocd.k8s.leinonen.ninja, user `admin`, password from
`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`.
