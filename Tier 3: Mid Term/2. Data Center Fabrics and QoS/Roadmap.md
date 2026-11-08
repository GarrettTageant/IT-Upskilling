Claude-generated roadmap:

# Roadmap: data center fabrics and QoS

## Assumptions and prerequisites

The roadmap assumes 8 hours per week, which gives 18 to 22 weeks. Your Linux and BGP depth sets the actual pace.

You must meet these prerequisites before Phase 1. If you do not, finish the Linux and BGP roadmaps first, because every project below fails in ways you cannot diagnose without them.

- Create network namespaces, veth pairs, VLAN subinterfaces, and LACP bonds with `ip`.
- Configure an eBGP session, `maximum-paths`, and a prefix filter from memory.
- Capture traffic with `tcpdump` or `tshark`, and read a hex dump of an Ethernet, IP, and UDP header.

## Lab platform

Use containerlab on a Linux host with Docker. Use these images:

| Image | Use |
|---|---|
| FRRouting (`quay.io/frrouting/frr`) | Underlay, EVPN, and Linux kernel VXLAN |
| Nokia SR Linux | A full EVPN-VXLAN implementation, free to run |
| Arista cEOS | MLAG and an EOS CLI, free download with an Arista account |
| Linux hosts | Traffic generation, `tc`, `nftables`, `iperf3` |

**Limitation:** Containers forward in software. They have no per-priority hardware queues, no shared buffer, and no PFC. You can verify the configuration, the control plane, the packet formats, and the software queuing behavior. You cannot verify PFC behavior in a virtual lab. Phase 5 states how to handle this.

## Phase overview

| Phase | Topic | Weeks |
|---|---|---|
| 1 | Clos topology and eBGP underlay | 3 |
| 2 | VXLAN and BGP EVPN | 5 |
| 3 | MLAG and EVPN multihoming | 3 |
| 4 | DiffServ, queuing, and ECN | 4 |
| 5 | Lossless Ethernet, PFC, and RoCEv2 | 3 |
| 6 | Capstone | 3 |

## Phase 1: Clos topology and eBGP underlay

**Study:** RFC 7938 (BGP in large-scale data centers). A 3-stage Clos fabric: every leaf connects to every spine, and no spine connects to another spine. Oversubscription ratio, ECMP hashing on the 5-tuple, RFC 8950 (IPv4 NLRI with an IPv6 next hop, used for unnumbered interfaces), and BFD.

**Project 1.1: Design calculation (paper).**
- Task: Given leaf switches with 48 x 25 GbE downlinks and 8 x 100 GbE uplinks, calculate the oversubscription ratio, the spine count, and the maximum number of server ports for 4 spines.
- Pass criterion: Every figure is shown with its derivation and verified by a second method.

**Project 1.2: Build the underlay.**
- Task: Build 2 spines and 4 leaves in containerlab with FRR. Use eBGP with one ASN per leaf and one shared ASN for the spines. Advertise a /32 loopback from each device.
- Pass criteria:
  1. Each leaf has 2 equal-cost paths to every other leaf loopback.
  2. A link failure converges in the time you measured and recorded with BFD enabled and disabled.
  3. Run 20 parallel `iperf3` flows between two hosts on different leaves, and show with interface counters that both spines carry traffic.

**Project 1.3: Unnumbered variant.**
- Task: Rebuild Project 1.2 with RFC 8950 unnumbered interfaces.
- Pass criterion: You explain, from a packet capture of the BGP OPEN message, which capability the peers negotiate.

## Phase 2: VXLAN and BGP EVPN

**Study:**
- RFC 7348: the 8-byte VXLAN header, the 24-bit VNI, UDP destination port 4789, and the source port derived from a hash of the inner headers.
- RFC 7432: the EVPN route types, the route distinguisher, the route target, and MAC mobility.
- RFC 8365: the use of EVPN with VXLAN encapsulation. RFC 7432 alone defines MPLS encapsulation.
- RFC 9136: route type 5.
- RFC 9135: integrated routing and bridging (IRB).

**Project 2.1: Static VXLAN.**
- Task: Connect two Linux hosts with `ip link add vxlan100 type vxlan id 100 dstport 4789 local <ip> remote <ip>`. Bridge it to a local interface.
- Pass criteria:
  1. A capture shows UDP destination port 4789 and VNI 100.
  2. With an underlay IP MTU of 1500, a 1500-byte inner IP packet fails. The outer IPv4 packet is 1550 bytes: 20 + 8 + 8 + 14 + 1500.
  3. You fix the failure by setting the underlay MTU and show a passing result.

**Project 2.2: Layer 2 EVPN.**
- Task: Add the BGP `l2vpn evpn` address family to the Project 1.2 fabric. Create 2 VNIs mapped to 2 VLANs.
- Pass criteria:
  1. `show bgp l2vpn evpn` lists type 2 and type 3 routes for each VNI.
  2. You decode one type 2 route and one type 3 route field by field, including the route target and the VNI label.
  3. You state how the fabric handles BUM traffic in your configuration: ingress replication or underlay multicast.

**Project 2.3: Symmetric IRB with type 5 routes.**
- Task: Add a tenant VRF with an L3VNI. Configure an anycast gateway: the same IP address and the same virtual MAC address on every leaf, in each VLAN.
- Pass criteria:
  1. A host in VLAN A reaches a host in VLAN B on a different leaf, and the capture shows the L3VNI in the VXLAN header.
  2. You move a host MAC address between leaves and show the MAC mobility sequence number increment in the extended community (RFC 7432).
  3. An external prefix injected as a type 5 route is reachable from every leaf.

**Project 2.4: Failure analysis.**
- Task: Break the fabric in 5 ways: wrong route target, VNI mismatch, underlay MTU too low, missing `advertise-all-vni`, and a spine failure.
- Pass criterion: For each fault, you record the symptom, the diagnostic command, and the root cause.

## Phase 3: MLAG and EVPN multihoming

**Study:** IEEE 802.1AX (LACP). The MLAG components: peer link, keepalive link, and shared system ID. MLAG is vendor-specific (Arista MLAG, Cisco vPC, Juniper MC-LAG). Split-brain behavior. EVPN multihoming with route types 1 and 4 (RFC 7432) as the standards-based alternative.

**Project 3.1: MLAG pair.**
- Task: Build an MLAG pair with cEOS. Connect a Linux host with a `802.3ad` bond to both peers.
- Pass criteria: The bond forms with the same LACP partner system ID on both links. Failing one peer, the peer link, and the keepalive link each produce the behavior you predicted in writing before the test.

**Project 3.2: MLAG under VXLAN.**
- Task: Run the Project 2.3 design with the MLAG pair as a single VTEP (shared virtual IP address).
- Pass criterion: You state which route types and next hops the pair advertises.

**Project 3.3: EVPN multihoming.**
- Task: On a platform that supports it, replace MLAG with an Ethernet segment identifier (ESI). Platform support varies, so verify it in the vendor documentation before you start.
- Pass criterion: You compare MLAG and EVPN multihoming in a table covering failure behavior, interoperability, and the configuration state each requires.

## Phase 4: DiffServ, queuing, and ECN

**Study:**
- RFC 2474: the 6-bit DSCP in the former ToS octet.
- RFC 2597 (AF) and RFC 3246 (EF).
- RFC 4594 (configuration guidelines).
- RFC 3168: ECN codepoints Not-ECT `00`, ECT(1) `01`, ECT(0) `10`, and CE `11`.
- RFC 2983 and RFC 6040: DSCP and ECN handling in tunnels.
- RFC 8257 (DCTCP).
- Strict priority, DWRR, tail drop, RED, and WRED. Microbursts and incast.

**Project 4.1: Classification and marking.**
- Task: Mark traffic with `nftables` (`ip dscp set`). Classify it into 3 classes with `tc`.
- Pass criteria: Per-class counters from `tc -s class show` match the traffic you generated. A capture confirms each DSCP value on the wire.

**Project 4.2: Scheduling.**
- Task: Create a 1 Gb/s bottleneck with `tc`. Apply strict priority for one class and weighted sharing for two others.
- Pass criterion: You compute the expected throughput of each class before the test, and the measured result matches within the tolerance you set in advance.

**Project 4.3: ECN marking.**
- Task: Apply a RED queue with the `ecn` option at the bottleneck. Enable `net.ipv4.tcp_ecn=1` on the hosts.
- Pass criteria: `tcpdump -v` shows CE-marked packets (`tos 0x3` in the low 2 bits). The `marked` counter in `tc -s qdisc` increases. You record the queue depth as the marking threshold changes.

**Project 4.4: DCTCP versus CUBIC.**
- Task: Run both congestion controllers through the same marking threshold.
- Pass criterion: You report queue occupancy and throughput for each controller and explain the difference from the DCTCP reaction to the fraction of CE-marked packets.

**Project 4.5: QoS through VXLAN.**
- Task: In the Phase 2 fabric, send marked traffic across the VXLAN tunnel. Test `tos inherit` and a fixed `tos` value on the VXLAN device.
- Pass criterion: Captures of the inner and outer headers show whether DSCP and ECN propagate in each configuration.

## Phase 5: Lossless Ethernet, PFC, and RoCEv2

**Study:**
- IEEE 802.1Qbb (PFC). The pause frame uses destination MAC `01-80-C2-00-00-01`, EtherType `0x8808`, opcode `0x0101`, a priority enable vector, and 8 timers.
- IEEE 802.1Qaz (ETS and DCBX).
- RoCEv2: UDP destination port 4791, with the InfiniBand Base Transport Header after the UDP header.
- DSCP-based PFC on routed links.
- Headroom: it depends on link speed, cable length, MTU, and device response time.
- PFC storms and PFC deadlock.
- The DCQCN paper (Zhu et al., SIGCOMM 2015) and "RDMA over Commodity Ethernet at Scale" (Guo et al., SIGCOMM 2016).

**Project 5.1: RoCEv2 packet format.**
- Task: Load the `rdma_rxe` module (Soft-RoCE), create an RDMA device on a veth interface, and run `ib_write_bw` from `perftest`.
- Pass criteria: A capture shows UDP destination port 4791 and the BTH. You identify the opcode, destination QP, and PSN fields.

**Project 5.2: PFC frame construction.**
- Task: Build a PFC frame for priority 3 with Scapy. Decode it in Wireshark.
- Pass criterion: Every field decodes to the value you set, including the pause time in quanta (512 bit times each).

**Project 5.3: Headroom and threshold design (paper).**
- Task: For a 100 GbE link with a 100 m fiber run and a 4,200-byte MTU, calculate the PFC headroom. Set the ECN marking thresholds and the PFC XOFF threshold.
- Pass criteria: The ECN threshold triggers before the PFC XOFF threshold, and you justify why the order matters. Every input is stated with its source in the device documentation.

**Project 5.4: Hardware validation (conditional).**
- Condition: You have 2 RDMA-capable NICs and a switch that supports PFC and ECN. Used hardware is the realistic route. Cost is the constraint, not skill.
- Task: Enable PFC on one priority and ECN on that queue. Generate incast with `ib_write_bw`.
- Pass criteria: Switch counters show pause frames sent and received, and no drops in the lossless queue. Then disable PFC and show the drops.
- If you do not have the hardware, you cannot complete this project. State that gap in your portfolio and do not claim PFC competence.

**Project 5.5: Failure-mode analysis (paper).**
- Task: Using the Guo et al. paper, describe how a PFC deadlock forms and what a PFC watchdog does.
- Pass criterion: You identify the topology condition that creates the cyclic buffer dependency.

## Phase 6: Capstone

**Project 6.1:** Build a 2-spine, 4-leaf fabric with the following:
- eBGP underlay with BFD.
- L2 and L3 EVPN with symmetric IRB and anycast gateways.
- One MLAG pair.
- A DSCP policy with 3 queues and ECN marking.

**Deliverables:**
1. A design document that gives the ASN scheme, the IP plan, the VNI plan, the route target scheme, and the QoS map.
2. A method of procedure with rollback criteria for adding a leaf.
3. A failure matrix: 10 faults, each with the expected behavior, the observed behavior, and the evidence.

**Pass criterion:** A reviewer can rebuild the fabric from the document alone.

## Completion standard

You have completed the roadmap when you can do all of the following without references. If you cannot, repeat the relevant phase.
- Draw the VXLAN-encapsulated frame and state each header's field sizes.
- Trace a type 2 route from host ARP to remote VTEP forwarding.
- Compute the PFC headroom for an arbitrary link.
