# Implementation reference

[Back to the README](../README.md)

This reference describes the CLI behavior implemented in `main.go` and `aws_collector.go`. It explains which collected fields reach each output, how security findings are produced, and what snapshot comparisons can detect.

## Resource and output coverage

JSON inventory and regional snapshots include all collected resource types below. Collection uses a single response from each list/describe operation; additional API pages are not retrieved.

| Resource | Collection scope | DOT / PNG / SVG | HTML nodes | CSV rows | Markdown detail tables |
| --- | --- | --- | --- | --- | --- |
| VPCs | Primary IPv4 CIDR, Name tag, default flag, VPC-level flow-log presence | Yes | Yes | Yes | Yes |
| Subnets | VPC, IPv4 CIDR, AZ, Name tag, `MapPublicIpOnLaunch` | Yes | Yes | Yes | Yes |
| EC2 instances | Running instances with VPC, subnet, and private IP metadata | Yes | Yes | Yes | Yes |
| Security groups | IPv4 ingress/egress ranges, protocol, port bounds, description | No | Count only | Yes | Count only |
| Load balancers | ELBv2 resources with ARN, name, and VPC; scheme, type, subnet IDs | Yes | Yes | Yes | Yes |
| RDS instances | Engine/version/class, VPC/subnet group, security group IDs, public/encrypted flags | No | Yes | Yes | Yes |
| NAT gateways | VPC/subnet, state, Name tag, public/private IP fields | No | Yes | Yes | Count only |
| Internet gateways | Attached VPC IDs and Name tag | No | Yes | Yes | Count only |
| Route tables | VPC, Name tag, main flag, selected route destinations/targets/state | No | No | No | Count only |
| VPC peerings | Requester/accepter VPC IDs, status, Name tag | No | No | No | Count only |
| Lambda functions | Name, ARN, runtime, and VPC configuration when present | No | VPC-attached only | Yes | Yes |
| EKS clusters | Name, ARN, status, VPC/subnets, cluster security group ID | No | Yes | Yes | Yes |
| ECS clusters | Name, ARN, status, active service count, running task count | No | Yes, unconnected | Yes | Yes |

Load balancer discovery uses `elasticloadbalancingv2`, covering ALB, NLB, and Gateway Load Balancer metadata when returned with a VPC. Classic Load Balancers are not collected. EC2 stopped/terminated instances, ECS task/service topology, EKS workloads, network ACLs, transit gateways, VPNs, and a full route or reachability model are outside the collected model.

Only the `Name` tag is retained for resource types that use `getNameTag`; security groups use their group names, and some services use their own identifiers. Arbitrary tags are not exported. CSV has columns `Type`, `Name`, `ID`, `VPC ID`, `Details`, and `Tags`; the `Tags` column is empty in every row.

## Collection and visualization limits

- **Pagination:** list/describe calls do not follow continuation tokens. Missing resources may reflect truncated API results rather than absence in AWS.
- **Partial scans:** failures describing VPCs, subnets, EC2 instances, or security groups abort collection for that region. Other service API failures generally log warnings and leave those arrays empty. Failed EKS cluster descriptions are skipped; ECS response failure entries are not surfaced individually.
- **Silent metadata failures:** errors from flow-log and EBS-volume lookups are ignored. A failed flow-log lookup leaves VPCs marked without flow logs; failed EBS lookups leave the instance's encryption flag at its initial `true` value.
- **Flow logs:** `FlowLogsEnabled` means a returned flow-log record references the VPC ID. Status and delivery health are not checked, and logs attached only to subnets or interfaces do not set this flag.
- **Subnet classification:** `IsPublic` mirrors `MapPublicIpOnLaunch`; route-table associations and Internet Gateway routes are not evaluated. It is not a reachability verdict.
- **IPv4 and rules:** security group collection omits IPv6 ranges, prefix lists, referenced security groups, and rule descriptions. Route collection retains the IPv4 destination plus gateway, NAT gateway, instance target, and state fields; other destinations/targets are omitted.
- **Topology:** graph edges express resource membership or attachment. DOT links VPCs to subnets/load balancers and subnets to instances. HTML adds subnet-to-NAT and VPC-to-Internet-Gateway/RDS/Lambda/EKS relationships. Neither renderer draws security group rules, routing paths, peerings, or load balancer targets.
- **Graph identifiers:** DOT leaves ordinary AWS hyphens in unquoted node identifiers and does not escape label text. PNG/SVG rendering can fail. HTML sanitizes identifiers, but Lambda/EKS names and RDS identifiers are not scoped by region; matching names across regions can collide. Edges to missing parent resources can also prevent a graph from loading.
- **HTML dependency:** the generated file embeds graph data and loads Cytoscape.js 3.28.1 from `cdnjs.cloudflare.com`. It has no bundled offline library. Labels are truncated; search checks the displayed label and internal node ID. The details panel omits false/empty values, so use JSON for full collected fields. The HTML map does not display security findings.

Regions are scanned sequentially. Merged resource names receive a `[region]` prefix, but resource order is not stable because regions are merged from a Go map. Do not interpret JSON ordering changes as infrastructure drift.

## Security checks

Checks are fixed in [security_checker.go](../security_checker.go) and run once per collected region. There are no custom ports, effective-permission analysis, reachability probes, compliance mappings, or automatic remediation. YAML severity thresholds do not filter results.

| Finding type | Severity | Implemented trigger |
| --- | --- | --- |
| Open Security Group | High | An ingress IPv4 range equals `0.0.0.0/0` and `FromPort` is `22`, `3389`, `21`, `23`, or `0`. |
| Overly Permissive Egress | Low | An egress rule has protocol `-1` and an IPv4 range of `0.0.0.0/0`. |
| Direct Public Exposure | Medium | At least one collected EC2 instance has a public IP and the region's collected load balancer list is empty. |
| Default VPC In Use | Low | A collected VPC has `IsDefault=true`; resource usage is not checked. |
| Missing VPC Flow Logs | Medium | The collected VPC has `FlowLogsEnabled=false`. |
| IMDSv2 Not Enforced | Medium | A collected running instance's metadata `HttpTokens` value is not `required`. |
| Unencrypted EBS Volume | Medium | At least one successfully described attached EBS volume is unencrypted. Reported against the instance, not the volume. |
| Publicly Accessible RDS | Critical | A collected database has `PubliclyAccessible=true`. |
| Unencrypted RDS Instance | High | A collected database has `Encrypted=false`. |
| Overly Permissive IAM Role | High | The instance-profile ARN contains `Admin`, `FullAccess`, `PowerUser`, or `AdministratorAccess` (case-sensitive). |

Ingress checks inspect only the start of a port range, without checking whether a risky port falls elsewhere inside it or whether the protocol is TCP/UDP. Missing port bounds become `0`, which is labeled “All Ports” by the checker. Database ports such as `3306` and `5432` have display-name helpers but are not included in the risky ingress-port checks. IPv6 public access is not evaluated.

The IAM finding is a name heuristic on the **instance profile ARN**, not a review of attached role policies. The public-exposure check is regional: any collected load balancer suppresses it, even if unrelated to the instance. An RDS public-access flag does not establish that security groups or routes allow access. These findings need human review, and a scan with no findings does not establish security or compliance.

Findings appear in terminal output, Markdown, and SARIF. JSON, CSV, diagrams, and snapshots do not contain the `Risk` records. Severity levels do not affect the process exit code.

## SARIF export

The exporter writes version `2.1.0` with one run, a rule per observed finding type, and a result per finding. Rule IDs replace spaces in the finding type with hyphens. Severity mapping is:

| Cloud NetMapper severity | SARIF level |
| --- | --- |
| Critical / High | `error` |
| Medium | `warning` |
| Low | `note` |

Locations use synthetic `aws://` URIs derived from the resource display name and ID, rather than source-code file locations. The exporter does not upload reports or enforce a CI threshold. Its driver version is hard-coded to `1.0.0` and is not a release lookup; its information URL still uses the older `msaadshabir` repository owner.

Consumer compatibility is not guaranteed: a scan with no findings serializes the uninitialized `rules` and `results` slices as JSON `null`, and synthetic location URIs are not URL-escaped. Validate or normalize the export for your SARIF consumer before using it in an upload or automated gate.

## Diff coverage

`DetectChanges` compares resources by ID, ARN, or service name within each regional snapshot. Additions and removals are tracked for every included type. Modifications use only these fields:

| Resource | Identity key | Fields used to detect modifications |
| --- | --- | --- |
| VPC | ID | Name, CIDR, flow-log flag |
| Subnet | ID | Name, CIDR, public flag |
| EC2 instance | ID | Instance type, public IP, instance-profile ARN (`IAMRole`), subnet ID |
| Security group | ID | Ingress rule count, egress rule count |
| Load balancer | ARN | Scheme |
| RDS instance | ID | Instance class, public-access flag |
| NAT gateway | ID | State |
| Internet gateway | ID | No modification checks; additions/removals only |
| Lambda function | Name | Runtime, VPC ID |
| EKS cluster | Name | Status, VPC ID |
| ECS cluster | Name | Active service count, running task count |
| Route table | — | Excluded from comparisons |
| VPC peering | — | Excluded from comparisons |

Changing a security group's ports, CIDRs, or protocols without changing rule counts is invisible to diff mode. EC2 private IP, security group attachments, IMDSv2, and EBS encryption changes are also not compared; RDS encryption changes are not compared. An instance-profile ARN change triggers an EC2 modification but is not included in the printed detail list. Use the raw inventories or snapshots when reviewing fields beyond this table.

An unchanged report means only that these comparisons found no differences in the collected data. Diff reports are written only when changes exist; they do not describe collection completeness. Malformed baseline contents beyond JSON syntax are not validated. Baselines should be intact snapshots generated by this CLI for the same account and region.

## Code structure

```text
main.go                    CLI flags, config overrides, regional scan loop,
                           resource merge, snapshot handling, output selection
aws_collector.go           CLI resource structs and AWS API collection
security_checker.go        Fixed risk checks and remediation text
visualizer.go              Smaller DOT topology for Graphviz
html_visualizer.go         HTML template, Cytoscape graph data, browser controls
report_generator.go        Markdown, CSV, and SARIF generation
diff_detector.go           Snapshot persistence and selected-field comparisons
security_checker_test.go   Selected risk, DOT text, and label/ID helper tests
config/config.go           YAML search/loading and built-in defaults
logger/logger.go           slog wrapper and verbosity parsing
providers/                 Separate provider abstraction and implementations
types/types.go             Resource types used by the provider package
```

The CLI loads configuration, scans each region through `getAWSResources`, checks risks, merges collected resources, compares previous snapshots if requested, saves snapshots if enabled, writes JSON/DOT, and generates the selected format. Snapshot writes happen before rendering, so a later PNG/SVG or export failure can still advance the baseline. Snapshot and diff-report write errors are warnings; primary JSON, DOT, or selected-output failures exit with an error.

The `providers.CloudProvider` interface and `types.CloudResources` model are separate from the CLI's `AWSResources` structs and direct collector. `main.go` does not call `providers.GetProvider`. The AWS provider has its own collection code and is not interchangeable evidence for CLI behavior. Azure and GCP provide static region lists and return errors from `GetResources`; no CLI cloud-provider selection is implemented.

Tests operate on in-memory fixtures and generated text. Live AWS collection, configuration precedence, snapshot comparisons, SARIF consumer validation, and Graphviz/browser rendering are not covered by the existing suite.
