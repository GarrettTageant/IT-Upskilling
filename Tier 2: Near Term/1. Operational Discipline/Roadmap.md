Claude-generated roadmap:

# Operational Discipline Roadmap

Learning roadmap and test projects for MOPs, maintenance windows, rollback criteria, peer review, ITIL 4 incident, problem and change practices, post-incident reviews, runbooks, and NetBox.

## 1. Ground rules

This roadmap has six phases and a capstone. Each phase lists study topics, projects, and pass criteria. A phase is complete when its projects pass. Reading is not completion.

Three constraints shape the design:

- **You probably do not own a change process at work.** As a placed contractor you follow the client’s process. The projects therefore use a lab you control and paper exercises based on your real fiber work.
- **Sanitize everything.** Hostnames, circuit IDs, customer names, cage layouts, and vendor details from the Pittock cage are likely confidential under your agreement. Check the agreement before you reuse anything. Invent all data.
- **Peer review needs a second person.** Self-review is weaker evidence, because authors do not see their own omissions. Use a classmate, an instructor, or a senior technician or NOC contact. If nobody is available, wait 48 hours and review against a written checklist, and mark the project as self-reviewed only.

Pace: about 5 hours per week, about 24 weeks.

Evidence: every artifact lives in a Git repository as Markdown or YAML and is merged through a reviewed pull request. The pull request history is your proof of peer review.

## 2. Phase 0: Lab and tooling (weeks 1 to 2)

Build the environment once. Every later project reuses it.

- A private Git repository with these directories: `mops/`, `runbooks/`, `changes/`, `incidents/`, `problems/`, `pir/`, `netbox/`.
- A Linux VM with Docker Engine and Docker Compose.
- A lab network in Containerlab using free router images (for example FRR): two core nodes, two access nodes, and one dual-homed server. Every lab MOP and fault injection uses this topology.
- NetBox deployed with the community netbox-docker project.
- Git skills: branching, pull requests, review comments, merge. This overlaps your automation roadmap.

**Pass criteria:** you destroy and rebuild the lab from the repository, and the rebuilt lab matches the documented topology.

## 3. Phase 1: NetBox as source of truth (weeks 2 to 6)

**Study topics**

- Object model: sites, locations, racks, device types, roles, devices, modules, interfaces, front ports and rear ports, cables and terminations, circuits and providers, power feeds.
- IPAM: prefixes, IP addresses, VRFs, VLANs.
- Tags, custom fields, config contexts, journal entries, the change log, object permissions.
- REST API with tokens, GraphQL, custom validation, event rules and webhooks, custom scripts and reports.
- The source-of-truth concept: for each data domain, define what is authoritative (intended state) and what is observed (actual state), and detect drift between them.

**Projects**

1. **Model a representative data hall.** Create a fictional site with 6 racks, about 20 devices, patch panels (front and rear ports), fiber trunks, cross-connects, and power feeds. Pass: every cable has two terminations; a second person traces a path from device A through two patch panels to device B using only NetBox; the labeling scheme is documented and encoded in the data.
2. **Data-quality rules.** Document naming conventions and required custom fields (owner, lifecycle status, change record ID). Enforce them with custom validators, or detect violations with a report. Pass: you seed 10 known errors and the report finds 10 of 10.
3. **Drift detection.** Compare NetBox interfaces and IP addresses against observed lab state. Start with `curl` and `jq` against the API. Move to Python with pynetbox when your Python skills allow. Pass: you inject 5 discrepancies and all 5 are found; there are no false positives; the tool is read-only.
4. **Authority statement.** Write one page that states what NetBox is authoritative for, what it is not, and the update rule: no change record closes until NetBox matches reality. Pass: the reviewer finds no data domain with an unassigned owner.

## 4. Phase 2: Methods of procedure (weeks 6 to 10)

**What a MOP contains**

- Header: title, ID, version, author, reviewer, approver, linked change record.
- Purpose, scope, and explicit out-of-scope items.
- Impact assessment: affected services, redundancy status, blast radius.
- Prerequisites: access, tools, materials, approvals, notifications sent.
- Pre-checks, each with the exact action and the expected result.
- Numbered steps. Each step has one action, the exact command or physical action, the expected result, and the verification method.
- Flags on irreversible steps and on the point of no return.
- Abort and rollback triggers, then rollback steps written to the same standard as the main steps.
- Post-checks, closeout steps (including the NetBox update), and escalation contacts.

**Writing standard:** the executor must not need to make a judgment that the author could have made in advance. Anything ambiguous is a defect.

**Safety:** MOPs that involve optics must state laser safety rules. Never look into a fiber end or an active port. Use a power meter to test for light.

**Projects**

1. **Fiber link migration MOP** (paper, sanitized, from your real experience). Move a link from one device port to another. Include end-face inspection and cleaning to IEC 61300-3-35 criteria, optic compatibility check, DOM transmit and receive readings compared with the optic datasheet, labeling per the ANSI/TIA-606 administration standard, and the NetBox update.
2. **Device decommission MOP.** Include dependency check (NetBox and observed state), proof the device is idle (interface counters, no active routes or sessions), power, copper, and fiber removal, label removal, inventory update, and NetBox lifecycle status change.
3. **Lab network MOP.** Replace a core link in Containerlab. Every pre-check and post-check is a command with an expected output (adjacency state, route counts, interface counters).
4. **Cold execution test.** Have another person execute your MOP exactly as written, without help from you. Record every question they ask. Each question is a defect. Then execute theirs and log every ambiguity you find. Pass: after defects are fixed, a second cold execution needs zero clarifications.

## 5. Phase 3: Change enablement, maintenance windows, rollback (weeks 10 to 14)

**ITIL 4 change enablement**

- Purpose: maximize the number of successful changes by assessing risk, authorizing changes, and managing the change schedule.
- Change types: **standard** (low risk, repeatable, pre-authorized, with a documented procedure), **normal** (assessed and authorized by a change authority sized to the risk), and **emergency** (expedited, for restoring service or urgent fixes).
- ITIL 4 uses the term change authority. A change advisory board is one possible form of it, not a requirement.
- Risk factors to assess: scope, blast radius, reversibility, complexity, detectability of failure, past failure rate, and timing.

**Maintenance window planning**

- Timeline in minutes, with the critical path identified.
- Roles: executor, verifier or scribe, NOC liaison, and an approver with stop-work authority.
- Dependencies and blackout periods checked against the change schedule.
- Notification plan: who is told, when, and through which channel.
- A go/no-go checkpoint before the first service-affecting step.
- A rollback reserve inside the window.

**Rollback criteria**

Write criteria as observable conditions, not judgment calls. Separate abort criteria (before commitment) from rollback triggers (after commitment). Mark the point of no return.

Every plan needs a latest rollback start time:

`latest rollback start = window end − (measured rollback duration + verification duration + buffer)`

If the change is not complete and verified by that time, you roll back. You must also test the rollback in the lab before you approve the change. An untested rollback is an assumption.

**Projects**

1. **Three change records.** Write one standard, one normal, and one emergency change in an ITSM tool (iTop or GLPI are open-source options with ITIL-oriented modules; verify that your version supports change objects) or as structured YAML in Git if tool setup takes more than a week. Each record has risk assessment, implementation plan, backout plan, test plan, schedule, and approvals. Pass: the reviewer approves or rejects each record without asking questions.
2. **Standard change conversion.** Select a repeated fiber task, such as installing a cross-connect between two already-provisioned patch panels. Write the procedure, the eligibility criteria that qualify a job as standard, the conditions that force escalation to a normal change, and the NetBox update requirement.
3. **Window plan with rollback drill.** Plan a 4-hour lab window with a minute-level timeline, a go/no-go checkpoint, and rollback times calculated from measured durations. Execute it while another person injects a fault at a time you are not told. Pass: the rollback starts because a written criterion was met, not because of intuition; service is restored within the plan; the post-check passes; the timeline was recorded during the event.

## 6. Phase 4: Incident management and runbooks (weeks 14 to 18)

**ITIL 4 incident management**

- Purpose: minimize the negative impact of incidents by restoring normal service operation as quickly as possible.
- Activities: logging, categorization, prioritization from impact and urgency, diagnosis, escalation (functional to a specialist team, hierarchical to management), workaround, resolution, closure.
- A major incident follows a separate procedure with a dedicated team.
- Restoring service is not the same as removing the cause. The cause belongs to problem management.
- ITIL 4 does not prescribe incident roles. The common roles (incident commander, communications lead, scribe) come from incident response frameworks such as the public PagerDuty incident response documentation and the Google SRE book.

**What a runbook contains**

- Trigger or symptom, and the severity it implies.
- Prerequisites and access.
- Diagnostic decision tree.
- Exact commands with expected outputs.
- Mitigation and resolution steps.
- Verification of recovery.
- Escalation path.
- Owner and last-tested date.

A runbook that has not been executed by someone other than its author is untested.

**Projects**

1. **Five runbooks:** optical link down or degraded (DOM check, clean and inspect, loopback test, optic swap, escalation), routing adjacency down, interface errors, MTU mismatch, and NetBox unavailable. Pass: another person resolves a seeded fault using only the runbook, within its stated time. Every deviation from the runbook is a runbook defect.
2. **Three incident simulations:** a degraded receive-power fiber fault; a failed change that causes an outage; and incorrect NetBox data that leads to the wrong port being disconnected. Define your impact and urgency matrix first. Pass: the timeline is written during the incident, not reconstructed afterward; priority is justified; communications updates follow a stated interval; escalations are documented.
3. **NetBox backup and restore.** Restore NetBox to a fresh VM from backup and validate object counts. Record the measured recovery time and the data loss. Pass: both numbers are documented in the NetBox runbook.

## 7. Phase 5: Problem management and post-incident reviews (weeks 18 to 21)

**ITIL 4 problem management**

- A problem is a cause, or potential cause, of one or more incidents.
- Problem identification: trend analysis of incidents.
- Problem control: analysis and a documented workaround.
- Error control: a known error record and tracking of the permanent fix through change enablement.

**Analysis methods and limits**

- Five Whys tends to produce a single linear cause. Complex failures usually have several contributing factors.
- Use fault trees, cause-and-effect diagrams, or contributing-factor analysis.
- Do not accept human error as a conclusion. Ask why the system allowed the error and why detection failed.

**Post-incident review standard**

- Blameless. The goal is corrective action, not attribution.
- Timeline built from artifacts: logs, tickets, chat, command history.
- Contributing factors, what worked, what failed, and where luck mattered.
- Metrics with stated definitions: time to detect, acknowledge, mitigate, and resolve. State each definition, because mean time to repair is ambiguous.
- Action items classified as prevent, detect, or mitigate. Each has an owner, a due date, and a verification method.
- Review held within a few business days, while evidence is available.

**Projects**

1. **PIR for each Phase 4 simulation.** Pass: the timeline is sourced from artifacts; at least two contributing factors are identified with evidence; no individual is blamed; every action item is specific, testable, and owned; at least one action changes a MOP, runbook, or NetBox validation rule; at least one action is closed with verification evidence.
2. **Problem record.** Select a recurring pattern from your simulations. Write the known error record and workaround, a permanent fix proposal, and a change record. Implement the fix in the lab. Pass: the recurrence is reproduced before the fix and does not recur after it.
3. **Critique a public postmortem.** Pick a published postmortem from a major provider and assess it against the standard above. Pass: a written critique that names at least three specific strengths or gaps.

## 8. Phase 6: Capstone (weeks 21 to 24)

**Scenario:** add a new access switch, dual-homed to both lab cores, end to end.

1. Pre-model the planned devices and cables in NetBox with planned status.
2. Open a normal change record with a risk assessment.
3. Write the MOP, including pre-checks, post-checks, and rollback. A peer reviews it through a pull request.
4. Plan the window and calculate the latest rollback start time.
5. Execute. An observer injects an undisclosed fault: a swapped cable, an MTU mismatch, or a low-power optic.
6. Open an incident. Decide on rollback or mitigation using only your written criteria.
7. Correct NetBox, write the PIR, open a problem record if warranted, and update the runbook.

**Pass criteria**

- The drift report shows zero differences at the end.
- Every decision has a timestamp and a recorded reason.
- The rollback or abort decision follows the written criteria.
- All artifacts cross-reference by ID: change, incident, problem, MOP version, and NetBox change log entry.
- A reviewer other than the author verifies the evidence.

## 9. Self-assessment scale

Score each skill: MOPs, window planning, rollback criteria, peer review, change, incident, problem, PIR, runbooks, NetBox.

- **0:** cannot produce the artifact.
- **1:** produces the artifact from a template, with major defects.
- **2:** produces a reviewed artifact with only minor defects.
- **3:** another person executes the artifact without clarification.

Target level 3 for MOPs, runbooks, and NetBox data quality. Level 2 is acceptable elsewhere until you work in a real change process.

## 10. References

- ITIL 4 Foundation publication (AXELOS and PeopleCert). The certification is optional. It proves vocabulary, not ability. Complete the projects before you pay for the exam.
- NetBox and pynetbox official documentation. NetBox is learned by building with it.
- Google SRE book and SRE workbook (free online): incident management and postmortem chapters.
- PagerDuty incident response documentation and the Atlassian incident handbook (both public).
- Etsy, Debriefing Facilitation Guide (public).
- IEC 61300-3-35, ANSI/TIA-606, ANSI/TIA-942. These standards are sold; confirm the current edition before you buy.

## 11. Limits of this roadmap

These projects show method and produce portfolio evidence. They do not show how you perform under production pressure, with real customers and real time limits. Only real operations show that. Use your datacenter work to observe how the client’s process differs from the one here, and note every difference, because the differences are the learning.
