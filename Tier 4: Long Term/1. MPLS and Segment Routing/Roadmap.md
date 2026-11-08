Claude-generated roadmap:

This roadmap has seven phases, each ending in a project with pass criteria. Phases 1 through 3 build the MPLS foundation. Phases 4 through 6 cover SR-MPLS, TI-LFA, and EVPN-VPWS. Phase 7 combines them in one design.

## Prerequisites

Phase 1 requires the following skills. If you cannot do them without references, close the gap first, because every later phase fails without them.

- Configure IS-IS or OSPF (single area is sufficient) and verify the adjacency, LSDB, and SPF result.
- Configure iBGP between loopbacks, including route reflectors, next-hop handling, and best-path selection.
- Configure VRF-Lite and explain why overlapping IPv4 prefixes coexist in separate VRFs.
- Read a packet capture in Wireshark.

Your coursework so far does not cover IS-IS, route reflectors, or MPLS. Treat the routing roadmap as a hard prerequisite.

## Lab platform

| Phases | Platform | Reason |
|---|---|---|
| 1 to 3 | containerlab with FRRouting (FRR) | FRR implements LDP and BGP VPNv4, and the Linux kernel forwards MPLS. |
| 4 to 7 | A vendor image (Cisco IOS XRv 9000 or Juniper vJunos-router) | As far as I know, FRR does not implement EVPN-VPWS or TI-LFA. Verify this against the release notes of the FRR version you use. |

Before you start each phase, confirm that your chosen image supports the features of that phase. Virtual router images consume substantial CPU and RAM, so check the current requirements before you design a topology. A virtual data plane gives no valid measurement of forwarding performance or repair time.

Reference list:

- Books: *MPLS Fundamentals* (De Ghein), *Segment Routing Part I* and *Part II* (Filsfils and others).
- Credential: CCNP Service Provider, exam 350-501 (SPCOR), covers this subject area.

## Phase 1: MPLS forwarding

**Study:**

- RFC 3031 (architecture) and RFC 3032 (label stack encoding).
- The label stack entry has four fields: a 20-bit label, a 3-bit traffic class (TC, RFC 5462), a 1-bit bottom-of-stack flag (S), and an 8-bit TTL.
- Reserved labels: 0 (IPv4 explicit null), 2 (IPv6 explicit null), 3 (implicit null).
- Label operations: push, swap, pop. Also forwarding equivalence class (FEC), label forwarding information base (LFIB), and penultimate hop popping (PHP).
- TTL propagation and the uniform and pipe models (RFC 3443).
- MPLS MTU and its interaction with IP MTU.

**Project 1: static LSPs**

1. Build four routers: PE1, P1, P2, PE2.
2. Configure static label bindings for a bidirectional LSP between PE1 and PE2 loopbacks, with PHP.
3. Before you run any command, write the expected label at every hop.

*Pass criteria:*

- A capture shows the expected labels, the S bit, and TTL decrement at each hop.
- You can state what traceroute shows with TTL propagation enabled and with it disabled.
- You can state what happens when you remove one binding.

## Phase 2: LDP

**Study:**

- RFC 5036: link Hello over UDP port 646 to 224.0.0.2, session over TCP port 646, and the transport address.
- Message classes: discovery, session, advertisement, notification.
- Label distribution modes: downstream unsolicited and downstream on demand; ordered and independent control; liberal and conservative retention. Identify which mode your platform uses.
- Targeted LDP sessions.
- LDP-IGP synchronization (RFC 5443).
- IPv6 LDP (RFC 7552).
- Session authentication with TCP MD5 (RFC 2385) or TCP-AO (RFC 5925), as the platform supports.
- Label allocation filtering.

**Project 2: LDP core**

1. Build a six-router core with an IGP and LDP.
2. Enable LDP-IGP synchronization.
3. Add one targeted LDP session.
4. Filter label allocation to loopback /32 prefixes only.

*Pass criteria:*

- You produce a failure test: disable LDP on one link while the IGP stays up, and show the traffic loss with sync disabled and with sync enabled.
- You document the evidence with `show` output and a capture.
- You can explain the difference between the LIB and the LFIB for one prefix.

## Phase 3: BGP/MPLS L3VPN (RFC 4364)

**Study:**

- Virtual routing and forwarding (VRF) instances.
- The route distinguisher (RD): 8 bytes, prepended to the IPv4 prefix to form a 12-byte VPN-IPv4 address.
- The route target (RT) extended community and its import and export policy.
- MP-BGP (RFC 4760) with AFI 1, SAFI 128. VPNv6 is covered in RFC 4659.
- The two-label stack: an outer transport label and an inner VPN label.
- PE-CE routing: static, eBGP (as-override, allowas-in, Site of Origin), OSPF (domain ID, DN bit, sham link, RFC 4577).
- Route reflectors and RT Constraint (RFC 4684).
- Inter-AS options A, B, and C (RFC 4364, section 10).

**Project 3: two-customer L3VPN**

1. Reuse the Project 2 core. Provision two customers with overlapping address space.
2. Use eBGP as the PE-CE protocol for one customer and OSPF for the other.
3. Use two route reflectors.

*Pass criteria:*

- Customers cannot reach each other.
- A capture shows the two-label stack, and you identify the VPN label and the transport label.
- You explain the effect of PHP on the label stack at the last P router.

**Project 4: hub-and-spoke and fault injection**

1. Implement hub-and-spoke with RT policy, plus one extranet shared-services VRF.
2. Write a script that applies one randomly selected fault from a list of at least eight, so that you do not know which fault was applied. Include RT mismatch, missing `mpls ip` on one link, MTU mismatch, and an RD collision.
3. Diagnose each fault from the data plane toward the control plane. Use `ping mpls` and `traceroute mpls` (RFC 8029).

*Pass criteria:*

- You diagnose seven of eight faults correctly without help.
- Each diagnosis has written evidence.

## Phase 4: SR-MPLS (RFC 8402)

**Study:**

- RFC 8402 (architecture) and RFC 8660 (SR-MPLS data plane).
- The IGP extensions: IS-IS (RFC 8667) and OSPFv2 (RFC 8665).
- The segment routing global block (SRGB) and the segment routing local block (SRLB).
- Prefix-SID, node-SID, and anycast SID, plus the prefix-SID flags (N, P, E).
- Adjacency-SID, which has local significance only.
- Using the same SRGB on every router is a recommended practice. Different SRGBs still work, because each router advertises its own SRGB.
- LDP interworking (RFC 8661) and the mapping server.
- SR Policy (RFC 9256): endpoint, color, candidate paths, preference, and binding SID.
- Flexible Algorithm (RFC 9350).
- On-demand next hop and automated steering for L3VPN routes that carry a color extended community.

**Project 5: replace LDP with SR-MPLS**

1. Enable IS-IS SR on the Project 3 core.
2. Migrate the transport from LDP to SR-MPLS. Run both in parallel and verify that the platform prefers SR.
3. Use a mapping server to give one LDP-only router a prefix-SID.

*Pass criteria:*

- Both L3VPNs keep working throughout the migration.
- The migration has a written method of procedure (MOP) with rollback criteria.
- A capture shows the transport label equal to the SRGB base plus the prefix-SID index.

**Project 6: SR Policy and Flex-Algo**

1. Create an SR Policy with an explicit SID list that avoids one link.
2. Steer one VPN prefix into the policy with a color.
3. Define a flexible algorithm that excludes links by affinity.

*Pass criteria:*

- A traceroute and a capture show the label stack of the policy.
- The policy path differs from the IGP shortest path.
- You show the behavior when the policy has no valid candidate path.

## Phase 5: TI-LFA

**Study:**

- Background: loop-free alternates (RFC 5286), remote LFA (RFC 7490), and LFA applicability (RFC 6571).
- TI-LFA: the repair path follows the post-convergence path. The repair list can use node-SIDs and adjacency-SIDs. The P-space and Q-space determine the repair list. Protection types are link, node, and shared risk link group (SRLG).
- The TI-LFA specification was developed as `draft-ietf-rtgwg-segment-routing-ti-lfa`. Verify whether it is now an RFC and under which number before you cite it.
- Failure detection with BFD (RFC 5880).
- Micro-loops after convergence (RFC 8333).

**Project 7: protection coverage**

1. Build a ring-based topology in which classic LFA gives incomplete coverage.
2. Enable LFA only and record which prefixes have no backup.
3. Enable TI-LFA and compare the coverage.
4. Add node protection and an SRLG.
5. Fail each link in turn.

*Pass criteria:*

- For each protected prefix, you predict the repair list before you read the platform output, then confirm it.
- Packet loss during each failure is recorded.
- You state the limit of the result: the 50 ms figure is a design target, not a protocol property, and a virtual data plane cannot validate it.

## Phase 6: EVPN-VPWS (RFC 8214)

**Study:**

- RFC 7432 (EVPN base) and RFC 8214 (EVPN-VPWS).
- The service uses the Ethernet A-D per EVI route (route type 1). The Ethernet Tag ID field carries the VPWS service instance identifier.
- No MAC learning occurs in the PE for this service.
- The Layer 2 Attributes extended community: the P and B flags and the L2 MTU field. Test the behavior of your platform on an L2 MTU mismatch.
- Redundancy: single-homed, single-active, and all-active, with the Ethernet Segment Identifier (ESI).
- Contrast: LDP pseudowires (RFC 4447, RFC 4448).

**Project 8: point-to-point service over SR-MPLS**

1. Provision one EVPN-VPWS service between two sites across the SR-MPLS core.
2. Multihome one CE to two PEs.
3. Fail one attachment circuit, then one PE.

*Pass criteria:*

- The BGP table shows the route type 1 entries and their attributes.
- The PE shows no MAC table entries for the service.
- A capture shows the service label.
- You measure the failover behavior and explain it from the route advertisements and withdrawals.
- You can state the control-plane difference between this service and an LDP pseudowire.

## Phase 7: capstone

Build one design and prove each requirement:

1. An SR-MPLS IS-IS core with TI-LFA on all links and two route reflectors.
2. Two L3VPN customers and one EVPN-VPWS service.
3. A migration MOP, including pre-checks, change steps, validation, and rollback criteria.
4. A fault-injection test with at least twelve faults across all five technologies.

*Pass criteria:*

- You diagnose ten of twelve faults without help.
- You complete the migration MOP in a lab window, with rollback exercised once.
- You explain every label in a capture of each service.

## Time and limits

At 8 to 10 hours per week, the planning estimate is as follows. These are estimates, not measurements.

| Phase | Weeks |
|---|---|
| Prerequisites (if gaps exist) | 4 to 8 |
| 1 | 2 |
| 2 | 2 to 3 |
| 3 | 4 to 5 |
| 4 | 5 to 6 |
| 5 | 3 |
| 6 | 4 |
| 7 | 3 to 4 |

The skills in this roadmap apply to service provider and large WAN networks. A lab proves that you understand the protocols. It does not replace production experience with scale, vendor defects, and change control, and a hiring manager will weigh it accordingly.
