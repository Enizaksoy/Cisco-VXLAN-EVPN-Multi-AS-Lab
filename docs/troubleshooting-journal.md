# Troubleshooting Journal

Real faults hit while building this fabric, in the order they appeared — each with the
evidence that identified it and the lesson it left behind.

## 1. Type-5 routes generated but never advertised

**Symptom:** VRF prefixes visible in the leaf's own BGP table; spine shows `Type-5: 0`
from every neighbor.

**Evidence:**
```
Path type: local, path is invalid(EVI down), no labeled nexthop
nve1  50001  Down  L3 [Customer-B]
```

**Root cause:** L3VNIs had `vrf context ... vni` and `member vni ... associate-vrf`, but
no core VLAN (`vlan 500 / vn-segment 50000`) and no `ip forward` SVI. Two of three legs
configured; the L3VNI had no bridge domain to ride.

**Lesson:** an *invalid* BGP path is never advertised — the control plane refuses to
promise what the data plane can't deliver. Diagnosis order for "route missing remotely":
source route flags first (`invalid/valid/best`), then `advertised-routes`, then the peer.

## 2. Copy-paste route-target typo

**Symptom (latent):** two tenants' L2VNIs importing each other's routes.

```
evpn
  vni 50 l2
    route-target import 65000:30     ← should be 65000:50
    route-target export 65000:30
```

Symmetric on two leaves, so vlan50 "worked" — while quietly cross-importing VNI 30.
**Lesson:** RTs are the tenant isolation boundary. Derive them mechanically from the VNI
(`65000:<VNI>`) and diff configs against the scheme; or use `route-target auto` where the
AS layout allows it.

## 3. vlan 503 instead of vlan 502

**Symptom:** one L3VNI still down after "everything" was fixed.

**Evidence:**
```
Vlan502   forward-enabled   protocol-down/link-down/admin-up
vlan 503
  vn-segment 50003          ← intended: vlan 502 / vn-segment 50002
```

**Lesson:** `protocol-down/link-down/admin-up` on an SVI is the fingerprint of "the SVI
object exists but the VLAN doesn't (or has no vn-segment)". VLAN and SVI are separate
objects on NX-OS — verify both.

## 4. %ARP-2-DUP_SRC_IP from an nve-peer

**Symptom:**
```
%ARP-2-DUP_SRC_IP: arp Source address of packet received from 5269.d61a.1b08
on Vlan20(nve-peer1) is duplicate of local, 172.16.20.254
```

**Root cause:** the same SVI IP on two leaves *without* `fabric forwarding mode
anycast-gateway` — each leaf claimed the shared IP with its own system MAC
(5269.d61a.1b08 is the other leaf's RMAC; the source being an RMAC, not a host MAC, is
what pins the diagnosis).

**Side effect worth knowing:** without anycast mode, HMM ignores the SVI's ARP entries →
no MAC-IP Type-2 routes, no /32 host routes, remote ARP suppression impossible.

## 5. DHCP relay on one leaf only

**Symptom (none — that's the point):** hosts on all leaves got leases with the relay on a
single leaf, because BUM flooding carried the Discovers across the stretched L2VNI.

**Verified failure:** with that leaf's nve down, hosts behind *other, healthy* leaves
could no longer obtain leases.

**Lesson:** "working" ≠ "working by design". Relay must be configured on every leaf
carrying the VLAN, with a per-leaf unique `source-interface` (see the DHCP deep dive).

## 6. Feature-asymmetry (from the L2 phase)

`feature dhcp` / relay enabled on some leaves only makes DHCP broadcasts die at the VTEP
boundary *without any counters* — ARP floods fine, DHCP doesn't (sup-redirect grabs
UDP 67/68). Fabric features must be symmetric: on every leaf or on none.

## 7. One-way route leaking (Internet VRF phase)

**Symptom:** tenant host's pings reached the internet router (request *and* reply visible
on the router), host saw timeouts.

**Root cause:** the Internet VRF imported only one tenant's RT (`65000:5001`); the tested
tenant's return route (172.16.10.0/24, RT 65000:5000) was never imported — replies died
at the border leaf. Every leak direction is an independent import: hub needs one import
line per tenant.

**Bonus find while diagnosing:** `redistribute direct route-map all` referenced a
route-map that didn't exist yet — NX-OS then redistributes *nothing*, silently. The
connected 172.16.111.0/30 never became a Type-5 until the map was defined.

## 8. `network 0.0.0.0/0` without a RIB default

**Symptom:** default route configured under the VRF's BGP AF, never advertised.

**Evidence:** `l0.0.0.0/0` present in `show bgp vrf Internet ipv4` but with **no `*>`** —
the network statement only picks up a route that already exists in the RIB. Adding
`ip route 0.0.0.0/0 172.16.111.2` under the VRF context made it valid, advertised — and
immediately opened tenant-to-tenant transit through the border leaf (see the route-leaking
doc, rule d).

## Meta-lessons

- Small integers are the enemy: three of six faults were single-digit typos (30↔50,
  502↔503, 50002↔50003). Mechanical naming schemes + config diffing beat careful typing.
- One `show nve vni` is worth an hour of BGP debugging: L2/L3 VNI state, VRF binding and
  `[--]` orphan markers all live there.
- Verify with packets when possible: the DUP_SRC_IP MAC, the Offer's giaddr destination
  and the RMAC on routed frames each settled an argument that configs alone could not.
