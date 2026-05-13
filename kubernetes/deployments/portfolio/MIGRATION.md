# Portfolio Migration Runbook

Migrating from old namespace to `portfolio` namespace on the same cluster.
Same domain (`rafael.duarte-correia.pt`) means Envoy Gateway will conflict if both
are live simultaneously — this runbook minimises downtime to ~1 minute.

## Prerequisites

- `kubectl` configured against the homelab cluster
- Know the old namespace name (run `kubectl get ns` if unsure)
- New stack already committed to git (ArgoCD will pick it up)

Set this once at the top of your shell session:
```bash
export OLD_NS=<your-old-namespace>   # e.g. portfolio-old, default, etc.
export NEW_NS=portfolio
```

---

## Phase 1 — Inspect the old deployment

```bash
kubectl get all,gateway,httproute -n $OLD_NS
kubectl get pods -n $OLD_NS
kubectl get secret -n $OLD_NS
```

Identify:
- The postgres pod name (e.g. `postgres-0` or `portfolio-db-xxx`)
- The postgres credentials secret name
- The Gateway resource name (needed for deletion)

---

## Phase 2 — Deploy new stack WITHOUT the Gateway

Before pushing to git, temporarily move gateway.yaml aside so ArgoCD
does not try to create a conflicting Gateway resource:

```bash
# In the Rafael-Homelab repo locally:
cd kubernetes/deployments/portfolio/manifests
mkdir -p .hold
mv gateway.yaml .hold/
git add -A && git commit -m "chore: deploy portfolio without gateway (pre-migration)"
git push
```

Wait for ArgoCD to sync and for the new postgres cluster and pods to be Ready:
```bash
kubectl get pods -n $NEW_NS -w
# Wait until portfolio-postgres-1, -2, -3 are Running and portfolio pod is Running
```

---

## Phase 3 — Dump the old database

```bash
# Find the old postgres pod
OLD_PG_POD=$(kubectl get pod -n $OLD_NS -l cnpg.io/cluster=<OLD_CLUSTER_NAME> \
  -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || \
  kubectl get pod -n $OLD_NS -l app=postgres \
  -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || \
  kubectl get pod -n $OLD_NS | grep -i postgres | awk '{print $1}' | head -1)
echo "Old postgres pod: $OLD_PG_POD"

# Get the old DB password
OLD_PG_SECRET=$(kubectl get secret -n $OLD_NS | grep -i db | awk '{print $1}' | head -1)
OLD_PG_PASS=$(kubectl get secret -n $OLD_NS $OLD_PG_SECRET \
  -o jsonpath='{.data.password}' | base64 -d)
OLD_PG_USER=$(kubectl get secret -n $OLD_NS $OLD_PG_SECRET \
  -o jsonpath='{.data.username}' | base64 -d 2>/dev/null || echo "portfolio")
OLD_PG_DB=$(kubectl get secret -n $OLD_NS $OLD_PG_SECRET \
  -o jsonpath='{.data.database}' | base64 -d 2>/dev/null || echo "portfolio")

echo "User: $OLD_PG_USER  DB: $OLD_PG_DB"

# Dump — writes portfolio_dump.sql to your local machine
kubectl exec -n $OLD_NS $OLD_PG_POD -- \
  bash -c "PGPASSWORD='$OLD_PG_PASS' pg_dump -U $OLD_PG_USER $OLD_PG_DB --no-owner --no-acl" \
  > /tmp/portfolio_dump.sql

wc -l /tmp/portfolio_dump.sql   # sanity check — should be non-zero
```

---

## Phase 4 — Copy media files

```bash
# Find the old web pod (has /app/media mounted)
OLD_WEB_POD=$(kubectl get pod -n $OLD_NS -l app=portfolio \
  -o jsonpath='{.items[0].metadata.name}')
echo "Old web pod: $OLD_WEB_POD"

# Copy media to local machine first
kubectl cp $OLD_NS/$OLD_WEB_POD:/app/media /tmp/portfolio-media

# Find new web pod
NEW_WEB_POD=$(kubectl get pod -n $NEW_NS -l app=portfolio \
  -o jsonpath='{.items[0].metadata.name}')
echo "New web pod: $NEW_WEB_POD"

# Copy from local into new pod
kubectl cp /tmp/portfolio-media $NEW_NS/$NEW_WEB_POD:/app/media
```

---

## Phase 5 — Import the database dump into new cluster

```bash
# Get new DB credentials from the CNPG-managed secret
NEW_PG_POD=$(kubectl get pod -n $NEW_NS -l cnpg.io/instanceRole=primary \
  -o jsonpath='{.items[0].metadata.name}')
echo "New postgres primary: $NEW_PG_POD"

NEW_PG_PASS=$(kubectl get secret -n $NEW_NS portfolio-db-secret \
  -o jsonpath='{.data.password}' | base64 -d)

# Copy dump into the new postgres pod and restore
kubectl cp /tmp/portfolio_dump.sql $NEW_NS/$NEW_PG_POD:/tmp/portfolio_dump.sql

kubectl exec -n $NEW_NS $NEW_PG_POD -- \
  bash -c "PGPASSWORD='$NEW_PG_PASS' psql -U portfolio -d portfolio -f /tmp/portfolio_dump.sql"

# Verify row counts look right
kubectl exec -n $NEW_NS $NEW_PG_POD -- \
  bash -c "PGPASSWORD='$NEW_PG_PASS' psql -U portfolio -d portfolio -c '\dt'"
```

---

## Phase 6 — The cutover (brief downtime ~1 minute)

This is the atomic swap. Run these commands back-to-back as fast as possible:

```bash
# 1. Delete the old namespace entirely (removes old Gateway + DNS entry)
kubectl delete namespace $OLD_NS

# 2. Restore gateway.yaml and push (ArgoCD will create new Gateway within seconds)
# In the Rafael-Homelab repo:
cd kubernetes/deployments/portfolio/manifests
mv .hold/gateway.yaml .
rmdir .hold
git add gateway.yaml && git commit -m "chore: restore portfolio gateway post-migration"
git push

# 3. Force an immediate ArgoCD sync (don't wait for the poll interval)
argocd app sync portfolio
# or via kubectl:
kubectl annotate application portfolio -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite
```

Watch the Gateway come up:
```bash
kubectl get gateway -n $NEW_NS -w
kubectl get certificate -n $NEW_NS -w   # TLS cert — takes ~30s
```

---

## Phase 7 — Verify and clean up

```bash
# Check the site is responding
curl -I https://rafael.duarte-correia.pt

# Check Wagtail admin loads
curl -I https://rafael.duarte-correia.pt/cms/

# Clean up local temp files
rm -rf /tmp/portfolio_dump.sql /tmp/portfolio-media
```

If anything looks wrong after the cutover, you still have `/tmp/portfolio_dump.sql`
locally as a fallback until you delete it.

---

## Notes

- **Wagtail superuser**: if you need to create a new superuser in the new cluster:
  ```bash
  kubectl exec -n portfolio -it $NEW_WEB_POD -- python manage.py createsuperuser
  ```
- **CNPG vs plain Postgres**: the pg_dump commands above work for both. If the old
  cluster also used CNPG, use `-l cnpg.io/instanceRole=primary` to find the pod.
- **ArgoCD Image Updater**: once the migration is done and the image updater is
  running, every push to `main` in the portfolio repo will auto-deploy within minutes.
