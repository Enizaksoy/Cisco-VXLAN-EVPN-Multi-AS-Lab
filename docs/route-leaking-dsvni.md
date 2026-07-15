# Route Leaking, Centralized Internet Access & Downstream VNI (DSVNI)

Built and verified live on this fabric: a shared **Internet VRF exists on one border leaf
only**, tenants reach it through RT-based route leaking, and the data plane works because
NX-OS honors **downstream-assigned VNIs** — proven by the `(Asymmetric)` flag in the RIB.

## 1. Design

```
                       Internet-Router (IOL)
                            │ vlan 511 · 172.16.111.0/30
                      ┌─────┴─────┐
                      │  Leaf-4   │   border leaf
                      │  VRF Internet · L3VNI 50010 (exists ONLY here)
                      └───────────┘
                            │ RT leaking
          ┌─────────────────┼─────────────────┐
     Customer-A        Customer-B        Customer-C
     (L3VNI 50000)     (L3VNI 50001)     (L3VNI 50002)
```

**Internet VRF (border leaf only):**

```
vrf context Internet
  vni 50010
  ip route 0.0.0.0/0 172.16.111.2            ! toward the internet router
  rd auto
  address-family ipv4 unicast
    route-target import 65000:5000 evpn      ! return paths: one import per tenant
    route-target import 65000:5001 evpn
    route-target import 65000:5002 evpn
    route-target export 65000:5010
    route-target export 65000:5010 evpn
router bgp 65004
  vrf Internet
    address-family ipv4 unicast
      network 0.0.0.0/0
      redistribute direct route-map all
```

**Each tenant VRF (all leaves):**

```
vrf context Customer-A
  address-family ipv4 unicast
    route-target import 65000:5010 evpn      ! receive the leaked default + 111.0/30
```

## 2. The DSVNI moment

Tenant leaves import routes whose **label is 50010 — a VNI they do not have**. Classic
(symmetric-only) VXLAN would be stuck here. NX-OS instead encapsulates with the VNI
**advertised by the egress VTEP** (downstream assignment, exactly like MPLS VPN labels)
and marks such routes explicitly:

```
Leaf-4# show ip route vrf Internet
172.16.10.0/24 ... segid: 50000 (Asymmetric) ... encap: VXLAN
172.16.30.0/24 ... segid: 50001 (Asymmetric) ... encap: VXLAN
172.16.50.0/24 ... segid: 50002 (Asymmetric) ... encap: VXLAN
```

The Internet VRF (local VNI 50010) forwards to each tenant using *that tenant's* L3VNI —
three different downstream VNIs from a single VRF. Confirmed working on Nexus 9000v
10.5(3) in CML.

> **Do not confuse with "asymmetric IRB"** (RFC 9135). Asymmetric IRB is an *IRB
> forwarding model* with no L3VNI at all: the ingress leaf routes straight into the
> destination's **L2VNI** and the egress only bridges — forward and return traffic ride
> different VNIs, and every leaf must carry every VLAN. NX-OS implements symmetric IRB
> only. DSVNI lives *inside* symmetric IRB: what is asymmetric is not the two directions
> of a flow but the **VNI numbers of two VRFs** exchanging leaked routes.

## 3. Rules this lab proved the hard way

**a) Every leak direction is an independent import.** With only `import 65000:5001`
configured on the Internet VRF, Customer-B worked while Customer-A pings died silently on
the return path — requests reached the internet router (its own routing was fine), replies
arrived at the border leaf and were dropped for lack of a route. Rule of thumb for
hub-and-spoke leaking: the hub carries **one import per tenant (N lines)**, every tenant
carries **one import of the hub RT**. Count the lines.

**b) RTs stick at origination — leaking does not cascade.** Customer-B's routes imported
into the Internet VRF keep their original `RT:65000:5001`; the hub's export RT is *not*
re-applied to them. Customer-A (importing only 5010) therefore never learns Customer-B's
prefixes through the hub. Tenant isolation survives RT leaking — by protocol design, not
by luck.

**c) `network 0.0.0.0/0` needs a real RIB route.** The statement is accepted and shows in
the BGP table as `l0.0.0.0/0` — but without `*>` (valid/best) until a static
`ip route 0.0.0.0/0 <next-hop>` (or an eBGP-learned default) exists in the VRF. Until
then, nothing is advertised. Reading the status column beats reading the prefix column.

**d) A leaked default re-opens tenant-to-tenant transit.** The moment the default became
valid, Customer-A could reach Customer-B — hairpinning through the border leaf
(A → default → Internet VRF → B's leaked route). Reachability is now decided by routing;
*permission* must be enforced by policy (ACL/firewall at the hub). Isolation moved from
the control plane to the policy plane the instant both tenants met in one VRF holding a
covering route.

## 4. Verification cheat-sheet

```
show bgp vrf Internet ipv4 unicast            ! *> on 0.0.0.0/0? redistributed connected there?
show bgp l2vpn evpn 0.0.0.0                   ! Type-5 default, RT:65000:5010, label 50010
show ip route vrf Internet                    ! per-tenant (Asymmetric) segids
show bgp l2vpn evpn 172.16.111.0              ! leaked prefix imported into tenant VRFs
show ip route vrf Customer-A <target>         ! ALWAYS query the specific destination —
                                              ! a busy table is not the same as the right route
```
