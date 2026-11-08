Claude-generated roadmap:

This roadmap has nine phases. It covers 16 weeks at 6 to 8 hours per week. Each phase ends with pass criteria that you can verify. Before phase 1, you must be able to use `systemd`, `journalctl`, `ip`, `ss`, `nftables`, and network namespaces on Linux. You must also be able to use `tshark` capture and display filters. The phases need these skills. Do not start with Grafana. A dashboard built on incorrect data gives incorrect conclusions.

## Phase 0: lab (week 1)

Build one Linux VM (Debian or Ubuntu LTS) with Docker and Containerlab. Install `net-snmp`, `rsyslog`, `softflowd`, `hsflowd`, `nfdump`, and `gnmic`. Run Containerlab with two Linux routers that run FRRouting and one Nokia SR Linux node. SR Linux provides gNMI. The Linux routers provide SNMP, syslog, and flow export. If you have access to real devices, substitute them.

**Pass criteria:**
- All nodes keep the same time through NTP, and the clock offset between nodes is under 100 ms. Every later phase correlates timestamps.
- You can capture traffic on each lab link with `tshark`.

## Phase 1: SNMP (weeks 2 and 3)

**Study:**
- SMI, MIB, and OID structure
- The `IF-MIB` `ifTable` and `ifXTable` (RFC 2863)
- The PDU types: Get, GetNext, GetBulk, Set, Response, Trap, Inform, and Report
- The difference between 32-bit and 64-bit counters

| Property | SNMPv2c | SNMPv3 |
|---|---|---|
| Specification | RFC 1901, RFC 3416 | RFC 3411 to RFC 3418 |
| Version field | 1 | 3 |
| Authentication | Community string, sent in cleartext | User-based Security Model (USM), RFC 3414 |
| Integrity and replay protection | None | HMAC authentication, plus engine boots and engine time |
| Privacy (encryption) | None | CBC-DES (RFC 3414), CFB128-AES-128 (RFC 3826) |
| Access control | Vendor-specific or RFC 3584 community mapping | View-based Access Control Model (VACM), RFC 3415 |
| Security levels | None | `noAuthNoPriv`, `authNoPriv`, `authPriv` |

USM authentication uses HMAC-MD5-96 or HMAC-SHA-96. RFC 7860 adds the HMAC-SHA-2 family. You should not use MD5 or DES. Requests use UDP port 161. Notifications use UDP port 162. A Trap is not acknowledged. An Inform is acknowledged. For a Trap, the sender is the authoritative engine. For an Inform, the receiver is the authoritative engine.

**Project 1A: v2c.**
1. Configure `snmpd` with a read-only community.
2. Walk `ifXTable` with `snmpwalk`.
3. Capture the exchange and extract the community string from the capture.
4. Write a script that polls `ifHCInOctets` twice and computes bits per second.

**Project 1B: v3.**
1. Create a USM user with `createUser` and grant access with `rouser <name> authPriv`.
2. Test all three security levels.
3. Poll with a wrong authentication key.
4. Capture the exchange and identify the Report PDU counter that the agent returns.
5. Confirm that the `scopedPDU` is encrypted.

**Pass criteria:**
- Your script handles a `Counter32` wrap. A 32-bit octet counter wraps after 34.36 seconds at 1 Gb/s and after 3.436 seconds at 10 Gb/s.
- You can explain why `ifHCInOctets` is the correct object for 10 Gb/s interfaces.
- You can state which security level each capture proves.

## Phase 2: syslog, RFC 5424 (week 4)

**Study:**
- The RFC 5424 header fields: `PRI`, `VERSION`, `TIMESTAMP`, `HOSTNAME`, `APP-NAME`, `PROCID`, `MSGID`, `STRUCTURED-DATA`, and `MSG`
- `PRI` equals facility × 8 + severity
- The `NILVALUE` (`-`)
- The three transports: UDP (RFC 5426), TLS (RFC 5425, port 6514), and TCP framing (RFC 6587)
- The legacy RFC 3164 format

A receiver must accept messages up to 480 octets. Inside a `PARAM-VALUE`, the characters `"`, `\`, and `]` must be escaped with a backslash.

**Project 2.**
1. Configure `rsyslog` to forward in RFC 5424 format with the `RSYSLOG_SyslogProtocol23Format` template.
2. Forward over UDP and then over TLS.
3. Write a parser that extracts every header field and every structured data element.
4. Add loss with `tc netem` and measure the UDP message loss rate.

**Pass criteria:**
- Your parser decodes the four example messages in RFC 5424 section 6.5 without error.
- Your parser handles escaped `]` characters.
- You can calculate `PRI` 34 as facility 4, severity 2.

## Phase 3: flow telemetry (weeks 5 and 6)

| Property | NetFlow v5 | NetFlow v9 | IPFIX | sFlow v5 |
|---|---|---|---|---|
| Specification | Cisco, no RFC | RFC 3954 (informational) | RFC 7011, RFC 7012 | sflow.org specification |
| Header version | 5 | 9 | 10 | 5 |
| Record format | Fixed | Template-based | Template-based | Fixed XDR structures |
| IPv6 | No | Yes | Yes | Yes |
| Method | Flow cache | Flow cache | Flow cache | Sampled packet headers and counter samples |
| Transport | UDP, no standard port | UDP, no standard port | SCTP (required), TCP, UDP | UDP |
| Port | Convention only | Convention only | 4739 (4740 for TLS and DTLS) | 6343 |

**Study:**
- IPFIX Template Sets (Set ID 2), Options Template Sets (Set ID 3), and Data Sets (Set ID 256 or higher)
- Observation Domain ID
- Active and inactive timeouts
- Sampling rate N and the sFlow counter sample

**Project 3A.**
1. Export traffic from `softflowd` as NetFlow v9 and as IPFIX.
2. Collect both with `nfcapd`.
3. Decode both with `tshark -Y cflow`.
4. Identify each template and the Data Set that uses it.

**Project 3B.**
1. Run `hsflowd` with 1-in-N sampling.
2. Capture the datagrams with `tshark -Y sflow`.
3. Generate a known volume of traffic with `iperf3`.
4. Compare the NetFlow, IPFIX, and sFlow volume estimates to the known volume.

**Pass criteria:**
- You document the error of each method.
- You can state that the sFlow sampling error at 95% confidence is at most 196 × √(1/c) percent, where c is the number of samples.
- You can explain why a UDP template that is lost makes the Data Sets that follow it undecodable.

## Phase 4: gNMI streaming telemetry (weeks 7 and 8)

**Study:**
- gRPC over HTTP/2 with TLS
- The four RPCs: `Capabilities`, `Get`, `Set`, and `Subscribe`
- The three subscription lists: `STREAM`, `ONCE`, and `POLL`
- The three stream modes: `SAMPLE`, `ON_CHANGE`, and `TARGET_DEFINED`
- YANG-modeled paths and OpenConfig models
- The encodings: `JSON`, `JSON_IETF`, `BYTES`, `PROTO`, and `ASCII`

gNMI has no mandatory default port. SR Linux uses 57400.

**Project 4.**
1. Run `gnmic capabilities` and `gnmic get` against the SR Linux node.
2. Use `gnmic set` to change an interface administrative state.
3. Subscribe to interface statistics in `SAMPLE` mode:

```bash
gnmic -a <node>:57400 -u admin -p <password> --skip-verify \
  subscribe --path "/interface[name=ethernet-1/1]/statistics" \
  --mode stream --stream-mode sample --sample-interval 10s
```

4. Subscribe to the interface operational state in `ON_CHANGE` mode.
5. Fail the link and record the delay until the update arrives.
6. Repeat the failure with SNMP polling and record that delay.

**Pass criteria:**
- You report both detection delays in seconds.
- You can state why the SNMP delay depends on the polling interval and the `ON_CHANGE` delay does not.

## Phase 5: Prometheus (weeks 9 and 10)

**Study:**
- The pull model
- The four metric types: counter, gauge, histogram, and summary
- Label cardinality
- PromQL `rate()` and `irate()`
- Recording rules
- `snmp_exporter` and its generator
- Alertmanager

Prometheus retains 15 days of data by default. Do not shorten this value, because phase 8 needs the data. A `rate()` window must contain at least two samples. You should use a window of at least four scrape intervals so that one failed scrape does not remove the result.

**Project 5.**
1. Deploy Prometheus and `snmp_exporter`.
2. Generate an SNMPv3 `if_mib` module and an `auth` entry.
3. Configure the scrape job:

```yaml
scrape_configs:
  - job_name: snmp
    metrics_path: /snmp
    params:
      auth: [lab_v3]
      module: [if_mib]
    static_configs:
      - targets: [192.0.2.1]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9116
```

4. Add the gNMI data with the `gnmic` Prometheus output.
5. Write a recording rule for interface utilization.

**Pass criteria:**
- Your PromQL query for bits per second matches the phase 1 script within 1%.
- You can show the active series count with `prometheus_tsdb_head_series`.
- You can show that a counter reset does not produce a negative rate.

## Phase 6: Grafana (week 11)

**Study:**
- Data sources, template variables, and panel units
- Provisioning from files under `/etc/grafana/provisioning/`

**Project 6.**
1. Provision the Prometheus data source and a Loki data source from files.
2. Send syslog to Loki with Grafana Alloy. Promtail reached end of life on March 2, 2026, so you must not use it.
3. Build one dashboard with a device variable and an interface variable.
4. Show bit rate in bits per second, errors per second, operational state, and a log panel.

**Pass criteria:**
- You can rebuild the dashboard from the provisioning files alone.
- Every panel uses the correct unit.

## Phase 7: LibreNMS or Zabbix (weeks 12 and 13)

| Property | LibreNMS | Zabbix |
|---|---|---|
| Focus | Network devices | General infrastructure and network devices |
| Device onboarding | SNMP auto-discovery | Templates and low-level discovery (LLD) |
| Default storage | RRD files | Relational database |
| Alert logic | Alert rules and transports | Trigger expressions, with recovery expressions |

Complete one tool fully. Use LibreNMS first, because your goals are network-focused. In the second tool, reproduce only the alert set from phase 8, to compare how each tool expresses thresholds.

**Project 7.**
1. Deploy the tool and discover the lab devices with SNMPv3.
2. Receive syslog from the lab nodes.
3. Receive an SNMP Trap with `snmptrapd`. In Zabbix, use `zabbix_trap_receiver.pl`.
4. Create one alert for a down interface and one for high utilization.

**Pass criteria:**
- Each alert delivers a notification and a recovery.
- You can state which data each tool polls and which data it receives passively.

## Phase 8: alert threshold design (weeks 14 and 15)

Phase 8 requires at least 14 days of retained data from phase 5. This period covers two weekly cycles.

**Study:**
- Alert on conditions that require action, not on every anomaly.
- Alert on symptoms before causes.
- Derive each threshold from the baseline and not from a guess.
- Require a duration. Prometheus uses `for:`, and Zabbix uses functions such as `min(/host/key,15m)`.
- Prevent flapping with hysteresis. Prometheus uses `keep_firing_for`, and Zabbix uses a recovery expression.
- Suppress dependent alerts with Alertmanager `inhibit_rules` or Zabbix trigger dependencies.
- Alert when data is missing, with `up == 0`, `absent()`, or Zabbix `nodata()`.
- Compute error rates as a ratio of packets, with a minimum packet rate, so low traffic does not produce noise.
- For optical interfaces, start from the alarm and warning thresholds stored in the transceiver (SFF-8472 for SFP and SFP+, SFF-8636 for QSFP). Read them with `ethtool -m`, `ENTITY-SENSOR-MIB` (RFC 3433), or OpenConfig transceiver paths.

Example rule:

```yaml
- alert: InterfaceInputUtilizationHigh
  expr: |
    (rate(ifHCInOctets[5m]) * 8 / (ifHighSpeed * 1e6)) > 0.85
      and on (instance, ifIndex) ifHighSpeed > 0
  for: 15m
  keep_firing_for: 10m
  labels:
    severity: warning
```

`ifHighSpeed` is in units of 1,000,000 bits per second. The `ifHighSpeed > 0` guard prevents division by zero on interfaces that report no speed.

**Project 8.** Write an alert catalog with at least eight alerts. For each alert, record the signal, the threshold, the evidence from the baseline, the duration, the severity, the recovery condition, and the action that the responder takes.

**Pass criteria:**
- Over a 7-day replay, no alert fires more than twice without a requiring action.
- You can defend each numeric threshold with a graph.

## Capstone (week 16)

Inject six faults: link down, link flap, 2% packet loss with `tc netem`, link saturation with `iperf3`, an exporter outage, and a burst of syslog messages. For each fault, record which signals detected it (SNMP, syslog, flow, gNMI, LibreNMS or Zabbix) and the time to detection of each signal.

**Pass criteria:**
- Every fault is detected by at least one alert.
- The exporter outage is detected by an alert about missing data.
- Your report states the false positives and false negatives for each alert.
