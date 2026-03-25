---
name: cisco-ucce-cli
description: Use when monitoring or troubleshooting Cisco UCCE 12.6 — agent states, queue stats, PG diagnostics, CVP config, VVB calls, system health checks, and infrastructure inventory. Covers all cisco-ucce CLI operations across AW, Finesse, CVP, Diagnostic Portal, and VVB APIs.
license: MIT
metadata:
  author: sieteunoseis
  version: "1.0.0"
---

# Cisco UCCE CLI

Read-only CLI for monitoring and troubleshooting Cisco Unified Contact Center Enterprise 12.6. Covers 5 API services: AW Config/Status, Finesse, CVP, Diagnostic Portal, and VVB.

## Setup

```bash
# Install globally
npm install -g cisco-ucce
# Or run without installing
npx cisco-ucce --help
```

### Configuration

```bash
# Add a cluster
cisco-ucce config add prod

# Edit config with hosts and credentials
# Config location: ~/.cisco-ucce/config.json
cisco-ucce config show

# Set active cluster
cisco-ucce config use prod

# Test connectivity
cisco-ucce config test
```

Config supports `<ss:ID:field>` placeholders for ss-cli integration (optional).

## Global Flags

```
--format table|json|toon|csv   (default: table)
--host <host>                  (override target host)
--cluster <name>               (use specific cluster)
--insecure                     (skip TLS verification)
--no-audit                     (disable audit logging)
```

## Common Workflows

### System Health Check

```bash
# Quick health check across all services
cisco-ucce doctor

# AW system-wide status
cisco-ucce status

# Version check across all PGs/Routers
cisco-ucce diag version --host all
```

### Agent & Queue Monitoring

```bash
# All agent states from Finesse
cisco-ucce finesse users

# Queue stats by ID (IDs = skill group IDs from AW)
cisco-ucce finesse queue 205001

# List all teams
cisco-ucce finesse teams

# Not-ready reason codes
cisco-ucce finesse reason-codes --category NOT_READY

# Find agents by team (pipe JSON to jq)
cisco-ucce agent list --format json | jq '.[] | select(.agentTeamName == "IT.CALL")'

# Count agents per team
cisco-ucce agent list --format json | jq 'group_by(.agentTeamName) | .[] | {team: .[0].agentTeamName, count: length}'
```

### PG / Router Diagnostics

The Diagnostic Portal runs on PGs (port 7890), Routers (port 7890), AW (port 8443), CUIC (port 8443), and Finesse (port 8443). Use `--host` to target a specific server, or `--host all` for all.

```bash
# What's running on a PG?
cisco-ucce diag list-processes --host pg1b.example.com

# Service states
cisco-ucce diag list-services --host pg1b.example.com

# Active alarms
cisco-ucce diag alarms --host pg1b.example.com

# Performance counters
cisco-ucce diag perf --host pg1b.example.com

# Network connections
cisco-ucce diag netstat --host pg1b.example.com

# IP configuration
cisco-ucce diag ipconfig --host pg1b.example.com --args "/all"

# Available log files
cisco-ucce diag log-files --host pg1b.example.com

# Product version
cisco-ucce diag version --host pg1b.example.com

# Check ALL hosts at once
cisco-ucce diag alarms --host all
```

### CVP Investigation

```bash
# CVP version
cisco-ucce cvp version

# All CVP nodes and types (CALL, MEDIA, VXML)
cisco-ucce cvp servers

# SIP/IVR/ICM config on a call server
cisco-ucce cvp properties --host cvp1a.example.com
```

CVP OPS (`/cvp-config/`) uses a dedicated API user. CVP call servers (`/cvp-orm/rest/`) use AW credentials.

### VVB Active Calls

```bash
# Active calls on first VVB
cisco-ucce vvb active-calls

# Active calls across ALL VVBs
cisco-ucce vvb active-calls --host all
```

### Configuration Audit & Export

```bash
# Export resources to CSV for comparison/reporting
cisco-ucce agent list --format csv > agents.csv
cisco-ucce skillgroup list --format csv > skillgroups.csv

# Full machine inventory (all servers in solution)
cisco-ucce inventory list

# Specific machine details
cisco-ucce inventory get 5000

# Precision queue routing
cisco-ucce precisionqueue list

# PG configuration with CTI addresses
cisco-ucce peripheralgateway list

# Dialed numbers
cisco-ucce dialednumber list

# Call types
cisco-ucce calltype list
```

### Scripting & Monitoring

```bash
# Monitor queue depth every 30s
watch -n 30 'cisco-ucce finesse queue 205001 --format json | jq .callsInQueue'

# Export all config resources
for resource in agent skillgroup calltype precisionqueue attribute; do
  cisco-ucce $resource list --format csv > "${resource}.csv"
done

# JSON output for pipe-friendly scripting
cisco-ucce finesse system-info --format json
```

## AW Config Resources

All support `list` and `get <id>` (except `department` which is list-only):

| Command                | API Resource         | Notes                         |
| ---------------------- | -------------------- | ----------------------------- |
| `agent`                | agent                | 3000+ agents, paginated       |
| `skillgroup`           | skillgroup           | IDs map to Finesse queue IDs  |
| `calltype`             | calltype             |                               |
| `precisionqueue`       | precisionqueue       |                               |
| `attribute`            | attribute            |                               |
| `expandedcallvariable` | expandedcallvariable | NOT `callvariable`            |
| `mediaroutingdomain`   | mediaroutingdomain   |                               |
| `bucketinterval`       | bucketinterval       |                               |
| `dialednumber`         | dialednumber         |                               |
| `peripheralgateway`    | peripheralgateway    | Includes CTI addresses        |
| `department`           | department           | List only, no get             |
| `inventory`            | machineinventory     | Full solution inventory       |
| `agentteam`            | agentteam            | May return 403 for some users |

Pagination: `--page N --page-size N`. Without these flags, auto-paginates all records.

## Diagnostic Portal Endpoints

All support `--host <hostname>` and `--host all`:

| Command                  | Endpoint                    | Notes                               |
| ------------------------ | --------------------------- | ----------------------------------- |
| `diag list-processes`    | ListProcesses               |                                     |
| `diag list-services`     | ListServices                |                                     |
| `diag version`           | GetProductVersion           |                                     |
| `diag license`           | GetProductLicense           |                                     |
| `diag netstat`           | GetNetStat                  |                                     |
| `diag ipconfig`          | GetIPConfig                 | `--args "/all"`                     |
| `diag perf`              | GetPerformanceInformation   | `--component <path>`                |
| `diag perf-counter`      | GetPerfCounterValue         |                                     |
| `diag trace-level`       | GetTraceLevel               |                                     |
| `diag alarms`            | GetAlarms                   |                                     |
| `diag trace-components`  | ListTraceComponents         |                                     |
| `diag trace-files`       | ListTraceFiles              | `<component>` required, `--hours N` |
| `diag log-components`    | ListLogComponents           |                                     |
| `diag log-files`         | ListLogFiles                | `<component>` required, `--hours N` |
| `diag app-servers`       | ListAppServers              |                                     |
| `diag config-categories` | ListConfigurationCategories |                                     |
| `diag config-category`   | GetConfigurationCategory    | `<category>` required               |
| `diag traceroute`        | GetTraceRoute               |                                     |
| `diag ping`              | GetPing                     |                                     |
| `diag download-trace`    | DownloadTraceFile           | `<component> <file>`, `-o path`     |
| `diag download-log`      | DownloadLogFile             | `<component> <file>`, `-o path`     |

### Trace & Log File Collection

```bash
# 1. Discover available trace components
cisco-ucce diag trace-components --host pg1a.example.com

# 2. List trace files for a component (last 12 hours default)
cisco-ucce diag trace-files "Peripheral Gateway 1A/opc" --host pg1a.example.com
cisco-ucce diag trace-files "CTI Server 1A/ctisvr" --hours 24

# 3. Download a trace file
cisco-ucce diag download-trace "Peripheral Gateway 1A/opc" "PG1A_opc_trace.zip" -o ./traces/

# 1. Discover available log components
cisco-ucce diag log-components --host pg1a.example.com

# 2. List log files (EventLog, Tomcat, ICMDBA, etc.)
cisco-ucce diag log-files "EventLog" --host pg1a.example.com
cisco-ucce diag log-files "Tomcat" --hours 24

# 3. Download a log file
cisco-ucce diag download-log "EventLog" "System.evtx.xml" -o ./logs/system-events.xml
```

## Server-Side Log Locations

When RDP/SSH access is needed, these are the common file paths:

### PG / Router (Windows)

| Path                                            | Contents                                    |
| ----------------------------------------------- | ------------------------------------------- |
| `C:\icm\<instance>\logfiles\`                   | ICM binary trace logs (opc, pim, jgw, etc.) |
| `C:\icm\<instance>\logfiles\dumplog\`           | Text-converted trace logs                   |
| `C:\icm\serviceability\diagnostics\logs\`       | Diagnostic Framework service logs           |
| `C:\icm\serviceability\wsccli\`                 | Unified System CLI tool and output          |
| `C:\icm\serviceability\wsccli\conf\devices.csv` | Multi-server collection config              |

Use `dumplog <process> /bt HH:MM /et HH:MM /ms /o` on the server to convert binary logs to text.

### AW / HDS (Windows)

| Path                                      | Contents                  |
| ----------------------------------------- | ------------------------- |
| `C:\icm\<instance>\logfiles\`             | AW/HDS service logs       |
| `C:\icm\<instance>\logfiles\dbworker\`    | Database replication logs |
| `C:\icm\serviceability\diagnostics\logs\` | Diagnostic Framework logs |

### CVP (Windows)

| Path                    | Contents                |
| ----------------------- | ----------------------- |
| `%CVP_HOME%\logs\`      | CVP application logs    |
| `%CVP_HOME%\logs\OAMP\` | OPS Console logs        |
| `%CVP_HOME%\conf\`      | CVP configuration files |

### Finesse / CUIC / VVB (VOS Linux)

| Path                                        | Contents                          |
| ------------------------------------------- | --------------------------------- |
| `/var/log/active/tomcat/logs/`              | Finesse/CUIC web application logs |
| `/var/log/active/platform/log/`             | Platform service logs             |
| Use `cisco-dime` CLI for VOS log collection | DIME-based log download           |

## Cluster Config Structure

```json
{
  "activeCluster": "prod",
  "clusters": {
    "prod": {
      "aw": {
        "hosts": ["aw-a.example.com"],
        "username": "<your-aw-user>",
        "password": "<ss:ID:password>"
      },
      "finesse": {
        "hosts": ["fin-a.example.com"],
        "username": "<your-finesse-user>",
        "password": "..."
      },
      "cvpOps": {
        "host": "cvpops.example.com",
        "port": 8111,
        "username": "<your-cvp-api-user>",
        "password": "..."
      },
      "cvpCallServers": { "hosts": ["cvp1a.example.com"], "port": 8111 },
      "diagnosticPortal": {
        "hosts": [
          { "host": "pg1a.example.com", "port": 7890 },
          { "host": "aw-a.example.com", "port": 8443 }
        ],
        "username": "<domain\\your-admin-user>",
        "password": "..."
      },
      "vvb": {
        "hosts": ["vvb1a.example.com"],
        "username": "<your-vvb-user>",
        "password": "..."
      },
      "insecure": true
    }
  }
}
```

Auth precedence: CLI flags > env vars (`CISCO_UCCE_USERNAME`/`CISCO_UCCE_PASSWORD`) > config file.
