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

### Immich

The `immich` Application (`apps/immich.yaml`) is a multi-source ArgoCD app:
the official Helm chart (`immich-app/immich-charts`) provides the
server/machine-learning/valkey deployments and the Ingress, while
`manifests/immich` supplies the pieces the chart doesn't manage — Postgres
(the chart dropped its Postgres subchart) and the photo/video library
storage.

The library still lives at `/data/immich` on the node, same as before, but
the chart requires an existing PVC rather than a raw hostPath, so
`manifests/immich/library-pv.yaml` + `library-pvc.yaml` wrap that hostPath
in a statically-provisioned PV/PVC pair (`storageClassName: manual`) that
the chart's `immich.persistence.library.existingClaim` points at. The PV
pins to node `kubrick` via node affinity — update that if the cluster
gains more nodes. Make sure `/data/immich` exists and has room before
syncing.

Immich's Postgres deployment and the chart-managed server container read
their DB credentials from an `immich-db` secret that isn't stored in git
(same reasoning as the Cloudflare token above). Create it before syncing
the `immich` Application:

```bash
kubectl create secret generic immich-db \
  --namespace immich \
  --from-literal=username=postgres \
  --from-literal=password=YOUR_STRONG_PASSWORD \
  --from-literal=database=immich
```

`username=postgres` matches Immich's own documented default, so a
`pg_dumpall`/`pg_restore` backup taken from a typical Immich instance
(including the official backup docs' example) restores here without
needing to pre-create a role or reassign object ownership.

The Immich app version is pinned via the root-level
`controllers.main.containers.main.image.tag` value in `apps/immich.yaml`
(shared by both the server and machine-learning components), independent
of the chart's `targetRevision` — the chart does not track every Immich
release, so bumping Immich almost never means bumping the chart too.

The chart source is the OCI registry (`oci://ghcr.io/immich-app/immich-charts/immich`),
not the old `https://immich-app.github.io/immich-charts` HTTP repo — the
maintainers retired that HTTP repo after chart `0.12.0` and publish only to
OCI now, so the HTTP repo's `index.yaml` is permanently stuck at `0.12.0`
and any chart `targetRevision` newer than that will fail to resolve
against it.

Immich v3 dropped `pgvecto.rs` support entirely (VectorChord is now the
only supported vector extension), but that only matters for instances
that hadn't already migrated to VectorChord on a pre-v1.133.0 install —
our Postgres image already ran VectorChord, so no database migration step
was needed there. The v3.0.3 upgrade did bump the recommended Postgres
image to `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`
(VectorChord `0.3.0` → `0.4.3`) in `manifests/immich/postgres-deployment.yaml`
— `immich-server` detects and applies the extension version bump itself on
startup, no manual `ALTER EXTENSION` needed, but take a fresh `pg_dumpall`
backup before syncing a Postgres image bump regardless.

### ArgoCD Ingress (argocd.blinde.net)

ArgoCD is exposed via TLS passthrough rather than edge termination, so
`argocd-server` keeps serving its own TLS (no `server.insecure` needed, and
`kubectl port-forward` + the `argocd` CLI keep working exactly as before):

- `manifests/argocd-ingress/certificate.yaml` — a cert-manager `Certificate`
  targeting the `argocd-server-tls` secret name, which `argocd-server`
  auto-detects and hot-reloads (no restart needed, even on renewal).
- `manifests/argocd-ingress/ingressroutetcp.yaml` — a Traefik
  `IngressRouteTCP` that passes TLS straight through to `argocd-server:443`
  based on SNI, instead of terminating it at the edge.

No manual steps needed here — both are managed by the `argocd-ingress`
Application like everything else. The cert currently uses
`letsencrypt-staging` — switch it to `letsencrypt-prod` in
`manifests/argocd-ingress/certificate.yaml` once it's confirmed working.