# n8n

Workflow automation, backed by a co-deployed Postgres 15 + Redis. The
chart is `8gears/n8n` from the 8gears OCI registry.

## Required out-of-band secrets

Two Secrets must exist in the `n8n` namespace **before** the Application
syncs (otherwise the pods will fail with missing-env errors):

```sh
# 1. n8n encryption key — used to encrypt credentials at rest.
#    Generate once and never lose it (rotating it invalidates all
#    stored credentials in n8n).
kubectl -n n8n create secret generic n8n-secrets \
  --from-literal=N8N_ENCRYPTION_KEY="$(openssl rand -hex 32)"
# ⚠ MIGRATING an existing install: do NOT generate a fresh key — reuse the
#    one the running pods already use, or every stored credential in the DB
#    becomes undecryptable. See "Migrating to this layout" below.

# 2. Postgres credentials — consumed by both the postgres container
#    (POSTGRES_*) and the n8n main process (DB_POSTGRESDB_PASSWORD).
kubectl -n n8n create secret generic postgres-n8n \
  --from-literal=POSTGRES_DB=n8n \
  --from-literal=POSTGRES_USER=n8n \
  --from-literal=POSTGRES_PASSWORD="$(openssl rand -base64 24)"
```

If the namespace doesn't exist yet, run `kubectl create ns n8n` first
(Argo CD will create it on first sync, but the Secrets must precede it,
so do it manually here).

## Notes

- The Postgres `Secret` is **not** rendered by the chart anymore (the old
  inline `extraManifests` Secret was removed to avoid leaking the
  password into git). Postgres still picks up `POSTGRES_DB/USER/PASSWORD`
  via `secretKeyRef`.
- `N8N_ENCRYPTION_KEY` is referenced by all three n8n tiers
  (`main`, `worker`, `webhook`) via `extraEnv.valueFrom.secretKeyRef`.
- `DB_POSTGRESDB_PASSWORD` overrides `db.postgresdb.password` from the
  chart values — n8n picks env vars over file config.

## Postgres storage (local on kube-master)

Postgres no longer goes through the `smb`/CIFS CSI. Its data sits on
kube-master's local disk at
`/srv/kubernetes/nfs/smb-csi/pvc-n8n-postgres-n8n-pg` (PGDATA =
`<that>/pgdata`) and is mounted straight in via `hostPath`. That directory
is the *same physical folder* the old `smb-pg` PVC served (kube-master is
also the Samba server, `192.168.99.44`), so the switch is in-place — no copy.

Consequences baked into the manifest:

- The pod is pinned to **kube-master** (`hostPath` is node-local).
- Strategy is **Recreate** — the old pod is fully gone before the new one
  starts, so two postmasters never touch one PGDATA.
- `hostPath.type: Directory` means a wrong/missing path makes the pod fail
  to schedule instead of silently initialising an empty database.

## Migrating to this layout (from the old inline-values / smb-pg app)

The live `n8n` Argo Application predates this repo state: its Helm values are
inline and hand-edited (plaintext secrets, smb-pg PVC). Registering the `ai`
app-of-apps makes git authoritative and flips it in one sync. Do the prep
first or you lose DB access (not the data — the data is safe via Retain +
hostPath-over-existing-files, but auth/credentials break):

```sh
# 0. Note the CURRENT encryption key (must be preserved):
kubectl -n n8n get application n8n -o yaml | grep -A1 N8N_ENCRYPTION_KEY
#   (currently: a_very_long_and_secure_key)

# 1. Create n8n-secrets with the EXISTING key (NOT a fresh one):
kubectl -n n8n create secret generic n8n-secrets \
  --from-literal=N8N_ENCRYPTION_KEY='a_very_long_and_secure_key'

# 2. Orphan the chart-rendered postgres-n8n Secret so prune won't delete it
#    (it already holds the right POSTGRES_DB/USER/PASSWORD):
kubectl -n n8n annotate secret postgres-n8n argocd.argoproj.io/tracking-id-

# 3. Register the app-of-apps (one-time, like the apps/mcp/platform roots):
kubectl apply -f ai/root.yaml

# 4. Watch the cutover. Recreate stops the old postgres, then the new
#    hostPath pod starts on kube-master against the existing pgdata.
kubectl -n n8n rollout status deploy/postgres-n8n
```

If postgres then refuses to start with *"data directory has invalid
permissions / wrong ownership"*, the on-disk files aren't owned by uid 999
(CIFS faked that via mount options; the real FS enforces it). One-time fix on
kube-master, data untouched:

```sh
chown -R 999:999 /srv/kubernetes/nfs/smb-csi/pvc-n8n-postgres-n8n-pg
chmod 700 /srv/kubernetes/nfs/smb-csi/pvc-n8n-postgres-n8n-pg/pgdata
```

### Why the data can't be lost here

- The old `smb-pg` PV is `reclaimPolicy: Retain` — pruning the PVC never
  deletes the share folder.
- The hostPath points at the existing, populated `pgdata`; Postgres only
  runs `initdb` on an *empty* dir, so it reuses the data as-is.
- Recreate + single replica means no concurrent writer.

### Rollback

Re-point the postgres `volumes[].data` back to the PVC
(`persistentVolumeClaim: { claimName: postgres-n8n-pg }`), restore the
`smb-pg` StorageClass + PVC manifests, drop the `nodeSelector`/`Recreate`.
The Retain'd PV `pvc-903f7fae-...` still binds the same share subdir.

The leftover Released PV `pvc-903f7fae-...` is harmless; delete it once the
local layout is confirmed (`kubectl delete pv pvc-903f7fae-...` removes only
metadata, share data stays).
