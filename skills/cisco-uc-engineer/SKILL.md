---
name: cisco-uc-engineer
description: Cisco Unified Communications engineer skill. Orchestrates cisco-axl, cisco-dime, cisco-perfmon, cisco-risport, cisco-support, cisco-ucce, cisco-yang, and audiocodes-cli CLIs for troubleshooting, provisioning, monitoring, and lifecycle management of UC infrastructure including AudioCodes SBCs. Use when working across multiple Cisco UC tools or diagnosing complex issues.
license: MIT
metadata:
  author: sieteunoseis
  version: "1.0.0"
---

# Cisco UC Engineer

Orchestration skill for Unified Communications. Connects cisco-axl, cisco-dime, cisco-perfmon, cisco-risport, cisco-support, and audiocodes-cli to solve cross-domain UC problems across Cisco CUCM and AudioCodes SBCs.

## Tool Detection

Before starting any workflow, check which tools are available:

```bash
# Check each tool — run all five, note which succeed
cisco-axl --version 2>/dev/null && echo "cisco-axl: available" || echo "cisco-axl: not installed"
cisco-dime --version 2>/dev/null && echo "cisco-dime: available" || echo "cisco-dime: not installed"
cisco-perfmon --version 2>/dev/null && echo "cisco-perfmon: available" || echo "cisco-perfmon: not installed"
cisco-risport --version 2>/dev/null && echo "cisco-risport: available" || echo "cisco-risport: not installed"
cisco-support --version 2>/dev/null && echo "cisco-support: available" || echo "cisco-support: not installed"
audiocodes-cli --version 2>/dev/null && echo "audiocodes-cli: available" || echo "audiocodes-cli: not installed"
```

Report what's available and what's missing. If a workflow requires a missing tool, tell the user:

> "This workflow needs cisco-dime which isn't installed. Install it with: `npm install -g cisco-dime`"

Adapt workflows to use only the tools that are installed. A partial toolkit is still useful — just skip steps that require missing tools and note what was skipped.

## Available Tools

| Tool             | Purpose                                                                            | npm Package      | Skill               |
| ---------------- | ---------------------------------------------------------------------------------- | ---------------- | ------------------- |
| `cisco-axl`      | CUCM configuration — phones, lines, route patterns, CSS, device pools, SIP trunks  | `cisco-axl`      | `cisco-axl-cli`     |
| `cisco-dime`     | CUCM log collection — SIP traces, SDL, audit logs, service logs                    | `cisco-dime`     | `cisco-dime-cli`    |
| `cisco-perfmon`  | Real-time performance counters — CPU, memory, call stats                           | `cisco-perfmon`  | `cisco-perfmon-cli` |
| `cisco-risport`  | Device registration status — phone reg, CTI, trunk status                          | `cisco-risport`  | `cisco-risport-cli` |
| `cisco-support`  | Cisco Support APIs — bugs, EoX, PSIRT, software, serial coverage                   | `cisco-support`  | `cisco-support-cli` |
| `audiocodes-cli` | AudioCodes Mediant VE SBC — call stats, alarms, KPIs, health checks, config backup | `audiocodes-cli` | `audiocodes-cli`    |

Each tool has its own skill with detailed command reference. Use those skills for tool-specific questions. This skill is for workflows that span multiple tools.

> Note: cisco-support uses separate Cisco API credentials (not CUCM credentials). It has its own config at `~/.cisco-support/config.json`.
> Note: audiocodes-cli uses separate device credentials (not CUCM credentials). It has its own config at `~/.audiocodes-cli/config.json`.

## Cluster Alignment

Each tool has its own config (`~/.cisco-axl/config.json`, `~/.cisco-dime/config.json`, etc.) with its own `activeCluster`. **Before any multi-tool workflow, verify all tools are pointing at the same CUCM host.**

```bash
# Check active cluster for each installed tool
cisco-axl config show 2>/dev/null
cisco-dime config show 2>/dev/null
cisco-perfmon config show 2>/dev/null
cisco-risport config show 2>/dev/null
```

**What to check:**

- Do all tools have the same host configured?
- Are the active clusters aligned (e.g., all pointing at "staging", not one at "staging" and another at "prod")?

**If misaligned:**

- Use `--cluster <name>` on each command to explicitly target the same cluster
- Or switch the active cluster: `cisco-dime config use staging`
- If a tool is missing the cluster entirely, add it:
  ```bash
  cisco-dime config add staging --host 10.0.0.1 --username admin --password secret --insecure
  ```

**Best practice:** When the user says "check phone X", always confirm which cluster/environment before running commands across multiple tools. Use the explicit `--cluster` flag in multi-tool workflows rather than relying on each tool's active cluster.

> Note: cisco-axl requires `--cucm-version` when adding a cluster. The other tools do not.

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
# 1. Quick snapshot of key counters
cisco-perfmon collect "Cisco CallManager" --counter "RegisteredHardwarePhones,CallsActive,CallsAttempted,CallsCompleted"
cisco-perfmon collect "Processor" --counter "% CPU Time"
cisco-perfmon collect "Memory" --counter "% Memory Used"

# 2. Live monitoring with sparklines (watch for trends)
cisco-perfmon watch "Cisco CallManager" --counter "CallsActive,RegisteredHardwarePhones" --interval 10

# 3. Cluster registration summary
cisco-risport summary

# 4. Check for errors in logs
cisco-dime select syslog --last 1h
```

**What to look for:**

- CPU consistently above 80% — possible resource issue
- RegisteredHardwarePhones dropping — phones losing registration
- CallsActive spiking — possible call loop or toll fraud
- Sparklines show trends — is the value stable, climbing, or oscillating?

### Call Load Testing / Monitoring

**Tools needed:** cisco-perfmon

```bash
# Monitor during a test window with sparklines
cisco-perfmon watch "Cisco CallManager" --counter "CallsActive,CallsAttempted,CallsCompleted,CallsInProgress" --interval 5 --duration 300

# Monitor trunk utilization
cisco-perfmon watch "Cisco SIP" --counter "CallsActive,CallsAttempted,CallsCompleted" --interval 10
```

### Syslog Error → Bug Search

**Tools needed:** cisco-dime, cisco-support

```bash
# 1. Pull syslogs to find errors
cisco-dime select syslog --last 1h --download --decompress

# 2. Look for error patterns — grep for alarm names, error codes, CSC bug IDs
# Common patterns: %CCM_CALLMANAGER-CALLMANAGER-3-DeviceTransientConnection
# If you find a CSCxx bug ID in the logs:
cisco-support bug get CSCvx12345

# 3. Search for known bugs matching the error
cisco-support bug search --keyword "DeviceTransientConnection" --pid "Cisco Unified Communications Manager"
cisco-support bug search --keyword "TransientConnection" --severity 1,2,3 --status open

# 4. Check if there's a PSIRT advisory related to the issue
cisco-support psirt --severity critical
```

**What to look for:**

- Error messages in syslog often contain Cisco bug IDs (CSCxx format)
- Alarm names map to known bugs — search for the alarm name as a keyword
- If a bug is found, check if it's fixed in a newer release

### Pre-Upgrade Research

**Tools needed:** cisco-support, cisco-axl

```bash
# 1. Check current CUCM version
cisco-axl sql query "SELECT version FROM componentversion WHERE softwarecomponent='callmanager'"

# 2. Get software suggestions for your platform
cisco-support software suggest --pid CUCM-LIC-15

# 3. Compare bugs between current and target version
cisco-support software compare --pid CUCM-LIC-15 --from 14.0 --to 15.0

# 4. Check for security advisories affecting target version
cisco-support psirt --severity critical --year 2026

# 5. Check if any hardware is end-of-life
cisco-support eox --pid CP-8841-K9
cisco-support serial warranty FJC12345678
```

### Hardware Lifecycle Check

**Tools needed:** cisco-support, cisco-axl

```bash
# 1. Get all phone models in the cluster
cisco-axl sql query "SELECT DISTINCT tm.name AS model, COUNT(*) AS cnt FROM device d JOIN typemodel tm ON d.tkmodel=tm.enum WHERE d.tkclass=1 GROUP BY tm.name ORDER BY cnt DESC"

# 2. Check EoX status for each model
cisco-support eox --pid CP-8841-K9
cisco-support eox --pid CP-7841-K9
cisco-support eox --pid CP-8865-K9

# 3. Check warranty for specific serial numbers
cisco-support serial coverage FJC12345678
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

### SBC Troubleshooting (AudioCodes)

**Tools needed:** audiocodes-cli, cisco-dime

```bash
# 1. Check SBC health
audiocodes-cli doctor --device <sbc>

# 2. Check active alarms (proxy sets down, IP groups blocked, cert expiry)
audiocodes-cli alarms list --device <sbc>

# 3. Check call statistics — are calls getting through?
audiocodes-cli calls list --device <sbc>

# 4. Check KPIs — license usage, DDoS stats, media quality
audiocodes-cli kpi show system/licenseStats/global --device <sbc>
audiocodes-cli kpi show network/ddosStats/global --device <sbc>

# 5. Backup SBC config for review
audiocodes-cli device backup --device <sbc>

# 6. Cross-reference with CUCM SDL logs to trace calls end-to-end
cisco-dime select "Cisco CallManager" --last 30m --all-nodes --include-active --download --decompress
```

**What to look for:**

- IP Group blocked alarms — proxy set connectivity issues between SBC and CUCM/carrier
- Call stats: attempted vs established — large gap means routing or far-end rejection
- "No Answer" vs "Other Reason" — no answer is far-end, other reason is often 4xx/5xx from carrier
- CUCM SDL logs show the SIP trunk name and response codes for calls leaving CUCM to the SBC

### CUCM-to-SBC Call Flow Investigation

**Tools needed:** audiocodes-cli, cisco-dime, cisco-axl

```bash
# 1. Check which SIP trunk CUCM uses for the route pattern
cisco-axl sql query "SELECT dnorpattern, routePartitionName FROM numplan WHERE dnorpattern LIKE '%<pattern>%'"

# 2. Pull CUCM call logs to see the INVITE flow
cisco-dime select "Cisco CallManager" --last 30m --download --decompress
# grep for the dialed number in calllogs

# 3. Check the SBC side — did it receive the call?
audiocodes-cli calls list --device <sbc>

# 4. Check SBC alarms — is the outbound IP group healthy?
audiocodes-cli alarms list --device <sbc>
```

## Diagnostic Decision Tree

When the user reports a problem, follow this order:

1. **Is it registered to CUCM?** → `cisco-risport query --mac <mac>`
2. **Is the config correct?** → `cisco-axl get Phone --name <name>`
3. **What do the logs show?** → `cisco-dime select sip-traces --last 30m`
4. **Is the system healthy?** → `cisco-perfmon collect "Cisco CallManager" --counter "CallsActive"`

5. **Are there known bugs?** → `cisco-support bug search --keyword "<error>" --pid "Cisco Unified Communications Manager"`
6. **Is the hardware end-of-life?** → `cisco-support eox --pid <model>`

Start broad, narrow down. Don't pull traces until you've checked the basics.

## Important Notes

- All tools support `--format json` for structured output — use this when you need to parse results programmatically or correlate data across tools.
- All tools support `--cluster <name>` to target a specific cluster — useful when the user has multiple CUCM environments.
- All CUCM tools (axl, dime, perfmon, risport) share the same CUCM host.
- When pulling logs with cisco-dime, note that timestamps from CUCM are in the server's timezone. Use `--timezone` to convert.
- Use `cisco-perfmon watch` for live monitoring with sparklines — great for observing trends during testing or incidents.
- Use `cisco-dime analyze sip-ladder` to render SIP call flow diagrams from SDL trace files.
- All tools have a `doctor` command for health checks — run `cisco-<tool> doctor` to verify connectivity.
