# NEI — the memory exhibit

The site for a neuroscience exhibit at Noite Europeia dos Investigadores,
Coimbra, on **Friday 25 September 2026**, 15:00–23:00. Two pages: a scroll-driven
explainer that runs itself on a screen at the station, and a shared board that
visitors change from their phones and that grows all evening.

Source: `github.com/rafaelcorreia/construction-by-augmentation` (the app repo's
`docs/deploy.md` is the other half of this document).

**The evening happens once.** Everything below is shaped by that: the database is
append-only, `prune` is off on the Argo app, and there is a half-hourly backup on
the day itself. A mistake here cannot be re-run next week.

## What runs

| Workload | Image | What it is |
| --- | --- | --- |
| `nei-web` (2 replicas) | `hydrodog11/nei-web` | Caddy serving the built site, proxying `/api` to `nei-api`. The only thing the public reaches. |
| `nei-api` (1 replica) | `hydrodog11/nei-api` | FastAPI: the board payload, the writes, the derivation. Its initContainer builds the database if it is empty. |
| `nei-postgres` (3 instances) | CNPG 17.2 | The board. Every thread a visitor lays. |

One origin serves the site and the API, because the board draws a QR code from
`location.origin` and the phone that scans it has to reach the same place.

### The bootstrap, and why it is an initContainer

`db/*.sql` — schema, concept pool, seeded board, frozen snapshot — ships **inside
the api image**, and `exhibit.bootstrap` applies it before anything serves.

It asks before it writes: if the `pins` table exists it does nothing at all, and
it never drops, truncates or deletes. That is what makes `kubectl rollout
restart` a complete deploy in a cluster with no image updater — an Argo PreSync
hook would only fire on a sync, so a restart-driven deploy could start new code
against an empty schema.

### Nothing calls outward during the event

No Wikidata, no Wikipedia, no Commons, no Google Fonts, and no visitor ever
reaches an AI. Images and fonts are served by the site itself. The concept pool
was warmed ahead of time and lives in Postgres. A visitor's IP is never stored.

## First deploy

### 1. Create the Bitwarden secrets

In Bitwarden **Secrets Manager**, before syncing anything:

| Key | Value |
| --- | --- |
| `rafael-nei-db-password` | `openssl rand -hex 24` |
| `rafael-nei-admin-password` | `openssl rand -hex 24`, or empty for no admin at all |
| `rafael-nei-backup-s3-key-id` | Garage key scoped to `porto-k8s-backup` |
| `rafael-nei-backup-s3-secret` | its secret |

A missing key fails the whole ExternalSecret, so create all four.

### 2. Publish the images

The app repo pushes both on every push to `main`
(`.github/workflows/docker-publish.yml`). It needs `DOCKERHUB_USERNAME` and
`DOCKERHUB_TOKEN` in that repo's secrets, and **both Docker Hub repositories must
be public** — there is no pull secret here, unlike nubiup.

```bash
# by hand, from the app repo, if CI is not wired yet
docker build -t hydrodog11/nei-api:latest -f api/Dockerfile .
docker build -t hydrodog11/nei-web:latest web
docker push hydrodog11/nei-api:latest && docker push hydrodog11/nei-web:latest
```

The api image's build context is the repository root, not `api/`: that is how
`db/*.sql` gets into it.

### 3. The name

`nei.duarte-correia.pt`, in three places that must agree: `tunnelbinding.yaml`,
`gateway.yaml` and `certificate.yaml`. Change all three together if it changes.

The tunnel is what makes the exhibit independent of the venue's wifi: visitors
are on mobile data in a street in Coimbra.

### 4. Sync

Push this repo and let the automated sync run, or hit **Sync** in the Argo UI.
Then, still in the UI: the four ExternalSecrets should read `SecretSynced`, and
the `nei-api` pod's **bootstrap** container log says what it applied.

On a database that already has a schema the bootstrap prints "the schema is
already there, nothing to do", which is the expected line on every deploy after
the first.

### 5. Check it end to end

```bash
curl -s https://nei.duarte-correia.pt/api/health/ready      # {"status":"ok","db":"ok"}
curl -s https://nei.duarte-correia.pt/api/board | head -c 200
# and from the app repo, the real check:
tools/smoke.sh https://nei.duarte-correia.pt
```

## Shipping a new version

`git push` on the app repo; CI builds and pushes `:latest` and `:<sha>`. Then, in
the Argo UI, select the `nei-api` and `nei-web` Deployments and hit **Restart**
(`kubectl -n rafael-homelab rollout restart deploy/nei-api deploy/nei-web` does
the same thing if you have a kubeconfig to hand).

`imagePullPolicy: Always` on `:latest` is what makes a restart a deploy. This
cluster has no image updater, so nothing happens on its own. To pin or roll
back, set the image to `hydrodog11/nei-api:<sha>` in `api-deployment.yaml` and
let Argo sync — that route needs no CLI at all, and leaves the running version
visible in git.

## During the evening

**Retuning.** `configmap.yaml` holds every tuning value the exhibit has. Edit,
commit, let Argo sync, then `rollout restart deploy/nei-api`. Nothing is lost:
the board is derived from `threads` on every read and `threads` is append-only.
`HALFLIFE_MIN` is the one worth watching — lower it if the board looks frozen by
21:00, raise it if it is thrashing.

**Backups.** Continuous WAL archiving to Garage, a base backup nightly at 02:40,
and `nei-postgres-event-backup` every half hour between 14:00 and 23:30 **on 25
September only**. Suspend or delete that one afterwards.

**The admin** is on the LAN only: `https://nei.duarte-correia.pt/admin`, reachable
from the house network, where the Pi-hole resolves the name and the Envoy gateway
routes `/admin` straight to `nei-api`. The tunnel points at `nei-web`, whose Caddy
has no route to `/admin` at all, so the public door cannot reach it.

It does not exist until `rafael-nei-admin-password` is non-empty — leave that
Bitwarden item empty for the night and set it afterwards. The panel then carries
**Download the evening**: the three tables of the study as one zip, so the export
needs no cluster access.

**Watching it.** The Argo UI has the logs of every pod, which is enough for the
night. The exhibit's own check is better than either:

```bash
tools/smoke.sh https://nei.duarte-correia.pt    # from the app repo
curl -s https://nei.duarte-correia.pt/api/board | head -c 200
```

## Afterwards

The evening is the data. Before anything else:

A base backup runs every half hour that evening already, and the WAL archive is
continuous, so the moment the doors close the evening is in Garage. If you want
one more by hand, the Argo UI can create a CNPG `Backup`, or `kubectl cnpg backup
nei-postgres` with a kubeconfig.

Then set `rafael-nei-admin-password`, restart the api from the Argo UI, and take
the export from `https://nei.duarte-correia.pt/admin` → **Download the evening**
(`tools/export_study.py` does the same thing from a compose stack). The `relation`
column is what strangers typed: read it before it goes anywhere public.

## What has not been verified

These manifests have been schema-validated offline (`kubeconform`, including the
CRD catalogue) but **never applied to the cluster** — the kubeconfig in this repo
is unauthenticated, so there was no server to dry-run against. Before the event,
run:

```bash
kubectl apply --dry-run=server -f kubernetes/deployments/nei/manifests
```

The three things most likely to need a correction are the Garage endpoint in
`postgres-cluster.yaml` (copied from nubiup), the `cnpg-system` namespace label
in `networkpolicy.yaml`, and whether `nei.duarte-correia.pt` is the name you
want.
