Claude-generated roadmap:

This roadmap has eight phases and 245 budgeted hours. One case study carries every project, so each artifact feeds the next one. The order differs from your list: SLO definition comes before failure-domain design, because a failure-domain design requires an availability target to design against.

## Limits of this roadmap

- Design skill comes mainly from operating production networks and from review by engineers who have seen designs fail. These projects produce verifiable design artifacts. They do not replace operating experience.
- Each project has pass criteria that a calculation or a measurement can check. If a project fails its criteria, repeat the project.
- Obtain a review of each artifact from a practising network engineer. A self-review does not expose your unstated assumptions.
- The lab projects use virtual devices. They cannot validate hardware forwarding performance, buffer depth, or table scale.
- If you substitute data from a real facility, obtain written permission first. Colocation and client network data can be covered by confidentiality terms.

## Sequence, effort, and prerequisites

| Phase | Subject | Budget (hours) | Prerequisite |
|---|---|---|---|
| 1 | Requirements analysis | 20 | None |
| 2 | SLO definition | 20 | Phase 1; basic Python |
| 3 | Failure-domain design | 40 | Phase 2; Linux networking for the lab project |
| 4 | Capacity planning | 30 | Phase 3 |
| 5 | Observability design | 40 | Phases 2 and 3; Prometheus, Grafana, Alertmanager basics |
| 6 | Cost modeling | 25 | Phases 3 and 4 |
| 7 | Vendor evaluation | 30 | Phases 1 and 6 |
| 8 | Capstone | 40 | Phases 1 to 7 |

The total is 245 hours. At 8 hours per week, this is 31 weeks, rounded up. Phases 1, 2, and 6 need no lab, so you can start them now.

The lab projects in Phases 3, 4, 5, and 7 assume working Linux skills: `ip`, `nftables`, `systemd`, and Bash. With limited Linux experience, finish that prerequisite first. Otherwise the lab tooling consumes the hours, not the design work.

## Case study baseline

Use this baseline in every phase. Replace invented values with measured values where you have them.

**Business brief (Phase 1 input):**

1. "Customers must never notice a network problem."
2. "Move the order-processing platform to a second site."
3. "The network budget is 1.2 million USD over 5 years."
4. "Go live in 9 months."
5. "Payment card data is in scope for PCI DSS."
6. "The business expects traffic to grow."

**Topology data (Phase 3 input):**

- Site A and Site B are colocation sites in the same metro area. One cloud region also exists.
- Circuit C1 (provider P1) crosses path segments S1, S2, and S5.
- Circuit C2 (provider P2) crosses path segments S3, S4, and S5.
- Segment S5 is one duct under a river crossing.
- Peak traffic grows 2% per month, compounded.

## Phase 1: Requirements analysis

**Learn:**

- Functional requirements, non-functional requirements, constraints, and assumptions are four different categories. A constraint is imposed. An assumption is believed and must be validated.
- Stakeholder identification: for each stakeholder, record the decision they own and the acceptance criteria they set.
- Traffic characterization: for each application, record source, destination, protocol, peak and mean bandwidth, latency tolerance, and loss tolerance.
- Requirement quality attributes in ISO/IEC/IEEE 29148: necessary, unambiguous, verifiable, and traceable.
- Requirement keywords in RFC 2119 as clarified by RFC 8174: `MUST`, `SHOULD`, and `MAY` carry defined meanings only in capital letters.
- Reference: Oppenheimer, *Top-Down Network Design*, the chapters on requirements analysis.

**Project 1: Requirements specification.** Convert the six brief statements into a specification. Deliver:

- A requirements table with ID, text, category, priority, and acceptance test.
- An assumptions register and a constraints register.
- An open-questions list addressed to named stakeholder roles.
- A traceability matrix with columns for requirement ID, design element, and verification method. Leave the design-element column empty until Phase 3.

**Pass criteria:**

- Each requirement is atomic and uses an RFC 2119 keyword.
- No requirement contains an unmeasurable word such as "never", "fast", or "highly available".
- Each requirement has an acceptance test that a third party can run.
- Brief statement 1 becomes at least one availability requirement and one latency or loss requirement, each with a number.

## Phase 2: SLO definition

**Learn:**

- A service level indicator (SLI) is the ratio of good events to valid events. A service level objective (SLO) is a target for that SLI over a window. A service level agreement (SLA) is a contract with consequences.
- Error budget = 1 − SLO. Burn rate = observed error ratio ÷ (1 − SLO).
- The availability of a service cannot exceed the product of the availabilities of its serial hard dependencies. Therefore the network SLO must be higher than the SLO of any service that depends on it.
- Allowed downtime: 99.9% allows 43.2 minutes per 30 days and 8.76 hours per 365 days. 99.99% allows 4.32 minutes per 30 days and 52.56 minutes per 365 days.
- Network SLI sources: probe success ratio, round-trip time percentile, packet loss ratio, one-way delay (RFC 7679), one-way loss (RFC 7680), and TWAMP (RFC 5357).
- Multiwindow, multi-burn-rate alerting from the *SRE Workbook*, chapter 5. For a 30-day window:

| Burn rate | Long window | Short window | Budget consumed | Action |
|---|---|---|---|---|
| 14.4 | 1 hour | 5 minutes | 2% | Page |
| 6 | 6 hours | 30 minutes | 5% | Page |
| 1 | 3 days | 6 hours | 10% | Ticket |

- Reference: *Site Reliability Engineering*, chapters 3 and 4; *The Site Reliability Workbook*, chapters 2 and 5.

**Project 2: SLO document with simulation.** Define SLIs and SLOs for four services: internet ingress, inter-site transport, cloud connectivity, and management-plane access. For each SLI, specify the numerator, the denominator, the measurement point, and the probe method. State the compliance window and an error-budget policy that names the actions taken when the budget is exhausted.

Then write a Python simulation. It generates 90 days of synthetic probe data at 10-second intervals, injects outages, computes SLI and remaining budget, and evaluates the three alert rules above.

**Pass criteria:**

- The computed error budget matches a hand calculation.
- A 10-minute full outage triggers the 14.4 alert.
- A single failed probe triggers no alert.
- A sustained 0.5% loss over 3 days is reported with its detection time and the alert that caught it, or is documented as undetected.

## Phase 3: Failure-domain design

**Learn:**

- A failure domain is the set of components that one failure can disable. Blast radius is the set of services affected by one failure.
- A shared risk link group (SRLG) is a set of links that can fail together because they share a resource (RFC 4202). Fate sharing is the condition where components fail together because of a common dependency.
- Failure layers: device, line card, power feed, circuit, duct, carrier, site, control plane, management plane, and configuration change. A software defect that exists on every device running one NOS version is a correlated failure.
- Availability: A = MTBF ÷ (MTBF + MTTR). Series: A = A₁ × A₂ × … × Aₙ. Independent parallel: A = 1 − (1 − A₁)(1 − A₂). Two independent 99.9% elements in parallel give 99.9999%.
- The parallel formula is valid only if failures are independent. In the case study, C1 and C2 share S5, so the circuit pair cannot exceed the availability of S5.
- Failure detection and recovery time: BFD detection time equals the detect multiplier times the negotiated interval (RFC 5880). IEEE 802.1AX LACP sends periodic frames every 1 second (short timeout) or every 30 seconds (long timeout).
- Gray failure is a partial failure, such as packet loss or CRC errors, that health checks do not report as down.
- Analysis methods: failure mode and effects analysis (IEC 60812) and fault tree analysis (IEC 61025). Facility redundancy ratings: ANSI/TIA-942 and the Uptime Institute Tier Standard.
- Reading:
  - Govindan et al., "Evolve or Die: High-Availability Design Principles Drawn from Google's Network Infrastructure" (SIGCOMM 2016).
  - Gill, Jain, and Nagappan, "Understanding Network Failures in Data Centers" (SIGCOMM 2011).
  - Meza et al., "A Large-Scale Study of Data Center Network Reliability" (IMC 2018).
  - Huang et al., "Gray Failure: The Achilles' Heel of Cloud-Scale Systems" (HotOS 2017).

**Project 3A: Topology and failure-domain inventory.** Design the two-site-plus-cloud topology. Deliver a diagram with failure domains marked and an inventory table with these columns: component, failure domain, blast radius, detection mechanism, recovery mechanism, and expected recovery time. Deliver an SRLG register built from the segment data in the baseline.

**Project 3B: Availability model.** Build a reliability block diagram. Compute end-to-end availability for each Phase 2 SLO. Source every MTBF and MTTR value and state whether it is a vendor prediction or a field measurement. Run a sensitivity analysis. If the design misses a Phase 2 target, change the design and recompute.

**Project 3C: Failure injection (lab).** In containerlab, build a two-spine, two-leaf fabric using FRRouting. Send a UDP stream with sequence numbers at a fixed interval. For a contiguous loss, outage duration equals lost packets times the send interval. Inject link-down, node-down, and BFD-timeout failures. Predict each outage from configured timers before you measure it.

**Pass criteria:**

- The SRLG register identifies S5 and quantifies its effect on the C1-and-C2 pair.
- The availability model states each independence assumption.
- For each injected failure, the report lists the predicted and measured outage. Each case where the measured value exceeds the prediction has a documented cause supported by a packet capture or a log.

## Phase 4: Capacity planning

**Learn:**

- Capacity has more dimensions than bandwidth: ports, FIB and TCAM entries, MAC and ARP/ND table entries, BGP session count, power, rack space, and cooling.
- Interface counters: 64-bit `ifHCInOctets` (IF-MIB, RFC 2863) must replace 32-bit `ifInOctets` at high speed. A 32-bit octet counter wraps after 34.4 seconds at 1 Gbit/s.
- A counter polled every 300 seconds yields the average rate over that interval. A 100 ms burst at line rate is not visible in that average.
- Percentile measurement: IP transit is commonly billed on the 95th percentile of 5-minute samples.
- Growth model: demand(t) = demand(0) × (1 + g)^t, with g as the monthly growth rate.
- Order trigger date = date when projected demand reaches the utilization threshold − procurement and provisioning lead time.
- Failure-case sizing: after the loss of the largest single failure domain, peak demand must not exceed the utilization threshold of the remaining capacity.
- M/M/1 mean time in system is 1 ÷ (μ − λ). The model assumes Poisson arrivals and exponential service times, which packet traffic does not satisfy. Use it only to show that delay grows without bound as utilization approaches 1.
- Link aggregation and ECMP distribute flows by hash. A small number of large flows can load one member link while others remain idle.

**Project 4: Capacity model.**

1. Generate 18 months of synthetic 5-minute utilization data. Include a diurnal pattern, a weekly pattern, and 2% monthly compound growth.
2. Fit the model on the first 12 months and compare its forecast with the withheld 6 months. Report mean absolute percentage error.
3. Compute the 95th percentile and the peak per link.
4. Compute the load on every link under each single failure from the Phase 3 inventory.
5. Report the month when each link first exceeds the threshold in the normal state and in the worst single-failure state.
6. Compute the order date for each link. Parameterize circuit and hardware lead times.
7. In the lab, run `iperf3` with 2, 8, and 64 parallel flows across a Linux bond and across ECMP paths. Report per-member utilization for each case.

**Pass criteria:**

- The model identifies the link that first exceeds its threshold, and the result matches a hand calculation for that link.
- The forecast error is reported and bounded, and the report states what the error implies for the order dates.
- The lab report shows the hash-imbalance effect with measured numbers.

## Phase 5: Observability design

**Learn:**

- Monitoring answers questions you defined in advance. Observability is the ability to infer internal state from outputs. A network design needs both.
- Signal sources: SNMP (use v3 where authentication and encryption are required), gNMI streaming telemetry, syslog (RFC 5424), IPFIX (RFC 7011), sFlow, active probes, and BFD and BGP session state.
- Worst-case detection time = scrape or sample interval + rule evaluation interval + alert `for` duration + notification grouping delay.
- Alerts on SLI burn rate indicate user-visible impact. Alerts on causes, such as interface errors, support diagnosis. Each alert must be actionable and must link to a runbook entry.
- The monitoring system must not share a failure domain with the data path it monitors. Provide an out-of-band path and a dead-man's switch alert that fires when the alerting pipeline stops.
- Resource analysis for interfaces: utilization, saturation, and errors (the USE method). *Site Reliability Engineering*, chapter 6, covers the four golden signals.
- Cardinality, retention, and downsampling determine storage and memory. Take the bytes-per-sample figure from the Prometheus storage documentation and verify it against your own measurement.

**Project 5A: Telemetry requirements matrix.** For each Phase 2 SLI and each Phase 3 failure mode, record the detecting signal, source, collection interval, designed time-to-detect, and retention. List each failure mode with no detecting signal as an explicit gap with an accepted-risk statement.

**Project 5B: Implementation and fault injection (lab).** Extend the Phase 3 containerlab topology.

- Export metrics with `node_exporter` and either `gnmic` or an SNMP exporter.
- Run blackbox probes between the nodes.
- Build Grafana dashboards and Alertmanager rules, including the Phase 2 burn-rate alerts.
- Size the monitoring system from series count, scrape interval, and retention, then compare with measured disk and memory use.
- Inject every Phase 3 failure and record measured time-to-detect.
- Inject a gray failure with `tc qdisc add dev <interface> root netem loss 2%`.
- Stop the collector and verify that the dead-man's switch alert fires.

**Pass criteria:**

- Measured time-to-detect is less than or equal to the designed value for each failure.
- An SLI-based alert detects the gray failure.
- Every alert rule maps to a runbook entry.
- The sizing prediction and the measurement are both reported, with the difference explained.

## Phase 6: Cost modeling

**Learn:**

- Capital expenditure: hardware, optics, cabling, perpetual licenses. Operating expenditure: support contracts, subscription licenses, colocation space and power, cross-connects, circuit recurring and non-recurring charges, transit, spares, monitoring tools, and staff time.
- Net present value: NPV = Σ CFₜ ÷ (1 + r)ᵗ, summed from t = 0 to T. The discount rate r is an organizational input. Do not assume a value.
- Unit costs: cost per port-month and cost per Gbit/s-month.
- Expected annual downtime cost = (1 − A) × 8760 hours × cost per hour of outage. Compare this value with the marginal cost of the added redundancy that raises A.
- Cloud and transit pricing mechanisms: egress charge per gigabyte, committed rate versus burst, and 95th-percentile billing.
- Sensitivity analysis ranks each input by its effect on the total. Scenario analysis varies several inputs together.

**Project 6: Five-year cost model.** Model three variants of the Phase 3 design: single site with dual homing, two sites active-passive, and two sites active-active. Use the availability result from Project 3B for each variant. Source every price input and record the source date. Deliver:

- An inputs sheet.
- Calculation sheets.
- NPV per variant.
- A sensitivity ranking of the five most influential inputs.
- The marginal cost per avoided downtime minute between variants.
- A comparison against the 1.2 million USD budget.

**Pass criteria:**

- No number is hard-coded inside a formula. All values come from the inputs sheet.
- The sum of line items equals each total.
- A reviewer changes one input, and the model output changes as the reviewer predicts.
- The recommendation states which input, if it changes by a stated amount, reverses the ranking.

## Phase 7: Vendor evaluation

**Learn:**

- Process: request for information, request for proposal, shortlist, proof of concept, decision.
- Separate mandatory pass/fail gates from weighted criteria. Set the weights before you score any candidate.
- Criteria: functional fit, performance, interoperability, operability (CLI, NETCONF, gNMI, and YANG or OpenConfig model support), software release quality, lifecycle dates (end of sale, end of support), support terms (response time, hardware replacement time), supply lead time, security process (vulnerability disclosure, signed images), total cost (Phase 6), and exit cost.
- Test methods: RFC 2544 for network interconnect device benchmarking, RFC 6349 for TCP throughput testing, and ITU-T Y.1564 for Ethernet service activation. These methods require a hardware traffic generator and hardware under test to give valid device results.
- Free lab images: FRRouting, Nokia SR Linux, and Arista cEOS. Verify each image's license and download terms before use.

**Project 7: Platform evaluation.**

1. Write the evaluation plan first. It contains gates, weights, test cases, and expected results. Commit it to Git before any test.
2. Run identical test cases on three candidates. Cover BGP with BFD and ECMP, failover behavior, rollback and commit behavior, telemetry paths available over gNMI, and configuration automation reproducibility.
3. Score the candidates.
4. Run a weight-sensitivity analysis. Report whether the top-ranked candidate changes as each weight varies across a stated range.
5. Write an exit-cost analysis.

Label each hardware-dependent criterion as "not evaluated" or as "vendor claim, unverified".

**Pass criteria:**

- The Git history shows weights fixed before testing.
- Another person can rerun the tests from your automation and obtain the same results.
- The report states whether the ranking is stable under weight changes.
- The report does not present any virtual-device result as a hardware performance result.

## Phase 8: Capstone

Assemble one design package for the case study:

- The Phase 1 requirements, the Phase 2 SLOs, the Phase 3 topology and availability model, the Phase 4 capacity plan, the Phase 5 telemetry design, the Phase 6 cost model, and the Phase 7 recommendation.
- A risk register.
- An architecture decision record for each major decision. Each record lists the alternatives considered and the reason for the choice.
- The completed traceability matrix from Phase 1.

Then run a failure exercise. A reviewer writes five failure scenarios that you have not seen. For each scenario, predict the outcome from your design documents, and then inject the failure in the lab.

**Pass criteria:**

- Every requirement traces to a design element and a verification result.
- Each of the five scenarios either matches your prediction or becomes a documented design defect with a corrected design.
- A practising engineer has reviewed the package, and the package records each review finding and your response.
