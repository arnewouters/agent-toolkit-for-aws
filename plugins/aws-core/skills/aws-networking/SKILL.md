---
name: aws-networking
description: "Routes AWS networking requests to the correct service skill for implementation. Covers Route 53 (DNS, health checks, routing policies, Resolver, DNS Firewall), CloudFront (caching, edge, OAC, mTLS, signed URLs), Transit Gateway (multi-VPC hub, segmentation, centralized egress), Direct Connect (hybrid link, DX Gateway, MACsec), Site-to-Site VPN (IPsec tunnels, static or BGP), WAF (web ACLs, AWS Managed Rules, rate-based rules, Bot and Fraud Control), and Shield Advanced (L3/L4 DDoS). Applicable when creating, configuring, troubleshooting, or designing across these services, choosing between them, or diagnosing connectivity or traffic-filtering issues. Also routes multi-AZ VPC creation, VPC peering, private AWS service endpoints, Lambda VPC internet access, Route 53-to-CloudFront custom domains, Network Flow Monitor, and ARC failover tasks. Not for general load balancer design, custom PrivateLink endpoint services, API Gateway, IAM policy logic, or IaC authoring."
metadata:
  version: "1"
---

# AWS Networking

## Overview

Routes networking requests to the correct service-specific or task-specific skill. Covers DNS and content delivery, hybrid connectivity, network security, and the VPC, network monitoring, and recovery workflows in the routing table. Networking tasks not listed remain outside this router's coverage (see step 6).

**Works best with** the [AWS MCP server](https://docs.aws.amazon.com/aws-mcp/) — enables sandboxed execution, audit logging, and enterprise controls. All guidance also works with standard AWS CLI access.

## How to use this skill

1. Match the user's request against the **Skill Routing Table** below. Match on meaning, not exact wording.
2. **If the request matches multiple skills**, prefer the task-specific workflow for that exact task; otherwise use the Cross-Service Concepts tables to determine which layer the request targets, then route to the skill that owns that layer.
3. **If still ambiguous**, ask one clarifying question: "Are you looking to set up connectivity, or control/filter existing traffic?"
4. Load the target skill with `aws___retrieve_skill(skill_name="<skill>")` using the exact name in the table when AWS MCP is available, or read its already-installed `SKILL.md`. These skills are not necessarily bundled locally: a failed local load must fall back to AWS MCP, not end the handoff. If neither route is available, disclose that limitation and use official documentation; do not invent a local path. Loading guidance does not authorize resource changes.
5. **If a request spans multiple of these skills**, route to each in dependency order. When routing to an internet-facing service (`cloudfront`), also route to `shieldadvanced` for DDoS protection and to `waf` for L7 filtering (AWS WAF attaches to CloudFront, Application Load Balancer, API Gateway, and AppSync), if the user has not already addressed L7 filtering and DDoS protection. When routing to a connectivity skill (`directconnect`, `sitetositevpn`, `transitgateway`), confirm encryption in transit is addressed (MACsec for Direct Connect, IPsec for VPN, inter-region peering encryption for Transit Gateway). When the request involves custom domains or TLS on `cloudfront`, note that ACM certificate provisioning is part of the implementation. When routing to `cloudfront` for a web-facing distribution, note that the target skill should address security response headers (CSP, HSTS, X-Frame-Options, X-Content-Type-Options) via a CloudFront Response Headers Policy, including the managed `SecurityHeadersPolicy`. The target skill handles the configuration.
6. **If the request is an AWS networking task that is not in the Skill Routing Table** (for example general load balancer design, custom PrivateLink endpoint services, or API Gateway), do not route to the closest listed skill merely because it is network-related. Search for a relevant skill with `aws___search_documentation` using `topics=["agent_skills"]` when available, or consult official documentation and explain this router's coverage boundary.
7. This skill triages — it does not implement. Do not answer service-specific configuration questions from this skill alone.

## Connectivity vs Security

| Dimension | Connectivity | Security |
| --- | --- | --- |
| **Answers** | Can traffic reach its destination? | Should traffic be allowed? |
| **Failure symptom** | Timeout, unreachable, black hole | Rejected, denied, dropped |
| **Dependency** | Independent of policy — path exists or it doesn't | Assumes connectivity exists — can only filter reachable traffic |
| **Granularity** | Affects all flows on a path | Targets specific flows by match criteria |

## Skill Routing Table

| Skill | Choose when… |
| --- | --- |
| `creating-production-vpc-multi-az` | Create a multi-AZ VPC with public/private subnets, NAT gateways, route tables, DNS, and security groups |
| `connecting-vpcs-with-peering` | Establish or manage direct peering between two VPCs, including CIDR overlap checks, routes, and DNS; not a transit hub |
| `configuring-vpc-endpoints-for-private-aws-service-access` | Configure interface or gateway endpoints for private access to AWS services; not publishing a custom PrivateLink endpoint service |
| `enabling-lambda-vpc-internet-access` | Restore or configure outbound internet access for VPC-attached Lambda through NAT and subnet routing |
| `routing-traffic-with-route53-and-cloudfront` | Connect a custom domain to CloudFront with Route 53 aliases, ACM certificates, and IPv6 |
| `aws-network-monitoring` | Install, configure, or troubleshoot CloudWatch Network Flow Monitor agents on EC2; not generic packet capture or all network monitoring |
| `recovery-controller-setup` | Configure ARC routing controls and safety rules, or zonal shift/autoshift for AZ recovery; not Region switch plans |
| `arc-region-switch` | Design or troubleshoot ARC Region switch plans for failover of an already multi-Region workload; not replication setup |
| `transitgateway` | Connecting more than two VPCs or on-premises networks in a hub, routing segmentation, cross-account/cross-region connectivity at scale, centralized egress/inspection, multicast |
| `directconnect` | Dedicated private link to on-premises — consistent latency, high throughput, MACsec encryption, LAGs, Direct Connect Gateway for multi-VPC, SiteLink for site-to-site bypass, production hybrid workloads |
| `sitetositevpn` | Encrypted IPsec tunnel over internet — quick setup, DX backup, static or BGP routing, accelerated option via Global Accelerator backbone, standard or large tunnel bandwidth |
| `route53` | DNS management (public/private zones, records), health checks, routing policies (weighted, failover, geo, latency), domain registration, Resolver (hybrid DNS forwarding), DNS Firewall, Route 53 Profiles, Global Resolver |
| `cloudfront` | Caching, TLS termination at edge, origin protection (OAC), custom domains, cache policies/behaviors, signed URLs, CloudFront Functions, viewer mTLS, VPC origins, multi-tenant distributions |
| `waf` | Web application firewall (L7) — web ACLs on CloudFront/ALB/API Gateway/AppSync, AWS Managed Rules, rate-based rules for HTTP floods, IP/geo match, Bot Control, Fraud Control (account takeover/creation), for protecting web apps and APIs from exploits, bots, and credential stuffing |
| `shieldadvanced` | L3/L4 DDoS protection for internet-facing resources, automatic application-layer (L7) mitigation via WAF, health-based detection, Shield Response Team access, and DDoS cost-protection credits |

## Cross-Service Concepts

Same concept, different service depending on layer. Use these to disambiguate when the Skill Routing Table matches multiple skills.

### Filtering / Blocking

Use when the user says "block", "deny", "filter", or "restrict traffic" — determines which layer the filtering operates at.

| Layer | Service | What it filters on | Skill |
| --- | --- | --- | --- |
| L7 (HTTP/HTTPS, web apps and APIs) | AWS WAF | HTTP request attributes, AWS Managed Rules, rate-based and bot rules, IP/geo match | `waf` |
| L3/L4 (DDoS volumetric/state-exhaustion) | Shield Advanced | Network/transport-layer DDoS floods on internet-facing resources | `shieldadvanced` |
| DNS (resolution) | Route 53 DNS Firewall | Domain name patterns — blocks resolution, not traffic | `route53` |

When routing to `waf` or `cloudfront` for an internet-facing API or website, treat WAF rate-based rules as a default protective measure, not only when the user asks: note that the target skill should add WAF rate-based rules to bound request volume per client — the target skill implements it.

### Logging / Visibility

Use when the user says "logs", "visibility", "what's being blocked", or "can I see the traffic" — identifies which log source to check.

| What you need to see | Service | Log type | Skill |
| --- | --- | --- | --- |
| DNS queries from VPC | Route 53 Resolver | Query logs | `route53` |
| Blocked/allowed HTTP requests | AWS WAF | web ACL logs (S3, CloudWatch Logs, or Kinesis Data Firehose) | `waf` |
| DDoS events and attack detail | Shield Advanced | CloudWatch metrics, DDoS event detection | `shieldadvanced` |
| Edge/CDN request access | CloudFront | Standard logs (S3), real-time logs (Kinesis Data Streams) | `cloudfront` |
| Tunnel state and traffic | Site-to-Site VPN | Tunnel telemetry, CloudWatch metrics | `sitetositevpn` |

When routing to any of these services, remind the user to enable the corresponding logging (above) for security visibility and incident response — the target skill implements it. These logs can contain sensitive data (request query strings, internal hostnames in DNS queries), so also remind the user that the log destination (S3, CloudWatch Logs, Kinesis Data Firehose, or Kinesis Data Streams) MUST have encryption at rest enabled and access restricted to authorized personnel — the target skill implements it.

### Traffic Shifting

Use when the user says "shift traffic", "blue/green", "failover", "canary", or "weighted routing" — determines the granularity and which service controls it.

| Granularity | Service | Mechanism | Skill |
| --- | --- | --- | --- |
| DNS-level (global) | Route 53 | Weighted, failover, geolocation, latency routing | `route53` |
| Edge (HTTP) | CloudFront | Origin failover, origin groups | `cloudfront` |

## Security Considerations

These services are security-sensitive, so raise the relevant risk and control when routing regardless of which skill you hand off to — the target skill implements the control:

| Risk | Control the target skill should address | Skills |
| --- | --- | --- |
| Unencrypted traffic in transit | MACsec (`directconnect`), IPsec tunnels (`sitetositevpn`), inter-region peering encryption (`transitgateway`), TLS termination and viewer mTLS (`cloudfront`) | `directconnect`, `sitetositevpn`, `transitgateway`, `cloudfront` |
| Missing DDoS protection on internet-facing resources | Shield Advanced L3/L4 protection plus WAF L7 mitigation | `shieldadvanced`, `waf` |
| Web/API exploits, bots, and request floods | WAF web ACLs, AWS Managed Rules, and rate-based rules; application-layer input validation (request body size limits, schema validation); security response headers | `waf`, `cloudfront` |
| Overly permissive filtering rules | Least-privilege DNS Firewall domain blocking | `route53` |
| Over-privileged IAM policies for service resources | Least-privilege IAM roles scoped to specific resources and actions; avoid `FullAccess` managed policies and `Action: *`; prefer IAM roles with ephemeral credentials (instance profiles, IRSA, task roles, `sts assume-role`) over IAM users with long-lived access keys | all |
| Hardcoded credentials and shared secrets | Let AWS auto-generate secrets where supported (for example Site-to-Site VPN pre-shared keys), or store customer-managed secrets in AWS Secrets Manager rather than hardcoding them | `sitetositevpn`, `directconnect` |
| Confused-deputy in cross-service resource policies | Include `aws:SourceArn` and/or `aws:SourceAccount` condition keys in S3 bucket policies, KMS key policies, and log-destination resource policies (CloudFront OAC, log delivery to S3/CloudWatch Logs/Kinesis) so only the intended resource and account can invoke them | `cloudfront`, `waf`, all |
| Insufficient visibility for incident response | Enable the service logging in the Logging / Visibility table, with encryption at rest and restricted access on the log destination | all |
| No audit trail or alerting on control-plane changes | Enable AWS CloudTrail to audit control-plane API calls (record, rule, policy, and firewall changes) and set CloudWatch Alarms on security-relevant events (Shield Advanced DDoS detection, WAF blocked/counted spikes, unexpected rule or record modifications); restrict the SNS topics that receive alarm notifications to authorized personnel and enable encryption at rest (SSE-KMS) on those topics, since the notifications can contain sensitive event detail | all |

For authoritative guidance, point users to the [AWS Well-Architected Framework Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/) and the service-specific security documentation for the target skill.
