Claude-generated roadmap:

# Packet analysis and troubleshooting roadmap

The roadmap has nine phases. Each phase has required knowledge, one project, and pass criteria. All projects run in one Linux network namespace lab. A phase passes only when every criterion is met with captured evidence.

Prerequisites: working use of Bash, `ip`, `ss`, network namespaces, and `nftables`. Phase 0 exercises all of them. Capture traffic only on networks and hosts that you own or have written authorization to monitor. Capture files can contain credentials and customer data, so handle them as sensitive.

| Phase | Topic | Estimated duration at 6 hours per week |
|---|---|---|
| 0 | Lab and capture position | 1 week |
| 1 | Capture filters and display filters | 1 week |
| 2 | ARP and DHCP | 1 week |
| 3 | DNS | 1 week |
| 4 | TCP behavior | 2 weeks |
| 5 | TLS handshakes | 1 week |
| 6 | MTU, MSS, and path MTU discovery | 2 weeks |
| 7 | `mtr` and path analysis | 1 week |
| 8 | Layered fault isolation and blind exam | 2 weeks |

## Phase 0: Lab and capture position

**Required knowledge**

- The capture point determines what the capture shows. On Linux, an ingress capture records frames before netfilter evaluates them. An egress capture records frames after the netfilter output and postrouting hooks. A frame that an egress rule drops never appears in the capture. A frame that an ingress rule drops does appear.
- Generic receive offload (GRO), generic segmentation offload (GSO), and TCP segmentation offload (TSO) merge segments before the capture point. A capture can therefore contain packets larger than the interface MTU. Such a packet does not prove an MTU fault.
- Run `tshark` with `-n`. Name resolution generates DNS queries that contaminate the capture.
- Hash each capture file with `sha256sum`. Record tool versions with `tshark --version`.

**Project: build the lab**

The topology is `client` — `r1` — `r2` — `server`. Run the script as root.

```bash
#!/usr/bin/env bash
set -euo pipefail
for ns in client r1 r2 server; do ip netns add "$ns"; ip -n "$ns" link set lo up; done
link() { # NS1 IF1 ADDR1 NS2 IF2 ADDR2
  ip link add "$2" type veth peer name "$5"
  ip link set "$2" netns "$1"; ip link set "$5" netns "$4"
  ip -n "$1" addr add "$3" dev "$2"; ip -n "$1" link set "$2" up
  ip -n "$4" addr add "$6" dev "$5"; ip -n "$4" link set "$5" up
}
link client c-r1   10.0.1.2/24  r1     r1-c   10.0.1.1/24
link r1     r1-r2  10.0.12.1/24 r2     r2-r1  10.0.12.2/24
link r2     r2-s   10.0.2.1/24  server s-r2   10.0.2.2/24
ip -n client route add default via 10.0.1.1
ip -n server route add default via 10.0.2.1
ip -n r1 route add 10.0.2.0/24 via 10.0.12.2
ip -n r2 route add 10.0.1.0/24 via 10.0.12.1
for ns in r1 r2; do ip netns exec "$ns" sysctl -qw net.ipv4.ip_forward=1; done
for p in client:c-r1 r1:r1-c r1:r1-r2 r2:r2-r1 r2:r2-s server:s-r2; do
  ip netns exec "${p%%:*}" ethtool -K "${p##*:}" tso off gso off gro off
done
```

**Pass criteria**

1. Send `ping -c 3 10.0.2.2` from `client` while capturing on all six interfaces.
2. Produce a table of the IP TTL at each capture point.
3. Explain each TTL value. The expected values are 64, 64, 63, 63, 62, and 62.

## Phase 1: Capture filters versus display filters

**Required knowledge**

| Property | Capture filter | Display filter |
|---|---|---|
| Syntax | `pcap-filter(7)` | Wireshark display filter language |
| Evaluation time | During capture, before storage | After dissection, on stored frames |
| Excluded packets | Lost permanently | Remain in the file |
| Field scope | Byte offsets and header fields that libpcap knows | Any dissected field, including `tcp.analysis.retransmission` |
| `tshark` option | `-f` | `-Y` |

- On Linux, libpcap attaches the compiled classic BPF program to the capture socket. The kernel discards non-matching packets before they reach user space.
- A port filter such as `tcp port 80` excludes IPv4 fragments other than the first fragment, because those fragments carry no TCP header.
- The BPF primitive `greater N` matches length greater than or equal to N, not strictly greater than N.
- The `vlan` keyword shifts the byte offsets for the filter expressions that follow it.
- The meaning of `!=` on fields that occur more than once in a frame, such as `ip.addr`, differs between Wireshark versions. Verify the behavior in your installed version.
- Best practice: you should capture broadly into a ring buffer (`-b duration:60 -b files:10`) and filter on read. A wrong capture filter destroys evidence that cannot be recovered.

**Project: filter equivalence**

Run `python3 -m http.server 8080` in `server`. Capture `curl` requests from `client`.

1. Write each of the following as a capture filter and as a display filter, or state that one form is impossible:
   - all traffic to or from `10.0.2.2`
   - TCP SYN segments without ACK
   - ICMP destination unreachable
   - UDP port 53 traffic
   - frames longer than 1400 bytes
   - DNS responses with RCODE 3 over UDP (use byte offsets, for example `udp[11] & 0xf`)
   - TLS ClientHello messages that carry a specific SNI
2. Verify that each pair selects the same packets. Compare packet counts with `capinfos` and `tshark -r file -Y filter | wc -l`.
3. Cause a lost-evidence case. On `r2`, add an `nft` rule that rejects traffic to port 8080 with `icmp type admin-prohibited`. Capture with `tcp port 8080` and show that the ICMP error is absent. Repeat the capture with a broader filter and show that the ICMP error is present.
4. Extract a table with `tshark -r file -T fields -e frame.number -e frame.time_relative -e ip.src -e ip.dst -e tcp.flags.str`.

**Pass criteria:** every filter pair is verified by count, the impossible cases are identified correctly, and the lost-evidence case is demonstrated.

## Phase 2: ARP and DHCP

**Required knowledge**

- ARP (RFC 826) uses EtherType `0x0806`. A request goes to `ff:ff:ff:ff:ff:ff` with a zero target hardware address. A reply is unicast.
- An ARP probe has sender IP `0.0.0.0` (RFC 5227). A gratuitous ARP has sender IP equal to target IP.
- A host resolves only the on-link next hop. The client ARPs for the gateway address, not for a remote destination.
- Linux neighbor states: `INCOMPLETE`, `REACHABLE`, `STALE`, `DELAY`, `PROBE`, `FAILED`, `PERMANENT`, `NOARP`. Use `ip neigh show`.
- DHCP (RFC 2131) uses UDP port 67 for the server and port 68 for the client. A DISCOVER is sent from `0.0.0.0:68` to `255.255.255.255:67`.
- DHCP message types in option 53: DISCOVER 1, OFFER 2, REQUEST 3, DECLINE 4, ACK 5, NAK 6, RELEASE 7, INFORM 8. The `xid` field correlates the messages of one exchange.
- Option 54 identifies the server. Option 50 carries the requested address. Option 51 carries the lease time.
- Renewal timers: T1 is 0.5 times the lease time, and T2 is 0.875 times the lease time. A client sends a unicast REQUEST at T1 and a broadcast REQUEST at T2.
- Without a response, a client retransmits with delays that start at 4 seconds and double up to 64 seconds, randomized by plus or minus 1 second.
- A router does not forward a limited broadcast. A relay agent forwards DHCP messages and sets `giaddr`.

**Project A: ARP**

1. Flush the neighbor table with `ip neigh flush all` in `client`. Ping the gateway and capture the ARP exchange. Record every ARP field.
2. Assign `10.0.1.2` to a second namespace on LAN A. Show the duplicate in the capture. Two ARP replies carry different sender MAC addresses for one sender IP.
3. Set `r1-c` down in `r1`. Show the `INCOMPLETE` to `FAILED` transition and the ping error that the client generates.
4. Change the client address to `10.0.1.2/8` and remove the default route. The client now ARPs for `10.0.2.2` directly. Show that no reply arrives. Then set `net.ipv4.conf.r1-c.proxy_arp=1` in `r1` and show that `r1` replies with its own MAC address.

**Project B: DHCP**

Run `dnsmasq` in `r1` with `--dhcp-range` and `--log-dhcp`. Use a client such as `dhclient` or `udhcpc` in `client`.

1. Capture a complete lease. Tabulate the four messages with `xid`, source, destination, and options.
2. Stop the server. Measure the DISCOVER retransmission intervals with `frame.time_delta_displayed`.
3. Exhaust the address pool. Show the missing OFFER.
4. Run a second DHCP server on the same LAN. Identify both servers by option 54.
5. Move the server behind a router. Show that the broadcast does not cross the router. Add a relay agent and show `giaddr`.

**Pass criteria:** each fault is identified from the capture alone, with frame numbers.

## Phase 3: DNS

**Required knowledge**

- Header fields: ID, QR, opcode, AA, TC, RD, RA, RCODE (RFC 1035).
- RCODE values: NOERROR 0, FORMERR 1, SERVFAIL 2, NXDOMAIN 3, NOTIMP 4, REFUSED 5.
- Queries use UDP port 53 first. A response with TC set requires a retry over TCP port 53 (RFC 7766).
- Without EDNS0, a UDP response is limited to 512 bytes. With EDNS0 (RFC 6891), the sender advertises its UDP payload size. 1232 bytes is the value that DNS Flag Day 2020 recommends.
- The `dig` defaults are `+time=5` and `+tries=3`. The glibc resolver defaults are `timeout:5` and `attempts:2`.
- DNS over TLS (port 853) and DNS over HTTPS (port 443) do not appear in the `dns` dissector.
- On a host with `systemd-resolved`, applications query the stub at `127.0.0.53`. The upstream query appears on the physical interface.
- Display filter fields: `dns.flags.rcode`, `dns.flags.truncated`, `dns.time`.

**Project**

Run `dnsmasq` in `server` with `--host-record` and a `--txt-record` whose response exceeds 512 bytes.

1. Capture one successful query. Record the query ID, flags, and `dns.time`.
2. Query a nonexistent name. Show NXDOMAIN.
3. Drop UDP port 53 on the server input chain. Show that the query arrives at the server capture, that no response leaves, and that the client retransmits at the timeout interval.
4. Run `dig +noedns +ignore big.lab TXT`. Show TC set. Run `dig +tcp` and show the TCP exchange.
5. Compute the resolver latency distribution for 100 queries with `tshark -T fields -e dns.time`.

**Pass criteria:** you distinguish no response, NXDOMAIN, and truncation from the capture alone, and you state the observed retransmission intervals.

## Phase 4: TCP behavior

**Required knowledge**

- Handshake: SYN, SYN-ACK, ACK. Each side chooses an initial sequence number. The SYN segments carry options: MSS (kind 2), window scale (kind 3), SACK permitted (kind 4), and timestamps (kind 8).
- Window scaling (RFC 7323): the shift count is 0 to 14. Scaling is in effect only if both SYN segments carry the option. The window field of a SYN segment is never scaled. The maximum window is 65,535 × 2^14 = 1,073,725,440 bytes.
- If a capture omits the SYN segments, Wireshark cannot determine the scale factor. It reports the factor as unknown (-1), and the calculated window size is not reliable.
- The retransmission timeout (RTO, RFC 6298) doubles after each consecutive timeout. Fast retransmit starts after three duplicate ACKs (RFC 5681). Selective acknowledgment (RFC 2018) reports received blocks.
- Wireshark analysis flags: `tcp.analysis.retransmission`, `.fast_retransmission`, `.spurious_retransmission`, `.duplicate_ack`, `.lost_segment`, `.out_of_order`, `.zero_window`, `.zero_window_probe`, `.window_full`. Wireshark computes these from the captured stream. They are not fields on the wire.
- A sequence gap means the segment was lost before the capture point or was not captured.
- Linux defaults: `net.ipv4.tcp_syn_retries` is 6, and the minimum RTO is 200 ms.
- Linux tools: `ss -ti` shows `rto`, `rtt`, `mss`, `cwnd`, and `retrans`. `nstat -az TcpRetransSegs` shows the system counter.

**Project A: handshake outcomes**

Capture four connection attempts to port 8080: listener present, no listener, `nft` drop, and `nft` reject with ICMP.

1. For each attempt, record the frames, flags, and options.
2. Measure the SYN retransmission intervals in the drop case. They start at 1 second and double.

**Project B: loss and retransmission**

Add `tc qdisc add dev r1-r2 root netem loss 2%` in `r1`. Run `iperf3 -c 10.0.2.2 -n 50M` from `client`. Capture at `c-r1` and `s-r2` at the same time.

1. Find one segment that appears in the client capture and not in the server capture.
2. Classify each retransmission as timeout-driven or fast retransmit.
3. Reconcile the capture counts with `ss -ti` and `nstat`.
4. Replace the loss with `loss 100%` for 5 seconds. Measure the RTO doubling.

**Project C: window scaling and receiver limits**

Add `delay 50ms` with `netem` on both `r1-r2` and `r2-r1`. The round-trip time is 100 ms.

1. Predict the throughput ceiling with scaling disabled in `client` (`net.ipv4.tcp_window_scaling=0`) for a transfer toward the client: 65,535 bytes ÷ 0.1 s = 655,350 bytes per second, which is 5.24 Mbit/s. Measure it with `iperf3 -R`.
2. Repeat with scaling enabled. Plot `tcp.analysis.bytes_in_flight` against the advertised window.
3. Run a server application that stops reading. Capture the zero window, the zero-window probes, and the window update.
4. Start a capture after the connection is established. Show the unknown scale factor.

**Pass criteria:** each measured value matches its prediction, or you explain the difference with evidence.

## Phase 5: TLS handshakes

**Required knowledge**

- TLS 1.2 (RFC 5246): ClientHello, ServerHello, Certificate, ServerKeyExchange, ServerHelloDone, ClientKeyExchange, ChangeCipherSpec, Finished. The Certificate message is visible in the capture unless the session resumes.
- TLS 1.3 (RFC 8446): after ServerHello, all handshake messages are encrypted. These are EncryptedExtensions, Certificate, CertificateVerify, and Finished. The ClientHello `legacy_version` is `0x0303`. The negotiated version is in the `supported_versions` extension.
- The SNI is extension `server_name` (type 0). ALPN is a separate extension.
- Alert descriptions: handshake_failure 40, bad_certificate 42, certificate_expired 45, unknown_ca 48, protocol_version 70, unrecognized_name 112.
- A TLS 1.2 alert that precedes ChangeCipherSpec is plaintext. A TLS 1.3 alert that follows the handshake keys is encrypted.
- Decryption uses an NSS key log file. Set `SSLKEYLOGFILE` for `curl`, or use `openssl s_client -keylogfile`. In `tshark`, use `-o tls.keylog_file:keys.log`.
- `openssl s_client` continues after a verification failure unless you add `-verify_return_error`.

**Project**

Create a self-signed certificate with a SAN. Run `openssl s_server -accept 4433 -www` in `server`.

1. Capture `openssl s_client -connect 10.0.2.2:4433 -servername server.lab -tls1_2` and the same command with `-tls1_3`. Tabulate the visible messages for each version.
2. Decrypt the TLS 1.3 capture with a key log. Identify the encrypted messages.
3. Cause these faults and identify each alert from the capture: `curl` without the lab CA, a server restricted to TLS 1.2 with a TLS 1.3-only client, and an SNI mismatch.
4. Drop the ClientHello at `r2`. Show the TCP retransmissions of the ClientHello.
5. Measure the TCP handshake time and the TLS handshake time separately.

**Pass criteria:** for each fault, you state the last successful handshake flight and the layer that failed.

## Phase 6: MTU, MSS, and path MTU discovery

**Required knowledge**

- MTU is the largest IP packet that a link carries. MSS is the largest TCP payload per segment, excluding IP and TCP headers.
- MSS is MTU − 20 − 20 = 1460 for IPv4 and MTU − 40 − 20 = 1440 for IPv6. The TCP timestamp option uses 12 bytes of each segment, so the payload is then 1448 bytes while the advertised MSS stays 1460.
- At MTU 1500, the largest untagged Ethernet II frame has `frame.len` 1514 when the capture excludes the frame check sequence.
- IPv4 path MTU discovery (RFC 1191): the sender sets DF. A router that cannot forward the packet returns ICMP type 3 code 4 with the next-hop MTU.
- IPv6 (RFC 8201): routers do not fragment. A router returns ICMPv6 type 2, Packet Too Big. The minimum IPv6 MTU is 1280.
- A path MTU discovery black hole occurs when a filter drops the ICMP message. Small packets succeed and large packets stall. Packetization layer discovery (RFC 4821) works without ICMP. In Linux it is `net.ipv4.tcp_mtu_probing` (0 off, 1 on black-hole detection, 2 always).
- `ping -M do -s 1472` sends a 1500-byte IPv4 packet. `tracepath` reports the path MTU.

**Project**

Set the MTU of `r1-r2` and `r2-r1` to 1400. The client and the server keep MTU 1500.

1. Run `ping -M do -s 1472 10.0.2.2` and `-s 1372` from `client`. Show the ICMP message and the next-hop MTU. Run `tracepath -n 10.0.2.2`.
2. Download a large file from the server with `curl`. Capture at `s-r2`. Show the full-size segment, the ICMP type 3 code 4 message, and the smaller retransmission. Show the cached route MTU with `ip route get 10.0.1.2` in `server`.
3. Drop ICMP destination unreachable in the server input chain. Show that the ICMP message still appears in the capture because the capture precedes netfilter. Show that the large transfer stalls and the server retransmits full-size segments with backoff.
4. Apply each remediation separately and verify it with a second capture:
   - allow the ICMP message
   - clamp the MSS on `r1` and `r2` (the SYN MSS must become 1360)
   - set `net.ipv4.tcp_mtu_probing=1` in `server`
5. Repeat steps 1 to 3 with IPv6. State each difference.

**Pass criteria:** you identify the black hole from a capture at one point, and you show the path MTU in the evidence.

## Phase 7: `mtr` and path analysis

**Required knowledge**

- `mtr` sends probes with increasing TTL. Each hop that decrements the TTL to zero returns ICMP time exceeded (type 11). The destination returns an echo reply.
- Useful options: `-r` report, `-w` wide, `-c` count, `-n` no DNS, `-b` names and addresses, `-z` AS numbers, `-T` TCP, `-P` port, `-u` UDP.
- Loss at an intermediate hop that does not continue to later hops is a rate limit on ICMP generation. Loss that begins at a hop and continues through the destination is forwarding loss.
- The first hop that shows forwarding loss is the hop beyond the lossy link.
- `mtr` measures round trips per hop. Run it from both ends to separate forward-path loss from reverse-path loss.

**Project**

1. Run `mtr -rwn -c 100 10.0.2.2` in `client` as the baseline.
2. Add `netem loss 20%` on `r1-r2`. Run `mtr` from `client` and from `server`. Name the faulty link and the direction.
3. Remove the loss. On `r1`, rate-limit ICMP time-exceeded output below the probe rate. Show loss at hop 1 and none at the destination.
4. Run `mtr -T -P 8080` and compare it with the ICMP result.

**Pass criteria:** you classify the two loss patterns correctly and name the link, not only the hop.

## Phase 8: Layered fault isolation and blind exam

**Isolation order**

Isolate in the order of connection dependency. Each step has an evidence source.

| Step | Layer or service | Evidence |
|---|---|---|
| 1 | Link | `ethtool IF`, `ip -s link`, optical power from `ethtool -m IF` |
| 2 | Address assignment | `ip addr`, capture filter `udp port 67 or udp port 68` |
| 3 | Next-hop resolution | `ip neigh`, capture filter `arp` |
| 4 | Route and path | `ip route get`, `ping`, `mtr` |
| 5 | Path MTU | `ping -M do -s`, `tracepath`, ICMP type 3 code 4 |
| 6 | Name resolution | `dig`, `dns` capture |
| 7 | Transport | `ss -tan`, handshake capture, `tcp.analysis.*` |
| 8 | TLS | `openssl s_client`, handshake capture, alerts |
| 9 | Application | `curl -v`, HTTP status |

**Evidence standard**

Every report must contain these items:

1. Symptom with the exact command and output.
2. Environment with tool versions.
3. Capture points, capture filters, file names, and SHA-256 hashes.
4. Timeline in UTC with frame numbers.
5. Hypotheses, each with a test that can refute it.
6. Findings, each with a frame number and the display filter that finds it.
7. Root cause: one layer and one mechanism.
8. Fix.
9. Verification: a second capture with the same filter that shows the symptom gone.

A conclusion without a frame number or command output does not count as evidence.

**Project: blind fault exam**

Write `fault.sh`. The script selects one fault with `shuf`, applies it, and does not print which fault it applied. Include at least 10 faults from Phases 2 to 7. Store the choice as a SHA-256 hash for later verification.

1. Run 10 trials. Produce one report per trial in the format above.
2. Unblind the trial only after the report is complete.

**Pass criteria:** at least 9 of 10 root causes are correct, and every report meets the evidence standard.
