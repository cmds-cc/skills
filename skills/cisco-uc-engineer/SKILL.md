---
name: cisco-uc-engineer
description: Cisco Unified Communications engineer skill. Orchestrates cisco-axl, cisco-dime, cisco-perfmon, cisco-risport, cisco-support, cisco-ucce, cisco-yang, audiocodes-cli, genesys-cli CLIs, and the cisco-cdr and cucm-avail MCP servers for troubleshooting, provisioning, monitoring, CDR analysis, and lifecycle management of UC infrastructure including AudioCodes SBCs and Genesys Cloud CX. Use when working across multiple Cisco UC tools or diagnosing complex issues.
license: MIT
metadata:
  author: sieteunoseis
  version: "1.0.0"
---

# Cisco UC Engineer

Orchestration skill for Unified Communications. Connects cisco-axl, cisco-dime, cisco-perfmon, cisco-risport, cisco-support, audiocodes-cli, genesys-cli, and the cisco-cdr and cucm-avail MCP servers to solve cross-domain UC problems across Cisco CUCM, AudioCodes SBCs, and Genesys Cloud CX.

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
genesys-cli --version 2>/dev/null && echo "genesys-cli: available" || echo "genesys-cli: not installed"

# Check cisco-cdr MCP server (requires MCP configuration, not a CLI tool)
# Verify by calling: cdr_health

# Check cucm-avail MCP server (requires MCP configuration, not a CLI tool)
# Verify by calling: cucm_servers
```

Report what's available and what's missing. If a workflow requires a missing tool, tell the user:

> "This workflow needs cisco-dime which isn't installed. Install it with: `npm install -g cisco-dime`"

Adapt workflows to use only the tools that are installed. A partial toolkit is still useful — just skip steps that require missing tools and note what was skipped.

## Available Tools

| Tool              | Purpose                                                                            | npm Package           | Skill               |
| ----------------- | ---------------------------------------------------------------------------------- | --------------------- | ------------------- |
| `cisco-axl`       | CUCM configuration — phones, lines, route patterns, CSS, device pools, SIP trunks  | `cisco-axl`           | `cisco-axl-cli`     |
| `cisco-dime`      | CUCM log collection — SIP traces, SDL, audit logs, service logs                    | `cisco-dime`          | `cisco-dime-cli`    |
| `cisco-perfmon`   | Real-time performance counters — CPU, memory, call stats                           | `cisco-perfmon`       | `cisco-perfmon-cli` |
| `cisco-risport`   | Device registration status — phone reg, CTI, trunk status                          | `cisco-risport`       | `cisco-risport-cli` |
| `cisco-support`   | Cisco Support APIs — bugs, EoX, PSIRT, software, serial coverage                   | `cisco-support`       | `cisco-support-cli` |
| `audiocodes-cli`  | AudioCodes Mediant VE SBC — call stats, alarms, KPIs, health checks, config backup | `audiocodes-cli`      | `audiocodes-cli`    |
| `genesys-cli`     | Genesys Cloud CX — conversations, BYOC trunks, queues, agents, edges, audit        | `genesys-cli`         | `genesys-cli`       |
| `cisco-cdr` (MCP) | CDR/CMR analysis — call search, call trace, quality analysis, statistics, health   | `cisco-cdr-processor` | `cisco-cdr-mcp`     |
| `cucm-avail` (MCP) | Outage/upgrade planning — failover simulation, CMG/AG safety, upgrade order, CER 911 subnet coverage | `CUCM-AVAIL` (Docker) | `cucm-avail-mcp` |

Each tool has its own skill with detailed command reference. Use those skills for tool-specific questions. This skill is for workflows that span multiple tools.

> Note: cisco-support uses separate Cisco API credentials (not CUCM credentials). It has its own config at `~/.cisco-support/config.json`.
> Note: audiocodes-cli uses separate device credentials (not CUCM credentials). It has its own config at `~/.audiocodes-cli/config.json`.
> Note: genesys-cli uses Genesys Cloud OAuth client credentials (not CUCM credentials). It has its own config at `~/.genesys-cli/config.json`.
> Note: cisco-cdr is an MCP server (not a CLI tool). It requires the cisco-cdr-processor service running with PostgreSQL. Configure the MCP endpoint in your Claude Code MCP settings.
> Note: cucm-avail is an MCP server (not a CLI tool) built into the CUCM-AVAIL app. Read tools are always on; sync/import tools need the server's `MCP_API_KEY`. Its data is only as fresh as the app's last AXL sync / RISPort poll.

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
  cisco-dime config add staging --host 10.0.0.1 --username admin --password "$CUCM_PASSWORD" --insecure
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

### CDR-Based Call Investigation

**Tools needed:** cisco-cdr (MCP), cisco-dime, cisco-axl

```
# 1. Search CDR for the reported call
cdr_search calling="5551234" called="5559999" last="2h"

# 2. Trace the specific call for all legs and CMR quality data
cdr_trace call_id="<globalcallid_callid from search>"

# 3. Run the returned cisco-dime SDL trace command for SIP analysis
# (cdr_trace returns a ready-to-run cisco-dime command)

# 4. Cross-reference device config if needed
cisco-axl get Phone --name <device_from_cdr> --format json
```

**What to look for:**

- Cause code — 16 is normal, anything else indicates an issue (see cause description in results)
- Multiple CDR legs — indicates call was transferred, forwarded, or hit a CTI route
- CMR quality data — MOS, jitter, latency, packet loss for audio quality issues
- Enrichment data — device descriptions, users, pools, locations help identify who/where

### CDR Quality Analysis

**Tools needed:** cisco-cdr (MCP), cisco-perfmon

```
# 1. Find poor-quality calls
cdr_quality mos_below=3.0 last="1d"

# 2. Check call volume and failure patterns
cdr_stats type="by_cause" last="1d"
cdr_stats type="by_device" last="1d"

# 3. Correlate with system health at the time
cisco-perfmon collect "Cisco CallManager" --counter "CallsActive,RegisteredHardwarePhones"
```

**What to look for:**

- Clusters of low-MOS calls from the same device or location — indicates a network issue in that segment
- High jitter/latency — WAN or QoS problem
- Packet loss — network congestion or misconfigured QoS
- Specific devices with high failure rates — configuration or hardware issue

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

**Tools needed:** cisco-support, cisco-axl (cucm-avail MCP for the outage plan, if available)

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

If the cucm-avail MCP server is available, plan the reboot order and confirm no phone loses service:

```
cucm_upgrade_plan                                # recommended node order with phone impact per step
cucm_simulate_outage servers=["<node>"]          # unregistered must be 0 for a clean step
cucm_registration_summary                        # phones already off their primary before you start
```

### Planned Outage / Maintenance Window

**Tools needed:** cucm-avail (MCP), cisco-risport

```
# 1. What breaks if these nodes go down?
cucm_simulate_outage servers=["dcrsub05","dcrsub06"]
# 2. Are side A/B CM groups independent? (no shared servers)
cucm_cm_groups
```

```bash
# 3. During the window, confirm nothing is still registered to the node you took down
cisco-risport query --node <down-node> --status Registered
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

### End-to-End Call Trace (CUCM → SBC → Genesys Cloud)

**Tools needed:** cisco-axl, cisco-dime, cisco-risport, audiocodes-cli, genesys-cli

```bash
# 1. Verify the CUCM line/device and call forward config
cisco-axl sql query "SELECT dnorpattern, description FROM numplan WHERE dnorpattern = '<extension>'"
cisco-axl sql query "SELECT cfadestination FROM callforwarddynamic cf JOIN numplan np ON cf.fknumplan = np.pkid WHERE np.dnorpattern = '<extension>'"

# 2. Check registration of the SIP trunk to the SBC
cisco-risport query --name "<trunk_name>"

# 3. Pull CUCM SDL traces and grep for the call
cisco-dime select "Cisco CallManager" --last 30m --all-nodes --include-active --download --decompress
# grep for the dialed number, Call-ID, and trunk name in SDL files

# 4. Check the AudioCodes SBC — did it receive and relay the call?
audiocodes-cli doctor --device <sbc>
audiocodes-cli calls list --device <sbc>
audiocodes-cli alarms list --device <sbc>

# 5. Check Genesys Cloud — did the call arrive?
genesys-cli doctor
genesys-cli conversations list --caller <calling_number> --last 1h
genesys-cli conversations list --callee <called_number> --last 1h

# 6. Check BYOC trunk health on Genesys side
genesys-cli trunks list
genesys-cli trunks metrics
```

**What to look for:**

- CUCM SDL: confirm INVITE sent to SBC IP, look for `call_active` state and `200 OK`
- AudioCodes: compare attempted vs established calls — large gap means rejection
- AudioCodes alarms: Proxy Set down alarms indicate Genesys connectivity issues
- Genesys conversations: matching caller/callee confirms end-to-end delivery
- Genesys trunks: `connected` + `inService=true` means BYOC trunk is healthy
- Blank `disconnectType` and `sipResponseCode` in Genesys = normal call handling

### Genesys Cloud Troubleshooting

**Tools needed:** genesys-cli

```bash
# 1. Check Genesys Cloud connectivity and org health
genesys-cli doctor

# 2. Check BYOC trunk status — are trunks connected?
genesys-cli trunks list
genesys-cli trunks metrics

# 3. Check Edge server status
genesys-cli edges list

# 4. Search conversation history for a specific call
genesys-cli conversations list --caller <number> --last 1h
genesys-cli conversations list --callee <number> --last 1h
genesys-cli conversations list --queue <queue_name> --last 1h

# 5. Get full detail for a specific conversation
genesys-cli conversations detail <conversationId>

# 6. Check agent availability
genesys-cli agents list

# 7. Check queue status
genesys-cli queues list

# 8. Review recent config changes
genesys-cli audit list
```

**What to look for:**

- BYOC trunks not `connected` or `inService=false` — SBC-to-Genesys SIP connectivity issue
- Edge servers OFFLINE — media processing issues
- Conversations with `disconnectType` values like `error` or `system` — abnormal terminations
- High `sipResponseCode` (4xx/5xx) — SIP-level failures
- Agent presence — are agents available to take calls?
- Queue member count — are agents assigned to the expected queues?

## Diagnostic Decision Tree

When the user reports a problem, follow this order:

1. **Is it registered to CUCM?** → `cisco-risport query --mac <mac>`
2. **Is the config correct?** → `cisco-axl get Phone --name <name>`
3. **What do the CDRs show?** → `cdr_search calling="<number>" last="2h"` (if cisco-cdr MCP is available)
4. **What do the logs show?** → `cisco-dime select sip-traces --last 30m`
5. **Is the system healthy?** → `cisco-perfmon collect "Cisco CallManager" --counter "CallsActive"`
6. **Was call quality poor?** → `cdr_quality mos_below=3.5 last="2h"` (if cisco-cdr MCP is available)

7. **Are there known bugs?** → `cisco-support bug search --keyword "<error>" --pid "Cisco Unified Communications Manager"`
8. **Is the hardware end-of-life?** → `cisco-support eox --pid <model>`
9. **Did the call reach Genesys?** → `genesys-cli conversations list --caller <number> --last 1h`
10. **Is the BYOC trunk healthy?** → `genesys-cli trunks list`
11. **Will a server outage take it down?** → `cucm_simulate_outage servers=["<node>"]` / `cucm_phone_status name="<device>"` (if cucm-avail MCP is available)

Start broad, narrow down. Don't pull traces until you've checked the basics. If the cisco-cdr MCP server is available, CDR search is a fast way to confirm whether a call happened and what the outcome was before diving into logs.

## Important Notes

- All tools support `--format json` for structured output — use this when you need to parse results programmatically or correlate data across tools.
- All tools support `--cluster <name>` to target a specific cluster — useful when the user has multiple CUCM environments.
- All CUCM tools (axl, dime, perfmon, risport) share the same CUCM host.
- When pulling logs with cisco-dime, note that timestamps from CUCM are in the server's timezone. Use `--timezone` to convert.
- Use `cisco-perfmon watch` for live monitoring with sparklines — great for observing trends during testing or incidents.
- Use `cisco-dime analyze sip-ladder` to render SIP call flow diagrams from SDL trace files.
- All tools have a `doctor` command for health checks — run `cisco-<tool> doctor` or `genesys-cli doctor` to verify connectivity.
- genesys-cli uses OAuth client credentials and connects to the Genesys Cloud Platform API. It supports `--org <name>` for multi-org environments and `--region <region>` to target different Genesys Cloud regions (e.g., usw2, use1, aps1).
- When tracing calls end-to-end across CUCM → SBC → Genesys, correlate by timestamp and caller/callee numbers across all three systems.
