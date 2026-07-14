# Design Notes — Multi-AS eBGP EVPN

## Why every leaf gets its own AS (and spines share one)

BGP's AS_PATH loop prevention is used as a free, config-less topology guard:

- **Spines share AS 65000** → a route that tries to transit leaf→spine→leaf→spine is rejected
  by the second spine (its own AS already in AS_PATH). Leaves can never become transit.
  Valley-free routing enforced by numbering alone.
- **Each leaf in its own AS** → a leaf rejects its own routes coming back (own AS in path);
  AS_PATH instantly shows which leaf originated any route.
- **ECMP stays simple**: both spines present identical AS_PATHs (`65000 6500x`), so BGP
  multipath works without `bestpath as-path multipath-relax`.

The rule of thumb: *devices that must not transit each other's traffic share an AS;
layers that must exchange routes use different AS numbers.*

## The spine is a VPNv4-style route reflector — without VRFs

The `l2vpn evpn` address family is the exact analog of `vpnv4` in MPLS L3VPN:

| MPLS L3VPN | VXLAN EVPN | Purpose |
|---|---|---|
| `address-family vpnv4` | `address-family l2vpn evpn` | VPN route transport |
| RD | RD | NLRI uniqueness across tenants |
| RT extcommunity | RT extcommunity | VRF import/export tagging |
| VPN label | VNI | data-plane tenant selector |
| PE | leaf (VTEP) | encap/decap, VRFs live here |
| RR (no VRFs) | spine (no VRFs) | blind route transport |
| P router | spine data plane | routes outer header only |

EVPN routes live in the spine's *default VRF* BGP table as opaque NLRI. The spine never
imports them — which is why `retain route-target all` is mandatory: the default BGP
behavior drops VPN routes whose RTs no local VRF imports.

`next-hop unchanged` is required because eBGP rewrites next-hop per hop, and in EVPN the
next-hop *is* the VXLAN tunnel destination (the remote VTEP loopback). The spine is in the
data path only as an IP router of the outer header — it is not a VTEP and cannot decap.

## Symmetric IRB — one L3VNI per tenant

Each tenant VRF needs exactly one L3VNI, and each L3VNI needs three legs on every leaf:

```
vrf context Customer-A → vni 50000          (intent)
interface nve1 → member vni 50000 associate-vrf   (tunnel membership)
vlan 500 → vn-segment 50000  +  SVI Vlan500 (vrf member + ip forward)   (the bridge domain it rides)
```

If the third leg is missing, `show nve vni` shows the L3VNI **Down** and every Type-5
route stays `invalid(EVI down)` — present in the local BGP table, advertised to no one.
BGP will not promise what the data plane cannot deliver.

Routed traffic between tenants' subnets uses the L3VNI (two labels in the Type-2:
`Received label <L2VNI> <L3VNI>`); same-subnet traffic bridges over the L2VNI.
The `Router MAC` extended community carries each leaf's system RMAC — the inner
Ethernet destination for routed VXLAN packets (it is *not* the anycast gateway MAC).

## Distributed anycast gateway

Every leaf hosting a tenant SVI runs:

```
fabric forwarding anycast-gateway-mac 0000.1111.2222     ! global, identical fabric-wide
interface Vlan10
  vrf member Customer-A
  ip address 172.16.10.254/24                            ! identical IP on every leaf
  fabric forwarding mode anycast-gateway
```

Without the per-SVI `fabric forwarding mode anycast-gateway` line, two leaves sharing the
SVI IP claim it with *different system MACs* → `%ARP-2-DUP_SRC_IP` from nve-peers, and
HMM (Host Mobility Manager) never injects ARP entries into L2RIB, so no MAC-IP Type-2
routes are generated. The ARP→HMM→L2RIB→BGP chain only runs on anycast-mode SVIs.

## RD vs RT (the one-liner that prevents hours of confusion)

- **RT** = VPN membership. Must *match* across leaves. Manual scheme here (`65000:<VNI>`)
  because `auto` derives `ASN:VNI`, which never matches when every leaf has a different ASN.
- **RD** = NLRI uniqueness. Must *differ* per leaf (`rd auto` = `router-id:VNI-offset`),
  so the same MAC advertised by two VTEPs stays two distinct paths on the spine
  (a prerequisite for ESI aliasing and fast mass-withdrawal).
