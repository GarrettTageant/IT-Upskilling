Claude-generated roadmap:

This roadmap has seven phases. Phases 1 to 5 each cover one platform. Phase 6 combines them in a multi-vendor project. Duration assumes 8 hours per week on one x86-64 Linux host with KVM support. The total is 21 weeks. If you have fewer hours, extend the durations. Do not skip the projects.

## Lab platform

Use containerlab as the primary platform. Topologies are YAML text files, so Git can track them and one file redeploys the same lab. Containerlab supports container-based nodes such as Arista cEOS, Juniper cRPD, and SONiC, and VM-based nodes such as Cisco Nexus 9000v and Juniper vJunos-switch. FRRouting runs as a `linux` kind node.

Use EVE-NG, GNS3, or Cisco Modeling Labs only when a required image has no containerlab path, or when you need a graphical interface.

Vendor images are not part of containerlab. Each vendor sets its own download and license terms. Verify those terms before Phase 2. VM-based nodes need more memory than container-based nodes. Read the containerlab page for each kind before you size the host.

## Configuration models to compare

You must be able to state, for each platform, where a change takes effect and how you undo it.

| Platform | Default behavior of a configuration command | Transactional mechanism | Rollback mechanism |
|---|---|---|---|
| Junos | Edits the candidate configuration | `commit check`, `commit`, `commit confirmed` | `rollback 0` to `rollback 49`, then `commit` |
| Arista EOS | Changes the running configuration immediately | `configure session`, `commit`, `abort`, `commit timer` | `configure replace` from a saved file |
| NX-OS | Changes the running configuration immediately | `configure session`, `verify`, `commit` | `checkpoint`, `rollback running-config checkpoint` |
| FRRouting | Changes the running configuration immediately in `vtysh` | None in the classic model | Restore `frr.conf`, then apply it with `frr-reload.py` |
| SONiC | Writes to CONFIG_DB, not to the saved file | None in the base `config` CLI | `config reload` from `config_db.json`; checkpoint and rollback depend on the release |

Recent FRRouting releases add a management daemon (`mgmtd`). Check the documentation for your FRRouting version to learn which commands use a candidate datastore.

## Phase 0: lab platform (week 1)

**Skills:** install Docker and containerlab, write a topology file, and use `containerlab deploy`, `inspect`, and `destroy`.

**Project 0: two-node OSPF lab.**
- Deploy two FRRouting nodes with a point-to-point link.
- Enable the required daemons in `/etc/frr/daemons`.
- Store the topology file and configurations in a Git repository.

*Pass criteria:* `show ip ospf neighbor` shows state `Full` on both nodes. After `containerlab destroy` and a redeploy, the same result appears with no manual steps.

```yaml
name: multivendor
topology:
  nodes:
    r1:
      kind: juniper_crpd
      image: crpd:<version>
    r2:
      kind: arista_ceos
      image: ceos:<version>
    r3:
      kind: linux
      image: quay.io/frrouting/frr:<tag>
  links:
    - endpoints: ["r1:eth1", "r2:eth1"]
    - endpoints: ["r2:eth2", "r3:eth1"]
```

## Phase 1: Junos and JNCIA-Junos (weeks 2 to 6)

**Skills:**
- Junos architecture: Routing Engine, Packet Forwarding Engine, control plane, forwarding plane, transit traffic, and exception traffic.
- CLI modes, navigation, and help.
- Hierarchy navigation with `edit`, `up`, `top`, `set`, `delete`, and `show`.
- Candidate and active configurations.
- `show | compare`, `show | display set`, and `show | display inheritance`.
- `load merge`, `load override`, `load replace`, and `load set`.
- Configuration groups with `apply-groups`.
- Rescue configuration.
- User accounts and login classes.
- Interface units and families.
- Static routing, OSPF, routing policy, and firewall filters.

cRPD has no packet forwarding engine and does not support every Junos feature. If cRPD rejects a command, record the rejection and repeat the test on vJunos-switch.

**Project 1.1: commit model drill.**
1. Enter an invalid configuration and show that `commit check` rejects it.
2. Run `commit confirmed 2` with a change that blocks your management session. Show the automatic rollback after the timer expires.
3. Roll back to the third previous configuration with `rollback 3` and `commit`. Include `show | compare rollback 3` output.
4. Save a rescue configuration, damage the configuration, and restore it.

*Pass criteria:* the evidence file contains the command and output for each of the four steps.

**Project 1.2: policy and filter.**
- Build three Junos devices running OSPF.
- Redistribute a static route with an OSPF export policy.
- Apply a loopback firewall filter that permits SSH from the management prefix, OSPF, and ICMP.

*Pass criteria:* the static route appears on the far device only after you commit the policy. Filter counters increase. Removing the final accept term causes the implicit discard to drop traffic, and you demonstrate this.

**Project 1.3: configuration groups.** Use one group to set the interface description format and NTP servers on all devices.

*Pass criteria:* `show | display inheritance` shows the inherited values on each device.

**Credential timing:** take the exam at the end of this phase.

## Phase 2: Arista EOS (weeks 7 to 9)

**Skills:**
- EXEC, privileged EXEC, and configuration modes.
- Configuration sessions: `configure session`, `commit`, `abort`, and `commit timer`.
- `configure replace`.
- `show running-config diffs`.
- eAPI, enabled with `management api http-commands`.
- eBGP configuration, including address-family activation of neighbors.

**Project 2.1: eBGP and sessions.**
- Build two cEOS nodes with an eBGP session.
- Make a change inside a configuration session, inspect it, and abort it.
- Make a second change and commit it with a commit timer.
- Retrieve `show ip bgp summary` as JSON through eAPI with `curl`.

*Pass criteria:* the aborted change is absent from the running configuration. The timed commit reverts without confirmation. The JSON output parses with `jq`.

**Project 2.2: translate Project 1.2.** Reproduce the route advertisement and the control-plane protection on EOS with a route-map, a prefix-list, and a control-plane ACL.

*Pass criteria:* the observable results match Project 1.2.

## Phase 3: NX-OS (weeks 10 to 12)

**Skills:**
- Feature enablement with `feature`, and `show feature`.
- The `management` VRF.
- Routed ports versus switchports.
- Configuration sessions.
- Checkpoints and rollback.
- OSPF and BGP configuration.

Feature support on Nexus 9000v depends on the release. Read the Cisco release notes before you plan any project that needs vPC or VXLAN.

**Project 3.1: feature gating.** Show that `router bgp` is rejected before `feature bgp` is enabled. Enable the feature and configure OSPF underlay with an eBGP session.

*Pass criteria:* the rejection and the working session are both documented.

**Project 3.2: checkpoint recovery.** Create a checkpoint, apply a disruptive change, and roll back to the checkpoint.

*Pass criteria:* `show running-config` after the rollback matches the configuration at the checkpoint.

## Phase 4: FRRouting (weeks 13 to 15)

**Skills:**
- The daemon model: `zebra`, `bgpd`, `ospfd`, `staticd`, and the `/etc/frr/daemons` file.
- `vtysh` and integrated configuration in `frr.conf`.
- `frr-reload.py`.
- Route-maps and prefix-lists.
- How `zebra` installs routes into the Linux kernel forwarding table.
- BGP unnumbered (`neighbor <interface> interface remote-as external`).

**Project 4.1: BGP unnumbered.** Build two FRRouting nodes with BGP unnumbered. Then attempt the same session between FRRouting and cEOS.

*Pass criteria:* the FRRouting-to-FRRouting session reaches `Established`. The FRRouting-to-EOS result is documented, including the cause of any failure.

**Project 4.2: eBGP policy requirement.** FRRouting enables `bgp ebgp-requires-policy` by default in current releases. Show that an eBGP session reaches `Established` and exchanges no prefixes until you attach inbound and outbound route-maps. Then disable the setting and compare.

*Pass criteria:* the evidence shows prefix counts before and after each change.

## Phase 5: SONiC (weeks 16 to 18)

**Skills:**
- The Redis databases (`CONFIG_DB`, `APPL_DB`, `STATE_DB`, `ASIC_DB`).
- The container set (`swss`, `syncd`, `bgp`, `teamd`, `lldp`).
- The `config` and `show` commands.
- `config_db.json`, `config save`, and `config reload`.
- FRRouting access through `vtysh` in the `bgp` container.

The `sonic-vs` image has no hardware ASIC. Forwarding features are limited. Read the containerlab kind page before you design a test around a forwarding feature.

**Project 5.1: persistence.**
1. Configure an interface address and a BGP neighbor with the `config` CLI.
2. Run `config reload` without saving. Show that the changes are gone.
3. Repeat the changes, run `config save`, and run `config reload`.

*Pass criteria:* the changes are absent in step 2 and present after step 3.

**Project 5.2: database inspection.** Use `sonic-db-cli CONFIG_DB keys '*'` to list the keys that your configuration created. Map each `config` command to the keys it wrote.

*Pass criteria:* a table of at least five command-to-key mappings.

## Phase 6: multi-vendor capstone (weeks 19 to 21)

**Project 6: eBGP underlay with four network operating systems.**
- Build a spine-and-leaf topology with at least four distinct network operating systems: Junos, EOS, FRRouting, and one of NX-OS or SONiC, as the host allows.
- Run eBGP with a prefix-list policy that permits only loopbacks.
- Store the topology file, device configurations, and a verification matrix in Git.
- Write a one-page rollback runbook for each platform from the table above.

**Fault injection tests:**
1. Fail a link and record the convergence evidence.
2. Apply a policy error on one device and recover it with that platform's rollback mechanism.
3. Rebuild the underlay on a fresh deployment from the repository and runbooks only.

*Pass criteria:* all three tests complete with evidence. The rebuild finishes within 3 hours without consulting vendor documentation for commands already in the runbooks.

## Credential: JNCIA-Junos

Juniper replaced JN0-105 with JN0-106 in April 2026. Confirm the exam code and objectives on Juniper's JNCIA-Junos page before you buy materials. Material labeled JN0-105 can be partly outdated.

The exam has 65 multiple-choice questions, lasts 90 minutes, and is delivered through Pearson VUE. It has no CLI simulations or labs. The seven objective domains are Networking Fundamentals, Junos OS Fundamentals, User Interfaces, Configuration Basics, Operational Monitoring and Maintenance, Routing Fundamentals, and Routing Policy and Firewall Filters.

The exam tests concepts. It does not verify that you can operate a device. The projects verify that. Do not use exam dump sites. They distribute leaked exam content, and using them violates certification program rules.

## Constraints

- **Image access:** if you cannot obtain an image legally, replace that platform's phase with the nearest available platform and record the substitution.
- **Host capacity:** if the host cannot run VM-based nodes, drop Phase 5 first, then Phase 3. SONiC on `sonic-vs` cannot validate forwarding behavior. NX-OS is the closest to the IOS syntax you already know, so it adds the least new material per hour.
- **Priority:** Phases 1, 2, and 4 transfer the most. Junos has the largest conceptual difference from IOS, so do not shorten Phase 1.
- **Prerequisite:** containerlab requires a Linux host. Complete shell, Docker, and package management basics before Phase 0.
