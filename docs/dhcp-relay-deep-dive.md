# DHCP Relay on a Stretched L2VNI — a Packet-Level Deep Dive

All findings below were verified live on this fabric with Wireshark captures.

## 1. BOOTP/DHCP header fields (RFC 951 / 2131)

| Field | Meaning | Filled by | Note |
|---|---|---|---|
| **ciaddr** | Client IP Address | client | only when it already has a valid lease (renew); 0.0.0.0 during initial DORA |
| **yiaddr** | Your IP Address | server | the address being offered |
| **siaddr** | Server IP Address | server | next-server (PXE) |
| **giaddr** | Gateway IP Address | relay | the relay's interface IP in the client subnet |
| **chaddr** | Client Hardware Address | client | 16-byte field, first 6 bytes = MAC on Ethernet |

## 2. Message addressing matrix

| Message | IP src → dst | giaddr |
|---|---|---|
| Discover/Request (client) | 0.0.0.0 → 255.255.255.255 | 0.0.0.0 |
| Relayed Discover (relay→server) | relay IP → server (unicast) | set by relay |
| **Offer/ACK to a relayed client** | **server → giaddr (unicast!)** | read by server |
| Offer/ACK, same-subnet client | server → broadcast / client | 0.0.0.0 |
| **Release** | **client IP → server, direct unicast** | not used — relay never involved |

> First step of any DHCP capture analysis: read option 53 (Message Type).
> A Release and an Offer on the same UDP ports have completely different addressing —
> we initially mistook a Release for a "reply bypassing the relay".

## 3. The relay is stateless — the state travels inside the packet

When the server's Offer arrives (addressed to the relay's own IP, UDP 67):

```
1. BOOTREPLY to my own address        → "this is a server reply"
2. read giaddr (172.16.40.254)        → matches my Vlan40 SVI → egress interface known
3. read flags (unicast / broadcast)   → delivery method known
4. build frame: dst MAC = chaddr, dst IP = yiaddr     ← NO ARP!
5. transmit into vlan 40. Nothing to remember.
```

The relay cannot ARP for yiaddr — the client hasn't accepted that address yet and would
not answer (chicken-and-egg). That is exactly why chaddr exists in the payload: the L2
destination is *read from the packet*, not resolved. Design pattern: *if you can't keep
state in the node, carry it in the packet.*

## 4. Why a single relay "works" on a stretched L2VNI — and why it's wrong

With vlan 40 stretched between Leaf-2 and Leaf-4 (same L2VNI) and the relay configured
**only on Leaf-2**, hosts behind *both* leaves get leases:

```
host-4 ──DISCOVER (broadcast)──► Leaf-4 ──BUM/ingress-replication──► Leaf-2
Leaf-2 SVI Vlan40 has the relay  → unicasts to server, giaddr = its SVI IP
server → Offer → giaddr (Leaf-2) → re-emitted into vlan 40 → floods back to host-4 ✓
```

It works as a *side effect of BUM flooding* — not by design. **Verified live:** with
Leaf-2's nve down, host-4 (whose own leaf was perfectly healthy) could no longer get a
lease. Single point of failure in the shape of "sometimes DHCP dies for the whole VLAN".

A second trap appears with the distributed anycast gateway: the SVI IP (= giaddr) is now
identical on all leaves, so the server's reply can be ECMP-delivered to a leaf that has
no relay configured. A packet addressed to the switch's *own* IP is not forwarded — it is
delivered locally, and with no UDP-67 listener it is silently dropped.

## 5. Correct design

On **every** leaf carrying the VLAN:

```
interface Vlan40
  ip dhcp relay address <server>
  ip dhcp relay source-interface loopback<N>    ! per-leaf unique, advertised in the VRF
ip dhcp relay information option                ! option 82
ip dhcp relay information option vpn            ! link-selection / VRF-aware
```

Unique giaddr (loopback) restores deterministic reply delivery; option 82 link-selection
tells the server which client subnet to allocate from since giaddr no longer lives in it.

## 6. Security: relay turns DHCP into punted control-plane traffic

Every Discover in a relay-enabled VLAN is punted to the CPU (rewriting giaddr is software
work). Axiom: **anything punted to the CPU is a DoS vector.** Layered defense:

| Layer | Mechanism | Effect |
|---|---|---|
| edge | `storm-control broadcast` | kills broadcast floods in the ASIC |
| edge | DHCP snooping (untrusted ports) | drops rogue OFFER/ACK from hosts |
| edge | `ip dhcp snooping verify mac-address` | frame src MAC must equal chaddr → defeats starvation tools that randomize chaddr only |
| last | CoPP (`copp profile strict`) | classes + policers: a DHCP flood cannot starve BGP/ARP (bulkhead pattern) |

Check: `show policy-map interface control-plane | inc dhcp|violate` — a rising `violated`
counter is your attack signal. `%COPP-2-COPP_NO_POLICY` in the log means no CoPP at all.

## 7. Reading captures in an EVPN fabric

The source MAC identifies *how* a packet arrived:

- host MAC → bridged (L2VNI)
- leaf **router MAC** (from the Type-5 `Router MAC` extcommunity) → routed (L3VNI, symmetric IRB)
- anycast gateway MAC (0000.1111.2222) → sent by a distributed gateway

A vlan40→vlan30 packet arriving with Leaf-1's RMAC as source was our packet-level proof
that symmetric IRB routing worked end-to-end. Repeated Discovers with the same
Transaction ID every ~3 s = a client nobody answers; find it before it finds you.
