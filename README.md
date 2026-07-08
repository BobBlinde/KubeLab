# KubeLab

## Setup

### Cloudflare API token for cert-manager

Both the `letsencrypt-staging` and `letsencrypt-prod` `ClusterIssuer`s (see
`manifests/cert-manager-issuers/cluster-issuer.yaml`) use the same DNS-01
secret, so it only needs to be created once. It must live in the
`cert-manager` namespace, since that's cert-manager's default cluster
resource namespace.

1. Create a Cloudflare API token scoped to `Zone:DNS:Edit` for the zone(s)
   you'll be issuing certs for (not the Global API Key).
2. Create the secret on the cluster:

   ```bash
   kubectl create secret generic cloudflare-api-token-secret \
     --namespace cert-manager \
     --from-literal=api-token=YOUR_CLOUDFLARE_API_TOKEN
   ```

This secret is created manually via `kubectl` and is intentionally not
stored in git — ArgoCD only manages resources defined in its source paths,
so it won't touch or prune this secret.

Also update the placeholder email in
`manifests/cert-manager-issuers/cluster-issuer.yaml` (`REPLACE_WITH_YOUR_EMAIL@example.com`)
before applying — consider using a non-personal/alias address rather than
committing your real one.

### ArgoCD Ingress (argocd.blinde.net)

ArgoCD itself isn't managed by GitOps in this repo (it's the bootstrap
layer), so exposing its UI via `apps/argocd-ingress.yaml` needs one manual
step outside git: ArgoCD serves TLS on its own by default, but the Ingress
(see `manifests/argocd-ingress/ingress.yaml`) terminates TLS at Traefik using
a cert-manager-issued cert, so `argocd-server` needs to be switched to plain
HTTP internally.

1. Set `server.insecure: "true"` on the `argocd-cmd-params-cm` ConfigMap:

   ```bash
   kubectl -n argocd patch configmap argocd-cmd-params-cm \
     --type merge -p '{"data":{"server.insecure":"true"}}'
   ```

2. Restart `argocd-server` to pick up the change:

   ```bash
   kubectl -n argocd rollout restart deployment/argocd-server
   ```

This needs to be redone if `argocd-cmd-params-cm` is ever reset (e.g. a
fresh ArgoCD install). The Ingress currently points at `letsencrypt-staging`
— switch it to `letsencrypt-prod` in
`manifests/argocd-ingress/ingress.yaml` once it's confirmed working.