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

- This chart runs `docker.io/aw1cks/openconnect` (image kept at `latest` but
  `IfNotPresent` so it uses the copy already cached on the node). Its entrypoint
  dials the tunnel and adds `MASQUERADE` on `tun127`. An init container sets
  `ip_forward=1`, `rp_filter=0` (the forwarded path is asymmetric: `eth0` in,
  `tun127` out) and `FORWARD ACCEPT` so the pod forwards client traffic.
- **nftables backend (why both containers relink `/sbin/iptables`):** the image
  defaults `/sbin/iptables` → `xtables-legacy-multi`, but newer kernels ship no
  legacy xtables modules — e.g. `kube-worker-3` (6.18 rpt-rpi, Debian/RPi trixie)
  is nft-only, so legacy MASQUERADE/FORWARD died with `can't initialize iptables
  table 'nat'`. Both the init and the openconnect container relink
  `/sbin/iptables` → `xtables-nft-multi` (the openconnect container wraps the
  image's env-driven `/vpn/entrypoint.sh`; separate container rootfs, so each must
  redo the relink). The nft backend works on every node's kernel, so the gateway
  is no longer kernel-bound.
- **Placement** is operator-controlled by a node label, not a hard-coded host:
  `kubectl label node <node> vpn-gateway-capable=true`. All four nodes qualify
  (nft works everywhere); the label just lets you pin or exclude nodes by hand.
- **One session per corp user — never two.** `strategy: Recreate` avoids a second
  tunnel transiently overlapping the old one (the concentrator routes one session
  per user and blackholes both when two are live).
- The `vpn.extraArgs` server-cert `pin-sha256:` is the concentrator's public cert
  fingerprint (anti-MITM), **not** a secret — it stays inline in `values.yaml`.
- A **headless Service** (`clusterIP: None`) publishes the gateway pod IP via DNS
  (the port is nominal; only the A record matters). `publishNotReadyAddresses:
  true` keeps the IP resolvable during gateway startup/reconnect.
- Each client pod (gitlab/confluence MCP) runs a small **route-manager** sidecar
  (defined in the client's own manifest, not this chart) that resolves the Service
  and keeps `ip route replace <cidr> via <gateway-pod-ip>` in place. The next-hop
  is a **pod IP**, which is only on-link — and thus a usable route — when the
  client shares the gateway's node, so the clients use **podAffinity** to
  co-locate on whichever node the gateway lands on. Off-node the route fails with
  `Network unreachable` (and `onlink` does not help: ARP for the cross-node
  next-hop never resolves), so corp traffic never reaches the VPN.
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
