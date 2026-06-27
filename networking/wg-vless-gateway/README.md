# wg-vless-gateway

Unlike the rest of this repo, this Application references a chart that
lives in **another git repository** —
[`Arbuzov/home-k8s-helm`](https://github.com/Arbuzov/home-k8s-helm),
path `arbuzov/networking/wg-vless-gateway`. Argo CD pulls the chart from
there at sync time.

That means edits to the *chart* require a push to `home-k8s-helm`
before Argo CD picks them up. Edits to the *Application* (this file)
take effect once it's applied to the cluster.

No inline credentials live in this Application; any secrets the chart
needs are managed inside `home-k8s-helm` (see that repo's chart
`values.yaml` and templates).
