---
name: cisco-uc-engineer
description: Cisco Unified Communications engineer skill. Orchestrates cisco-axl, cisco-dime, cisco-perfmon, and cisco-risport CLIs for troubleshooting, provisioning, and monitoring UC infrastructure. Use when working across multiple Cisco UC tools or diagnosing complex issues.
license: MIT
metadata:
  author: sieteunoseis
  version: "1.0.0"
---

# Cisco UC Engineer

Orchestration skill for Cisco Unified Communications. Connects cisco-axl, cisco-dime, cisco-perfmon, and cisco-risport to solve cross-domain UC problems.

## Tool Detection

Before starting any workflow, check which tools are available:

```bash
# Check each tool — run all four, note which succeed
cisco-axl --version 2>/dev/null && echo "cisco-axl: available" || echo "cisco-axl: not installed"
cisco-dime --version 2>/dev/null && echo "cisco-dime: available" || echo "cisco-dime: not installed"
cisco-perfmon --version 2>/dev/null && echo "cisco-perfmon: available" || echo "cisco-perfmon: not installed"
cisco-risport --version 2>/dev/null && echo "cisco-risport: available" || echo "cisco-risport: not installed"
```

Report what's available and what's missing. If a workflow requires a missing tool, tell the user:

> "This workflow needs cisco-dime which isn't installed. Install it with: `npm install -g cisco-dime`"

Adapt workflows to use only the tools that are installed. A partial toolkit is still useful — just skip steps that require missing tools and note what was skipped.

## Available Tools

| Tool | Purpose | npm Package | Skill |
|---|---|---|---|
| `cisco-axl` | CUCM configuration — phones, lines, route patterns, CSS, device pools, SIP trunks | `cisco-axl` | `cisco-axl-cli` |
| `cisco-dime` | CUCM log collection — SIP traces, SDL, audit logs, service logs | `cisco-dime` | `cisco-dime-cli` |
| `cisco-perfmon` | Real-time performance counters — CPU, memory, call stats | `cisco-perfmon` | `cisco-perfmon-cli` |
| `cisco-risport` | Device registration status — phone reg, CTI, trunk status | `cisco-risport` | `cisco-risport-cli` |

Each tool has its own skill with detailed command reference. Use those skills for tool-specific questions. This skill is for workflows that span multiple tools.

## Cluster Configuration

All tools share the same cluster pattern. If the user has configured one tool, the same CUCM host likely works for others:

```bash
# Check existing configs
cisco-axl config list
cisco-dime config list

# If one is configured but another isn't, suggest reusing the same credentials
# Note: cisco-axl requires --cucm-version, others don't
```

## Troubleshooting Workflows

### Phone Not Registering

**Tools needed:** cisco-axl, cisco-risport, cisco-dime

```bash
# 1. Check phone config in CUCM
cisco-axl get Phone --name SEP001122334455 --format json

# 2. Check current registration status
cisco-risport query --mac 001122334455

# 3. Pull recent SIP traces for this device
cisco-dime select sip-traces --last 30m --download
```

**What to look for:**
- Device pool and CUCM group — is the phone pointing at the right server?
- CSS — does the phone have a calling search space assigned?
- SIP traces — look for 401/403 (auth issues), 503 (server unavailable), or no response

### Call Routing Issue

**Tools needed:** cisco-axl, cisco-dime

```bash
# 1. Check the dialed pattern
cisco-axl sql query "SELECT dnorpattern, routePartitionName FROM numplan WHERE dnorpattern LIKE '%<pattern>%'"

# 2. Check the CSS chain
cisco-axl get CallingSearchSpace --name <css_name> --format json

# 3. Check route patterns and route lists
cisco-axl list RoutePattern --search "pattern=%<digits>%"

# 4. Pull SIP traces to see the actual call flow
cisco-dime select sip-traces --last 1h --download
```

**What to look for:**
- Is the pattern in a partition that's in the calling CSS?
- Route pattern → route list → route group → trunk — follow the chain
- SIP traces show the actual INVITE routing and any redirects/rejects

### Call Quality / One-Way Audio

**Tools needed:** cisco-perfmon, cisco-dime, cisco-risport

```bash
# 1. Check system health
cisco-perfmon collect "Cisco CallManager" --counter "CallsActive,RegisteredHardwarePhones"

# 2. Check device registration (partially registered = codec issues)
cisco-risport query --mac <mac>

# 3. Pull traces for the specific call
cisco-dime select sip-traces --last 30m --download

# 4. If packet captures are available
cisco-dime select "Packet Capture Logs" --last 1h
```

**What to look for:**
- SDP in SIP traces — are both sides agreeing on codec and media IP?
- One-way audio often means NAT/firewall issue — check the media IPs in SDP
- MTP/transcoder insertion — look for MTP resources in the call flow

### Performance Investigation

**Tools needed:** cisco-perfmon, cisco-risport, cisco-dime

```bash
# 1. Collect key performance counters
cisco-perfmon collect "Cisco CallManager" --counter "RegisteredHardwarePhones,CallsActive,CallsAttempted,CallsCompleted"
cisco-perfmon collect "Processor" --counter "% CPU Time"
cisco-perfmon collect "Memory" --counter "% Memory Used"

# 2. Get device summary
cisco-risport summary

# 3. Check for errors in logs
cisco-dime select "Cisco CallManager" --last 1h
cisco-dime select syslog --last 1h
```

### Audit / Change Investigation

**Tools needed:** cisco-dime, cisco-axl

```bash
# 1. Pull audit logs to see who changed what
cisco-dime select audit --last 1d --download

# 2. Look at a specific object's current config
cisco-axl get Phone --name <device> --format json

# 3. Compare with expected config
cisco-axl sql query "SELECT * FROM device WHERE name='<device>'"
```

## Provisioning Workflows

### New Phone Setup

**Tools needed:** cisco-axl

```bash
# 1. Add the line
cisco-axl add Line --data '{"pattern":"1001","routePartitionName":"PT_INTERNAL","description":"John Doe"}'

# 2. Add the phone
cisco-axl add Phone --data '{"name":"SEP001122334455","product":"Cisco 8841","class":"Phone","protocol":"SIP","devicePoolName":"DP_HQ","lines":{"line":{"index":"1","dirn":{"pattern":"1001","routePartitionName":"PT_INTERNAL"}}}}'

# 3. Verify registration
cisco-risport query --mac 001122334455
```

### Bulk Provisioning

**Tools needed:** cisco-axl

```bash
# From CSV
cisco-axl add Phone --csv phones.csv
cisco-axl add Line --csv lines.csv

# Verify all registered
cisco-risport summary --model "Cisco 8841"
```

## Diagnostic Decision Tree

When the user reports a problem, follow this order:

1. **Is it registered to CUCM?** → `cisco-risport query --mac <mac>`
2. **Is the config correct?** → `cisco-axl get Phone --name <name>`
3. **What do the logs show?** → `cisco-dime select sip-traces --last 30m`
4. **Is the system healthy?** → `cisco-perfmon collect "Cisco CallManager" --counter "CallsActive"`

Start broad, narrow down. Don't pull traces until you've checked the basics.

## Important Notes

- All tools support `--format json` for structured output — use this when you need to parse results programmatically or correlate data across tools.
- All tools support `--cluster <name>` to target a specific cluster — useful when the user has multiple CUCM environments.
- All CUCM tools (axl, dime, perfmon, risport) share the same CUCM host.
- When pulling logs with cisco-dime, note that timestamps from CUCM are in the server's timezone. Use `--timezone` to convert.
