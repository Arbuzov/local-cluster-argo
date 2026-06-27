# wstunnel

This Application references a chart from a separate git repo —
[`Arbuzov/home-k8s-helm`](https://github.com/Arbuzov/home-k8s-helm),
path `arbuzov/networking/wstunnel`. Argo CD pulls the chart from there
at sync time.

Edits to the *chart* require a push to `home-k8s-helm` before
Argo CD picks them up. Edits to the *Application* (this file) take
effect once it's applied to the cluster.
