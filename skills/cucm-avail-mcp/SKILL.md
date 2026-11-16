---
name: cucm-avail-mcp
description: Use when planning or assessing CUCM outages, upgrades and failover through the cucm-avail MCP server — simulating servers going down, CM group / Availability Group safety, device pool and phone impact, upgrade order, phone registration and failover status, and 911 subnet coverage against Cisco Emergency Responder (CER). Covers all cucm_* MCP tools.
license: MIT
metadata:
  author: sieteunoseis
  version: "1.0.0"
---

# CUCM-AVAIL MCP Server

An MCP server built into [CUCM-AVAIL](https://github.com/sieteunoseis/CUCM-AVAIL), a CUCM availability planner. It knows the cluster's servers, Call Manager Groups (CMGs), device pools, phones and their live RISPort registrations, SIP trunks, and (optionally) CER IP subnets, and can simulate what happens when servers go down.

Data comes from the app's own syncs: **AXL** (servers, CMGs, pools, phones, trunks — manual), **RISPort** (registrations — polled on a schedule) and **CER** (subnets — manual). Answers are only as fresh as the last sync/poll; check `cucm_registration_summary` → `lastPoll` when freshness matters.

## Setup

The endpoint is `/mcp` on the CUCM-AVAIL host (Streamable HTTP).

```bash
claude mcp add -s user --transport http cucm-avail https://<cucm-avail-host>/mcp
# when the server sets MCP_API_KEY (enables action tools, and requires the key for everything):
claude mcp add -s user --transport http cucm-avail https://<cucm-avail-host>/mcp \
  --header "Authorization: Bearer <MCP_API_KEY>"
```

Verify with `cucm_servers` — it should list the cluster nodes.

## Concepts

| Term | Meaning |
|---|---|
| CMG | Call Manager Group: ordered list of servers (P1 primary, P2/P3 backups) a device pool's phones register to |
| AG | Availability Group: CMGs that use the **exact same server set**. Two AGs that share **no** servers are independent — losing every server in one doesn't touch the other (a "safe pair", e.g. call-center side A/B) |
| re-register | Phone's current server goes down; it moves to the next available server in its CMG |
| unregistered | No server left in its CMG — the phone loses service |
| already failed over | Phone's CMG primary is down/selected and it's already on a backup (outage in progress) |
| inCmGroup vs callManagerActive | Role vs live state. A node in a CMG is call-processing even if CallManager is down right now |

## Read tools (always available)

| Tool | Use it for | Key params |
|---|---|---|
| `cucm_servers` | Cluster nodes, node type, in a CMG, CallManager active | — |
| `cucm_cm_groups` | Each CMG's members in priority order + AG label | — |
| `cucm_availability_groups` | AGs with servers, member CMGs, phone counts | — |
| `cucm_device_pools` | Pools with CMG and phone count | `cmGroup` (optional filter) |
| `cucm_registration_summary` | Phones per server now, phones not on their primary, last poll | — |
| `cucm_simulate_outage` | **What breaks if these servers go down** — totals, per-CMG, top impacted subnets, trunk/gateway/service impact | `servers` (names, short or FQDN), `topSubnets` |
| `cucm_phone_status` | One device: registration, CMG, pool, IP/subnet; optional "where would it go" | `name`, `ifServersDown` |
| `cucm_upgrade_plan` | Recommended upgrade/reboot order with per-step phone impact | — |
| `cucm_subnets` | Defined subnets, phone counts, CER status | `cerStatus` (any/match/inside/mismatch/missing), `search`, `minPhones`, `limit` |
| `cucm_unmapped_phones` | Phones in no defined subnet, grouped by containing CER subnet or guessed /24 (911 gaps) | — |
| `cucm_trunks` | SIP trunks and their per-server registration | — |

## Action tools (only when the server has `MCP_API_KEY`)

| Tool | Effect |
|---|---|
| `cucm_poll_risport` | Starts a registration poll (background) |
| `cucm_sync_axl` | Full AXL resync from CUCM (takes a while) |
| `cucm_sync_cer` | Pulls CER IP subnets (~30 s) |
| `cucm_import_cer_subnets` | Adds CER subnets to the app (name = ERL, description = CER location) |
| `cucm_replace_with_cer` | "Use CER" for app subnets: replaces each with CER's overlapping subnets, or rolls it into a wider CER subnet already defined. Atomic per subnet |

**Always confirm with the user before `cucm_import_cer_subnets` or `cucm_replace_with_cer`**, listing exactly which CIDRs change. These edit the subnet list that drives 911-location and impact reports. Syncs/polls are safe but heavy — don't loop them.

## Workflows

### Planned outage / maintenance window
1. `cucm_servers` — confirm the exact node names.
2. `cucm_simulate_outage servers=["dcrsub05"]` — report `willReRegister`, `unregistered` (must be 0 for a clean window), affected CMGs and subnets, trunk/gateway impact.
3. For a pair: simulate both together; any `unregistered` > 0 means some CMG has no surviving member.
4. `cucm_registration_summary` — note phones already off their primary before you start.

### Upgrade order
`cucm_upgrade_plan` gives the sequence; sanity-check a step with `cucm_simulate_outage` on that step's servers.

### "Is CMG X independent of CMG Y?" / side A/B
`cucm_cm_groups` → compare server sets. Disjoint sets (different AGs that share no server) = safe pair. Any shared server = a single failure hits both.

### Single phone
`cucm_phone_status name="SEP…"`; add `ifServersDown=[...]` to see where it would land.

### 911 / CER subnet coverage
- `cucm_subnets cerStatus="missing" minPhones=1` — phone subnets CER doesn't know (real gaps).
- `cucm_subnets cerStatus="mismatch"` — CER splits ours into narrower subnets; `useCerTargets` shows what "Use CER" would add.
- `inside` = ours sits inside a wider CER subnet (e.g. an Expressway /32 inside CER's /24) — usually fine.
- `cucm_unmapped_phones` — `inCer` rows are importable; `notInCer` rows are phones no CER subnet covers.

## Interpreting results

- Totals partition phones: `mapped + noIp + sharedIp + unmappedWithIp = phones`. `noIp` is mostly unregistered phones (no current or last-known IP).
- Shared-IP phones (many phones on one IP, e.g. Expressway/MRA, Citrix) count under a subnet defined for them but are never used for CER suggestions.
- "broad" CER subnets (wider than /22) are flagged and skipped by bulk actions.
- Server names accept short (`dcrsub05`) or FQDN; unknown names are returned in `unknownServers` rather than failing the call.

## Troubleshooting

- Tool list has no action tools → the server has no `MCP_API_KEY`.
- HTTP 401 → key set on the server but missing/wrong in the client header.
- `cucm_subnets` shows no `cerStatus` → CER not configured or never synced (`cer` is null / `lastSync` empty).
- Numbers look stale → check `lastPoll`; AXL data changes only after a Sync AXL.
