# Cloud NetMapper

[![Go](https://img.shields.io/badge/Go-1.25.1%2B-00ADD8?logo=go)](https://go.dev/doc/install)
[![AWS](https://img.shields.io/badge/AWS-Cloud-FF9900?logo=amazon-aws)](https://aws.amazon.com/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Cloud NetMapper is a Go command-line tool for collecting AWS network resource metadata, visualizing resource relationships, and reporting common security configuration risks. It scans one account at a time using read-only AWS API calls and writes reports locally.

- Scan selected regions or discover the regions enabled for your account.
- Explore an interactive HTML map with search, resource filters, details, and PNG export.
- Export collected metadata as JSON, CSV, or Markdown, and security findings as SARIF.
- Save regional snapshots and compare selected resource changes between scans.

The CLI supports **AWS only**. Azure and GCP files are placeholders. Collection does not paginate API responses, and some API failures produce partial results. Treat reports as an aid to infrastructure review; see the [coverage and limitations reference](docs/reference.md) before interpreting an empty inventory or a clean scan.

## Documentation

- [Build and quick start](#quick-start)
- [AWS credentials and permissions](#aws-credentials-and-permissions)
- [CLI options](#command-line-options)
- [Configuration](#configuration)
- [Outputs](#output-files)
- [Snapshot comparisons](#compare-scans)
- [Troubleshooting](#troubleshooting)
- [Resource coverage, security checks, diff fields, and code structure](docs/reference.md)
- [Copyable IAM policy](docs/iam-policy.json)

## Quick start

You need [Go 1.25.1 or newer](https://go.dev/doc/install), AWS credentials, and the [read permissions](docs/iam-policy.json) used by the collector. Go downloads the dependencies pinned in `go.mod` and `go.sum` during the first build.

[Graphviz](https://graphviz.org/download/) is needed only for `--format png` or `--format svg`. HTML generation does not require Graphviz; viewing the HTML map requires a browser that can load Cytoscape.js from `cdnjs.cloudflare.com`.

```bash
git clone https://github.com/saadshabir/cloud-netmapper.git
cd cloud-netmapper
mkdir -p bin
go build -o ./bin/cloud-netmapper .

# With AWS credentials already configured:
./bin/cloud-netmapper --region us-east-1 --format html --output-dir ./output
```

Open `output/network_map.html` in your browser. The scan also writes `aws_resources.json`, `network_map.dot`, and regional snapshots. Review warnings in the terminal before relying on the report.

Build from source for your platform. The `bin` path keeps the build separate from the binary already tracked at the repository root.

## AWS credentials and permissions

The application uses the [AWS SDK for Go v2 default credential chain](https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/configure-gosdk.html), including shared profiles, environment credentials, and supported IAM role credentials. The AWS CLI is optional for the scanner itself; it is useful for configuring profiles and signing in with IAM Identity Center.

For an existing named profile:

```bash
export AWS_PROFILE=netmapper

# Optional: confirm which account and principal the CLI will use.
aws sts get-caller-identity

./bin/cloud-netmapper --region us-east-1 --format markdown --output-dir ./output/netmapper
```

Replace `netmapper` with your profile name. If using IAM Identity Center, configure and authenticate the profile first:

```bash
aws configure sso --profile netmapper
aws sso login --profile netmapper
```

Environment credentials take precedence over shared profile credentials. Temporary environment credentials also require `AWS_SESSION_TOKEN`. There is no `--profile` flag; select profiles with `AWS_PROFILE`.

The [IAM policy example](docs/iam-policy.json) lists the read/list actions called by the CLI collector, including `ec2:DescribeRegions` for `--all-regions`. It grants no resource changes. Apply it to the scanning identity through your normal IAM process; organization policies and permission boundaries may still restrict access. The collector does not read IAM policy documents.

For region discovery, set a starting region in your AWS profile or `AWS_REGION`:

```bash
AWS_REGION=us-east-1 ./bin/cloud-netmapper --all-regions --format json --output-dir ./output/netmapper
```

`--all-regions` uses [EC2 DescribeRegions](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeRegions.html) without `AllRegions=true`, so it discovers enabled regions. Its discovery request uses the SDK region, independently of `--region` and the application's YAML region list.

## Command-line options

Use `./bin/cloud-netmapper --help` for the flag parser's usage text. Both single and double hyphens are accepted. Defaults below describe application behavior when no configuration file is found.

| Option | Behavior / default |
| --- | --- |
| `--region us-east-1,eu-west-1` | Comma-separated scan regions; overrides YAML `regions`. Default: `us-east-1`. |
| `--all-regions` | Discover enabled regions; takes precedence over `--region` and YAML. Default: `false`. |
| `--format html` | One of `png`, `svg`, `html`, `json`, `csv`, `markdown`, `sarif`. Default: `png`. |
| `--output-dir ./output` | Directory for reports and snapshots; created if needed. Default: current directory. |
| `--verbosity debug` | `debug`, `info`, `warn`, or `error`. Default: `info`; unrecognized values also use `info`. |
| `--config ./scan.yaml` | Explicit YAML file; otherwise search the working directory, then home directory. |
| `--diff` | Compare each collected region with its latest snapshot in the output directory. Default: `false`. |
| `--save-snapshot=false` | Disable both timestamped and latest snapshot writes. Default: snapshots enabled. |

Boolean values use `=false` to disable a flag. A run selects one report format; separate invocations collect AWS data again. Unknown format strings produce a warning and leave the JSON/DOT files, without rendering an additional format.

```bash
# Selected regions, combined into one report
./bin/cloud-netmapper --region us-east-1,eu-west-1 --format html --output-dir ./output/netmapper

# Security findings for a downstream SARIF consumer
./bin/cloud-netmapper --region us-east-1 --format sarif --output-dir ./output/netmapper

# Diagnostic logging with no snapshot update
./bin/cloud-netmapper --region us-east-1 --format json --verbosity debug --save-snapshot=false
```

Security findings are printed regardless of the selected output format or log level. Log messages go to stderr; findings, diff summaries, and the completion message go to stdout. Findings do not cause a nonzero exit code. A successful exit can also include skipped regions or incomplete service data; CI consumers must inspect the logs and report contents. The completion message lists requested regions, even when some were skipped.

## Configuration

Copy the [example configuration](.cloud-netmapper.yaml.example) and adjust it:

```bash
cp .cloud-netmapper.yaml.example .cloud-netmapper.yaml
./bin/cloud-netmapper --config .cloud-netmapper.yaml
```

```yaml
regions:
  - us-east-1

output:
  format: html
  directory: ./output
```

Without `--config`, the first existing file in this order is loaded:

1. `.cloud-netmapper.yaml` in the current working directory
2. `.cloud-netmapper.yml` in the current working directory
3. `~/.cloud-netmapper.yaml`
4. `~/.cloud-netmapper.yml`

Omitted fields retain their built-in defaults. Nonempty CLI flags override YAML output settings; region precedence is `--all-regions`, then `--region`, then YAML. Relative paths are resolved from the process's working directory, including paths inside a configuration stored elsewhere.

Only `regions`, `output.format`, and `output.directory` affect the CLI through YAML. `security.severity_threshold` is parsed but does not filter findings. YAML `log_level` and `security.risky_ports` are ignored; use `--verbosity` for logging, and see the [fixed security checks](docs/reference.md#security-checks). Unknown YAML keys are silently ignored. A nonexistent explicit config path falls back to defaults; malformed YAML or other read errors stop the run.

## Output files

Every run that reaches report generation writes `aws_resources.json` and `network_map.dot`, then creates the selected report. Files with fixed names are overwritten on subsequent runs; files from previously selected formats are left in place and may be stale.

| Format | Selected file | Contents |
| --- | --- | --- |
| `html` | `network_map.html` | Interactive map of selected resource relationships; CDN dependency. |
| `json` | `aws_resources.json` | All fields in the collected resource model, without security findings. |
| `csv` | `resources.csv` | Summary rows for selected resource types, without security findings. |
| `markdown` | `report.md` | Resource counts, selected detail tables, and findings with remediation text. |
| `sarif` | `security_report.sarif` | Security findings exported with SARIF 2.1.0 fields; see [compatibility limits](docs/reference.md#sarif-export). |
| `png` | `network_map.png` | Graphviz rendering of the smaller DOT topology. |
| `svg` | `network_map.svg` | Graphviz rendering of the smaller DOT topology. |

Output coverage differs by format; the [coverage table](docs/reference.md#resource-and-output-coverage) shows what is included. DOT generation currently leaves hyphens in unquoted node identifiers, which can prevent PNG/SVG rendering for ordinary AWS IDs. Use HTML for visualization or JSON/Markdown for review; see [Graphviz troubleshooting](#troubleshooting).

The merged JSON has top-level resource arrays such as `VPCs`, `Subnets`, and `Instances`, using Go field names. Unpopulated slices may be `null`. Resource `Name` values receive a `[region]` prefix; there are no separate account, region, timestamp, or scan-completeness fields in this file. Per-region snapshots retain the original names and include region and timestamp metadata.

Reports contain infrastructure identifiers, IP addresses, names, and configuration details. Store them with appropriate access controls. Use a separate output directory for each account/profile so snapshot baselines do not mix accounts.

## Compare scans

```bash
# Establish a regional baseline.
./bin/cloud-netmapper --region us-east-1 --format json --output-dir ./output/netmapper

# Compare, then advance the baseline to this scan.
./bin/cloud-netmapper --region us-east-1 --format json --output-dir ./output/netmapper --diff

# Compare without advancing the baseline.
./bin/cloud-netmapper --region us-east-1 --format json --output-dir ./output/netmapper --diff --save-snapshot=false
```

Snapshots are written to:

```text
output/netmapper/snapshots/scan_us-east-1_YYYYMMDD_HHMMSS.json
output/netmapper/snapshots/scan_us-east-1_latest.json
```

`--diff` loads the regional `latest` file before saving the current scan. Missing baselines are reported and skipped; unreadable or invalid JSON baselines produce warnings. When changes are detected, the CLI prints them and writes `diff_report_us-east-1.md`. A run with no changes does not rewrite or remove an old diff report.

Comparisons cover selected fields, not every configuration change. Route tables and VPC peerings are excluded, and security groups are compared by rule counts only. See the [diff coverage table](docs/reference.md#diff-coverage). Permission failures, pagination gaps, and the running-only EC2 filter can appear as removals; a removal is an inventory difference, not proof that a resource was deleted.

Timestamped snapshots accumulate without automatic cleanup. Names have one-second precision and can overwrite if scans of the same region finish within the same second. Baselines are keyed by region and directory, not account identity.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| Credentials missing or expired | Check `AWS_PROFILE` and the SDK credential chain. Renew SSO sessions with `aws sso login --profile netmapper`; include a session token for temporary environment credentials. |
| `--all-regions` cannot resolve an endpoint | Set a region in the AWS profile or `AWS_REGION`; YAML/`--region` do not configure the discovery client. |
| Access denied or service data missing | Compare the failing action with the [policy](docs/iam-policy.json). Check regional service availability and organization restrictions. Review all warnings; some metadata lookups fail silently. |
| No output from a region | VPC, subnet, instance, or security group API errors skip that region. If every region fails, the CLI exits with an error. |
| `dot` executable missing | Install Graphviz and ensure `dot` is on `PATH`, or select `html`, `json`, `csv`, `markdown`, or `sarif`. |
| PNG/SVG fails despite Graphviz being installed | Inspect `network_map.dot`; unquoted identifiers such as `vpc_vpc-123` violate [DOT identifier syntax](https://graphviz.org/doc/info/lang.html). Labels containing quotes or backslashes can also break rendering. Use HTML or correct the DOT file before rendering it manually. |
| HTML map is blank | Check the browser console and CDN access. Incomplete collections can also create edges whose parent nodes are absent. |
| HTML nodes collide across regions | Lambda, EKS, and RDS node IDs depend on names/identifiers without regional scoping. Generate a separate HTML map per region. |
| Unexpected default settings | Verify the working directory, config search order, and file existence; unknown keys and missing config paths do not trigger validation errors. |
| Old output looks unchanged | Only the selected report is regenerated. Inspect the file timestamp and use consistent output directories for comparisons. |
| Build cannot fetch modules | Check network/proxy access and your Go version, then run `go mod download`. |

## Development

```bash
go test ./...
go vet ./...
go build -o ./bin/cloud-netmapper .
```

The existing tests cover selected security rules, DOT text generation, and label/ID helpers. They do not call AWS, validate Graphviz rendering, or exercise an end-to-end cloud scan. See the [code structure](docs/reference.md#code-structure) for the execution path and provider boundaries.

## License

[MIT](LICENSE).
