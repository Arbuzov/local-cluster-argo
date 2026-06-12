# mcp-gitlab

GitLab MCP server ([iwakitakuma/gitlab-mcp](https://hub.docker.com/r/iwakitakuma/gitlab-mcp))
at `/mcp/gitlab`, talking to the corp GitLab at `gitlab.corp.example`.

History: this service was originally deployed by a hand-created Argo CD
Application (chart `mcp-gitlab` 0.3.1) that was later deleted without its
finalizer, leaving the Deployment/Service/Ingress orphaned in the cluster.
This Application re-adopts those resources (same names and selectors) and
moves the GitLab token out of plain pod env into a Secret.

## VPN sidecar + hostAliases

GitLab lives behind the corp VPN, so the pod runs the same OpenConnect
sidecar as the confluence app (`mcp/atlassian/application-confluence.yaml`),
reusing the shared `mcp-atlassian-vpn-credentials` Secret.

Unlike `*.corp.example` names, `gitlab.corp.example` exists **only in corp
DNS** (corp-dns.example, 10.20.30.21), which neither cluster DNS nor
the sidecar's resolver can reach — the chart's `hostAliases` value pins it
to 10.20.0.156 (`corp-host.example`) in `/etc/hosts` instead. If GitLab
moves hosts, update that IP.

> **Known issue (2026-06-12):** the `VPN_GROUP_A` tunnel group on
> `vpn.corp.example` authenticates and establishes, but passes no TCP —
> not to corp subnets, not to the internet (only UDP/53 to public resolvers
> gets through). The confluence sidecar has the same problem, so until the
> ASA group policy / vpn-filter is fixed corp-side, both MCP servers accept
> MCP sessions but fail actual API calls. The other ASA groups are no help
> for a sidecar: `DUO` needs MFA + client cert (it's what AnyConnect on the
> workstation uses, against `vpn.corp.example`), `VPN_GROUP_C` rejects
> these credentials, `VPN_GROUP_D` wants a client certificate.

## Required out-of-band secrets

Two are shared with the atlassian apps (`mcp-atlassian-vpn-credentials`,
`mcp-basic-auth` — see `mcp/atlassian/README.md`). One is specific to this
app:

```sh
# GitLab personal access token (api scope)
kubectl -n mcp create secret generic mcp-gitlab-credentials \
  --from-literal=GITLAB_PERSONAL_ACCESS_TOKEN='<your-gitlab-pat>'
```
