Claude-generated roadmap:

# Linux networking and administration

## Ground rules

1. Use a virtual machine with snapshots. Debian 12 or later and Ubuntu Server 24.04 LTS both provide the required tools.
2. Install these packages: `jq`, `tcpdump`, `ethtool`, `iperf3`, `shellcheck`, `gawk`, `nftables`. The `iproute2` package is installed by default.
3. Make firewall and `sshd` changes from the hypervisor console, not from an SSH session. An incorrect rule or setting locks out the remote session.
4. Type every command. Do not paste solutions.
5. A project passes only when every pass criterion succeeds as command output. A script that ran without errors does not pass.
6. Network configuration made with `ip` is volatile and does not survive a reboot. Phase 9 handles persistence.
7. Use manual pages as the primary reference: `bash(1)`, `sed(1)`, `jq(1)`, `systemd.service(5)`, `journalctl(1)`, `sshd_config(5)`, `ip(8)`, `ss(8)`, `ip-netns(8)`, `bridge(8)`, `nft(8)`. For bonding, use the kernel document `Documentation/networking/bonding.rst`.

CNA 130 can overlap Phases 1 to 3, and CNA 210 may overlap Phase 8. Compare each syllabus against this roadmap and combine work where the material matches.

| Phase | Topic | Estimated duration at 8 hours per week |
|---|---|---|
| 1 | Bash, `sed`, `awk`, `jq` | 3 weeks |
| 2 | `systemd`, `journalctl` | 2 weeks |
| 3 | SSH key authentication | 2 weeks |
| 4 | `ip`, `ss` | 2 weeks |
| 5 | Network namespaces | 2 weeks |
| 6 | VLAN subinterfaces | 2 weeks |
| 7 | LACP bonding | 2 weeks |
| 8 | `nftables` | 3 weeks |
| 9 | Capstone | 3 weeks |

The durations are estimates for a learner with limited Linux experience. The total is approximately 21 weeks.

## Phase 1: Shell and text processing

**Learn**
- Bash: quoting and word splitting, parameter expansion (`${var:-default}`, `${var%suffix}`), indexed and associative arrays, `[[ ]]` compared with `[ ]`, exit status, functions, `trap`, `getopts`, here-documents, process substitution, `mktemp`.
- `set -euo pipefail`: learn each option and its exceptions. For example, `set -e` is ignored inside conditions.
- `sed`: the `s` command and its flags, line addresses, `-n` with `p`, `-E`, and `-i` with a backup suffix.
- `awk`: pattern-action pairs, `$1` to `$NF`, `-F`, `BEGIN` and `END`, associative arrays, `printf`. On Debian and Ubuntu the default `awk` is `mawk`. Use `gawk` explicitly for gawk-only features.
- `jq`: path expressions, `select`, `map`, `group_by`, `--arg`, `--argjson`, `-r`, `@csv`.

**Project 1: interface report tool.** Write `ifreport.sh [-j] [-i IFACE]`. It reports, for each interface, the name, operational state, MAC address, IPv4 addresses, and receive and transmit byte counters.
- Take state and addresses from `ip -j addr` and process them with `jq`.
- Take counters from `/proc/net/dev` and process them with `awk`.
- With `-j`, emit valid JSON built with `jq`, not string concatenation.
- Add a separate `sed` command that replaces the address of one named host in a copy of `/etc/hosts` and keeps a backup.

**Pass criteria**
- `shellcheck` reports no findings.
- The script exits with status 2 for an invalid option and status 1 for a nonexistent interface.
- `-j` output passes `jq empty`.
- Counters in two consecutive runs do not decrease.
- The `sed` command changes only the line for the named host.

## Phase 2: `systemd` and `journalctl`

**Learn**
- Unit types: service, timer, target, path, socket.
- `Type=` values: `simple`, `exec`, `forking`, `oneshot`, `notify`.
- Ordering (`After=`) and dependency (`Wants=`, `Requires=`) are independent settings.
- `Restart=`, `RestartSec=`, `StateDirectory=`, `OnCalendar=`, `Persistent=`.
- `systemctl status|cat|edit|enable --now|daemon-reload|list-timers|--failed`. Use drop-ins from `systemctl edit` instead of editing vendor units.
- `systemd-analyze verify` and `systemd-analyze security`.
- Hardening: `NoNewPrivileges=`, `ProtectSystem=`, `PrivateTmp=`, `DynamicUser=`.
- `journalctl`: `-u`, `-b`, `-b -1`, `-p`, `--since`, `--until`, `-f`, `-k`, `-o json`, `-o cat`, field matches such as `_SYSTEMD_UNIT=`, `--disk-usage`, `--vacuum-time`.
- Storage: with the default `Storage=auto` in `journald.conf`, the journal is persistent only if `/var/log/journal` exists. Without persistence, `-b -1` returns nothing.

**Project 2: scheduled service.** Run `ifreport.sh` as `ifreport.service` (`Type=oneshot`) and trigger it every 5 minutes with `ifreport.timer` (`Persistent=true`). The service writes a JSON snapshot into a directory that `StateDirectory=` creates. Add hardening directives until `systemd-analyze security` reports a lower exposure score than the unhardened unit. Record each directive you added.

**Pass criteria**
- `systemctl list-timers` shows the timer with a next run time.
- Make the script exit with status 1. The unit appears in `systemctl --failed`, and `journalctl -u ifreport -p err` shows the failure.
- Write one `journalctl` command for each of these: warnings and above from the previous boot, kernel messages from the last 10 minutes, and the last run of the unit as JSON with `MESSAGE` and `__REALTIME_TIMESTAMP` extracted with `jq`.

## Phase 3: SSH key authentication

**Learn**
- `ssh-keygen -t ed25519`, `authorized_keys`, and the permission requirements that `StrictModes yes` enforces. Group and others must not have write access to the home directory, `~/.ssh`, or `authorized_keys`.
- `ssh-agent`, `ssh-add`, `~/.ssh/config`, `known_hosts`, host key fingerprints (`ssh-keygen -lf`).
- `sshd_config`: `PasswordAuthentication`, `KbdInteractiveAuthentication`, `PermitRootLogin`, `AllowUsers`, `Match`.
- Validate with `sshd -t` and inspect the effective configuration with `sshd -T`.
- Drop-in files in `/etc/ssh/sshd_config.d/`: `sshd` uses the first value it obtains for each keyword. On Debian and Ubuntu, the `Include` line is at the top of the main file, so drop-ins take precedence.
- Debugging: `ssh -vvv` on the client and the journal on the server. The unit is `ssh.service` on Debian and Ubuntu and `sshd.service` on the RHEL family.
- Forwarding: `-L`, `-R`, `-D`, `-J`.

**Project 3: hardened access between two hosts.** Use two virtual machines. On the server, configure key-only authentication for user `ops`, disable root login, and restrict logins with `AllowUsers ops`. On the client, create a dedicated passphrase-protected key and a `~/.ssh/config` alias. Start `python3 -m http.server --bind 127.0.0.1 8000` on the server and reach it through `-L`.

**Pass criteria**
- `ssh -o PubkeyAuthentication=no ops@HOST` fails with `Permission denied (publickey)`.
- `ssh root@HOST` fails.
- `sshd -T` shows `passwordauthentication no`.
- `curl` against the local end of the tunnel returns the server's loopback-only page.
- The failed attempts appear in the server journal.
- Introduce a syntax error in a drop-in file. `sshd -t` detects the error before any reload.

## Phase 4: `ip` and `ss`

**Learn**
- `ip link|addr|route|neigh|rule`, and the options `-br`, `-j`, `-d`, `-s`.
- Route selection: longest prefix match. `ip route get ADDRESS` shows the selected route.
- Routing tables: `local` (255), `main` (254), `default` (253). Policy routing with `ip rule`.
- Neighbor states: `REACHABLE`, `STALE`, `DELAY`, `PROBE`, `FAILED`.
- `ss`: `-t -u -l -n -p -a -e -i -o -s`, `-H`, state filters, and `sport` and `dport` expressions. `ss -tni` shows `rtt`, `cwnd`, and retransmission counters. `-p` shows other users' processes only with privilege. `ss` has no JSON output, so use `awk` to parse it.

**Project 4: network audit script and routing experiments.**
1. Write `netaudit.sh`. It outputs JSON with the default route and next hop, the listening TCP and UDP sockets with their owning processes, the count of established connections per remote address, and the neighbor entries in state `FAILED`.
2. Create `dummy0` with `192.0.2.1/24` (RFC 5737 TEST-NET-1). Create a second dummy interface with `192.0.2.0/25`. Before you run `ip route get 192.0.2.50`, write down the predicted output.
3. Add a rule with `ip rule add from 192.0.2.1 lookup 100` and a route in table 100. Predict and then verify the route for that source address.

**Pass criteria**
- Every predicted `ip route get` output matches the actual output, including the interface and source address.
- `ss` with a filter shows only the `http.server` socket and its PID.
- The script output is valid JSON.

## Phase 5: Network namespaces

**Learn**
- `ip netns add|del|list|exec`, `veth` pairs (`ip link add ... type veth peer name ...`), `ip link set DEV netns NS`, Linux bridges (`ip link add br0 type bridge`, `ip link set DEV master br0`), `bridge fdb show`.
- The loopback interface in a new namespace is down. Run `ip link set lo up`.
- Each namespace has its own interfaces, routing tables, neighbor table, `sysctl` network settings (including `net.ipv4.ip_forward`), and `nftables` ruleset.
- Deleting a namespace deletes the `veth` ends inside it, together with their peers.

**Project 5: routed topology.** Create namespaces `h1`, `r1`, and `h2` on two subnets with `r1` between them. Write a bridged variant in which a bridge connects `h1` and a third host `h3` on the same subnet. Write `lab5.sh up|down` and make it idempotent.

**Pass criteria**
- `h1` cannot reach `h2` until forwarding is enabled inside `r1`. Capture with `tcpdump -ni` to show the echo requests arrive at `r1` and are not forwarded. Then enable forwarding with `ip netns exec r1 sysctl -w net.ipv4.ip_forward=1` and show the ping succeeds.
- In the bridged variant, `bridge fdb show` lists the learned MAC addresses of `h1` and `h3`.
- Running `up` twice produces the same final state.
- After `down`, `ip netns list` is empty and `ip -br link` shows no leftover `veth` interfaces.

## Phase 6: VLAN subinterfaces

**Learn**
- IEEE 802.1Q: TPID `0x8100`, 3-bit priority, 12-bit VLAN ID. Usable IDs are 1 to 4094. A native or untagged VLAN does not carry a tag on the wire.
- Subinterface: `ip link add link IF name IF.10 type vlan id 10`. `ip -d link show` displays the VLAN protocol and ID.
- VLAN-aware bridge: `ip link add br0 type bridge vlan_filtering 1`. Add VLANs with `bridge vlan add dev PORT vid 10 [pvid untagged]`. Inspect with `bridge vlan show`. The default PVID is 1 on every port. Remove VLAN 1 from ports where it is not intended.
- `tcpdump -e -n -i IF` shows `vlan 10` in tagged frames.

**Project 6: router on a stick.** Build a switch namespace `sw` with a VLAN-aware bridge, hosts `h10` and `h20` on access ports (PVID 10 and PVID 20, untagged), and a trunk `veth` to a namespace `rtr`. In `rtr`, create subinterfaces `.10` and `.20` on the trunk, enable forwarding, and set the host default gateways. Write a table of the intended `bridge vlan show` output before building.

**Pass criteria**
- `h10` reaches `h20` through `rtr`.
- A capture on the trunk shows frames tagged `vlan 10` and `vlan 20`. A capture on the access ports shows no tags.
- `bridge vlan show` matches the table.
- After you remove VLAN 20 from the trunk port, `h20` is unreachable. After you restore it, `h20` is reachable.

## Phase 7: LACP bonding

**Learn**
- IEEE 802.1AX (originally 802.3ad, clause 43). LACPDUs use EtherType `0x8809` (slow protocols), destination MAC `01:80:C2:00:00:02`.
- Bonding modes: `balance-rr` (0), `active-backup` (1), `balance-xor` (2), `broadcast` (3), `802.3ad` (4), `balance-tlb` (5), `balance-alb` (6).
- Mode 4 parameters: `miimon`, `lacp_rate` (`slow`: 30 s, `fast`: 1 s), `xmit_hash_policy` (default `layer2`; also `layer2+3` and `layer3+4`), `min_links`.
- Mode 4 requires that each member report speed and duplex (check with `ethtool`) and that the partner also run LACP.
- A single flow always uses one member link.
- Inspect with `/proc/net/bonding/bond0` and `ip -d link show bond0`.

**Project 7: back-to-back LACP bond.** Create namespaces `a` and `b`. Connect them with two `veth` pairs. In each namespace, create `bond0` in mode `802.3ad` with `lacp_rate fast` and `xmit_hash_policy layer3+4`, and enslave one end of each pair. Assign addresses.

**Pass criteria**
- In both namespaces, `/proc/net/bonding/bond0` shows both members with the same `Aggregator ID`, `MII Status: up`, and a `Partner Mac Address` other than `00:00:00:00:00:00`.
- `tcpdump -e -n -i MEMBER 'ether proto 0x8809'` shows one LACPDU per second per member.
- Eight parallel TCP flows increase the transmit counters (`ip -s link`) on both members. One flow increases the counter on one member only.
- With `xmit_hash_policy layer2` and two fixed MAC addresses, all traffic uses one member. Verify and explain why.
- Set one member down while a ping runs. Record the packet loss. Set the member up and show it rejoins the aggregator.

This lab tests protocol behavior only. It does not test switch-side configuration or physical link negotiation.

## Phase 8: `nftables`

**Learn**
- Objects: table, chain, rule, set, map. Families: `ip`, `ip6`, `inet`, `arp`, `bridge`, `netdev`.
- Base chains: `type filter|nat|route hook prerouting|input|forward|output|postrouting priority N; policy accept|drop;`.
- Several base chains can attach to the same hook and run in priority order. The `accept` verdict ends evaluation of the current base chain only. A later base chain can still drop the packet.
- Matches: `ct state`, `iifname`, `oifname`, `ip saddr`, `tcp dport`, `icmp type`, `icmpv6 type`. Statements: `counter`, `log`, `limit rate`, `masquerade`, `dnat to`.
- Sets with `flags interval` and `timeout`.
- Tools: `nft -f`, `nft -c -f` (check without applying), `nft list ruleset`, `nft -a` (handles). A file loaded with `nft -f` applies as one atomic transaction.
- Tracing: set `meta nftrace set 1` and run `nft monitor trace`.
- Persistence on Debian: `/etc/nftables.conf`, loaded by `nftables.service`.
- On the lab machine, no other firewall manager (`ufw`, `firewalld`, Docker) can modify the ruleset.

**Project 8: stateful router firewall.** On `rtr` from Phase 6, add an `inet filter` table and an `ip nat` table. Add an `uplink` namespace that hosts a TCP server.
- Input policy `drop`. Allow `ct state established,related`, drop `ct state invalid`, allow ICMP echo with a rate limit, and allow the ICMPv6 neighbor discovery types. Allow SSH only from a named set of addresses with a timeout.
- Forward policy `drop`. Allow `h10` to reach `h20` on TCP 8080 only. Allow replies through `ct state`.
- Masquerade traffic from the VLAN subnets toward `uplink`.

**Pass criteria**
- `nft -c -f ruleset.nft` succeeds before loading.
- A test matrix (source, destination, protocol, port, expected result), written before testing, matches every result from `ip netns exec ... nc|curl|ping`.
- Rule counters increase as predicted.
- `nft monitor trace` shows one packet passing through each chain.
- A capture on the `uplink` side shows the translated source address.

## Phase 9: Capstone

**Project 9: branch-site simulator.** Write `lab.sh up|down|status|test`. The topology combines the previous phases:
- The `rtr` namespace connects to `sw` through a two-member LACP bond, with VLAN subinterfaces on the bond (`bond0.10`, `bond0.20`).
- The `nftables` inter-VLAN policy from Phase 8 applies.
- A server namespace runs a second `sshd` with its own configuration file and key-only access.
- `lab.service` (`Type=oneshot`, `RemainAfterExit=yes`) runs `up` at boot and `down` on stop.
- `lab-health.timer` runs a health check that reads `ip -j`, `ss`, and `/proc/net/bonding` with `jq`, `awk`, and `sed`, and writes JSON to the journal. The check logs failures at priority `err`.

**Pass criteria**
- `lab.sh test` exits with status 0 only when all of these hold: the bond aggregate is established, the VLAN reachability matrix matches the design, the firewall matrix matches the design, and key-only SSH to the server namespace works.
- Fault injection: set one bond member down, remove one VLAN from the trunk, and delete one firewall rule. The health check detects each fault, logs it at `err`, and the unit shows failed. After each repair, `lab.sh test` returns to status 0.
- After a reboot with no manual action, `systemctl status lab.service` shows `active (exited)` and `lab.sh test` exits with status 0.

Persistent configuration through `systemd-networkd`, NetworkManager, or `netplan` differs by distribution and is outside this roadmap. After Phase 9, learn the method that your target distribution uses. For `systemd-networkd`, the VLAN and bond equivalents are `.netdev` files with `Kind=vlan` and `Kind=bond`.

## Coverage

| Required item | Phases |
|---|---|
| Bash scripting | 1, 2, 4, 5, 9 |
| `sed`, `awk`, `jq` | 1, 2, 4, 9 |
| `systemd`, `journalctl` | 2, 3, 9 |
| SSH key authentication | 3, 9 |
| `ip`, `ss` | 4, 5, 6, 7 |
| Network namespaces | 5, 6, 7, 8, 9 |
| VLAN subinterfaces | 6, 9 |
| LACP bonding | 7, 9 |
| `nftables` | 8, 9 |

Reading about a topic is not evidence of ability. If a pass criterion needs a solution, repeat the project one week later with different VLAN IDs, subnets, and ports. The project counts as passed only when you complete it without a solution.
