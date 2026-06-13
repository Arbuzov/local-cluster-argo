# mcp-gitlab

GitLab MCP server ([zereight050/gitlab-mcp](https://hub.docker.com/r/zereight050/gitlab-mcp),
v2.1.20) at `/mcp/gitlab` on `dev.whitediver.keenetic.link`, talking to the corp
GitLab at `gitlab.asatnet.net` over the VPN sidecar.

> **Not** the same as `gitlab.mcp.sandbox.asatnet.net` (192.168.67.56) — that is a
> separate gitlab-mcp instance in the asatnet sandbox, unrelated to this deployment.
> This one is reached at `https://dev.whitediver.keenetic.link/mcp/gitlab`.

History: this service was originally deployed by a hand-created Argo CD
Application (chart `mcp-gitlab` 0.3.1) that was later deleted without its
finalizer, leaving the Deployment/Service/Ingress orphaned in the cluster.
This Application re-adopts those resources (same names and selectors).

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

## VPN sidecar + hostAliases

GitLab lives behind the corp VPN, so the pod runs the same OpenConnect
sidecar as the confluence app (`mcp/atlassian/application-confluence.yaml`),
reusing the shared `mcp-atlassian-vpn-credentials` Secret.

Unlike `*.spacebridge.com` names, `gitlab.asatnet.net` exists **only in corp
DNS** (DCMASTER.asatnet.net, 192.168.30.21), which neither cluster DNS nor
the sidecar's resolver can reach — the chart's `hostAliases` value pins it
to 192.168.8.156 (`zeus-1.asatnet.net`) in `/etc/hosts` instead. If GitLab
moves hosts, update that IP.

> **Gotcha — two concentrators (2026-06-12):** the `VPN_THROUGHT` tunnel
> group exists on both `asa.spacebridge.com` (107.161.14.242) and
> `asa1.spacebridge.com` (.243), but only **asa1** passes traffic. On asa
> the tunnel authenticates and establishes, then blackholes all TCP (only
> UDP/53 to public resolvers gets through) — that kept both this and the
> confluence MCP dead until the sidecars were pointed at asa1. The other
> groups are no help for a sidecar: `DUO` needs MFA + client cert (what
> AnyConnect on the workstation uses), `VPN_LOCAL` rejects these
> credentials, `VPN_RADIUS` wants a client certificate.

## Required out-of-band secrets

Two are shared with the atlassian apps (`mcp-atlassian-vpn-credentials`,
`mcp-basic-auth` — see `mcp/atlassian/README.md`). Two are specific to this app:

```sh
# GitLab personal access token (api scope) — read by the token-injector sidecar
kubectl -n mcp create secret generic mcp-gitlab-credentials \
  --from-literal=GITLAB_PERSONAL_ACCESS_TOKEN='<your-gitlab-pat>'

# Stateless session-sealing key (base64url, >=32 bytes). Must stay STABLE across
# restarts — rotating it forces every client to re-initialise once:
kubectl -n mcp create secret generic mcp-gitlab-stateless \
  --from-literal=OAUTH_STATELESS_SECRET="$(openssl rand -base64 32 | tr '+/' '-_' | tr -d '=')"
```
