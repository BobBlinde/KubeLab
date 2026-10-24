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

#### Backups (weekly, to OneDrive)

`manifests/immich/backup-cronjob.yaml` runs [restic](https://restic.net)
every Sunday at 04:00 UTC and backs up the library volume (`/data/immich`
on the node) to a restic repo in OneDrive (folder `kubelab-backups/immich`), using
rclone as restic's OneDrive backend. The repo deliberately has no password
(`--insecure-no-password`), so anyone with access to the OneDrive account
can read the backup. The database
is covered by the same snapshot: Immich's built-in database backup
(Administration → Settings → Backup) writes a daily `pg_dump` to
`/data/immich/backups` at 02:00, so keep that enabled. `thumbs/` and
`encoded-video/` are excluded because Immich can regenerate them.
Retention is 8 weekly + 12 monthly snapshots.

One-time setup:

1. On a machine with a browser, run `rclone config`, add a remote named
   exactly `onedrive` (type `onedrive`) and log in. This writes
   `~/.config/rclone/rclone.conf`.
2. Create the secret:

   ```bash
   kubectl create secret generic rclone-onedrive \
     --namespace immich \
     --from-file=rclone.conf=$HOME/.config/rclone/rclone.conf
   ```

   The secret is named generically because other services' backup jobs
   can reuse the same OneDrive login. Secrets are namespaced, though, so
   create a copy in each namespace that gets a backup job. Give each
   service its own restic repo under `kubelab-backups/` (e.g.
   `rclone:onedrive:kubelab-backups/bookorbit`).

3. Start the first run instead of waiting for Sunday. This first upload
   is the slow one:

   ```bash
   kubectl -n immich create job --from=cronjob/immich-backup immich-backup-manual
   kubectl -n immich logs -f job/immich-backup-manual
   ```

On its first run the job copies `rclone.conf` from the secret onto the
`immich-backup-state` PVC, because rclone has to write refreshed OneDrive
tokens back to it. That means **updating the secret won't take effect** by
itself. If you re-authorize OneDrive, also delete the PVC copy (it gets
recreated from the secret on the next run):

```bash
kubectl -n immich run rm-rclone --rm -it --restart=Never --image=busybox \
  --overrides='{"spec":{"volumes":[{"name":"s","persistentVolumeClaim":{"claimName":"immich-backup-state"}}],"containers":[{"name":"c","image":"busybox","command":["rm","/s/rclone/rclone.conf"],"volumeMounts":[{"name":"s","mountPath":"/s"}]}]}}'
```

The OneDrive token stays valid as long as the job runs at least once
every 90 days, which the weekly schedule easily covers.

To restore, run restic anywhere rclone has the same remote configured,
e.g. `restic --insecure-no-password -r rclone:onedrive:kubelab-backups/immich snapshots`
and `restic --insecure-no-password -r rclone:onedrive:kubelab-backups/immich restore latest --target ./restore`.
To get the database back, load the newest `backups/immich-db-backup-*.sql.gz`
from the restored files (see Immich's backup and restore docs).

### Paperless-ngx (paperless.blinde.net)

`apps/paperless.yaml` deploys `manifests/paperless`: Paperless-ngx itself,
plus its own Postgres and Valkey (the task broker Paperless requires).
Data, media (the documents) and the consume folder are separate
`local-path` PVCs.

Paperless and its Postgres read their settings from a `paperless-secrets`
secret that isn't stored in git. Create it before syncing the app:

```bash
kubectl create secret generic paperless-secrets \
  --namespace paperless \
  --from-literal=postgres-user=paperless \
  --from-literal=postgres-password=YOUR_STRONG_PASSWORD \
  --from-literal=postgres-db=paperless \
  --from-literal=secret-key="$(python3 -c 'import secrets;print(secrets.token_urlsafe(64))')" \
  --from-literal=admin-user=bob \
  --from-literal=admin-password=YOUR_ADMIN_PASSWORD
```

`admin-user`/`admin-password` are only used to create the superuser on
first start. Changing them later does nothing; change the password in
Paperless instead. To read the initial password:

```bash
kubectl -n paperless get secret paperless-secrets -o jsonpath='{.data.admin-password}' | base64 -d
```

To import documents, copy them into the consume folder. Paperless picks
them up, runs OCR, and removes them from the folder once imported:

```bash
kubectl -n paperless cp ./some-file.pdf deploy/paperless:/usr/src/paperless/consume/
```

OCR language is English (`PAPERLESS_OCR_LANGUAGE`). For Dutch documents
set it to e.g. `nld+eng` and add `PAPERLESS_OCR_LANGUAGES=nld`, since the
image doesn't ship Dutch by default.

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