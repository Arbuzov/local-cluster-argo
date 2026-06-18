# wstunnel

This Application references a chart from a separate git repo —
[`Arbuzov/home-cluster-helm`](https://github.com/Arbuzov/home-cluster-helm),
path `arbuzov/networking/wstunnel`. Argo CD pulls the chart from there
at sync time.

Edits to the *chart* require a push to `home-cluster-helm` before
Argo CD picks them up. Edits to the *Application* (this file) take
effect once it's applied to the cluster.
