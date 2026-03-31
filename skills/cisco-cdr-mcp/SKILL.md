---
name: cisco-cdr-mcp
description: Use when querying Cisco CUCM call detail records via the cisco-cdr MCP server — searching CDRs, tracing calls, finding poor-quality calls, generating call statistics, and checking processor health. Covers all cisco-cdr MCP tools for CDR/CMR analysis with AXL enrichment.
license: MIT
metadata:
  author: sieteunoseis
  version: "1.0.0"
---

# Cisco CDR MCP Server

An MCP server for querying Cisco Unified Communications Manager (CUCM) Call Detail Records (CDR) and Call Monitoring Records (CMR). Part of the [cisco-cdr-processor](https://github.com/sieteunoseis/cisco-cdr-processor) project.

## Setup

The MCP server runs as part of the cisco-cdr-processor service. It must be running and accessible before using these tools.

### Adding the MCP Server

Add the cisco-cdr server to Claude Code:

```bash
claude mcp add -s user --transport http cisco-cdr https://cdr-backend.tuce.ohsu.edu/mcp
```

Or manually add to your MCP configuration:

```json
{
  "mcpServers": {
    "cisco-cdr": {
      "type": "url",
      "url": "https://cdr-backend.tuce.ohsu.edu/mcp"
    }
  }
}
```

### Verifying Connectivity

Use the health check tool to confirm the MCP server is reachable and the database is populated:

```
cdr_health
```

This returns database record counts, file processing activity, enrichment cache stats, and pending file count. If it fails, verify the cisco-cdr-processor service is running.

## Available MCP Tools

### cdr_search — Search CDR Records

Search by calling/called number, device name, cause code, or time range. Returns records with cause/codec descriptions and AXL enrichment (device descriptions, users, pools, locations, types, models).

**Parameters:**

| Parameter | Type   | Required | Description                                            |
| --------- | ------ | -------- | ------------------------------------------------------ |
| `calling` | string | no       | Calling party number (partial match supported)         |
| `called`  | string | no       | Final called party number (partial match supported)    |
| `device`  | string | no       | Originating or destination device name (partial match) |
| `cause`   | string | no       | Cause code value (e.g. "16" for normal clearance)      |
| `last`    | string | no       | Relative time range: 30m, 2h, 1d, 7d (default: 24h)    |
| `start`   | string | no       | Start timestamp (ISO 8601)                             |
| `end`     | string | no       | End timestamp (ISO 8601)                               |
| `limit`   | number | no       | Maximum records to return (default 100, max 1000)      |

**Examples:**

```
# Search by calling number
cdr_search calling="5551234"

# Search by called number in the last hour
cdr_search called="5559999" last="1h"

# Search by device name
cdr_search device="SEP001122334455" last="7d"

# Search for failed calls (cause code != 16)
cdr_search cause="47" last="24h"

# Search with absolute time range
cdr_search calling="5551234" start="2026-03-30T08:00:00Z" end="2026-03-30T17:00:00Z"
```

### cdr_trace — Trace a Call

Trace a specific call by `globalcallid_callid`. Returns all CDR legs, associated CMR quality records, and a ready-to-run `cisco-dime` SDL trace command with timestamps buffered 30 seconds before/after the call.

**Parameters:**

| Parameter        | Type   | Required | Description                                                  |
| ---------------- | ------ | -------- | ------------------------------------------------------------ |
| `call_id`        | string | yes      | The globalcallid_callid value to trace                       |
| `callmanager_id` | string | no       | globalcallid_callmanagerid to narrow to a specific CUCM node |

**Examples:**

```
# Trace a call by ID
cdr_trace call_id="12345678"

# Trace narrowed to a specific CUCM node
cdr_trace call_id="12345678" callmanager_id="2"
```

**Workflow tip:** First use `cdr_search` to find the call, then use the `globalcallid_callid` value from the results to trace it. The returned `sdl_trace_command` can be run directly with cisco-dime to download the SDL traces for deep SIP analysis.

### cdr_quality — Find Poor-Quality Calls

Join CDR and CMR data to find calls with poor quality metrics. Filter by MOS score, jitter, latency, or packet loss thresholds. Results are sorted by MOS score (lowest first).

**Parameters:**

| Parameter       | Type   | Required | Description                                         |
| --------------- | ------ | -------- | --------------------------------------------------- |
| `mos_below`     | number | no       | MOS LQK below this value (default: 3.5)             |
| `jitter_above`  | number | no       | Jitter (ms) above this value                        |
| `latency_above` | number | no       | Latency (ms) above this value                       |
| `loss_above`    | number | no       | Packet loss count above this value                  |
| `last`          | string | no       | Relative time range: 30m, 2h, 1d, 7d (default: 24h) |
| `start`         | string | no       | Start timestamp (ISO 8601)                          |
| `end`           | string | no       | End timestamp (ISO 8601)                            |
| `limit`         | number | no       | Maximum records to return (default 100, max 1000)   |

**Examples:**

```
# Find calls with MOS below 3.0 in the last 24 hours
cdr_quality mos_below=3.0

# Find calls with high jitter in the last week
cdr_quality jitter_above=50 last="7d"

# Find calls with packet loss in a specific time window
cdr_quality loss_above=10 start="2026-03-30T08:00:00Z" end="2026-03-30T17:00:00Z"

# Combine thresholds
cdr_quality mos_below=3.5 jitter_above=30 latency_above=200 last="1d"
```

### cdr_stats — Aggregate Statistics

Generate aggregated CDR statistics for call volume analysis, top caller/callee identification, failure analysis, and device/location breakdowns.

**Parameters:**

| Parameter  | Type   | Required | Description                                                                      |
| ---------- | ------ | -------- | -------------------------------------------------------------------------------- |
| `type`     | enum   | yes      | `volume`, `top_callers`, `top_called`, `by_cause`, `by_device`, `by_location`    |
| `last`     | string | no       | Relative time range: 30m, 2h, 1d, 7d (default: 24h)                              |
| `start`    | string | no       | Start timestamp (ISO 8601)                                                       |
| `end`      | string | no       | End timestamp (ISO 8601)                                                         |
| `interval` | enum   | no       | Time bucket for volume: `minute`, `hour`, `day`, `week`, `month` (default: hour) |
| `limit`    | number | no       | Maximum rows to return (default 20, max 500)                                     |

**Stat types:**

- **volume** — Call volume over time with call counts, average duration, normal/failed breakdown
- **top_callers** — Calling numbers ranked by call count with total/average duration
- **top_called** — Called numbers ranked by call count with total/average duration
- **by_cause** — Failure reasons with call counts per cause code (with descriptions)
- **by_device** — Originating devices with call counts, average duration, and failure rates
- **by_location** — Locations/pools with call counts, average duration, failure counts, and failure rate percentage

**Examples:**

```
# Call volume by hour for the last 24 hours
cdr_stats type="volume"

# Call volume by day for the last week
cdr_stats type="volume" last="7d" interval="day"

# Top 10 callers today
cdr_stats type="top_callers" last="1d" limit=10

# Failure analysis — which cause codes are most common?
cdr_stats type="by_cause" last="7d"

# Device breakdown
cdr_stats type="by_device" last="1d"

# Location breakdown
cdr_stats type="by_location" last="7d"
```

### cdr_health — Processor Health Check

Check the health of the CDR processor service. Returns database record counts, oldest/newest timestamps, recent file processing activity, enrichment cache stats, and pending files.

**Parameters:** None

**Example:**

```
cdr_health
```

**Returns:**

- `cdr` — Total CDR record count, oldest and newest timestamps
- `cmr` — Total CMR record count
- `file_processing_last_hour` — Files processed, records inserted, errors in the last hour
- `enrichment_cache` — Total cache entries and expired entries
- `incoming_files_pending` — Unprocessed files in the incoming directory
- `status` — Health status ("ok")
- `timestamp` — Current ISO timestamp

## Common Workflows

### Investigate a Reported Call Issue

```
# 1. Search for the call by phone number
cdr_search calling="5551234" called="5559999" last="2h"

# 2. Trace the specific call for full detail
cdr_trace call_id="<globalcallid_callid from search>"

# 3. Check call quality metrics
cdr_quality mos_below=3.5 last="2h"

# 4. Run the returned cisco-dime SDL trace command for deep packet analysis
```

### Daily Call Quality Report

```
# 1. Check overall call volume and failure rate
cdr_stats type="volume" last="1d" interval="hour"

# 2. Identify failure hotspots
cdr_stats type="by_cause" last="1d"

# 3. Find the worst quality calls
cdr_quality mos_below=3.0 last="1d"

# 4. Check which devices have the most issues
cdr_stats type="by_device" last="1d"
```

### Toll Fraud Detection

```
# 1. Check for unusual call volume spikes
cdr_stats type="volume" last="7d" interval="hour"

# 2. Look for top callers — suspicious high volume from a single number
cdr_stats type="top_callers" last="1d" limit=20

# 3. Look for top called destinations — premium rate or international
cdr_stats type="top_called" last="1d" limit=20

# 4. Search for calls to suspicious prefixes
cdr_search called="011%" last="1d" limit=500
cdr_search called="900%" last="1d"
```

### Capacity Planning

```
# 1. Weekly call volume trends
cdr_stats type="volume" last="7d" interval="day"

# 2. Monthly trends
cdr_stats type="volume" start="2026-03-01T00:00:00Z" end="2026-03-31T23:59:59Z" interval="day"

# 3. Location-based utilization
cdr_stats type="by_location" last="7d"

# 4. Device utilization
cdr_stats type="by_device" last="7d"
```

## Integration with Other UC Tools

The cisco-cdr MCP server works best when combined with other Cisco UC tools:

- **cdr_trace → cisco-dime**: The trace tool generates a ready-to-run `cisco-dime` SDL trace command. Use it to download and analyze the SIP signaling for a specific call.
- **cdr_search → cisco-axl**: Cross-reference device names from CDR results with cisco-axl to check device configuration, CSS, device pool, and route pattern setup.
- **cdr_quality → cisco-perfmon**: When quality issues are found, use cisco-perfmon to check real-time system health (CPU, memory, call counters) to correlate with the time of the poor-quality calls.
- **cdr_stats by_device → cisco-risport**: Cross-reference high-failure-rate devices from stats with cisco-risport to check their current registration status.

## Important Notes

- The MCP server requires a running PostgreSQL database with CDR data loaded by the cisco-cdr-processor file watcher.
- Time ranges use the format `30m`, `2h`, `1d`, `7d` for relative ranges, or ISO 8601 timestamps for absolute ranges.
- The processor supports up to 5 CUCM clusters via AXL enrichment. Enrichment adds device descriptions, users, pools, locations, types, and models to CDR records.
- Enrichment data is cached with a configurable TTL (default 24 hours). Use `cdr_health` to check cache stats.
- CDR data retention is configurable (default 90 days). Older records are automatically purged.
- All search parameters support partial matching — you don't need to provide the full number or device name.
