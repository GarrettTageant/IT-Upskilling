Claude-generated roadmap:

This roadmap has eight phases in dependency order. Time synchronization comes first because DNSSEC signatures and X.509 certificates have validity intervals that validators evaluate against the local clock. DNS comes before DHCP because DHCP supplies DNS configuration and can update DNS records. PKI comes before ACME because ACME automates issuance by a CA whose behavior you must already understand.

## Rules for every phase

1. Meet every pass criterion without notes on protocol behavior. Man pages and RFCs are allowed.
2. Run the failure injection in each phase. A working configuration does not prove that you understand the protocol.
3. Break one component at a time. Write the predicted symptom before you trigger the fault. Record the observed symptom and the diagnosis.
4. Capture packets for every protocol that you configure. Compare each captured field with the RFC definition.
5. Keep the lab isolated from production networks.

| Phase | Topic | Planning estimate (6–8 hours per week) |
|---|---|---|
| 0 | Lab and tooling | 1–2 weeks |
| 1 | NTP and `chrony` | 1 week |
| 2 | DNS: authoritative and recursive | 2–3 weeks |
| 3 | DHCPv4, option 82, DHCPv6, DDNS | 3 weeks |
| 4 | DNSSEC and split-horizon DNS | 2–3 weeks |
| 5 | X.509 and PKI | 2–3 weeks |
| 6 | ACME | 2 weeks |
| 7 | Integration capstone | 2 weeks |

## Phase 0: Lab and tooling

You need working Linux administration skills before Phase 1. Phase 0 can overlap with your Linux coursework.

**Skills required:** `systemctl`, `journalctl`, `ip`, `ss`, `tcpdump`, package management, file permissions, and enabling IPv4 and IPv6 forwarding with `sysctl`.

**Lab specification**

| Component | Specification |
|---|---|
| Hypervisor | KVM/libvirt, Proxmox VE, VirtualBox, or VMware Workstation |
| Guest OS | Debian stable or Ubuntu LTS |
| Segments | `srv` (`10.10.0.0/24`), `cli-a` (`10.10.1.0/24`), `cli-b` (`10.10.2.0/24`). Add one IPv6 `/64` per segment from a ULA `/48` (RFC 4193). |
| `r1` | Linux router with one interface per segment. It forwards both IP versions and runs the DHCP relay. |
| `root1` | BIND serving the lab root zone `.` and the zone `test.` |
| `ns1`, `ns2` | BIND primary and secondary for `lab.test.` |
| `rec1` | Unbound validating recursive resolver |
| `dhcp1` | Kea DHCPv4, DHCPv6, and DDNS |
| `ca1` | OpenSSL CA and `step-ca` |
| `web1` | `nginx` and an ACME client |
| `c1`, `c2`, `c3` | Clients, at least one per client segment |
| Time servers | Four hosts running `chrony`. These can share VMs with the roles above. |

The hypervisor must not run its own DHCP service on the lab segments. A second DHCP server on a segment invalidates every DHCP test.

Use a private root zone instead of the real root. You then control every key and every delegation in the chain of trust. Name hosts under `.test` (RFC 6761).

**Project 0.1: Build the lab**

Pass criteria:
- `c1` reaches `dhcp1` through `r1` over both IP versions, with static addresses.
- A `tcpdump` capture on `r1` shows traffic on each segment.
- The hypervisor DHCP service is disabled on every lab segment.

## Phase 1: NTP and `chrony`

**Objectives**
- List the NTPv4 header fields: LI, VN, Mode, Stratum, Poll, Precision, Root Delay, Root Dispersion, Reference ID, and four timestamps (RFC 5905).
- Compute offset θ = ((T2 − T1) + (T3 − T4)) / 2 and delay δ = (T4 − T1) − (T3 − T2) from one captured exchange.
- Explain stratum assignment. Stratum 1 is a server attached to a reference clock. A server that synchronizes to stratum *n* advertises stratum *n* + 1. Stratum 16 means unsynchronized.
- Explain the symbols in `chronyc sources -v`: `*`, `+`, `-`, `x`, `~`, `?`.
- Distinguish a step from a slew. Explain the `makestep` directive.

**Project 1.1: Stratum hierarchy.** Configure three tiers of `chrony` servers and a client.

Pass criteria:
- `chronyc tracking` on each tier shows a stratum one greater than its upstream source.
- A packet capture shows the Reference ID of a stratum 2 or higher server equal to its upstream IPv4 address (RFC 5905).
- You compute θ and δ by hand from a captured exchange and match `chronyc ntpdata`.

**Project 1.2: Falseticker.** Configure the client with four servers. Set the clock of one server 10 minutes ahead, with the same advertised stratum as the others.

Pass criteria:
- `chronyc sources -v` marks the incorrect server `x`.
- You explain why a client needs at least three sources to detect one falseticker, and why four are recommended.

**Project 1.3: Failure injection**
- Block UDP 123 from the client to its preferred server. Record the failover and the `Reach` register.
- Offset the clock by 5 seconds after the `makestep` update limit is exhausted. Record whether `chronyd` steps or slews, and why.

## Phase 2: DNS, authoritative and recursive

**Objectives**
- Distinguish authoritative server, recursive resolver, forwarder, and stub resolver (RFC 9499).
- Identify the header flags AA, RD, RA, TC, AD, and CD, and the RCODEs NOERROR, NXDOMAIN, SERVFAIL, and REFUSED.
- Explain iterative resolution, referrals, glue, and TTL.
- Explain negative caching (RFC 2308). The negative TTL is the lesser of the SOA record TTL and the SOA MINIMUM field.
- Explain EDNS(0) (RFC 6891), truncation, and TCP fallback (RFC 7766).

**Project 2.1: Delegation and zone transfer.** Build `.` and `test.` on `root1`, and `lab.test.` on `ns1` and `ns2`. Include SOA, NS, A, AAAA, CNAME, MX, TXT, SRV, PTR, and CAA records. Add IPv4 and IPv6 reverse zones.

Pass criteria:
- `named-checkzone` reports no errors.
- `ns2` serial matches `ns1` after a zone change. NOTIFY (RFC 1996) and IXFR (RFC 1995) appear in the capture.
- A zone transfer without the TSIG key (RFC 8945) fails.
- `dig +norecurse` against `ns1` returns the `aa` flag.

**Project 2.2: Recursive resolver.** Configure Unbound on `rec1` with root hints that point to `root1`. Restrict `access-control` to the lab subnets.

Pass criteria:
- A capture on `rec1` shows the sequence for an uncached name: referral from `root1`, referral for `test.`, answer from `ns1`.
- A repeated query returns from cache with a decreasing TTL.
- You observe the effect of QNAME minimisation (RFC 9156) in the captured query names.

**Project 2.3: Role separation**
- `ns1` returns REFUSED for a name outside its zones.
- `rec1` returns REFUSED for a client outside the permitted subnets.

**Project 2.4: Failure injection**
- Create a lame delegation. Record the resolver behavior.
- Create a response larger than the EDNS buffer size. Show the TC flag and the TCP retry.
- Block TCP port 53 and record the failure.

## Phase 3: DHCP, option 82, DHCPv6, and DDNS

**Objectives**
- Describe the DHCPv4 exchange (Discover, Offer, Request, Acknowledgment), UDP ports 67 and 68, and the `giaddr` field (RFC 2131).
- Explain relay behavior. The relay sets `giaddr` and unicasts to the server. The server selects the subnet from `giaddr`.
- Explain option 82 (RFC 3046). The relay appends it. The server echoes it. The relay removes it before delivery to the client. Identify sub-option 1 (Agent Circuit ID), sub-option 2 (Agent Remote ID), RFC 3527 sub-option 5 (Link Selection), and RFC 5107 sub-option 11 (Server Identifier Override).
- Describe DHCPv6 messages, UDP ports 546 and 547, and multicast address `ff02::1:2` (RFC 8415). Distinguish DUID, IA_NA, IA_TA, and IA_PD.
- Explain stateful and stateless DHCPv6, and how the Router Advertisement M and O flags (RFC 4861) select between them. DHCPv6 does not supply a default router.

**Project 3.1: DHCPv4 through a relay.** Run `kea-dhcp4` on `dhcp1`. Configure the relay on `r1` for both client segments. Use `dhcrelay` from the ISC DHCP package (upstream maintenance ended in 2022) or a Cisco IOS device. Use a client other than ISC `dhclient`, such as `dhcpcd` or `systemd-networkd`.

Pass criteria:
- The capture shows a client broadcast, a relay unicast to the server with `giaddr` set, and a server reply to the relay.
- Each client receives an address from the subnet that matches its relay interface.

**Project 3.2: Option 82.** Enable option 82 on the relay.

Pass criteria:
- A capture on the `srv` segment shows sub-options 1 and 2. A capture on the client segment shows no option 82 in the reply.
- A Kea host reservation by `circuit-id` assigns a fixed address to `c1`.
- A Kea client class that uses `relay4[1].hex` selects a pool.
- You document what happens to a Discover with no option 82.

Failure injection: forge a client-originated option 82 with `scapy`. Document the relay policy (`dhcrelay -m` modes) or Cisco DHCP snooping behavior that stops it, and state the threat.

**Project 3.3: DHCPv6.** Run `kea-dhcp6` and `radvd`. Configure `cli-a` as stateful (M=1) and `cli-b` as stateless (M=0, O=1). Relay with `dhcrelay -6`.

Pass criteria:
- Captures show Solicit, Advertise, Request, and Reply, and show Relay-Forw and Relay-Repl with Interface-ID (option 18) or link-address.
- The default router comes from the Router Advertisement, not from DHCPv6.
- The stateless client sends an Information-Request and receives no address.
- You identify the client DUID in the Kea lease file.

Stretch: delegate an IA_PD prefix to `r1`.

**Project 3.4: Services and DDNS.** Deliver DNS and NTP through DHCP. In DHCPv4, use options 6, 15, 42, and 119. In DHCPv6, use options 23, 24, and 56. Configure `kea-dhcp-ddns` with a TSIG key to update `ns1`.

Pass criteria:
- `c1` uses the option 42 server in `chrony`.
- A lease creates A, AAAA, and PTR records, plus a DHCID record (RFC 4701). Lease expiry removes them.
- A wrong TSIG key causes a logged update failure.

## Phase 4: DNSSEC and split-horizon DNS

**Objectives**
- Explain DNSKEY, RRSIG, DS, NSEC, NSEC3, and NSEC3PARAM records.
- Trace the chain of trust from a trust anchor to a signed answer.
- Define the four validation states: secure, insecure, bogus, and indeterminate (RFC 4033). Explain the DO, AD, and CD bits.
- Select algorithms per RFC 8624 (13 and 15 are recommended). Set NSEC3 parameters per RFC 9276: zero iterations and an empty salt.
- Perform a ZSK pre-publication rollover and a KSK double-DS rollover (RFC 6781).

**Project 4.1: Sign the lab.** Sign `.`, `test.`, and `lab.test.` with BIND `dnssec-policy` (BIND 9.16 or later). Generate each DS record with `dnssec-dsfromkey` and publish it in the parent. Configure the root key as an Unbound `trust-anchor`.

Pass criteria:
- `delv -a <anchor-file> @rec1 www.lab.test A +vtrace` reports a fully validated answer.
- `dig +dnssec @rec1` returns the `ad` flag.
- An NXDOMAIN response contains the NSEC or NSEC3 records that prove the name does not exist.

**Project 4.2: Failure injection.** For each fault, predict the result, then record the RCODE, the `ad` flag, the Extended DNS Error (RFC 8914), and the result with `+cd`.
- Expired RRSIG, created by advancing the `rec1` clock. This links to Phase 1.
- DS record with a wrong digest in the parent.
- DS record removed from the parent. The zone becomes insecure.
- A manual KSK rollover with no validation outage.

**Project 4.3: Split-horizon.** Configure BIND `view` statements or Unbound `view:` clauses. The `internal` view returns private addresses. The `external` view returns different data.

Pass criteria:
- `dig -b <source-address>` returns different answers from the two views.
- You demonstrate the effect of reordering BIND views. BIND selects the first view whose `match-clients` matches.
- Recursion is enabled only in the internal view.
- You reproduce the DNSSEC conflict. A validating resolver that has a DS record from the parent treats an unsigned internal answer as bogus. Resolve it by signing both views, or by an Unbound `domain-insecure` or BIND `rndc nta` entry. Document the trade-off.

## Phase 5: X.509 and PKI

**Objectives**
- Identify each field of an X.509 v3 certificate (RFC 5280): `tbsCertificate`, `signatureAlgorithm`, `signatureValue`, and the extensions `basicConstraints`, `keyUsage`, `extendedKeyUsage`, `subjectAltName`, `nameConstraints`, `authorityKeyIdentifier`, and `subjectKeyIdentifier`.
- Distinguish path building (RFC 4158) from path validation (RFC 5280, section 6).
- Explain hostname verification. Clients match the SAN `dNSName` or `iPAddress` entry, not the Common Name (RFC 9525).
- Explain revocation by CRL, OCSP (RFC 6960), and OCSP stapling (RFC 6066).

**Project 5.1: Three-tier hierarchy.** Create a self-signed offline root with `pathlen:1`, an intermediate with `pathlen:0`, and a leaf for `web1.lab.test` with a SAN. Use ECDSA P-256 keys.

Pass criteria:
- `openssl verify -CAfile root.pem -untrusted intermediate.pem leaf.pem` succeeds.
- You explain every field in `openssl x509 -noout -text` for all three certificates.

**Project 5.2: Deployment.** Configure `nginx` with the leaf and the intermediate. Do not include the root. Install the root on the client with `update-ca-certificates`.

Pass criteria:
- `openssl s_client -connect web1.lab.test:443 -servername web1.lab.test -verify_return_error` ends with `Verify return code: 0 (ok)`.
- You explain why the server must send the intermediate and why it need not send the root.

**Project 5.3: Failure matrix.** For each case, construct the certificate, predict the error, run `openssl verify` or `s_client`, and record the exact error string.
- Expired and not yet valid (use `-attime`).
- Missing SAN, and hostname mismatch.
- `serverAuth` missing from `extendedKeyUsage`.
- Intermediate with `CA:FALSE`.
- Path length exceeded.
- `nameConstraints` violation.
- Intermediate not sent by the server.
- Unknown issuer.

**Project 5.4: Revocation.** Generate a CRL with `openssl ca -gencrl`. Verify with `-crl_check`. Run an OCSP responder with `openssl ocsp`.

Pass criteria: a revoked leaf fails verification, and the error string matches your prediction.

Stretch: configure NTS on `chrony` (RFC 8915, NTS-KE on TCP port 4460) with a certificate from this CA.

## Phase 6: ACME

**Objectives**
- Trace the RFC 8555 sequence: directory, `newNonce`, `newAccount`, `newOrder`, authorization, challenge, finalize with CSR, certificate download.
- Explain JWS request protection: the `nonce`, `url`, and `jwk` or `kid` header fields, and POST-as-GET.
- State the order states: pending, ready, processing, valid, invalid.
- Compute the key authorization: `token || "." || base64url(SHA-256(JWK thumbprint))`.
- Distinguish the challenges.

| Challenge | Port | Proof |
|---|---|---|
| `http-01` | TCP 80 | The key authorization at `/.well-known/acme-challenge/<token>` |
| `dns-01` | UDP and TCP 53 | A TXT record at `_acme-challenge.<name>` containing `base64url(SHA-256(keyAuthorization))` |
| `tls-alpn-01` (RFC 8737) | TCP 443 | An `acmeIdentifier` certificate extension, with ALPN `acme-tls/1` |

Wildcard identifiers require `dns-01`.

- Explain CAA records (RFC 8659) and the ACME parameters `accounturi` and `validationmethods` (RFC 8657). A CA must validate DNSSEC when it looks up CAA records.

**Project 6.1: ACME CA.** Configure `step-ca` with an ACME provisioner that issues from your Phase 5 intermediate.

Pass criteria: `curl` retrieves the directory object, and a `HEAD` request to `newNonce` returns a `Replay-Nonce` header.

**Project 6.2: Issuance.** Obtain a certificate with each method. Use `certbot --server` for `http-01`. Use the `certbot-dns-rfc2136` plugin for `dns-01`, with a TSIG key restricted to `_acme-challenge` names by BIND `update-policy`. Use `lego` or `acme.sh` for `tls-alpn-01`, because `certbot` does not implement it. Also obtain a wildcard certificate.

Pass criteria:
- Every certificate chains to your Phase 5 root.
- You match the challenge exchange in the CA log with the capture.

Stretch: compute the `dns-01` TXT value by hand from the token and your account key.

**Project 6.3: Automation.** Set the certificate lifetime to 24 hours. Configure a renewal timer and a deploy hook that reloads `nginx`.

Pass criteria: three consecutive automatic renewals complete with no failed client connection.

Reason for the exercise: the CA/Browser Forum Baseline Requirements limit publicly trusted certificate validity to 200 days for certificates issued on or after 15 March 2026, 100 days from 15 March 2027, and 47 days from 15 March 2029. Manual renewal does not scale to those lifetimes.

**Project 6.4: Failure injection.** Record the HTTP status and the ACME problem type for each fault.
- TXT record not published.
- Port 80 blocked.
- CAA record `issue ";"`, then a CAA record with the wrong `accounturi`.
- Bogus DNSSEC data for `lab.test.`, with the CA using the validating resolver `rec1`.
- Wrong account key.
- Rejected nonce (`urn:ietf:params:acme:error:badNonce`) and the client retry.

Stretch: write an ACME client that completes `http-01` against `step-ca` without a client library for ACME. A JOSE library is allowed.

## Phase 7: Integration capstone

**Project 7.1: Cold start.** A freshly booted client on `cli-a` must complete every item below with no manual action.

1. Obtain IPv4 and IPv6 configuration through the relay: address, DNS server, NTP server, and search domain.
2. Synchronize its clock with `chrony` from the DHCP-supplied server.
3. Resolve `web1.lab.test` through `rec1` with DNSSEC validation, and receive the view for its subnet.
4. Connect to `https://web1.lab.test` with certificate validation against your root.
5. Show a certificate issued and renewed by ACME.

Write a run book that lists the verification command and the expected output for each step.

**Project 7.2: Fault injection exam.** Apply each fault in a fresh lab state. Write the diagnostic procedure before the fix. Identify the failing layer from command output only.

1. Offset the clock of the `rec1` host by 10 minutes.
2. Remove the DS record for `lab.test.` from `test.`.
3. Disable option 82 insertion on the relay.
4. Remove the intermediate from the `nginx` certificate file.
5. Add a CAA record that forbids your CA.
6. Swap the order of the two BIND views.
7. Set the Router Advertisement M flag on the segment that requires stateless DHCPv6.
8. Replace the DDNS TSIG key on `ns1`.

Fault 1 affects DNSSEC validation, TLS validation, and ACME issuance simultaneously. Identify the single root cause, not three separate problems.

**Pass criteria for the roadmap:** Project 7.1 completes on a clean boot. All eight faults are diagnosed with command-output evidence and no outside help beyond man pages and RFCs.
