Claude-generated roadmap:

## Scope and gap

The exam code is 200-901. Cisco renamed the DevNet track to the Automation track on February 3, 2026. The associate exam is now 200-901 v1.1 CCNAAUTO, and its exam topics did not change. DEVASC is the retired exam name. The exam lasts 120 minutes.

Your tool list covers only part of the exam. The published topics name Ansible, Terraform, and Cisco NSO as the automation tools to describe. The topic fragments I retrieved do not name Netmiko, NAPALM, Nornir, Jinja2, or TextFSM. Confirm this against the official exam topics PDF on Cisco's learning site.

This roadmap therefore has two goals. Phases 0 to 8 build the skills you listed. Phase 9 covers exam content that your list omits. Completing only Phases 0 to 8 leaves domains 2.0, 3.0, and 4.0 unprepared, which is 50% of the exam.

| Exam domain | Weight | Covered by your list | Phase |
|---|---|---|---|
| 1.0 Software Development and Design | 15% | Partly: Python, data formats, Git | 0, 1, 2, 8 |
| 2.0 Understanding and Using APIs | 20% | No | 9 |
| 3.0 Cisco Platforms and Development | 15% | No | 9 |
| 4.0 Application Deployment and Security | 15% | No | 9 |
| 5.0 Infrastructure and Automation | 20% | Partly: Ansible, Python | 3 to 7, 9 |
| 6.0 Network Fundamentals | 15% | No | 9 (review only) |

Netmiko, NAPALM, Nornir, TextFSM, and Jinja2 are used in employer environments. Time spent on them beyond the exam topics is a career investment, not exam preparation.

## Lab and rules

**Lab requirements**
- A Linux environment (native, virtual machine, or WSL 2) with Python 3, `venv`, Git, and a code editor.
- At least three SSH-reachable network devices. Use Cisco IOS XE as the primary target, because the exam is Cisco-specific. Options are Cisco Modeling Labs or a Cisco public sandbox. Verify current licensing, node limits, and availability before you commit.
- Containerlab with Arista cEOS is optional. The NAPALM `eos` driver requires eAPI to be enabled on the device.

**Rules for every project**
- Keep all code in Git. Credentials must not enter the repository. Use environment variables or `ansible-vault`.
- Each project must include a `README.md` that states how to run the project and shows the evidence for each pass criterion.
- Do not start the next phase until every pass criterion of the current phase is met.
- From Phase 3 onward, make every change on a branch.

## Effort budget

The budget is a planning figure. Re-estimate it after Phase 1 from your measured pace.

| Phase | Topic | Planned hours |
|---|---|---|
| 0 | Environment and Git fundamentals | 8 |
| 1 | Python for network scripting | 40 |
| 2 | JSON, YAML, XML | 12 |
| 3 | Netmiko and TextFSM | 24 |
| 4 | Jinja2 | 16 |
| 5 | NAPALM | 24 |
| 6 | Nornir | 24 |
| 7 | Ansible network modules | 32 |
| 8 | Git branching and pull-request review | 16 |
| 9 | Exam gap domains | 40 |
| 10 | Capstone and exam readiness | 30 |
| | Total | 266 |

You have limited Python experience, so Phase 1 is the largest risk to the schedule. The Linux course you are taking this quarter covers the prerequisites for Phase 0. Do not wait for the later Python course to start Phase 1.

## Phase 0: environment and Git fundamentals

**Learn**
- Python virtual environments and `requirements.txt`.
- Git: working tree, staging area, commit, `status`, `log`, `diff`, `restore`, `.gitignore`, `switch -c`, `merge`, and `push`, `pull`, and `fetch`.
- Conflict resolution.

**Project 0:** Create the repository `netauto-lab` with a `.gitignore` that excludes the virtual environment and Python caches. Create a deliberate merge conflict between two branches and resolve it.

**Pass criteria**
- `git log --graph --oneline` shows two branches merged.
- A search of the full history with `git log -p` finds no credentials.

## Phase 1: Python for network scripting

**Learn**
- Built-in types: `str`, `int`, `bool`, `list`, `tuple`, `dict`, and `set`.
- Control flow, functions, modules, classes, and exceptions (`try`, `except`, `else`, `finally`).
- File I/O with `with`, the `argparse`, `logging`, `ipaddress`, `re`, and `csv` modules, and unit tests with `unittest` or `pytest`. The exam covers test-driven development.

**Project 1:** Write `subnet_report.py`. It reads a CSV file of IPv4 prefixes. For each prefix, it outputs the network address, broadcast address, and usable host count. It also flags overlapping prefixes.

**Pass criteria**
- The usable host count is correct for `/30`, `/31`, and `/32` prefixes. A `/31` prefix has two usable addresses (RFC 3021), and a `/32` prefix has one.
- Invalid rows are reported with their line number, and the script does not stop.
- No unhandled traceback occurs for any input, including an empty file.
- A unit test suite passes, with at least one test per function.

## Phase 2: JSON, YAML, XML

**Learn**
- JSON (RFC 8259) has no comments. YAML supports comments and uses indentation. XML has attributes and namespaces.
- Use `json`, `yaml.safe_load`, and `xml.etree.ElementTree`. Never use `yaml.load` on untrusted input.
- PyYAML implements YAML 1.1. Unquoted `no`, `NO`, `yes`, `on`, and `off` load as booleans.
- The mapping of each format to Python `dict` and `list` objects.

**Project 2:** Load a YAML interface inventory, validate its required keys, and write it as JSON and as XML. Then parse a saved NETCONF `<rpc-reply>` that contains `ietf-interfaces` data (RFC 8343). Extract each interface name and its `oper-status`.

**Pass criteria**
- The XML parser handles the namespace `urn:ietf:params:xml:ns:yang:ietf-interfaces` correctly.
- A JSON round trip produces equal data.
- A test demonstrates and documents that the YAML value `NO` loads as `False`, and shows the fix.

## Phase 3: Netmiko and TextFSM

**Learn**
- Netmiko: `ConnectHandler` with `device_type='cisco_ios'`, `enable`, `send_command`, `send_config_set`, `send_config_from_file`, and `save_config`.
- SSH key options `use_keys` and `key_file`, and the exceptions `NetmikoTimeoutException` and `NetmikoAuthenticationException`.
- TextFSM: `Value` definitions with the options `Filldown`, `Required`, `List`, and `Key`. A template also has a `Start` state, rules that begin with `^`, and the actions `Record`, `Clear`, `Next`, and `Continue`.
- The ntc-templates repository and the `use_textfsm=True` parameter.

**Project 3a:** Collect `show version` and `show ip interface brief` from all lab devices using an inventory file from Phase 2. Parse the output with ntc-templates. Write a CSV report with hostname, software version, and the count of interfaces in `up/up` state.

**Project 3b:** Write your own TextFSM template for `show ip interface brief`, without copying the ntc-templates version.

**Project 3c:** Push interface descriptions with `send_config_set`. Capture the configuration before and after the change, and print a `difflib` diff.

**Pass criteria**
- An unreachable device and a device with wrong credentials each produce one logged error. The remaining devices still complete.
- Credentials come from environment variables.
- Your template matches the ntc-templates output on at least three captured outputs.
- Running Project 3c twice does not produce a second change.

## Phase 4: Jinja2

**Learn**
- `Environment`, `FileSystemLoader`, and `render`.
- Syntax: `{{ }}`, `{% %}`, and `{# #}`. Control structures: `for` with `loop.index`, and `if`, `elif`, and `else`.
- Built-in filters, a custom filter, `trim_blocks`, `lstrip_blocks`, `StrictUndefined`, macros, `include`, and `extends` with `block`.

**Project 4:** Generate complete configurations for four devices from one YAML data model and one template set. Each configuration includes the hostname, interfaces with IP addresses, an OSPF process, and NTP.

**Pass criteria**
- Rendering with `StrictUndefined` fails when you remove a required variable on purpose.
- An invalid IP address in the data model is rejected by a validation step before rendering.
- Rendered output matches a stored reference configuration with zero differences.
- After you apply the output with Netmiko, every rendered line appears in `show running-config`. Exceptions are lines that IOS does not display when they set a default value. List those exceptions explicitly.

## Phase 5: NAPALM

**Learn**
- Drivers: `ios`, `nxos`, `nxos_ssh`, `eos`, and `junos`. Use `get_network_driver` and the context manager.
- Getters: `get_facts`, `get_interfaces`, `get_interfaces_ip`, `get_lldp_neighbors`, and `get_config`.
- Configuration methods: `load_merge_candidate`, `load_replace_candidate`, `compare_config`, `commit_config`, `discard_config`, and `rollback`.
- A replace operation on the `ios` driver needs a full configuration file and SCP enabled on the device.

**Project 5:** Write a compliance auditor and a change workflow.
- The auditor compares `get_facts`, `get_interfaces`, and `get_lldp_neighbors` to an intended-state YAML file, and reports pass or fail per device.
- The change workflow runs `load_merge_candidate`, then `compare_config`, then waits for explicit confirmation, then runs `commit_config`. If you reject the diff, it runs `discard_config`.

**Pass criteria**
- A manual drift that you introduce is detected.
- The second run of the change workflow shows an empty diff.
- The script never commits when the diff does not match the diff you expect.
- You demonstrate `rollback` on a committed change.

## Phase 6: Nornir

**Learn**
- Nornir 3.x: `InitNornir`, the `SimpleInventory` files `hosts.yaml`, `groups.yaml`, and `defaults.yaml`, and inheritance from host to group to defaults.
- Task functions, `Result`, `AggregatedResult`, `nr.run`, and `nr.filter` with `F`.
- Failed-host handling: Nornir skips failed hosts in later `run` calls unless you reset them.
- Plugin packages: `nornir_netmiko`, `nornir_napalm`, `nornir_jinja2`, and `nornir_utils`.

**Project 6:** Combine Projects 3, 4, and 5 into one workflow. The workflow renders configurations, runs a NAPALM dry run, applies the change, and verifies the result with Netmiko and TextFSM. The inventory must have at least two groups with group-level data.

**Pass criteria**
- A filter by group runs tasks only on that group.
- A device with a wrong password fails without stopping other devices, and the final report lists it.
- The process exits with a non-zero status if any host failed.

## Phase 7: Ansible network modules

**Learn**
- Inventory, playbooks, plays, tasks, handlers, `when`, `loop`, `register`, `block` and `rescue`, roles, and variable precedence.
- The connection plugin `ansible.netcommon.network_cli`, with `ansible_network_os: cisco.ios.ios`, `ansible_become: true`, and `ansible_become_method: enable`.
- Modules: `cisco.ios.ios_facts`, `ios_command`, `ios_config`, `ios_interfaces`, `ios_l3_interfaces`, and `ios_vlans`.
- Resource module states: `merged`, `replaced`, `overridden`, `deleted`, `gathered`, `rendered`, and `parsed`.
- The `ios_config` parameters `lines`, `parents`, `src`, `backup`, and `save_when`.
- `ansible.netcommon.cli_parse` with the `ansible.netcommon.ntc_templates` parser, `--check`, `--diff`, `ansible-vault`, and `ansible-lint`.

**Project 7**
- Playbook A gathers facts and parsed command output, and writes a Jinja2-rendered inventory report.
- Playbook B manages VLANs and interfaces with resource modules and `host_vars`. It uses `gathered` to read state and `rendered` to preview commands offline.
- Role `base_config` reuses the Phase 4 templates and applies them with `ios_config` and `src`.

**Pass criteria**
- A second run reports `changed=0` for every host.
- `ansible-lint` reports no violations.
- A run with `--check --diff` changes nothing on the devices.
- Secrets are in `ansible-vault` files, not plaintext.

## Phase 8: Git branching and pull-request review

**Learn**
- Feature-branch workflow, `merge` (fast-forward, merge commit, squash) and `rebase`, `rebase -i`, `cherry-pick`, `stash`, and tags.
- `revert` versus `reset`.
- Branch protection (required reviews, required status checks, no force pushes), `CODEOWNERS`, and a CI check with `yamllint`, `ansible-lint`, and `pytest`.

A pull request review requires a second person. Use a study partner, or run Gitea or Forgejo locally with two users.

**Project 8:** Apply branch protection to the repository from Phases 1 to 7. Direct commits to `main` must be blocked. Complete five pull requests:
1. A feature.
2. A bug fix.
3. A change that receives "changes requested" and is revised.
4. A change with a merge conflict that you resolve by rebasing.
5. A `git revert` of a merged defective change.

**Pass criteria**
- Every merged pull request has at least one review comment from the second user.
- A pull request with a deliberate lint error cannot merge.
- The history of `main` contains no force pushes.

## Phase 9: exam gap domains

**Domain 2.0, APIs**
- Learn REST constraints, HTTP methods and status codes, authentication types, pagination, rate limiting, and webhooks.
- Project 9a: Use Python `requests` against a Cisco platform API in a sandbox. Authenticate, paginate, and handle HTTP 429 using the `Retry-After` header.

**Domain 5.0, model-driven programmability**
- Learn YANG (RFC 7950), NETCONF (RFC 6241, SSH port 830 by default), and RESTCONF (RFC 8040).
- Project 9b: On IOS XE, use RESTCONF with the media type `application/yang-data+json` to create, read, and delete `Loopback100`. Repeat with `ncclient` over NETCONF.

**Domain 3.0, Cisco platforms**
- The topics name Meraki, Cisco Catalyst Center, ACI, Cisco Catalyst SD-WAN, and NSO as network management platforms. Use the official topics PDF for the remaining platforms.
- Write a one-page capability summary for each platform.

**Domain 4.0, deployment and security**
- Learn Docker (`Dockerfile`, image, container), CI/CD stages, and the OWASP Top 10, including injection, authentication failures, and sensitive data exposure.
- Project 9c: Containerize the Phase 6 tool. Run it as a non-root user. Pass credentials at run time with environment variables.

**Domain 5.0, Terraform and NSO**
- The exam requires you to describe these tools. Write a one-page capability comparison of Ansible, Terraform, and Cisco NSO. Include declarative versus procedural behavior and state handling.

**Domain 6.0, network fundamentals**
- Your completed networking courses cover this domain. Review it with timed practice questions. No project applies.

**Pass criteria**
- Projects 9a and 9b complete with logged evidence.
- `docker history` on the image from Project 9c shows no credentials.
- You can explain each written summary without notes.

## Phase 10: capstone and exam readiness

**Capstone:** Build one repository that holds a YAML source of truth for devices, VLANs, and interfaces. A Nornir pipeline renders Jinja2 templates, previews the diff with NAPALM, applies after approval, and verifies with TextFSM. A second path reaches the same state with Ansible. All changes go through pull requests with CI.

**Pass criteria**
- One command takes the lab from base configuration to the target state.
- A second run changes nothing.
- A manual drift is detected and reported.

**Exam readiness**
- Complete full-length timed practice exams within 120 minutes.
- Do not advance to scheduling until you score at least 85% on three consecutive attempts. This margin is set by this roadmap, not by Cisco.
- Review each missed question against the official exam topics, and avoid brain dumps, which violate Cisco testing policy.
