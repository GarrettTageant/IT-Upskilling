Claude-generated roadmap:

# Applied network security roadmap

The roadmap has six phases. Each phase builds on the lab from the previous phase, and each project has pass criteria that you define before you build. Phases 1 to 3 apply to most network security roles. Phase 4 (BCP 38, RTBH, Flowspec) applies mainly to service providers, hosting providers, and large datacenter or enterprise edge networks. Phases 1 to 3 overlap Linux and network security coursework, so schedule them first.

Durations are planning allowances that assume 8 to 10 hours per week. The total is about 30 to 36 weeks.

| Phase | Topics | Planned duration |
|---|---|---|
| 0 | Prerequisites and lab | 3 to 4 weeks |
| 1 | Stateful firewall policy | 5 to 6 weeks |
| 2 | IPsec with IKEv2, WireGuard | 5 to 6 weeks |
| 3 | 802.1X with RADIUS and EAP, TACACS+, CIS Benchmark hardening | 6 to 8 weeks |
| 4 | BCP 38 with uRPF, RTBH, BGP Flowspec | 6 to 8 weeks |
| 5 | Integrated capstone | 4 weeks |

## Phase 0: prerequisites and lab

You must be able to do the following before Phase 1:

- Build network namespaces, VLAN subinterfaces, and `nftables` rulesets on Linux.
- Configure eBGP and iBGP sessions, communities, route maps, and prefix lists (RFC 4271).
- Create a certificate authority and issue X.509 certificates with OpenSSL.
- Capture and read packets in Wireshark and `tcpdump`.

Lab environment: one Linux host that runs containerlab, FRRouting (FRR), strongSwan, WireGuard, FreeRADIUS, `hostapd`, `wpa_supplicant`, `tac_plus-ng`, and ExaBGP or GoBGP. Use Git for every configuration.

**Gate project.** Build three autonomous systems with FRR. Each autonomous system announces one prefix, and a community-based route map changes local preference.

Pass criteria:
- `show bgp` output matches the expected best path for every prefix.
- The route map changes the best path when you add the community.
- A packet capture shows the BGP OPEN, UPDATE, and KEEPALIVE messages.

## Phase 1: stateful firewall policy

Learn:
- Netfilter hooks: prerouting, input, forward, output, postrouting.
- `nftables` tables, base chains, chain priority, sets, maps, and counters.
- Connection tracking states: `new`, `established`, `related`, `invalid`, `untracked`.
- Required ICMP: ICMPv4 type 3 code 4 for path MTU discovery, and ICMPv6 types 2 and 133 to 137 for path MTU discovery and Neighbor Discovery.
- PAN-OS: zones, top-down first-match security policy, App-ID, the candidate and running configuration with `commit`, and the implicit rules `intrazone-default` (allow) and `interzone-default` (deny).
- FortiOS: interface and zone policies, top-down evaluation, implicit deny, and VDOMs.

**Project 1A: vendor-neutral policy in `nftables`.**
1. Write a policy specification: three zones (inside, DMZ, outside) and a flow table with source, destination, protocol, port, and action.
2. Implement the specification on a Linux router with three namespaces.
3. Write a test matrix of at least 20 cases, each with an expected result.

Pass criteria:
- Every case in the matrix passes, including IPv6.
- `invalid` packets are dropped.
- No unsolicited inbound connection reaches inside or DMZ.
- `conntrack -L` shows the expected entries.
- `nft -f` loads the ruleset atomically.

**Project 1B: the same specification on PAN-OS and FortiOS.** Re-run the Project 1A test matrix against both platforms. Document every divergence, such as PAN-OS matching on application instead of port.

Both vendors restrict free use of their virtual appliances. Verify the current evaluation terms on the vendor sites before you plan this project. If you cannot obtain an evaluation license, skip Project 1B. The `nftables` result still stands.

## Phase 2: IPsec with IKEv2 and WireGuard

Learn IKEv2 (RFC 7296):
- The four exchanges: `IKE_SA_INIT`, `IKE_AUTH`, `CREATE_CHILD_SA`, `INFORMATIONAL`.
- UDP ports 500 and 4500, and the 4-octet non-ESP marker on port 4500 for NAT traversal.
- Traffic selectors, ESP (RFC 4303), and rekeying.
- Algorithm guidance: RFC 8247.
- Certificate authentication with signature authentication (RFC 7427).
- Route-based tunnels on Linux with an XFRM interface (`ip link add type xfrm ... if_id`).

Learn WireGuard:
- The Noise construction `Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s`.
- Static public keys and the lack of algorithm negotiation.
- Cryptokey routing: `AllowedIPs` is the outbound routing table and the inbound source filter.
- `REKEY_AFTER_TIME` of 120 s and `REJECT_AFTER_TIME` of 180 s.
- Per-packet overhead of 60 bytes with an IPv4 outer header and 80 bytes with an IPv6 outer header (outer IP, UDP 8, WireGuard header 16, authentication tag 16).

**Project 2A: IKEv2 site-to-site with strongSwan.**
1. Authenticate with certificates from your own CA.
2. Use AES-256-GCM and a Diffie-Hellman group.
3. Define traffic selectors for two subnets.

Pass criteria:
- `swanctl --list-sas` shows an established IKE SA and Child SA.
- A capture shows `IKE_SA_INIT`, `IKE_AUTH`, and ESP packets.
- With short lifetimes, a rekey completes without dropped pings.
- Deliberate misconfigurations produce the expected notify types in the capture: `NO_PROPOSAL_CHOSEN` (14), `AUTHENTICATION_FAILED` (24), `TS_UNACCEPTABLE` (38).

**Project 2B: IKEv2 interoperability.** Establish the same tunnel between strongSwan and PAN-OS or FortiOS. The same licensing note as Project 1B applies. Pass when traffic passes in both directions and you have documented each proposal mismatch that you resolved.

**Project 2C: WireGuard hub with three peers.**

Pass criteria:
- `wg show` reports a recent handshake for each peer.
- A packet whose inner source address is outside the peer's `AllowedIPs` is dropped.
- A capture shows only UDP outside the tunnel.
- You compute the tunnel MTU from the overhead values above and confirm it with `ping -M do -s`.
- You complete a key rotation with no tunnel outage longer than your stated tolerance.

**Project 2D: comparison report.** Measure throughput with `iperf3` over IKEv2 and WireGuard. Compare them in a table with these attributes: standardization, algorithm agility, authentication method, per-packet overhead, and measured throughput.

## Phase 3: access control, device administration, hardening

### 802.1X with RADIUS and EAP

Learn:
- Roles: supplicant, authenticator, and authentication server.
- The Extensible Authentication Protocol over LAN (EAPOL) frame: EtherType `0x888E`, destination MAC `01-80-C2-00-00-03`.
- Standards: IEEE 802.1X-2020, EAP (RFC 3748), RADIUS (RFC 2865, accounting in RFC 2866), RADIUS support for EAP (RFC 3579), and VLAN assignment attributes (RFC 3580 and RFC 2868).
- RADIUS ports: UDP 1812 for authentication and 1813 for accounting.
- Dynamic authorization: RFC 5176 on UDP 3799.
- RADIUS over TLS: RFC 6614.
- EAP methods: EAP-TLS (RFC 5216, TLS 1.3 in RFC 9190), EAP-TTLS (RFC 5281), and PEAP, which has no RFC.
- MAC Authentication Bypass: a MAC address is not a credential and can be spoofed.
- CVE-2024-3596 (Blast-RADIUS): RADIUS over UDP is exposed when an Access-Request lacks `Message-Authenticator`.

**Project 3A: EAP-TLS with VLAN assignment.**
1. Use `hostapd` with `driver=wired` as the authenticator (verify that your distribution build supports it) or a managed switch that supports 802.1X.
2. Use FreeRADIUS as the authentication server.
3. Use `wpa_supplicant` with `-D wired`, and `eapol_test` for scripted tests.

Pass criteria:
- A valid certificate returns Access-Accept with the VLAN attributes.
- An expired certificate, a revoked certificate, and a certificate from a different CA are each rejected. You locate the reason for each rejection in `freeradius -X` output.
- A Disconnect-Request (RFC 5176) removes the authorization of an active session.
- A capture identifies the Access-Challenge sequence and `Message-Authenticator`.

**Project 3B: supplicant validation failure.** Configure a supplicant without `ca_cert` and record that it authenticates against a server that presents a certificate from an unrelated CA. Fix the configuration and record that authentication then fails. Use your own lab only.

### TACACS+

Learn:
- RFC 8907, TCP port 49.
- Separate authentication, authorization, and accounting.
- Per-command authorization.
- Encryption of the whole packet body, compared with RADIUS, which encrypts only the `User-Password` attribute.
- Command authorization support differs by platform. Verify support on each platform you test. Cisco IOS and Arista EOS support it. Linux hosts can use `pam_tacplus` for authentication and accounting.

**Project 3C: role-based device administration.**
1. Deploy `tac_plus-ng` with three roles: read-only, operator, administrator.
2. Enforce per-command authorization on a platform that supports it.
3. Configure a local fallback account.

Pass criteria:
- Each role can run only its permitted commands.
- The accounting log records user, time, and command for every command.
- A server outage triggers the local fallback.
- A capture shows an encrypted body.
- You test whether the server refuses packets that carry the unencrypted flag.

### CIS Benchmark hardening

Learn:
- Level 1 and Level 2 profiles, and Server and Workstation profiles.
- Each recommendation is Automated or Manual to assess.
- Assessment tools: CIS-CAT Lite (free, limited benchmark coverage) and OpenSCAP with SCAP Security Guide. Verify that a profile exists for your distribution and release.
- A benchmark version must match the operating system version.
- CIS also publishes benchmarks for PAN-OS and FortiOS. Verify the versions against your appliance versions.

**Project 3D: baseline, remediate, re-scan.**
1. Scan a fresh Linux server VM against the Level 1 Server profile.
2. Remediate with an Ansible playbook.
3. Re-scan.
4. Repeat for the strongSwan, FreeRADIUS, and TACACS+ hosts.

Pass criteria:
- The second playbook run reports `changed=0`.
- Every remaining failure has a written exception with the risk accepted.
- Your Phase 1 to 3 test suites still pass after hardening.

A benchmark score does not measure security. Treat the exceptions list and the regression results as the deliverable.

## Phase 4: BCP 38, RTBH, and Flowspec

### BCP 38 with uRPF

Learn:
- Ingress filtering (RFC 2827, BCP 38) and multihomed ingress filtering (RFC 3704, BCP 84).
- uRPF modes: strict, feasible-path, and loose. Enhanced feasible-path uRPF is in RFC 8704.
- Strict mode drops legitimate traffic when routing is asymmetric. Loose mode with a default route in the FIB accepts every source.
- Linux IPv4: `net.ipv4.conf.<interface>.rp_filter` set to `0` (off), `1` (strict), or `2` (loose). The kernel uses the larger of the `all` and per-interface values.
- Linux IPv6: the `nftables` rule `fib saddr . iif oif missing drop`.

**Project 4A: ingress filtering at a customer edge.**
1. Connect a customer namespace to an FRR provider router.
2. Generate spoofed-source packets with `scapy` or `hping3` inside the isolated lab.
3. Apply strict uRPF.
4. Add a second customer uplink with asymmetric return routing.
5. Replace strict mode with feasible-path or loose mode, or with an ACL.

Pass criteria:
- Spoofed packets are dropped before the filter change and after it.
- Counters confirm the drops.
- Strict mode drops legitimate asymmetric traffic.
- You document the mode that you chose and why.

Never send spoofed packets outside an isolated lab.

### Remotely triggered blackhole filtering (RTBH)

Learn:
- Destination-based RTBH (RFC 3882). A trigger router announces the victim `/32` with a next hop that resolves to a discard route. The `NO_EXPORT` community limits propagation.
- Destination-based RTBH makes the victim address unreachable for all sources.
- Source-based RTBH (RFC 5635) combines loose uRPF with a discard route for the attacker source. It does not stop traffic with randomized spoofed sources.
- The well-known `BLACKHOLE` community `65535:666` (RFC 7999) for customer-to-provider triggering.
- An upstream must accept a `/32` only from a customer that owns the covering prefix.

**Project 4B: RTBH in FRR.**
1. Build a trigger router, two edge routers, and one upstream.
2. Configure a discard route for `192.0.2.1/32`.
3. Announce `203.0.113.50/32` with next hop `192.0.2.1` and `NO_EXPORT`.
4. Implement the provider-side `65535:666` policy with a prefix list.

Pass criteria:
- The route table on each edge router shows the blackhole.
- Traffic to the victim stops at the edge.
- Withdrawal restores forwarding within BGP convergence time.
- A `/32` outside the customer prefix is rejected.
- You test whether Linux `rp_filter=2` drops packets whose source resolves to a `blackhole` route, and you record the result.

### BGP Flowspec

Learn:
- RFC 8955 (IPv4) and RFC 8956 (IPv6). The SAFI values are 133 (unicast) and 134 (VPN).
- Match components: destination and source prefix, protocol, ports, ICMP type and code, TCP flags, packet length, DSCP, fragment.
- Actions in extended communities: traffic-rate (`0x8006`, where rate `0` discards), traffic-action, redirect, and traffic-marking.
- Validation (RFC 8955, section 6): the originator of the flow route must match the originator of the best unicast route for the destination.
- Enforcement is platform-specific and limited by hardware filter capacity.

**Project 4C: originate, validate, and enforce.**
1. Use ExaBGP or GoBGP to originate a rule that discards UDP source port 123 toward `203.0.113.50/32`.
2. Decode the NLRI in Wireshark.
3. Send a rule from a peer that does not originate the covering unicast route.
4. Enforce on a platform that installs Flowspec rules. FRR documents Flowspec support, but you must verify the feature status and enforcement backend for your installed version.

Pass criteria:
- The capture shows `MP_REACH_NLRI` with AFI 1 and SAFI 133.
- The non-originator rule is rejected.
- Generated matching traffic is dropped and non-matching traffic passes.
- You write a table that compares Flowspec and RTBH on granularity, collateral impact, and platform requirements.

If you cannot enforce Flowspec in your lab, the control-plane results still apply. Record that enforcement is untested.

## Phase 5: capstone

Build two sites and one provider edge. Store every configuration in Git, and require peer review for each change.

Requirements:
- Perimeter policy from the Phase 1 specification.
- IKEv2 site-to-site tunnel, with BGP over the tunnel.
- WireGuard for remote administration.
- 802.1X on access ports with VLAN assignment.
- TACACS+ for device administration.
- CIS Level 1 hardening on all Linux hosts.
- uRPF at the customer edge.
- RTBH and Flowspec triggered from one trigger host.

Deliverables:
- Threat model.
- Policy specification.
- Test matrix.
- Hardening report.
- Runbook for DDoS response, with rollback criteria for each mitigation.

Pass criteria, each tested in the lab:
- Spoofed packets are dropped at the edge.
- A UDP reflection flood is dropped by Flowspec.
- A `/32` flood is contained by RTBH.
- A rogue device on an access port is denied.
- A compromised administrator credential can be traced through TACACS+ accounting records.

Ask a second person to review the runbook and to run two of the tests without your help.
