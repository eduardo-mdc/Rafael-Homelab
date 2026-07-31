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
| `nubiup-beat` | Celery beat — the scheduler, exactly one replica |
| `nubiup-postgres` | CloudNativePG, 3 instances |
| `nubiup-redis` | Celery broker `/0`, results `/1`, Channels layer `/2` |

Everything is one container image, `hydrodog11/biohub-up`, with different
entrypoints.

## Shipping a new version

**`argocd-image-updater` is not installed in this cluster.** The
`argocd-image-updater.argoproj.io/*` annotations on this app — and on portfolio,
cladewright and palworld — currently do nothing. A merge to `main` publishes a new
`:latest` to Docker Hub, and then nothing happens until you roll it out:

```bash
kubectl -n rafael-homelab rollout restart deploy/nubiup-web deploy/nubiup-beat
```

That is a complete deploy by itself. `imagePullPolicy: Always` re-pulls `:latest`,
and the web pod's `migrate` initContainer applies migrations and re-seeds before any
container serves traffic.

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

| Secret key | Value |
|---|---|
| `rafael-nubiup-django-secret-key` | Long random string: `python -c 'import secrets;print(secrets.token_urlsafe(64))'` |
| `rafael-nubiup-db-username` | Exactly **`nubiup`** — see below |
| `rafael-nubiup-db-password` | Strong password. Avoid `@ : / #`, it is used in a Postgres DSN |

The username is not a free choice. `postgres-cluster.yaml` declares
`bootstrap.initdb.owner: nubiup` and `database: nubiup`, and the *same* secret
supplies Django's `POSTGRES_USER`. Any other value and CNPG creates one role while
Django authenticates as another — the pod then fails on `role "..." does not exist`,
which reads like a password problem.

Verify they resolved before going further — both should report `SecretSynced`:

```bash
kubectl -n rafael-homelab get externalsecret
kubectl -n rafael-homelab get secret nubiup-app-secret nubiup-db-secret
```

### 2. Publish the image

Two separate things, both required.

**a. Repository secrets in `website_nubiup`.** `DOCKERHUB_USERNAME` and
`DOCKERHUB_TOKEN` (an access token from Docker Hub → Account Settings → Personal
access tokens, with Read & Write). Without them the workflow's login step fails and
nothing is published. Nothing is needed in *this* repo.

**b. The Docker Hub repository must end up PUBLIC.**
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

### 3. Sync

The Application is registered in `kubernetes/deployments/kustomization.yaml`, so
Argo picks it up. The web pod's `migrate` initContainer runs migrations and seeds the
bilingual page tree plus the five staff groups before any container serves traffic.

### 4. Create the first superuser

Nothing bootstraps an admin — deliberately, so there is never a default password
reachable from the internet.

```bash
kubectl -n rafael-homelab exec deploy/nubiup-web -c web -it -- \
  python manage.py createsuperuser
```

Then sign in at `https://nubi.duarte-correia.pt/cms/` and assign staff groups
under *Settings → Groups*.

### 5. Fix the contact form addresses

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

To enable it, add `rafael-nubiup-email-host-user` / `-password` to
`external-secret.yaml`, then set in `configmap.yaml`:

```yaml
EMAIL_HOST: smtp.example.com
EMAIL_PORT: "587"
EMAIL_USE_TLS: "true"
DEFAULT_FROM_EMAIL: NUBI UP <hello@the-real-domain>
```

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
| `configmap.yaml` | `DJANGO_ALLOWED_HOSTS`, `CSRF_TRUSTED_ORIGINS`, `PUBLIC_SITE_URL`, `WAGTAIL_BASE_URL`, `WAGTAIL_SITE_HOSTNAME` |
| `gateway.yaml` | listener `hostname`, HTTPRoute `hostnames` |
| `certificate.yaml` | `dnsNames` |
| `tunnelbinding.yaml` | `fqdn` |

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

# Django shell
kubectl -n rafael-homelab exec deploy/nubiup-web -c web -it -- python manage.py shell

# Re-run the seed / re-sync Drive resources
kubectl -n rafael-homelab exec deploy/nubiup-web -c web -it -- python manage.py create_initial_pages
kubectl -n rafael-homelab exec deploy/nubiup-web -c web -it -- python manage.py sync_drive_resources

# Why did the last deploy fail? (migrations run here)
kubectl -n rafael-homelab logs deploy/nubiup-web -c migrate

# Roll out a newly published image
kubectl -n rafael-homelab rollout restart deploy/nubiup-web deploy/nubiup-beat
```

### Notes

- **`/media/` over the tunnel bypasses nginx.** The TunnelBinding points at
  `nubiup-web:8000` directly, so tunnel traffic never passes through the HTTPRoute
  and Django serves `/media/` itself. Correct, just less efficient. LAN traffic
  resolves via Pi-hole to the gateway and does get the nginx path.
- **Newsletter and certificate sends need the worker and Redis healthy.** If a
  queued campaign never leaves, check the `worker` container logs before suspecting
  SMTP.
- **Google Drive sync** is not configured. It needs a service-account JSON in
  `GOOGLE_DRIVE_SERVICE_ACCOUNT_JSON_B64` (base64), which belongs in Bitwarden as
  `rafael-nubiup-drive-service-account-b64` wired through `external-secret.yaml` —
  never in the ConfigMap. Until then `sync_drive_resources` raises
  `DriveCredentialError` and the resources page stays empty. Setup steps are in the
  app repo at `docs/GOOGLE_DRIVE_SETUP.md`.
