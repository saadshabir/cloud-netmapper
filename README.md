# Cloud NetMapper

[![Go Version](https://img.shields.io/badge/Go-1.25.1+-00ADD8?style=flat&logo=go)](https://go.dev/)
[![AWS](https://img.shields.io/badge/AWS-Cloud-FF9900?style=flat&logo=amazon-aws)](https://aws.amazon.com/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg?style=flat)](LICENSE)
[![Graphviz](https://img.shields.io/badge/Graphviz-Required-5E5086?style=flat)](https://graphviz.org/)

> Visualize your AWS network topology and detect security risks automatically

Discover VPCs, subnets, instances, security groups, load balancers, and more - then generate visual network diagrams with built-in security analysis.

---

## Features

**Auto-Discovery** - Map your entire AWS network across multiple regions  
**Visual Diagrams** - Generate PNG, SVG, or interactive HTML topology maps  
**Security Scanning** - Detect open ports, misconfigurations, and compliance issues  
**Multi-Format Export** - JSON, CSV, Markdown, SARIF for CI/CD integration  
**Change Detection** - Compare scans to detect infrastructure drift  
**Configuration File** - YAML-based persistent settings  
**Multi-Cloud Ready** - Extensible architecture for AWS, Azure, and GCP

---

## Supported Resources

| Category       | Resources                                                                  |
| -------------- | -------------------------------------------------------------------------- |
| Networking     | VPCs, Subnets, Route Tables, Internet Gateways, NAT Gateways, VPC Peerings |
| Compute        | EC2 Instances, Lambda Functions                                            |
| Containers     | EKS Clusters, ECS Clusters                                                 |
| Databases      | RDS Instances                                                              |
| Load Balancing | ALB, NLB, CLB                                                              |
| Security       | Security Groups (Ingress and Egress rules)                                 |

---

## Quick Start

### Prerequisites

```bash
# macOS
brew install go graphviz

# Ubuntu/Debian
sudo apt install golang graphviz
```

> **Requirements:** Go 1.25.1+, AWS CLI configured, Graphviz (for PNG/SVG output)

### Install

```bash
git clone https://github.com/msaadshabir/cloud-netmapper.git
cd cloud-netmapper
go build
```

### Run

```bash
aws configure                    # Set up AWS credentials
./cloud-netmapper               # Scan us-east-1 region
./cloud-netmapper --all-regions # Scan all AWS regions
```

---

## Command Line Options

```
Usage: cloud-netmapper [options]

Options:
  --region string       AWS region to scan (comma-separated for multiple)
  --all-regions         Scan all available AWS regions
  --format string       Output format: png, svg, json, csv, markdown, html, sarif
  --output-dir string   Output directory for generated files (default: current dir)
  --verbosity string    Log verbosity: debug, info, warn, error (default: info)
  --config string       Path to configuration file
  --diff                Compare with previous scan and show changes
  --save-snapshot       Save snapshot for future diff comparisons (default: true)

Examples:
  # Scan specific regions with HTML output
  ./cloud-netmapper --region us-east-1,eu-west-1 --format html

  # Scan all regions and generate SARIF for CI/CD
  ./cloud-netmapper --all-regions --format sarif

  # Detect changes since last scan
  ./cloud-netmapper --diff --region us-east-1

  # Use configuration file
  ./cloud-netmapper --config .cloud-netmapper.yaml
```

---

## Configuration File

Create `.cloud-netmapper.yaml` in your project directory:

```yaml
# Regions to scan (overridden by --region or --all-regions)
regions:
  - us-east-1
  - us-west-2
  - eu-west-1

# Output settings
output:
  format: html # png, svg, json, csv, markdown, html, sarif
  directory: ./output # Output directory for generated files

# Logging
log_level: info # debug, info, warn, error

# Security rules (ports to flag as risky when open to 0.0.0.0/0)
security:
  risky_ports:
    - 22 # SSH
    - 3389 # RDP
    - 21 # FTP
    - 23 # Telnet
    - 3306 # MySQL
    - 5432 # PostgreSQL
    - 27017 # MongoDB
    - 6379 # Redis
```

---

## Output Formats

| Format     | File                    | Description                            |
| ---------- | ----------------------- | -------------------------------------- |
| `png`      | `network_map.png`       | Static network topology diagram        |
| `svg`      | `network_map.svg`       | Scalable vector diagram                |
| `html`     | `network_map.html`      | Interactive Cytoscape.js visualization |
| `json`     | `aws_resources.json`    | Complete resource inventory            |
| `csv`      | `resources.csv`         | Spreadsheet-compatible export          |
| `markdown` | `report.md`             | Human-readable report                  |
| `sarif`    | `security_report.sarif` | Security findings for CI/CD tools      |

---

## Security Checks

| Check                 | Severity | Description                           |
| --------------------- | -------- | ------------------------------------- |
| Open SSH/RDP/FTP      | High     | Dangerous ports open to 0.0.0.0/0     |
| Default VPC           | Medium   | Using default VPC (not recommended)   |
| Missing Flow Logs     | Medium   | VPC flow logs not enabled             |
| IMDSv2 Not Enforced   | Medium   | Instance metadata v1 still allowed    |
| Unencrypted EBS       | Medium   | EBS volumes without encryption        |
| Public RDS            | Critical | Database publicly accessible          |
| Unencrypted RDS       | High     | Database without encryption           |
| Overly Permissive IAM | High     | Admin/FullAccess roles on instances   |
| Unrestricted Egress   | Low      | Security groups allowing all outbound |

---

## Change Detection (Diff Mode)

Track infrastructure changes between scans:

```bash
# First scan - saves snapshot
./cloud-netmapper --region us-east-1

# Later scan - shows changes
./cloud-netmapper --region us-east-1 --diff
```

Output:

```
=== Changes detected in us-east-1 ===
Added: 2, Removed: 1, Modified: 3
  [+] EC2 Instance: web-server-3 (i-0abc123) - New t3.medium instance
  [-] EC2 Instance: legacy-app (i-0def456) - t2.micro instance terminated
  [~] Security Group: app-sg (sg-789) - Rules changed: ingress 3->5
```

A detailed diff report is saved to `diff_report_{region}.md`.

---

## IAM Permissions

Minimum required permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeInstances",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeRouteTables",
        "ec2:DescribeNatGateways",
        "ec2:DescribeInternetGateways",
        "ec2:DescribeVpcPeeringConnections",
        "ec2:DescribeFlowLogs",
        "ec2:DescribeVolumes",
        "ec2:DescribeRegions",
        "elasticloadbalancing:DescribeLoadBalancers",
        "rds:DescribeDBInstances",
        "lambda:ListFunctions",
        "eks:ListClusters",
        "eks:DescribeCluster",
        "ecs:ListClusters",
        "ecs:DescribeClusters"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Project Structure

```
cloud-netmapper/
  main.go               # Entry point and CLI handling
  aws_collector.go      # AWS resource collection
  security_checker.go   # Security risk analysis
  visualizer.go         # DOT file generation
  html_visualizer.go    # Interactive HTML output
  report_generator.go   # Markdown, CSV, SARIF exports
  diff_detector.go      # Change detection between scans
  config/
    config.go           # YAML configuration loading
  logger/
    logger.go           # Structured logging (slog)
  providers/
    provider.go         # CloudProvider interface
    aws_provider.go     # AWS implementation
    azure_provider.go   # Azure placeholder
    gcp_provider.go     # GCP placeholder
  types/
    types.go            # Shared type definitions
```

---

## Testing

```bash
go test -v ./...
```

---

## Troubleshooting

**AWS credentials not found**  
Run `aws configure` or set `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`

**Graphviz not installed**  
`brew install graphviz` (macOS) or `sudo apt install graphviz` (Linux)

**Module errors**  
Run `go mod tidy`

**Permission denied errors**  
Ensure your IAM user/role has the required permissions listed above

---

## Roadmap

- [x] Multi-region scanning
- [x] Interactive HTML visualization
- [x] SARIF output for CI/CD
- [x] Change detection
- [x] Configuration file support


---

## License

MIT
