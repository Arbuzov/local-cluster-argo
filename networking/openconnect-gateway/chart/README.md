# openconnect-gateway

A single shared **OpenConnect (Cisco AnyConnect) VPN gateway**. One pod holds
**one** session to the corp concentrator (`vpn.corp.example`, group
`VPN_GROUP_A`) and NATs forwarded traffic onto its `tun127`. Selected client
pods route the corp subnet(s) through it instead of each running their own
OpenConnect sidecar.

## Why

The gitlab and confluence MCP pods used to each run an OpenConnect sidecar that
logged in as the **same corp user**. The concentrator routes only **one** session
per user — with two tunnels up, **both get blackholed**. Centralizing to one
session removes the conflict by construction.

## How it works

- This chart runs `docker.io/aw1cks/openconnect` (its entrypoint dials the tunnel
  and adds `MASQUERADE` on `tun127`). An init container sets `ip_forward=1`,
  `rp_filter=0`, and `FORWARD ACCEPT` so the pod forwards client traffic.
- **Pinned to `kube-master`** (`nodeSelector`): the RPi worker kernels lack the
  `iptable_nat` module (`can't initialize iptables table 'nat'`), which
  MASQUERADE needs.
- A **headless Service** publishes the gateway pod IP via DNS.
- Each client pod runs a small **route-manager** sidecar (defined in the client's
  own manifest, not this chart) that resolves the Service and keeps
  `ip route replace <cidr> via <gateway-pod-ip>` in place. The client carries the
  route so it self-repairs if the gateway pod IP changes.
- A liveness probe (`tun127` has an address) restarts the gateway if the tunnel
  drops (the image runs openconnect with `--background`, so the process can die
  while the container lingers).

## Required out-of-band secret

Reuses the existing `mcp-atlassian-vpn-credentials` Secret (keys
`USERNAME`/`PASSWORD`) in the **destination namespace** (the Application deploys
into `mcp`). See `mcp/atlassian/README.md` in `local-cluster-argo`.

## Corp subnets

`values.yaml: routedCidrs` documents the corp subnets this gateway fronts
(default `10.20.0.0/24`, covering `gitlab.corp.example`=10.20.0.156 and
`confluence.corp.example`=10.20.0.181). The actual route is installed by the
client route-manager sidecars; extend both when adding corp subnets.
