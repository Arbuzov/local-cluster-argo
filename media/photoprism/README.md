# photoprism

## Required out-of-band secrets

The ingress is fronted by HTTP basic-auth; create the htpasswd Secret
before first sync:

```sh
htpasswd -cb auth <user> <password>
kubectl -n photoprism create secret generic photoprism-basic-auth --from-file=auth
```

The OneDrive sidecar needs an authenticated rclone/onedrive config the
first time it starts — that's done interactively from inside the pod
(`onedrive --auth-files`) and persisted to the `onedrive-config` PVC.
No secret material lives in this manifest.

`PHOTOPRISM_PUBLIC: "true"` disables PhotoPrism's own login screen since
the ingress already does basic-auth.
