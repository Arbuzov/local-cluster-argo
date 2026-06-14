# arc-operator (actions-runner-controller)

Self-hosted GitHub Actions runner controller.

## Required out-of-band secret

The controller authenticates to GitHub with a Personal Access Token (PAT).
Create the Secret manually — do **not** put the token in git.

```sh
kubectl -n arc-system create secret generic controller-manager \
  --from-literal=github_token='ghp_xxxxxxxxxxxxxxxxxxxx'
```

The token needs the `repo` and `admin:org` scopes (per upstream docs).

The Application disables the chart's `authSecret.create` path so it
references this Secret instead of inlining the token in `helm.values`.
