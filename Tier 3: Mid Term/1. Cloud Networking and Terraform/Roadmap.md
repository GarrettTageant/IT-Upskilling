Claude-generated roadmap:

## Structure and constraints

This roadmap has seven phases (0 to 6) and takes 20 weeks, assuming 8 hours per week. Each phase ends with a project. A project passes only when every pass criterion has recorded evidence in a Git repository. Evidence means command output, a packet capture, or a flow log record.

Four constraints affect the plan:

1. **Physical circuits.** Direct Connect and ExpressRoute need a port or a partner-provided circuit. A personal account cannot complete a real build without paying for one. Phase 5 therefore tests BGP and VLAN behavior on an emulated provider edge, and tests provisioning only through documentation review and Terraform code. The emulation does not validate the Letter of Authorization and Connecting Facility Assignment (LOA-CFA) process, the cross connect, or provider activation of a circuit.
2. **Changing numbers.** Quotas change without notice. For example, Azure raised the default NSG limit to 2,000 security rules per NSG in May 2026. Retrieve every quota from current vendor documentation at the start of each project. The cited values in this roadmap were retrieved on 26 October 2026.
3. **Hourly billing.** NAT gateways, Transit Gateway attachments, Site-to-Site VPN connections, Azure VPN gateways, and ExpressRoute gateways accrue charges while they exist. Before the first apply, set an AWS Budgets alert and an Azure Cost Management budget. Run `terraform destroy` at the end of every session.
4. **Certification fit.** See the last section.

### Construct mapping

| Function | AWS | Azure |
|---|---|---|
| Regional virtual network | VPC. Each subnet exists in one Availability Zone. | VNet. Each subnet spans all Availability Zones of the region. |
| Subnet-level packet filter | Network ACL: stateless, allow and deny rules, rule numbers 1 to 32766 | NSG associated with the subnet |
| Interface-level packet filter | Security group: stateful, allow rules only | NSG associated with the NIC: stateful, allow and deny rules, priority 100 to 4096 |
| Multi-network routing | Transit Gateway | VNet peering (not transitive) with a hub network virtual appliance (NVA) |
| IPsec VPN with BGP | Site-to-Site VPN on a Transit Gateway or virtual private gateway | VPN Gateway, route-based |
| Private circuit | Direct Connect | ExpressRoute |

## Phase 0: prerequisite check (weeks 1 to 2)

Phases 3 to 5 require Linux routing and IPsec troubleshooting. Do not start paid cloud work until Project 0 passes.

Study:

- CIDR arithmetic and longest-prefix match.
- eBGP session establishment, the AS_PATH, LOCAL_PREF, and MED attributes, communities, and best-path selection.
- IKEv2 (RFC 7296), ESP (RFC 4303), NAT traversal (RFC 3948), and the difference between policy-based and route-based VPNs.
- On Linux: `ip`, `ss`, `nftables`, network namespaces, XFRM interfaces.

**Project 0: BGP over IPsec between two Linux hosts**

Build two Linux hosts (virtual machines or network namespaces) with a strongSwan IKEv2 tunnel on a route-based XFRM interface. Run FRRouting eBGP across the inside tunnel addresses.

Pass criteria:

1. `swanctl --list-sas` shows an established IKE SA and Child SA.
2. `show bgp summary` shows state Established and one prefix received from each peer.
3. A capture shows IKE_SA_INIT and IKE_AUTH on UDP 500 and ESP (IP protocol 50) data traffic. No cleartext of the inner traffic appears.
4. After you delete the Child SA, traffic recovers through rekeying, and you explain which timer controlled the recovery.

## Phase 1: Terraform and AWS VPC (weeks 3 to 5)

Study:

- Terraform workflow (`init`, `plan`, `apply`, `destroy`), HCL types, provider version constraints, and the `.terraform.lock.hcl` file.
- State, and remote state locking with the S3 backend (`use_lockfile`, which requires Terraform 1.11 or later; verify).
- Variables, locals, outputs, `count` versus `for_each`, modules, data sources, lifecycle arguments, and `moved` and `import` blocks.
- VPC components: CIDR blocks, subnets, route tables, internet gateway, NAT gateway, gateway versus interface VPC endpoints, DHCP option sets, flow logs, and Reachability Analyzer.

**Project 1: reusable VPC module**

Write a module that creates a VPC across two Availability Zones with public, private, and isolated subnets. Include a NAT gateway toggle, an S3 gateway endpoint, and flow logs.

Pass criteria:

1. A second `terraform plan` after `apply` reports no changes.
2. `terraform destroy` removes every resource, and a tag-based CLI query returns none.
3. Reachability Analyzer reports reachable for each permitted path and unreachable for each blocked path.
4. Flow logs contain both ACCEPT and REJECT records from a deliberate test.
5. A `terraform test` file asserts at least three module outputs.
6. `tflint` and a policy scanner (Checkov or Trivy) run without unexplained findings.

## Phase 2: security groups, network ACLs, and NSGs (weeks 6 to 7)

Study these exact limits and behaviors:

- A new security group has no inbound rules and one outbound allow-all rule. Rules are allow-only, and the security group evaluates all rules without order.
- The default is 60 rules per security group and 5 security groups per network interface (adjustable to 16), and the product of these two quotas cannot exceed 1,000.
- A network ACL has a default of 20 rules, and this quota applies separately to inbound and outbound rules.
- Network ACLs apply only to traffic that enters or leaves a subnet. Traffic between two interfaces in the same subnet is not evaluated by the network ACL.
- Network ACLs are stateless. Return traffic needs an explicit rule for the client ephemeral port range. Linux clients use the range in `/proc/sys/net/ipv4/ip_local_port_range`.
- Azure NSGs are stateful and support allow and deny. For inbound traffic, the subnet NSG is evaluated before the NIC NSG. For outbound traffic, the order reverses. Both NSGs must allow the flow.
- Azure flow logging is moving from NSG flow logs to virtual network flow logs. Verify the current retirement dates before choosing one.

**Project 2: behavioral proof in both clouds**

Build a client and a TCP listener in each cloud. Use `aws_vpc_security_group_ingress_rule` resources, not inline rules.

Pass criteria. Each test has recorded evidence (client output plus a flow log record or Network Watcher IP flow verify result):

1. **Statefulness.** The security group allows inbound TCP 8080. The network ACL allows inbound 8080 but omits the outbound ephemeral range. The connection fails. After you add the range, the connection succeeds.
2. **Explicit deny.** A network ACL deny rule for one /32 blocks that source while the security group allows it. Swapping rule numbers changes the result.
3. **Same-subnet exemption.** A deny-all inbound network ACL does not block traffic between two instances in the same subnet. The same network ACL blocks a third instance in a different subnet.
4. **Security group reference.** A rule with a security group as the source permits only members of that group.
5. **Azure NSG order.** A subnet NSG deny overrides a NIC NSG allow. Lower priority numbers win.
6. **Write-up.** One table lists each behavior in AWS and Azure, with the evidence file name.

## Phase 3: Transit Gateway and Azure hub-and-spoke (weeks 8 to 10)

Study:

- Transit Gateway attachment types: VPC, VPN, Direct Connect gateway, peering, and Connect.
- Route table association versus propagation. An attachment associates with one route table and can propagate to several.
- A VPC attachment uses one subnet per Availability Zone. A CIDR that overlaps an existing propagated CIDR is not propagated. Peering attachments use static routes.
- Appliance mode keeps both directions of a flow on the same appliance interface. Without it, return traffic can use another Availability Zone and fail stateful inspection.
- Quotas: Each transit gateway allows 5,000 attachments, 20 route tables, and 10,000 combined routes (route tables and combined routes can be increased).
- Azure: peering is not transitive. Hub routing needs an NVA, a user-defined route with next hop type VirtualAppliance, and IP forwarding enabled on both the Azure NIC and the Linux kernel (`net.ipv4.ip_forward=1`).

**Project 3A: segmented Transit Gateway**

Create three VPCs (prod, dev, shared) with `for_each` over a map. Use two Transit Gateway route tables. The spoke table is associated with prod and dev and holds only the shared propagation. The shared table receives propagation from all three.

Pass criteria:

1. Prod to shared and dev to shared succeed.
2. Prod to dev fails, and Reachability Analyzer reports the cause.
3. A blackhole route blocks one prefix, and you demonstrate the effect.
4. Stretch: an inspection VPC with a Linux `nftables` forwarder fails across Availability Zones without appliance mode and succeeds with it. A packet capture shows the asymmetric return path.

**Project 3B: Azure hub-and-spoke**

Pass criteria:

1. Two spokes cannot reach each other through plain peering.
2. Spoke-to-spoke succeeds through the hub NVA with user-defined routes.
3. `azurerm` provider configuration includes `subscription_id` (required in 4.x; verify).

## Phase 4: BGP over IPsec (weeks 11 to 14)

Study:

- IKEv2 exchanges (IKE_SA_INIT, IKE_AUTH, CREATE_CHILD_SA), proposals, perfect forward secrecy, lifetimes, rekeying, dead peer detection (RFC 3706), and NAT-T on UDP 4500.
- Route-based tunnels with traffic selectors of 0.0.0.0/0, and eBGP on TCP 179 between inside tunnel addresses.
- Tunnel overhead, path MTU discovery, and TCP MSS clamping.
- AWS: each Site-to-Site VPN connection has two tunnels. Each tunnel supports up to 1.25 Gbps, ECMP aggregates tunnels, and ECMP does not work on static-routing connections. A customer gateway can advertise 1,000 dynamic routes to a Transit Gateway VPN.
- Azure: VPN Gateway default BGP ASN is 65515. Azure accepts a custom APIPA BGP address range. Confirm that range and the AWS inside-tunnel /30 rules in current documentation.

**Project 4: AWS Site-to-Site VPN with an emulated customer gateway**

Deploy strongSwan and FRRouting on an EC2 instance in a second VPC (standing in for the on-premises site). Create `aws_customer_gateway` and `aws_vpn_connection` with `transit_gateway_id` and dynamic routing.

Pass criteria:

1. IKE SA, Child SA, and BGP are established on both tunnels.
2. `iperf3 -P 8` flows increment the packet counters of both SAs (`ip -s xfrm state`), which proves ECMP.
3. Blocking tunnel 1 with `nftables` moves traffic to tunnel 2. You measure packet loss with continuous probes and explain it using the configured DPD and BGP hold timers.
4. `ping -M do -s` finds the largest unfragmented payload. An `nftables` MSS clamp rule reduces the MSS in a captured SYN.

**Project 5: AWS-to-Azure BGP over IPsec**

Connect an AWS VPN connection to an Azure VPN Gateway.

Pass criteria:

1. At least two tunnels reach BGP Established.
2. Each cloud learns the other cloud's VPC or VNet prefixes through BGP.
3. Stretch: Azure active-active with two public IPs and four tunnels.
4. Destroy the Azure gateway at the end of the session. Gateway creation takes tens of minutes, so plan sessions accordingly.

## Phase 5: Direct Connect and ExpressRoute (weeks 15 to 17)

Study, AWS:

- Dedicated versus hosted connections, link aggregation groups, and 802.1Q VLAN per virtual interface (VIF).
- Private, public, and transit VIFs. A dedicated connection supports 51 VIFs in total (50 private or public plus 1 transit), and a hosted connection supports one VIF. A Direct Connect gateway associates with up to 6 transit gateways (not adjustable).
- The default inbound allocation is 100 prefixes per address family on a private or transit VIF, and prefix controls can change it.
- With a transit gateway association, only the configured allowed prefixes are advertised to on-premises. With a virtual private gateway association, the list filters to the same or smaller CIDRs.
- BGP MD5 authentication, BFD, and the AWS local preference communities 7224:7100, 7224:7200, and 7224:7300 (verify). Direct Connect is not encrypted by default. Read the options for MACsec and for IPsec over Direct Connect.

Study, Azure:

- Circuit SKUs, private peering versus Microsoft peering, primary and secondary /30 links per peering, and the Microsoft ASN 12076.
- Private peering supports 4,000 IPv4 prefixes by default and 10,000 with ExpressRoute Premium.
- ExpressRoute virtual network gateways, FastPath, Global Reach, and ExpressRoute Direct.

**Project 6: emulated provider edge in containerlab**

Build two FRR nodes. One represents your edge and one represents the cloud edge. Create two VLAN subinterfaces with an eBGP session on each.

Pass criteria:

1. Both sessions are Established with MD5 authentication. BFD sessions are Up.
2. Community-based LOCAL_PREF policy moves the best path when you change the community. `show bgp ipv4 unicast <prefix>` documents both states.
3. Link failure convergence is measured with BFD and without BFD, and compared with the configured timers.
4. A `maximum-prefix 100` limit tears down the session when you advertise 101 prefixes. You then record how the AWS documentation describes the real behavior.
5. Terraform for `aws_dx_gateway` and `aws_dx_gateway_association` with `allowed_prefixes` passes `terraform validate` and `plan`. The write-up explains the transit gateway versus virtual private gateway difference.

**Project 7: hybrid design review**

Write a design for two on-premises sites, one AWS Region, and one Azure region. Use dual Direct Connect locations with a Site-to-Site VPN backup, and ExpressRoute with a coexisting VPN.

Pass criteria:

1. A route preference table is verified by the experiments in Projects 4 to 6, not copied from documentation.
2. A failure-mode table lists each single failure, the expected path, and the detection mechanism.
3. A method of procedure (MOP) includes rollback criteria and passes review by another person.

## Phase 6: capstone (weeks 18 to 20)

**Project 8: pipeline-driven hybrid environment**

Deploy the Phase 3 to Phase 5 environment from one repository using GitHub Actions or GitLab CI.

Pass criteria:

1. Pull requests run `fmt`, `validate`, `tflint`, a policy scan, and `plan`. Apply requires manual approval.
2. Pipelines authenticate to AWS and Azure through OpenID Connect (OIDC) federation. No long-lived access keys exist in the repository or the CI variables.
3. Remote state uses locking in S3 and in an Azure Storage backend.
4. A scheduled `terraform plan -refresh-only` detects a deliberately introduced manual change.
5. A hand-built resource is adopted into state with an `import` block and no recreation.
6. Modules are versioned with Git tags, and a `terraform test` suite passes in CI.

## Certification checkpoints

- **Terraform Associate (004), after Phase 3.** The exam is one hour long and tests Terraform 1.12. The objectives include collaboration and governance features such as workspaces and projects, which this roadmap does not cover. Study them separately.
- **AWS Certified Advanced Networking – Specialty (ANS-C01), after Phase 5.** The exam validates the design, implementation, management, and security of AWS and hybrid networks at scale. The exam also covers services this roadmap excludes, such as DNS with Route 53 and load balancing. Do not schedule it on this roadmap alone.
- **Azure Network Engineer Associate (AZ-700).** Verify the current exam code and status on Microsoft Learn before you plan for it.
