# Nubi UP

Bilingual public website (Django + Wagtail) with a React SPA staff area on a DRF
API. Source: [`website_nubiup`](https://github.com/rafaelcorreia/website_nubiup).

**Currently deployed at `nubi.duarte-correia.pt`** — a placeholder host. The
project's own domain has not been bought yet; see [Moving to the real
domain](#moving-to-the-real-domain).

---

## What runs

| Workload | What it is |
|---|---|
| `nubiup-web` (pod, 3 containers) | `web` Daphne/ASGI · `worker` Celery · `media` nginx serving `/media/`. Plus a `migrate` initContainer |
| `nubiup-next` | The **public site** — a Next server, 2 replicas, rolling updates |
| `nubiup-proxy` | The **front door** — nginx, 2 replicas. Owns the path split; both the Gateway and the tunnel point here |
| `nubiup-beat` | Celery beat — the scheduler, exactly one replica |
| `nubiup-postgres` | CloudNativePG, 3 instances |
| `nubiup-redis` | Celery broker `/0`, results `/1`, Channels `/2`, Django cache `/3` |

**Two images, not one:**

| Image | Runs |
|---|---|
| `hydrodog11/biohub-up` | web, worker, beat, migrate — same image, different entrypoints |
| `hydrodog11/biohub-up-web` | `nubiup-next` |

### Django does not serve the public site

That is the thing to know before reading anything else here. Django serves
`/cms/`, `/staff/`, `/api/v2/` and `/documents/`; a Next server renders every
public page by calling Django's content API. The **ConfigMap in `proxy.yaml`**
decides which backend gets a path, and it mirrors the app repo's
`nginx/default.conf` — that file is the readable reference.

| Path | Backend |
|---|---|
| `/en/…`, `/pt/…`, `/_next/…`, `/api/forms/…` | `nubiup-next` |
| `/media/…` | `nubiup-media` (the nginx container in the app pod) |
| everything else | `nubiup-web` |

**The split lives in one place, and it is not the HTTPRoute.** It used to be, and
that could only ever serve LAN traffic: the Cloudflare tunnel does not enter
through the Gateway, so rules written there are invisible to the public
internet — which is why `tunnelbinding.yaml` sat on a placeholder. Both the
HTTPRoute and the TunnelBinding are now a single hop to `nubiup-proxy`, so LAN
and WAN traverse the same table and cannot drift apart.

The two images are published from the same commit, tagged with the same SHA, and
**must be rolled out together** — a public site expecting an API field the
deployed Django does not serve renders broken pages. The app repo's
`docs/deploy.md` is the full contract.

## Shipping a new version

**`argocd-image-updater` is not installed in this cluster.** The
`argocd-image-updater.argoproj.io/*` annotations on this app — and on portfolio,
cladewright and palworld — currently do nothing. A merge to `main` publishes a new
`:latest` to Docker Hub, and then nothing happens until you roll it out:

```bash
kubectl -n rafael-homelab rollout restart \
  deploy/nubiup-web deploy/nubiup-next deploy/nubiup-beat deploy/nubiup-proxy
```

That is a complete deploy by itself. `imagePullPolicy: Always` re-pulls `:latest`,
and the web pod's `migrate` initContainer applies migrations and re-seeds before any
container serves traffic.

**All four, in one command.** Restarting `nubiup-web` alone leaves the public site
running the previous build against a freshly migrated database — the exact skew the
same-SHA tagging exists to prevent. (`nubiup-proxy` carries no application code and
rarely needs it, but it costs nothing and keeps the command one thing to remember.)

This is why migrations are an **initContainer** rather than an Argo PreSync hook: a
PreSync hook only fires when a manifest changes in git, so a restart-driven deploy
would have started new code against an un-migrated schema. Installing
argocd-image-updater later would automate the restart, and the annotations are
already in place for it — the initContainer stays correct either way.

### Why web, worker and media share a pod

They need the same two filesystems — Wagtail media, and the document vault that
web writes uploads to and the worker writes generated certificate PDFs to. This
cluster is **RWO-only**: `iscsi-zfs` (democratic-csi/iSCSI) is the only storage
class, and every one of the ~46 PVCs in the cluster is `ReadWriteOnce`. Volumes
therefore cannot be shared between pods, but containers within a pod share them
freely.

The trade is **one replica and `Recreate`** — an RWO volume can't attach to an old
and new pod simultaneously, so a deploy has a ~20-40s gap. For a student
organisation's site that is the right call; making it seamless means moving media
and the vault to object storage first (Garage already runs in this cluster, in
namespace `garage-host`), which is an application change in `website_nubiup`, not a
manifest change here.

> `kubernetes/examples/pvc.yaml` still describes Longhorn as the default storage
> class. That is stale — there is no Longhorn in this cluster.

### The vault is not public

Vault documents have no URL by design: `PrivateFileSystemStorage.url()` raises, and
every read is streamed by an authenticated Django view with `Cache-Control: private,
no-store`. The nginx container mounts **only** the media volume. Do not add a vault
mount to it.

---

## First deploy

### 1. Create the Bitwarden secrets

In **Bitwarden Secrets Manager** — not the password vault; they are different
products and the vault is not what this reads. They must live in the organization
and project the `bitwarden-secretstore` ClusterSecretStore is pinned to (org
`325ac27e-91d4-4ca0-bd60-b3cb00fb665f`, project
`57f38d98-f6eb-4c54-a3ce-b3cc01497ae9`); a correctly-named secret in another project
will not resolve.

| Secret key | Value | Needed for |
|---|---|---|
| `rafael-nubiup-django-secret-key` | Long random string: `python -c 'import secrets;print(secrets.token_urlsafe(64))'` | Django `SECRET_KEY` |
| `rafael-nubiup-db-password` | Strong password. Avoid `@ : / #`, it is used in a Postgres DSN | Postgres |
| `rafael-nubiup-redis-password` | Same generator, same character caveat — it goes into four `redis://` URLs | Redis auth |
| `rafael-nubiup-dockerhub-username` | The Docker Hub account name | Image pull |
| `rafael-nubiup-dockerhub-token` | A **read-only** Docker Hub access token, not the account password | Image pull |
| `rafael-nubiup-backup-s3-key-id` | Garage key id, scoped to the `porto-k8s-backup` bucket | Postgres backups |
| `rafael-nubiup-backup-s3-secret` | The matching Garage secret | Postgres backups |
| `rafael-nubiup-drive-service-account-b64` | `base64 -w0 drive-sa.json` — one line, no newlines | Drive sync (optional) |

`rafael-nubiup-db-username` is **no longer read**. It used to be, and sourcing a
role *name* from a secret store only created a way to get it wrong: a random value
there yields `password authentication failed for user "dC0j4Qak…"`, which reads
like a password problem and is not one. The role name is now a literal in
`external-secret.yaml`, beside the comment explaining why.

The username is not a free choice. `postgres-cluster.yaml` declares
`bootstrap.initdb.owner: nubiup` and `database: nubiup`, and the *same* secret
supplies Django's `POSTGRES_USER`. Any other value and CNPG creates one role while
Django authenticates as another — the pod then fails on `role "..." does not exist`,
which reads like a password problem.

Verify they resolved before going further — both should report `SecretSynced`:

```bash
kubectl -n rafael-homelab get externalsecret
kubectl -n rafael-homelab get secret \
  nubiup-app-secret nubiup-db-secret nubiup-redis-secret \
  nubiup-dockerhub nubiup-backup-s3-secret
```

**Redis, Docker Hub and the backup credentials are not optional.** Redis rejects
every connection without `nubiup-redis-secret` (it sets `--requirepass` from it,
and the four `redis://` URLs are templated from the same value), the pull secret
is what keeps the private image pullable, and without the Garage key the CNPG
cluster archives no WAL. `nubiup-drive-secret` is the one that can wait — the
site works without Drive sync, the Resources page is just empty.

### 2. Publish the image

Two separate things, both required.

**a. Repository secrets in `website_nubiup`.** `DOCKERHUB_USERNAME` and
`DOCKERHUB_TOKEN` (an access token from Docker Hub → Account Settings → Personal
access tokens, with Read & Write). Without them the workflow's login step fails and
nothing is published. Nothing is needed in *this* repo.

**b. BOTH Docker Hub repositories must end up PUBLIC** (or both covered by the
pull secret — see below). There are two now: `hydrodog11/biohub-up` and
`hydrodog11/biohub-up-web`. Checking one and forgetting the other leaves the
public site in `ImagePullBackOff` while Django comes up fine, which reads as a
Next problem and is not one.

`hydrodog11/biohub-up` does not exist yet; the first push creates it, and Docker Hub
creates new repositories using your account's *default repository privacy* setting —
which for many accounts is **private**.

That matters because **nothing in this cluster has an imagePullSecret**: no pod, no
`default` ServiceAccount, no registry secret in the namespace. `hydrodog11/portfolio`,
`/cladewright` and `/palworld` are all public and pulled anonymously. If
`biohub-up` lands private, every pod sits in `ImagePullBackOff` with an
authentication error that reads like broken credentials rather than wrong visibility.

So after the first successful push, check it:

```bash
curl -s https://hub.docker.com/v2/repositories/hydrodog11/biohub-up/ \
  | python3 -c 'import sys,json;print("private:", json.load(sys.stdin)["is_private"])'
```

If it says `private: True`, either flip it to public in the Docker Hub UI, or add an
imagePullSecret — but note that would make this the only app here needing one.

### 3. The tunnel — nothing to fill in

`tunnelbinding.yaml` needs no per-cluster value. It targets an ordinary Service:

```yaml
target: http://nubiup-proxy.rafael-homelab.svc.cluster.local:80
```

**This used to be the one value here that could not be written blind**, and the
reason is worth keeping. Tunnel traffic does not pass through the HTTPRoute; it
goes wherever the TunnelBinding points. While Django rendered the public site,
`nubiup-web:8000` was correct. Once Next took the public pages over, the split
mattered — and the split lived in the HTTPRoute, which the tunnel cannot see. The
apparent fix, routing the tunnel through the Gateway, needs the Service Envoy
provisions, whose name carries a generated hash and had to be read out of a
running cluster with `kubectl`. So the manifest carried a `REPLACE-ME`, and a
manifest with a placeholder in it is not a deployment.

Moving the split into `nubiup-proxy` removed the lookup entirely. The tunnel now
targets a workload by a name that is known from the manifests alone — the same
shape portfolio, cladewright and umami all use.

If you ever point this somewhere else, remember what the failure looks like: the
tunnel is how the **public** reaches this site, while LAN traffic resolves via
Pi-hole to the cluster ingress. Get it wrong and the site looks correct from the
office and is broken from everywhere else.

### 4. Sync

The Application is registered in `kubernetes/deployments/kustomization.yaml`, so
Argo picks it up. The web pod's `migrate` initContainer runs migrations and seeds the
bilingual page tree plus the five staff groups before any container serves traffic.

### 5. Create the first superuser

Nothing bootstraps an admin — deliberately, so there is never a default password
reachable from the internet.

```bash
kubectl -n rafael-homelab exec deploy/nubiup-web -c web -it -- \
  python manage.py createsuperuser
```

Then sign in at `https://nubi.duarte-correia.pt/cms/` and assign staff groups
under *Settings → Groups*.

### 6. Fix the contact form addresses

`create_initial_pages` seeds the ContactPage with
`PLACEHOLDER-EMAIL@example.com` as both to- and from-address. These are **database
rows, not settings** — editing environment variables will not touch them. Change
them in the CMS on the ContactPage, or the form silently mails nowhere.

---

## Email is not configured

`EMAIL_HOST` is deliberately unset, so Django uses the console backend and all mail
goes to the pod log. That is correct until the domain and a real mailbox exist.

Not working until then: password reset (`/cms/password_reset/`), the contact form,
newsletter sends, and certificate delivery. Everything else is unaffected.

**Turning it on is uncommenting, not writing.** Both halves are already in the
manifests, commented out, and every Django workload (web, worker, beat, migrate)
already loads `nubiup-env` and `nubiup-app-secret` — no Deployment changes:

1. Create `rafael-nubiup-email-host-user` and `rafael-nubiup-email-host-password`
   in Bitwarden. **Do this first.** An ExternalSecret that names a key which does
   not exist fails as a whole, and `nubiup-app-secret` also carries
   `DJANGO_SECRET_KEY` — so uncommenting early breaks Django, not just email.
2. Uncomment the two `EMAIL_HOST_*` entries in `external-secret.yaml`.
3. Uncomment and fill the email block in `configmap.yaml`: `EMAIL_HOST`,
   `EMAIL_PORT`, `EMAIL_USE_TLS`, `DEFAULT_FROM_EMAIL`, `NEWSLETTER_REPLY_TO`.
4. Once Argo has synced, restart `nubiup-web` and `nubiup-beat`. Environment from
   `envFrom` is read when a container starts, so a synced ConfigMap changes
   nothing in a pod that is already running — mail keeps going to the log and it
   looks like the settings are wrong.
5. Change the ContactPage to- and from-addresses in the CMS (see above). Those are
   database rows, and no variable reaches them.

`DEFAULT_FROM_EMAIL` matters: unset, it defaults to a literal
`PLACEHOLDER-EMAIL@example.com`, which is deliverable-looking and undeliverable.
Note also that `apps/newsletter/url_utils.py` refuses to send over real SMTP while
`PUBLIC_SITE_URL` points at localhost — a guard, not a bug.

---

## Moving to the real domain

The hostname appears in **four files**. Change all of them in one commit, or the
certificate and the gateway disagree and the listener serves no TLS:

| File | What to change |
|---|---|
| `configmap.yaml` | `DJANGO_ALLOWED_HOSTS`, `CSRF_TRUSTED_ORIGINS`, `PUBLIC_SITE_URL`, `WAGTAIL_BASE_URL`, `WAGTAIL_SITE_HOSTNAME`, `FRONTEND_PREVIEW_URL` |
| `gateway.yaml` | listener `hostname`, HTTPRoute `hostnames` |
| `certificate.yaml` | `dnsNames` |
| `tunnelbinding.yaml` | `fqdn` |
| `deployment.yaml` | the `Host` header on both probes |
| `next-deployment.yaml` | the `Host` header on both probes |

The two probe entries are the ones that get missed, and they do not fail in a way
that looks like a hostname problem: the pod is fine, serves correctly by hand, and
every probe comes back 400 because `ALLOWED_HOSTS` no longer contains the value
kubelet is sending. The applications themselves need nothing — both learn the host
from each request.

Then:

1. Add the domain to Cloudflare (DNS-01 is how cert-manager proves ownership — it
   will retry forever against a domain you don't control).
2. Sync. `create_initial_pages` rewrites the Wagtail `Site` row from
   `WAGTAIL_SITE_HOSTNAME` on every sync, so `page.full_url` and the sitemap follow
   automatically.
3. Keep the old hostname in `DJANGO_ALLOWED_HOSTS` for a while if anything is
   bookmarked against it.

**On HSTS:** `config/settings/prod.py` sets `SECURE_HSTS_SECONDS` to one year, so
the first successful load pins HTTPS in every visitor's browser for the whole year.
That is safe here because TLS is terminated at the gateway with a real Let's Encrypt
certificate — but it is why step 1 comes before the first public request on a new
domain, not after.

---

## Operations

```bash
# Logs, per container
kubectl -n rafael-homelab logs deploy/nubiup-web -c web -f
kubectl -n rafael-homelab logs deploy/nubiup-web -c worker -f
kubectl -n rafael-homelab logs deploy/nubiup-beat -f
# The public site. A blank or 500ing page is usually here, not in Django.
kubectl -n rafael-homelab logs deploy/nubiup-next -f

# Django shell
kubectl -n rafael-homelab exec deploy/nubiup-web -c web -it -- python manage.py shell

# Re-run the seed / re-sync Drive resources
kubectl -n rafael-homelab exec deploy/nubiup-web -c web -it -- python manage.py create_initial_pages
kubectl -n rafael-homelab exec deploy/nubiup-web -c web -it -- python manage.py sync_drive_resources

# Why did the last deploy fail? (migrations run here)
kubectl -n rafael-homelab logs deploy/nubiup-web -c migrate

# Roll out newly published images — all three together, always
kubectl -n rafael-homelab rollout restart \
  deploy/nubiup-web deploy/nubiup-next deploy/nubiup-beat deploy/nubiup-proxy

# Is a public page actually rendering? (bypasses the gateway and the tunnel)
kubectl -n rafael-homelab exec deploy/nubiup-next -- \
  node -e "fetch('http://127.0.0.1:3000/en/',{headers:{Host:'nubi.duarte-correia.pt'}}).then(r=>console.log(r.status))"

# Django's own health, as the probes see it
kubectl -n rafael-homelab exec deploy/nubiup-web -c web -- \
  python -c "import urllib.request;print(urllib.request.urlopen(urllib.request.Request('http://127.0.0.1:8000/readyz',headers={'Host':'nubi.duarte-correia.pt'})).read().decode())" 
```

### Notes

- **The tunnel and the Gateway both hit `nubiup-proxy`, so both paths route
  identically.** The tunnel used to point at `nubiup-web:8000` directly, which
  meant tunnel traffic bypassed the HTTPRoute and Django served `/media/` itself —
  tolerable while Django also served the public site, and not afterwards:
  bypassing the HTTPRoute meant bypassing the Next split, so every public page
  came back from a Django with no template for it. Routing the tunnel through
  Envoy would have fixed that at the cost of a Service name only a running
  cluster can tell you. One front door fixes it with neither. See "The tunnel"
  above.
- **`/cms/` and `/staff/` are reachable from outside the LAN, on purpose.** Nubi UP
  is a student organisation with staff who work from wherever they are; an admin
  surface that only answers on the LAN is one most of them cannot use. Wagtail
  login, `StaffRequiredMixin`, the DRF permission classes and django-axes are what
  hold, not network position.
- **The probes are HTTP now, and they send an explicit `Host` header.** They used
  to be TCP checks, on the reasoning that `ALLOWED_HOSTS` rejects the pod IP
  kubelet sends. That diagnosis was right and the cure was wrong: a TCP check
  passes on a Daphne that has lost its database and answers 500 to everything.
  Django exposes `/healthz` (liveness, touches nothing) and `/readyz` (readiness,
  round-trips Postgres and Redis). **Never point liveness at `/readyz`** — with one
  replica on an RWO volume, a Postgres blip would restart the pod and take the
  whole site down instead of degrading it.
- **Newsletter and certificate sends need the worker and Redis healthy.** If a
  queued campaign never leaves, check the `worker` container logs before suspecting
  SMTP.
- **Google Drive sync** is wired but needs its secret. `external-secret.yaml`
  now provides `GOOGLE_DRIVE_SERVICE_ACCOUNT_JSON_B64` to the web and worker
  containers from Bitwarden's `rafael-nubiup-drive-service-account-b64`; create
  that item and the sync starts working. Never the ConfigMap — it is a private key. Until then `sync_drive_resources` raises
  `DriveCredentialError` and the resources page stays empty. Setup steps are in the
  app repo at `docs/GOOGLE_DRIVE_SETUP.md`.
