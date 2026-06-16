# mcp-atlassian (Jira + Confluence)

Two sibling Applications:

- `application-jira.yaml`        → Jira MCP at `/mcp/jira`
- `application-confluence.yaml`  → Confluence MCP at `/mcp/confluence`

Confluence lives behind the corporate VPN, so its pod reaches it through the
shared `openconnect-gateway` (see [`networking/openconnect-gateway/`](../../networking/openconnect-gateway/)),
**not** a per-pod VPN sidecar — see **Confluence corp routing** below. Jira is
reachable directly and needs none of this.

## Confluence corp routing (shared VPN gateway)

`confluence.corp.example` (10.20.0.181, in 10.20.0.0/24) is only reachable
over the corp VPN. Instead of running its own OpenConnect tunnel, the Confluence
pod routes that subnet through the shared **`openconnect-gateway`** pod — one corp
session for all clients (per-pod tunnels all logged in as the same user, and the
concentrator routes only one session per user, so two live tunnels blackholed each
other).

- **`route-manager` sidecar:** resolves the gateway's headless Service and keeps
  `ip route replace 10.20.0.0/24 via <gateway-pod-ip>` in place, so the route
  self-repairs if the gateway pod IP changes. `confluence.corp.example` still
  resolves via cluster DNS; only the L3 path moves to the gateway.
- **`podAffinity` (co-location):** the route's next-hop is the gateway **pod IP**,
  which is only on-link — and thus a usable route — when this pod shares the
  gateway's node. So the pod uses `podAffinity` to follow the gateway onto whichever
  node it lands on (the gateway is label-gated, not pinned to a host). Off-node the
  route fails with `Network unreachable` (and `onlink` doesn't help — ARP for the
  cross-node next-hop never resolves) and corp traffic never reaches the VPN.

VPN credentials, cert pinning, node placement, and gateway internals are documented
in [`networking/openconnect-gateway/chart/README.md`](../../networking/openconnect-gateway/chart/README.md).

## Required out-of-band secrets

Three Secrets in the `mcp` namespace — create before first sync:

```sh
# Jira API token + username (Atlassian PAT, NOT the password)
kubectl -n mcp create secret generic mcp-atlassian-jira-credentials \
  --from-literal=JIRA_USERNAME='<your-atlassian-username>' \
  --from-literal=JIRA_API_TOKEN='<your-atlassian-api-token>'

# Confluence — same shape
kubectl -n mcp create secret generic mcp-atlassian-confluence-credentials \
  --from-literal=CONFLUENCE_USERNAME='<your-atlassian-username>' \
  --from-literal=CONFLUENCE_API_TOKEN='<your-atlassian-api-token>'

# OpenConnect VPN credentials, used by the shared openconnect-gateway (its
# chart maps USERNAME/PASSWORD onto the USER/PASS env vars the image expects)
kubectl -n mcp create secret generic mcp-atlassian-vpn-credentials \
  --from-literal=USERNAME='<your-vpn-username>' \
  --from-literal=PASSWORD='<your-vpn-password>'
```

The Atlassian Cloud "API token" is generated at
https://id.atlassian.com/manage-profile/security/api-tokens — it is not
the same as your account password.

## Basic-auth on the ingress

Both ingresses front the MCP servers with HTTP basic-auth via the
shared `mcp-basic-auth` Secret (htpasswd format) in the `mcp` namespace.
That Secret is also created out-of-band:

```sh
htpasswd -cb auth <user> <password>
kubectl -n mcp create secret generic mcp-basic-auth --from-file=auth
```
