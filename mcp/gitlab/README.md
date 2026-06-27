# mcp-gitlab

GitLab MCP server ([zereight050/gitlab-mcp](https://hub.docker.com/r/zereight050/gitlab-mcp),
v2.1.20) at `/mcp/gitlab` on `dev.whitediver.keenetic.link`, talking to a
self-hosted GitLab behind a corporate VPN over the shared
[`openconnect-gateway`](../../networking/openconnect-gateway/).

> **Employer-specific routing is not in git.** The real GitLab URL, the
> VPN subnet, and the `/etc/hosts` pin live only in out-of-band Secrets and a
> local overlay (this app is held back from the app-of-apps and applied
> push-based — see **Out-of-band routing** below). The committed manifest
> carries placeholders only.

## Image & auth (2.1.x: remote-auth + stateless + token-injector)

The maintained image is `zereight050/gitlab-mcp` (the older `iwakitakuma/gitlab-mcp`
froze at 2.0.19). On 2.1.x a static server-side `GITLAB_PERSONAL_ACCESS_TOKEN` over
`STREAMABLE_HTTP` is **rejected at startup** — the server requires
`REMOTE_AUTHORIZATION=true` (token per request via a `Private-Token` header) or full
OAuth. We use `REMOTE_AUTHORIZATION` and supply the token from inside the pod, so
clients stay unchanged:

- **`token-injector` sidecar** (`nginx`) owns the chart's service port (3002 — the
  named `http` port the Service and probes target) and reverse-proxies to the app on
  `127.0.0.1:3003` (`PORT=3003`), adding `Private-Token` read from the
  `mcp-gitlab-credentials` Secret. Injection happens below the Service, so every
  client and route works with no change and the PAT never lands in git.
  `proxy_buffering off` + HTTP/1.1 keep the streamable-HTTP/SSE responses flowing.
- **`OAUTH_STATELESS_MODE=true`** seals the session into the `Mcp-Session-Id`
  (`v1.sid.…`) with the stable `OAUTH_STATELESS_SECRET`, so sessions **survive pod
  restarts** — clients don't have to reconnect/re-initialise. (Without it a restart
  drops the in-memory session and clients hang on the stale id until a manual
  `/mcp` reconnect.)
- **`HOST=0.0.0.0`** is required: 2.1.x defaults `HOST` to `127.0.0.1` (since 2.0.21),
  which would make the app unreachable from the injector and the probes.

The app container holds no GitLab token; `mcp-gitlab-credentials` is read only by the
injector. To roll back to the 2.0.x static-PAT model, `git revert` the bump — Argo
returns to `iwakitakuma/gitlab-mcp:2.0.19` (both Secrets are left intact).

## Corp routing (shared VPN gateway)

GitLab lives behind a corporate VPN. Like the confluence app, this pod does **not**
run its own OpenConnect tunnel — it routes the corp subnet through the shared
[`openconnect-gateway`](../../networking/openconnect-gateway/) pod (reusing the
`mcp-atlassian-vpn-credentials` Secret):

- a **`route-manager`** sidecar keeps `ip route replace <corp-subnet> via
  <gateway-pod-ip>` pointed at the gateway's headless Service. The subnet is
  injected as `CORP_CIDR` from the `mcp-corp-routing` Secret, so it never lands
  in git;
- **`podAffinity`** co-locates this pod with the gateway, because the route's
  pod-IP next-hop is only on-link when both share a node — off-node it fails with
  `Network unreachable` and corp traffic never reaches the VPN (previously it
  worked only because the scheduler happened to co-locate them).

The self-hosted GitLab exists **only in corp DNS**, which neither cluster DNS nor
the sidecar's resolver can reach — so the pod pins the hostname to its corp-VPN IP
via `hostAliases`. That mapping is employer-specific, so it lives in the local
overlay (below), not in git.

> **Lesson learned — pick the right tunnel group:** the corp VPN concentrator
> exposed several tunnel groups; only one actually passed traffic. The others
> authenticated and established, then blackholed all TCP (only public-resolver
> UDP/53 got through), which kept both this and the confluence MCP dead until the
> sidecars were pointed at the working group. Others required MFA + a client cert
> (what the desktop AnyConnect uses) or rejected these credentials outright.

## Out-of-band routing (held back, push-based)

Because the corp GitLab URL and `hostAliases` are employer-specific, this app is
**excluded** from the `mcp` app-of-apps and applied directly, like the
`networking/` apps:

```sh
kubectl apply -f mcp/gitlab/application.local.yaml
```

`application.local.yaml` is your real manifest (gitignored): it adds the
`hostAliases` block pinning the corp hostname to its VPN IP. The committed
`application.yaml` is a placeholder template with `hostAliases: []`.

## Required out-of-band secrets

Two are shared with the atlassian apps (`mcp-atlassian-vpn-credentials`,
`mcp-basic-auth` — see `mcp/atlassian/README.md`). The rest are specific to this
app:

```sh
# GitLab API URL + personal access token (api scope). The URL is kept out of git;
# the token is read by the token-injector sidecar.
kubectl -n mcp create secret generic mcp-gitlab-credentials \
  --from-literal=GITLAB_API_URL='https://<corp-gitlab>/api/v4' \
  --from-literal=GITLAB_PERSONAL_ACCESS_TOKEN='<your-gitlab-pat>'

# Corp VPN subnet routed via the shared gateway (shared with the confluence app)
kubectl -n mcp create secret generic mcp-corp-routing \
  --from-literal=CIDR='<corp-subnet-cidr>'

# Stateless session-sealing key (base64url, >=32 bytes). Must stay STABLE across
# restarts — rotating it forces every client to re-initialise once:
kubectl -n mcp create secret generic mcp-gitlab-stateless \
  --from-literal=OAUTH_STATELESS_SECRET="$(openssl rand -base64 32 | tr '+/' '-_' | tr -d '=')"
```
