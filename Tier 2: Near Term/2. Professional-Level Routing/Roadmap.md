Claude-generated roadmap:

The routing material below is a mix of exam content and practice that no Cisco exam covers. CCNP Enterprise requires two exams: the core exam ENCOR (350-401) and one concentration exam. The concentration exam that matches your topic list is ENARSI (300-410). ENARSI is a 90-minute exam on implementing and troubleshooting advanced routing, VPN services, infrastructure security, infrastructure services, and automation. Its blueprint lists OSPF v2/v3, EIGRP, internal and external BGP, redistribution, route maps, and a describe-level BFD objective.

The retrieved blueprint text does not list IS-IS, RPKI route origin validation (ROV), IRR, MANRS, or PeeringDB. The BGP objective was truncated in the retrieved text, so route reflectors, communities, and ECMP may or may not appear as separate items. I did not retrieve the ENCOR blueprint, which covers topics outside routing. Verify both blueprints on cisco.com before you register, because Cisco states that blueprints can change without notice.

## What the credential does and does not validate

| Topic | In the retrieved ENARSI blueprint |
|---|---|
| Multi-area OSPF (v2 and v3) | Yes |
| EIGRP (not in your list) | Yes |
| Redistribution, route maps, filtering | Yes |
| eBGP and iBGP | Yes |
| BFD | Describe level only |
| IPv6 routing | OSPFv3 and EIGRP IPv6; BGP for IPv6 not confirmed |
| IS-IS | No |
| RPKI, IRR, MANRS, PeeringDB | No |

Phases 6 and 7 below are internet-edge and service provider practice. A CCNP Enterprise certificate does not prove that skill. Phase 2 (EIGRP) is on the exam but absent from your list.

## Lab platform

- **FRRouting (FRR) in containerlab on Linux.** FRR implements OSPFv2/v3, IS-IS, BGP, BFD, and RPKI. RPKI loads as a `bgpd` module (`-M rpki`), so confirm that your build includes it. FRR syntax resembles Cisco IOS XE syntax but is not identical.
- **A Cisco IOS XE virtual router (for example, in Cisco Modeling Labs).** Use it for Phases 1 to 5 so that you practice the syntax and `show` output that the exam tests. IOS XE images need much more memory than FRR containers.
- **Do not use Packet Tracer** for these labs. Packet Tracer simulates a subset of IOS behavior.
- Use the private ASN ranges from RFC 6996 for all labs. Use documentation prefixes from RFC 5737 (IPv4) and RFC 3849 (IPv6).

## Schedule

Durations are my estimates at 8 hours per week, not Cisco figures.

| Phase | Subject | Weeks |
|---|---|---|
| 0 | Baseline check | 1 |
| 1 | Multi-area OSPF | 5 |
| 2 | EIGRP | 2 |
| 3 | Route control and redistribution | 3 |
| 4 | BGP | 10 |
| 5 | IPv6 routing | 3 |
| 6 | IS-IS | 4 |
| 7 | RPKI, IRR, PeeringDB, MANRS | 5 |
| 8 | Capstone | 5 |

The total is about 38 weeks, before ENCOR preparation, which this roadmap does not cover.

## Phase 0: baseline check

You must be able to do the following without a reference: VLSM subnetting, longest-prefix match, administrative distance (AD) comparison, recursive next-hop lookup, and single-area OSPFv2 adjacency.

**Project 0.** Build six routers with static routes and one OSPF area. Predict the forwarding path for 10 source-destination pairs from the routing tables only. Verify each prediction with `traceroute`. Pass criterion: 10 of 10 correct. If you miss any, repeat the phase before continuing.

## Phase 1: multi-area OSPF

**Topics**
- LSA types 1, 2, 3, 4, 5, and 7, and which area types carry each type
- Area types: backbone, standard, stub, totally stubby, NSSA, and totally NSSA
- ABR and ASBR roles, and the NSSA translator election (RFC 3101)
- DR and BDR election, network types, and the adjacency states (Down, Init, 2-Way, ExStart, Exchange, Loading, Full)
- Cost and reference bandwidth, external metric types E1 and E2, inter-area summarization (`area range`), and external summarization (`summary-address`)
- OSPFv2 cryptographic authentication (RFC 5709)
- OSPFv3 (RFC 5340): per-link operation, link-local addressing, the Link-LSA and Intra-Area-Prefix-LSA, and address families (RFC 5838)

**Project 1.1.** Build area 0 plus a stub area, a totally stubby area, and an NSSA. Place an ASBR in the NSSA that redistributes static routes. For every router, produce a table that lists each LSA type in its link-state database and explains why that type is present or absent. Show the Type 7 to Type 5 translation, and identify which ABR performs it and why.

**Project 1.2.** Use a script to inject eight faults in random order: area ID mismatch, MTU mismatch, hello/dead timer mismatch, authentication mismatch, network type mismatch, duplicate router ID, stub flag mismatch, and an unintended passive interface. Identify each fault from `show` output only. Record the exact output that proves each fault.

## Phase 2: EIGRP

**Topics:** DUAL, the feasibility condition, successor and feasible successor, stuck-in-active, classic mode versus named mode, wide metrics, K values, stub routers, authentication, and summarization.

**Project 2.** Build one topology where a feasible successor exists and one where none exists. Fail the primary path in each. Use a packet capture to show the difference in query and reply messages.

## Phase 3: route control and redistribution

**Topics**
- Default AD values: connected 0, static 1, eBGP 20, EIGRP internal 90, OSPF 110, IS-IS 115, RIP 120, EIGRP external 170, iBGP 200
- Route maps (`match`, `set`, `continue`), prefix lists with `ge` and `le`, distribute lists, and route tags
- Seed metrics for redistribution, and the `subnets` keyword for OSPF redistribution in IOS
- Loops and suboptimal routing from mutual redistribution at more than one point

ENARSI also tests policy-based routing and VRF-Lite. Neither is in your list, so study them separately.

**Project 3.** Configure mutual OSPF and EIGRP redistribution at two points with no protection. Capture the resulting loop or suboptimal path. Fix it with route tags and route-map filters. Fail one redistribution point to prove the fix. Deliver before and after routing tables.

## Phase 4: BGP

**Fundamentals (RFC 4271)**
- TCP port 179, and the message types OPEN, UPDATE, NOTIFICATION, and KEEPALIVE
- The states Idle, Connect, Active, OpenSent, OpenConfirm, and Established
- The Cisco default timers: keepalive 60 seconds, hold time 180 seconds
- The default eBGP TTL of 1, and the options for loopback peering (multihop, GTSM per RFC 5082)
- The rule that a router does not advertise an iBGP-learned route to another iBGP peer
- The next-hop behavior of eBGP versus iBGP, and `next-hop-self`
- 4-octet ASNs (RFC 6793)

**Path attributes**
- Well-known mandatory: ORIGIN, AS_PATH, NEXT_HOP
- Well-known discretionary: LOCAL_PREF, ATOMIC_AGGREGATE
- Optional transitive: AGGREGATOR, COMMUNITY
- Optional non-transitive: MED, ORIGINATOR_ID, CLUSTER_LIST
- WEIGHT is Cisco-proprietary, local to the router, and not carried in UPDATE messages.

**Best-path selection on Cisco IOS XE.** A path whose next hop is unreachable is not eligible. Among eligible paths, the decision order is:
1. Highest WEIGHT
2. Highest LOCAL_PREF
3. Locally originated path
4. Shortest AS_PATH
5. Lowest ORIGIN (IGP, then EGP, then incomplete)
6. Lowest MED (compared only between paths from the same neighboring AS by default)
7. eBGP over iBGP
8. Lowest IGP metric to the next hop
9. Oldest eBGP path
10. Lowest router ID (or ORIGINATOR_ID when present)
11. Shortest CLUSTER_LIST
12. Lowest neighbor address

Other vendors differ in this order, so state the vendor when you write the order down.

**Communities.** Standard communities (RFC 1997) include NO_EXPORT, NO_ADVERTISE, and NO_EXPORT_SUBCONFED. Also study extended communities (RFC 4360), large communities (RFC 8092), BLACKHOLE (RFC 7999), and GRACEFUL_SHUTDOWN (RFC 8326). Cisco IOS XE does not send communities to a neighbor unless you configure `send-community`.

**Route reflectors (RFC 4456).** An RR reflects client routes to clients and non-clients. It reflects non-client routes to clients only. It reflects eBGP routes to both. An RR advertises only its best path, which reduces path diversity. Loop prevention uses ORIGINATOR_ID and CLUSTER_LIST. The default cluster ID is the RR router ID.

**ECMP.** Multipath installs several paths but still advertises one best path. Paths must tie on all attributes through the IGP metric step. eBGP multipath requires the same neighboring AS by default, and `bgp bestpath as-path multipath-relax` removes that requirement.

**BFD.** BFD uses RFC 5880 and RFC 5881 (single-hop, UDP 3784) and RFC 5883 (multihop, UDP 4784). Echo mode uses UDP 3785. The detection time is the detect multiplier multiplied by the negotiated transmit interval.

**Prefix filtering.** Study prefix lists with `ge` and `le`, AS-path filters, route maps, and `maximum-prefix`. Also study bogon and private-ASN filtering, and the common convention of rejecting IPv4 prefixes longer than /24 and IPv6 prefixes longer than /48 in the global table. FRR enables `bgp ebgp-requires-policy` by default (RFC 8212), so FRR drops eBGP routes in both directions until you configure a policy.

**Project 4.1: sessions and next hops.** Build AS 65001 with three iBGP routers peering over loopbacks on an IGP, and eBGP to two other ASNs. Demonstrate the iBGP next-hop failure and fix it. Demonstrate that loopback eBGP peering fails until you configure multihop. Capture an OPEN message and decode its capabilities.

**Project 4.2: best path.** Build a prefix with enough candidate paths that each of decision steps 1 through 8 decides the winner once. Before you read the output, write down which path wins and which step decides. Pass criterion: all predictions correct, with `show bgp` output as evidence.

**Project 4.3: policy.** Implement customer, peer, and transit relationships with communities. Set LOCAL_PREF in the order customer, then peer, then transit. Do not export peer or transit routes to other peers or transit. Add inbound prefix lists, AS-path filters, and `maximum-prefix`. Test a route leak: a customer announces provider routes, and your filters must block them. Provide `show` output as evidence.

**Project 4.4: route reflectors.** Replace the full mesh with two RRs that share a cluster ID. Show ORIGINATOR_ID and CLUSTER_LIST on a reflected route. Show the path-diversity loss from best-path-only reflection. Fail one RR and prove that traffic continues.

**Project 4.5: ECMP and BFD.** Dual-home one AS to a neighbor with two eBGP sessions and enable multipath. Generate 50 flows with different source ports and show the distribution on interface counters. Then break forwarding without dropping carrier (for example, with a packet filter between the routers). Measure BGP failover time without BFD and with BFD. Record both times.

## Phase 5: IPv6 routing

**Topics:** OSPFv3 address families, EIGRP named-mode IPv6, MP-BGP for IPv6 (RFC 4760), the global and link-local next hop (RFC 2545), IPv6 sessions versus IPv4 sessions that carry IPv6 routes, and IPv6 prefix lists.

**Project 5.** Convert the Project 4.3 topology to dual stack. Pass criterion: identical policy outcomes in both address families, plus a written list of every difference that you encountered.

## Phase 6: IS-IS

**Topics**
- ISO/IEC 10589 and RFC 1195, and the NET format with area ID, system ID, and N-selector
- Level 1, Level 2, and Level 1/2 routers
- DIS election on broadcast links (no BDR)
- LSP, CSNP, and PSNP behavior
- Narrow versus wide metrics (narrow allows 63 per interface, wide is defined in RFC 5305)
- The attached (ATT) bit and the default route it creates in Level 1
- Route leaking (RFC 5302)
- IPv6 support (RFC 5308) and multi-topology (RFC 5120)
- IS-IS runs directly over the data link layer, not over IP.

**Project 6.** Rebuild the Phase 1 topology with IS-IS. Use Level 1 areas and Level 1/2 border routers. Show the ATT-bit default route, then replace it with route leaking. Capture a Hello and a CSNP and decode the system ID fields. Write a table that compares OSPF and IS-IS on area boundaries, database contents, adjacency formation, and metric handling.

## Phase 7: routing security and peering data

**RPKI and ROV.** RFC 6480 defines the architecture. A Route Origin Authorization (ROA, RFC 6482) binds a prefix, a maximum length, and an origin ASN. RFC 6811 defines the validation states Valid, Invalid, and NotFound. The router receives validated data through the RPKI-to-Router protocol (RFC 8210). ROV validates the origin ASN only and does not validate the AS_PATH, so a forged-origin hijack still passes. RFC 9319 covers the use of maxLength.

**IRR.** Routing Policy Specification Language (RPSL, RFC 2622) objects include `route`, `route6`, `aut-num`, `as-set`, `route-set`, and `mntner`. IRR data is not cryptographically signed, and its quality varies by database. Use `bgpq4` to generate prefix lists from an `as-set`.

**PeeringDB.** It records networks, exchange points, and facilities. Relevant network fields include the IRR as-set, the prefix-count information, and the peering policy.

**MANRS.** The network operator actions are Filtering, Anti-spoofing (RFC 2827, RFC 3704), Coordination, and Global Validation.

**Project 7.1: ROV.** Deploy Routinator with Simplified Local Internet Number Resource Management (SLURM, RFC 8416) assertions to emulate ROAs. Connect a router over RTR. Simulate a wrong-origin announcement and a more-specific announcement that exceeds maxLength. Show that your policy rejects both. Also run Routinator against the real trust anchors and compare the validated data with your lab ROAs.

**Project 7.2: IRR filters.** Query an `aut-num` and an `as-set` with `whois`. Write a script that expands a customer `as-set` with `bgpq4`, renders a prefix list, and regenerates it on demand. Test: the customer announces a prefix that is not in its `as-set`, and the filter rejects it.

**Project 7.3: PeeringDB.** Query the PeeringDB API for a network, an exchange point, and a facility. Find the exchange points and networks present at the facility where you work. Extract the values you would use for `maximum-prefix` and the IRR `as-set`. Write one peering-request message that uses the stated peering policy.

**Project 7.4: MANRS audit.** Map each of the four actions to concrete configuration in your Phase 8 lab. Provide the command output that proves each control.

## Phase 8: capstone

Build a five-AS lab: your AS, two customers, one peer, and one transit provider. Use IS-IS or OSPF as the IGP, two route reflectors, dual-stack addressing, and the policy from Project 4.3. Add IRR-generated customer filters, ROV over RTR, BFD, ECMP, and maximum-prefix limits.

Run four tests and keep evidence for each:
1. A link failure, with the convergence time measured.
2. A route reflector failure.
3. A simulated prefix hijack that ROV rejects.
4. A simulated route leak that your filters block.

Deliverables:
- A Git repository with all configurations.
- A method of procedure for adding a customer.
- A method of procedure for decommissioning a transit link.
- A test report.

## Limits and sequencing

- You can run Phases 0 and 1 during your current Cisco III course. I do not have its syllabus, so compare its OSPF coverage against Phase 1 and skip what it duplicates.
- Your Linux course supports the FRR and containerlab work. Do not start Phase 4 until Phase 3 passes, because BGP policy debugging depends on reading route maps and prefix lists fluently.
- Lab competence does not equal production competence. A lab does not reproduce full-table scale, vendor software defects, or the behavior of other networks' policies. Treat Phase 7 especially as procedural practice, not proof of operational readiness.
- ENCOR is a separate study effort with a wider scope. The CCNP Enterprise credential requires it, and this roadmap does not prepare you for it.
