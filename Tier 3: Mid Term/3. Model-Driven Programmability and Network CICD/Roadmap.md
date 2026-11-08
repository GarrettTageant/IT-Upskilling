Claude-generated roadmap:

The roadmap has eight phases after a prerequisite gate. Each phase ends in projects with pass/fail criteria, and a capstone pipeline integrates all of them. Time estimates assume 8 hours per week; scale them to your actual hours.

## Constraints that affect the plan

1. Batfish and pyATS produce value only if you can read the configurations and device state they analyze. A `bgpSessionStatus` result is useless if you cannot explain the BGP finite state machine states. Your fiber and datacenter work gives you the physical layer. You need configuration-plane fluency (routing, switching, ACLs) before Phase 6.
2. YANG model coverage differs by vendor and by software release. Expect most of your debugging time to go to unsupported paths, deviations, and vendor-specific encodings, not to protocol mechanics.
3. No single free platform supports every protocol in this roadmap. Verify the supported management interfaces of each platform and release in its documentation before you select it. Platform selection is in the table below.
4. pyATS and Genie support Linux and macOS. A Windows host requires WSL2 or a Linux virtual machine.

## Lab platforms

| Purpose | Platform | Notes |
|---|---|---|
| Lab orchestration | containerlab on a Linux host with Docker | Declarative topology files in YAML; deployable inside a CI job |
| NETCONF, gNMI, YANG | Nokia SR Linux in containerlab | Native YANG models; verify NETCONF and gNMI are enabled in the image you pull |
| RESTCONF, pyATS parsing | Cisco IOS XE (DevNet Sandbox, Cisco Modeling Labs, or a vrnetlab-built image) | Verify RESTCONF support and gNMI support for your release |
| Batfish | `batfish/allinone` container | Batfish does not require direct access to network devices, and runs on any operating system that supports Docker |
| CI/CD | GitLab CI or GitHub Actions with a self-hosted runner | Hosted runners cannot reach a private lab network |

## Phase 0: prerequisite gate (4 to 8 weeks, depending on your starting point)

Do not start Phase 1 until you pass every item.

1. Write a Python script that uses functions, a virtual environment, `argparse`, exception handling, `json`, `PyYAML`, and an XML parser, with one `pytest` test.
2. Use Git branches, merge requests or pull requests, and conflict resolution without assistance.
3. Authenticate over SSH with keys. Run, exec into, and network Docker containers.
4. Deploy a three-node topology with containerlab from a YAML file.
5. Configure and verify an OSPF adjacency and an eBGP session on a device CLI, then explain each state transition you observed.

## Phase 1: YANG (RFC 7950), 2 weeks

Study RFC 7950 (YANG 1.1), RFC 7951 (JSON encoding), RFC 8525 (YANG library), and RFC 8342 (NMDA, which adds the `<intended>` and `<operational>` datastores). Cover `container`, `list` and `key`, `leaf`, `leaf-list`, `choice`, `grouping`, `augment`, `deviation`, `must` (XPath 1.0), `when`, `identity`, `feature`, and `config false`.

- **Project 1.1:** Write a YANG module for a static route table. Use a keyed `list`, a `must` constraint that rejects a next hop equal to the route's own prefix, and a `leaf` with a `range` restriction. **Pass:** `pyang` and `yanglint` report zero errors, and `pyang -f tree` output matches your design.
- **Project 1.2:** Write six instance documents (three valid, three invalid, each in JSON). **Pass:** `yanglint` accepts all valid documents and rejects each invalid document for the specific constraint you intended.
- **Project 1.3:** Download `ietf-interfaces` (RFC 8343) and the vendor-native module for the interface on your lab device. **Pass:** you identify, from the models alone, which nodes are `config false`, and you explain the difference between the same interface in the IETF model and the vendor model.

## Phase 2: NETCONF (RFC 6241), 3 weeks

Study RFC 6241 (operations and capabilities), RFC 6242 (SSH transport, TCP port 830, subsystem `netconf`), RFC 6243 (with-defaults), and RFC 8526 (NMDA operations `get-data` and `edit-data`). Cover base:1.0 and base:1.1 framing, the `:candidate`, `:confirmed-commit`, `:validate`, and `:rollback-on-error` capabilities, the `operation` attribute values (`merge`, `replace`, `create`, `delete`, `remove`), and `rpc-error` structure.

- **Project 2.1:** Open a session with `ssh -s -p 830 <host> netconf`. Complete the `<hello>` exchange by hand and issue one `<get-config>`. **Pass:** you state which framing the session negotiated and why.
- **Project 2.2:** Use `ncclient` to read configuration with a subtree filter, then an XPath filter if the server advertises `:xpath`. **Pass:** the output of both filters is identical, or you document why the server rejected one.
- **Project 2.3:** Execute each of the five `edit-config` operations against the same list entry. **Pass:** a table in your notes lists the starting state, operation, result, and `rpc-error` (if any) for each.
- **Project 2.4:** Perform a change through the candidate datastore with `<lock>`, `<validate>`, and `<commit><confirmed/>`. Let the confirm timeout expire. **Pass:** the device reverts, and you document the default `confirm-timeout` value.

## Phase 3: RESTCONF (RFC 8040), 2 weeks

Study RFC 8040, RFC 8072 (YANG Patch), and RFC 8527 (NMDA for RESTCONF). Cover root resource discovery through `/.well-known/host-meta`, the `data`, `operations`, and `yang-library-version` resources, the media types `application/yang-data+json` and `application/yang-data+xml`, the query parameters `content`, `depth`, `fields`, `filter`, `insert`, `point`, `with-defaults`, `start-time`, and `stop-time`, and the semantic difference among `POST`, `PUT`, and `PATCH`.

- **Project 3.1:** Discover the RESTCONF root with `curl` and TLS verification enabled. **Pass:** you retrieve the root path from the server response and do not hard-code it.
- **Project 3.2:** Perform create, read, replace, merge, and delete on one resource with `curl`, then repeat with Python `requests`. **Pass:** you record the HTTP status code returned by each method and explain each.
- **Project 3.3:** Make a conditional update that uses the `ETag` and `If-Match` headers. Modify the resource from a second client between your read and write. **Pass:** the server rejects the stale write.
- **Project 3.4:** Send one request with a `fields` parameter and one with `depth`. **Pass:** you explain the difference between the two responses in terms of the YANG tree.

## Phase 4: gNMI and OpenConfig, 3 weeks

Study the gNMI specification in the `openconfig/reference` repository. Cover the `Capabilities`, `Get`, `Set`, and `Subscribe` RPCs, the encodings (`JSON`, `JSON_IETF`, `BYTES`, `PROTO`, `ASCII`), the `Set` operation order (delete, then replace, then update, applied as one transaction), and the `Subscribe` modes (`STREAM` with `SAMPLE`, `ON_CHANGE`, or `TARGET_DEFINED`; `ONCE`; `POLL`). gNMI runs over gRPC with TLS. Default ports vary by vendor, so record the port from the vendor documentation.

- **Project 4.1:** Run `gnmic -a <host>:<port> -u <user> -p <pass> --skip-verify capabilities`. Use `--skip-verify` in the lab only. **Pass:** you list the supported models and encodings, and state which encoding you selected and why.
- **Project 4.2:** Perform `get` on one path in the OpenConfig model and the same data in the native model. **Pass:** you document each path difference.
- **Project 4.3:** Send one `set` request that contains a delete, a replace, and an update. Make the last operation invalid. **Pass:** you demonstrate that no part of the request was applied.
- **Project 4.4:** Subscribe to interface counters in `SAMPLE` mode and to interface operational state in `ON_CHANGE` mode. Shut down an interface. **Pass:** the `ON_CHANGE` update arrives without a sampling delay, and you record the sample interval you used.

## Phase 5: protocol comparison, 1 week

- **Project 5.1:** Apply the same change (create a loopback interface with an address) through NETCONF, RESTCONF, and gNMI on one platform. **Pass:** a table compares request size, number of round trips, atomicity guarantee, error detail, rollback support, and idempotency for each protocol, based on your own measurements and the RFC or specification text.

## Phase 6: pyATS, 3 weeks

Use Cisco IOS XE for this phase. Genie parser coverage differs by operating system and by command, so verify coverage for each command before you depend on it. Study the testbed YAML format, the Unicon connection library, AEtest (`CommonSetup`, `Testcase` with `setup`, `test`, and `cleanup`, `CommonCleanup`), `device.parse()`, `device.learn()`, `Diff`, and Easypy job files.

- **Project 6.1:** Write a testbed file for three devices. **Pass:** `pyats validate testbed` succeeds and `pyats parse "show version" --testbed-file ...` returns structured data.
- **Project 6.2:** Write an AEtest script that verifies every BGP neighbor is `Established` and every expected OSPF neighbor is `FULL`. **Pass:** the script fails when you shut a neighbor interface, and the failure message names the neighbor.
- **Project 6.3:** Capture state before a change and after it, then compute a `Diff`. **Pass:** the diff reports only the changes you intended, and you add exclusions for volatile fields such as uptime counters.
- **Project 6.4:** Write one custom parser for a command that Genie does not cover. **Pass:** the parser has a schema and a passing unit test.

## Phase 7: Batfish pre-change validation, 3 weeks

Run the `batfish/allinone` container, and install `pybatfish` in a virtual environment. Study snapshots, reference snapshots, and these questions: `initIssues`, `parseWarning`, `undefinedReferences`, `unusedStructures`, `bgpSessionCompatibility`, `bgpSessionStatus`, `routes`, `traceroute`, `reachability`, `differentialReachability`, `testFilters`, `searchFilters`, `filterLineReachability`, `compareFilters`, `detectLoops`, and `multipathConsistency`. Study the assertion helpers in the pybatfish documentation.

Batfish analyzes configuration files and models the forwarding behavior that those files produce. It does not observe live device state. It does not detect hardware faults, optics faults, or software defects. It reports unsupported configuration lines as warnings, and you must read those warnings.

- **Project 7.1:** Upload the lab configurations as a snapshot. **Pass:** you resolve every `initIssues` result or document why you accepted it.
- **Project 7.2:** Seed five faults into a copy of the configurations (a mismatched BGP AS number, an undefined route-map reference, a shadowed ACL line, an MTU mismatch, a missing network statement). **Pass:** Batfish detects at least four, and you identify which one it cannot detect and explain why.
- **Project 7.3:** Make a planned change on a branch. Run `differentialReachability` between the baseline snapshot and the changed snapshot. **Pass:** the reported differences match your change plan, and any unexpected flow blocks the merge in your script's exit code.
- **Project 7.4:** Find one defect that Batfish passes and pyATS fails (for example, a neighbor that is configured correctly but whose interface is administratively down on the live device). **Pass:** you write two sentences that state which tool detects which class of defect.

## Phase 8: CI/CD pipelines, 5 weeks

Study one platform first, then port the minimum pipeline to the other. For GitLab CI: `.gitlab-ci.yml`, `stages`, `rules`, `needs`, `artifacts`, `services`, `environment`, `when: manual`, `resource_group`, and protected and masked variables. For GitHub Actions: `.github/workflows/*.yml`, triggers (`pull_request`, `push`, `workflow_dispatch`), jobs and steps, environments with required reviewers, secrets, and self-hosted runners.

- **Project 8.1, validation pipeline:** On every merge request, run `yamllint`, JSON Schema validation of the intent file, `yanglint` validation of rendered instance data, and a rendering check of the Jinja2 templates. **Pass:** each fault class you inject fails the correct job.
- **Project 8.2, Batfish gate:** Add a job that starts Batfish, runs your assertions and `differentialReachability`, and fails on any violation. **Pass:** the pipeline blocks a merge request that introduces an unintended reachability change.
- **Project 8.3, ephemeral lab:** Add a job that deploys the topology with containerlab, applies the candidate change, runs pyATS tests, and destroys the lab even when a test fails. **Pass:** no containers remain after a failed run.
- **Project 8.4, controlled deployment:** Add a manual-approval job that applies the change through NETCONF with `<commit><confirmed/>`, runs the pyATS post-check, and issues the confirming `<commit>` only if the post-check passes. **Pass:** a deliberately bad change reverts without human action.
- **Project 8.5, evidence:** Store the pre-state, post-state, diff, Batfish results, and job log as pipeline artifacts. **Pass:** an auditor can reconstruct what changed, who approved it, and what the validation results were from the artifacts alone.
- **Project 8.6, port:** Rebuild Projects 8.1 and 8.2 on the other CI platform. **Pass:** a table maps each concept (trigger, runner, secret, approval gate, artifact) between the two platforms, and your pipeline runs.

## Capstone and exit criteria

Build a repository that holds a YAML intent file for a three-node lab. A merge request that changes the intent file must produce rendered configuration, pass static validation, pass Batfish analysis, deploy to an ephemeral lab, pass pyATS tests, wait for approval, deploy with a confirmed commit, and store evidence.

You have met the objective when you can do each of the following without reference material:

1. Explain the difference between `PUT` and `PATCH` in RESTCONF and between `replace` and `merge` in NETCONF, with the resulting datastore content for each.
2. Select NETCONF, RESTCONF, or gNMI for a stated requirement and defend the choice from protocol properties.
3. Write a YANG module with a `must` constraint and a `deviation`.
4. State which defect classes Batfish detects, which defect classes pyATS detects, and which defect classes neither detects.
5. Inject a fault of each class into the capstone and show which pipeline stage stops it.
