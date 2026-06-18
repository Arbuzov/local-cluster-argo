# smb (samba server)

In-cluster SMB share at `192.168.99.44:445`. The CSI driver
(`smb.csi.k8s.io`) and most workloads consume these shares via the
`smbcreds` Secret in the `smb` namespace; that Secret carries the same
user/password as the one configured below.

## Sensitive values

The samba-helm chart renders `configmap.config.data.config.yml` as a
plain ConfigMap. The auth block used to carry the cleartext samba
password — but the `crazy-max/samba` image **interpolates `${VAR}`
placeholders in `config.yml` at runtime**, so the committed config now
uses `${SAMBA_USER}` / `${SAMBA_PASSWORD}` (in the `auth:` entry and the
`valid users` / `admin users` overrides for the `smb-csi` share). Those
env vars are injected from the existing `smbcreds` Secret via
`env[].valueFrom.secretKeyRef`, so the password never appears in git nor
in the rendered ConfigMap.

`smbcreds` (namespace `smb`) is the **single source of truth** for the
samba credentials: the same Secret is read by the `smb.csi.k8s.io`
StorageClasses, by every workload that mounts an SMB share, and now by
the samba server itself.

## Concrete steps for this repo

Make sure the `smbcreds` Secret exists in the `smb` namespace:

```sh
kubectl -n smb create secret generic smbcreds \
  --from-literal=username='<samba-user>' \
  --from-literal=password='<samba-password>'
```

Then apply the Application normally — there is no longer an
`application.local.yaml`:

```sh
kubectl apply -f storage/smb/application.yaml
```

To rotate: update `smbcreds` and `kubectl rollout restart` the samba
Deployment (the image expands `${VAR}` only at container start).
