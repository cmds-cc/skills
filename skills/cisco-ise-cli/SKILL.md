---
name: cisco-ise-cli
description: Use when managing Cisco ISE via the cisco-ise CLI — endpoints, guests, network devices, sessions, RADIUS/TACACS monitoring, and identity management operations.
---

# cisco-ise CLI

CLI for Cisco ISE (Identity Services Engine) 3.1+ targeting day-to-day operations and troubleshooting.

## Setup

Configure a cluster (one-time):

```bash
cisco-ise config add <name> --host <host> --username <user> --password '<ss:ID:password>' --insecure
cisco-ise config test
```

Or use environment variables:

```bash
export CISCO_ISE_HOST=<host>
export CISCO_ISE_USERNAME=<user>
export CISCO_ISE_PASSWORD=<password>
```

Secret Server references are supported: `<ss:ID:field>` (requires ss-cli).

## Command Groups

| Command | Description |
|---------|-------------|
| `config` | Manage ISE cluster configurations (add/use/list/show/remove/test/update/clear-cache) |
| `endpoint` | Manage endpoints (list/search/add/update/delete, CSV bulk) |
| `guest` | Manage guest users (list/search/create/extend/suspend/reinstate/delete/portals) |
| `network-device` | Manage NADs (list/search/get/add/update/delete) |
| `session` | Active sessions (list/search/disconnect/reauth) |
| `radius` | RADIUS monitoring (failures with human-readable reasons, live polling) |
| `tacacs` | TACACS+ monitoring (failures/live/command-sets/profiles) |
| `identity-group` | List identity groups (--type endpoint/user) |
| `auth-profile` | List/get authorization profiles |
| `trustsec` | TrustSec SGTs and SGACLs (read-only) |
| `deployment` | ISE deployment nodes and status (read-only) |

## Common Workflows

### List all endpoints

```bash
cisco-ise endpoint list --insecure
cisco-ise endpoint search --mac AA:BB:CC:DD:EE:FF --insecure
```

### Add an endpoint (with dry-run)

```bash
cisco-ise endpoint add --mac AA:BB:CC:DD:EE:FF --group "Profiled" --dry-run --insecure
cisco-ise endpoint add --csv endpoints.csv --insecure
```

### Check RADIUS authentication failures

```bash
cisco-ise radius failures --last 1h --insecure
cisco-ise radius failures --last 30m --user jdoe --insecure
```

### Live RADIUS monitoring

```bash
cisco-ise radius live --insecure
```

### Manage guest users

```bash
cisco-ise guest portals --insecure
cisco-ise guest create --first "John" --last "Doe" --email "john@example.com" --portal "Sponsored Guest Portal (default)" --insecure
cisco-ise guest list --insecure
```

### Check active sessions

```bash
cisco-ise session list --insecure
cisco-ise session search --mac E2:7C:7E:5B:F0:E0 --insecure
cisco-ise session disconnect E2:7C:7E:5B:F0:E0 --insecure
```

### Manage network devices

```bash
cisco-ise network-device list --insecure
cisco-ise network-device add --name "switch01" --ip 10.0.0.1 --radius-secret '<ss:ID:radius-secret>' --insecure
```

### View deployment info

```bash
cisco-ise deployment nodes --insecure
cisco-ise deployment status --insecure
```

## MAC Address Formats

Any common format is accepted and automatically normalized:
- `AA:BB:CC:DD:EE:FF` (colon-separated)
- `AA-BB-CC-DD-EE-FF` (dash-separated)
- `AABB.CCDD.EEFF` (Cisco dot notation)
- `aabbccddeeff` (bare hex)

## Output Formats

- `--format table` (default) — human-readable
- `--format json` — for scripting/parsing
- `--format toon` — token-efficient for AI agents (recommended)
- `--format csv` — for spreadsheets

## Key Flags

- `--insecure` — required for self-signed ISE certs (most environments)
- `--dry-run` — show HTTP method, URL, and payload without executing
- `--read-only` — block all write operations (human-in-the-loop confirmation)
- `--cluster <name>` — target a specific cluster
- `--no-cache` — bypass 5-minute response cache
- `--debug` — enable verbose logging
- `--no-audit` — disable audit trail logging

## RADIUS Troubleshooting Workflow

When a user reports a connectivity or authentication problem, follow this workflow. Always use `--format json` so you can parse results programmatically.

### Step 1: Identify the device

Get the MAC address from the user, or find it from active sessions:

```bash
cisco-ise session list --format json
cisco-ise session search --mac <mac> --format json
cisco-ise session search --user <username> --format json
```

### Step 2: Run troubleshoot

```bash
cisco-ise radius troubleshoot --mac <mac> --last 1d --format json
```

This returns the full auth history with: pass/fail, matched policy rules, failure reasons, auth protocol, VLAN assignment, and ISE server.

### Step 3: Analyze and correlate

Based on the troubleshoot output, run follow-up commands:

**If auth is failing — check the user:**
```bash
cisco-ise internal-user get <username> --format json
```
- Is `enabled` false? → `cisco-ise internal-user update <user> --enable`
- Is the user missing? → `cisco-ise internal-user add --user-name <user> --user-password <pass> --group <group>`

**If NAD not found (11007) — check network device:**
```bash
cisco-ise network-device list --format json
```
- Is the device missing? → `cisco-ise network-device add --name <name> --ip <ip> --radius-secret <secret>`

**If shared secret mismatch (11036, 22040) — verify NAD config:**
```bash
cisco-ise network-device get <device-name> --format json
```
- Check `authenticationSettings.radiusSharedSecret` matches the device config.

**If CoA failing (5417, 11213) — check CoA port:**
```bash
cisco-ise network-device get <device-name> --format json
```
- Check `coaPort`. Cisco uses 1700, RFC standard is 3799. UniFi uses 3799.

**If certificate rejected (12520) — check deployment:**
```bash
cisco-ise deployment nodes --format json
```
- Verify EAP certificate. Client must trust the signing CA.

**If authorization denied (15039) — check policy:**
```bash
cisco-ise auth-profile list --format json
cisco-ise identity-group list --format json
```
- User may not match any authorization rule. Check group membership.

### Step 4: Verify the fix

After making changes, ask the user to reconnect, then re-run:
```bash
cisco-ise radius troubleshoot --mac <mac> --last 30m
```
Confirm the latest auth shows PASS.

### Step 5: Common failure code reference

| Code | Issue | Quick Fix |
|------|-------|-----------|
| 5411 | EAP timeout — no client response | Check supplicant config, certificate trust |
| 5417 | CoA failed | Check NAD CoA port and shared secret |
| 11007 | NAD not found | Add NAD: `cisco-ise network-device add` |
| 11036 | Invalid Message-Authenticator | Shared secret mismatch — update NAD |
| 12520 | Client rejected ISE cert | Install CA cert on client or fix ISE EAP cert |
| 15039 | Rejected by authz profile | Add authorization rule for this user/group |
| 22040 | Wrong password or shared secret | Reset password or fix shared secret |
| 22056 | User not found | Add user or check auth policy identity store |
| 22061 | User disabled | `cisco-ise internal-user update <user> --enable` |
| 24408 | AD auth failed — wrong password | Check AD credentials, also check shared secret if PAP |

The CLI has 311 ISE failure codes mapped in `cli/utils/failure-reasons.js` with causes and remediation. The `radius troubleshoot` command outputs these automatically.

## Internal User Management

```bash
cisco-ise internal-user list
cisco-ise internal-user get <name>
cisco-ise internal-user add --user-name <name> --user-password <pass> --group <group>
cisco-ise internal-user update <name> --enable|--disable|--group <group>|--user-password <pass>
cisco-ise internal-user delete <name>
```

## Agent-Safe Deployment

When giving AI agents access to the cisco-ise CLI, use these layers of protection. Only ISE-side RBAC is truly unbypassable — the others add friction but a determined agent with shell access could work around them.

### Layer 1: ISE Read-Only Admin (recommended — unbypassable)

Create a dedicated ISE admin account with **ERS Operator** (read-only) instead of **ERS Admin** (read-write). The ISE server itself rejects all write API calls regardless of what the CLI or agent does.

In ISE admin GUI:
1. Administration > System > Admin Access > Administrators > Admin Users
2. Create a new admin (e.g., `cli-reader`)
3. Assign to **ERS Operator** group (read-only ERS access)
4. Optionally add **MNT Admin** for monitoring/troubleshooting

```bash
cisco-ise config add prod --host <host> --username cli-reader --password '<ss:ID:password>' --insecure
```

This is the only protection that cannot be bypassed by any client-side mechanism. The ISE server enforces the restriction.

For write operations, use a separate cluster config with ERS Admin credentials that only humans access:
```bash
cisco-ise config add prod-admin --host <host> --username cli-admin --password '<ss:ID:password>' --read-only --insecure
```

### Layer 2: CLI Read-Only Flag (human-in-the-loop)

```bash
cisco-ise config add prod --host <host> --username <user> --password '<ss:ID:password>' --read-only --insecure
```

Write operations require typing a random 8-character hex string in an interactive TTY. Non-interactive environments (agents, scripts, pipes) are blocked entirely because `process.stdin.isTTY` is false.

**Limitation:** An agent with shell access could edit `~/.cisco-ise/config.json` directly to remove the `readOnly` flag.

### Layer 3: Separate Credentials

Keep admin/write credentials in Secret Server, not in the config file:
```bash
cisco-ise config add prod --host <host> --username <user> --password '<ss:ID:password>' --insecure
```

The agent never sees the actual password — it's resolved at runtime from Secret Server via `ss-cli`. Rotate credentials in Secret Server without touching the CLI config.

### Recommended Setup for Agent Access

| Account | ISE Role | CLI Config | Used By |
|---------|----------|------------|---------|
| `cli-reader` | ERS Operator + MNT Admin | `prod` cluster | AI agents (read + troubleshoot) |
| `cli-admin` | ERS Admin | `prod-admin` cluster, `--read-only` | Humans only (writes need TTY confirmation) |
| `sponsor` | Sponsor (internal user) | `--sponsor-user` in config | Guest management |

### Data Exposure by Command

Before granting agent access, understand what data each command exposes. Use this to decide which ISE RBAC permissions to grant.

**Low sensitivity — safe for most agents:**

| Command | Data Exposed |
|---------|-------------|
| `deployment nodes` | ISE hostnames, roles, services |
| `deployment status` | Node status |
| `identity-group list` | Group names and descriptions |
| `auth-profile list` | Authorization profile names |
| `trustsec sgt list` | SGT names and descriptions |
| `trustsec sgacl list` | SGACL names |
| `tacacs command-sets` | TACACS command set names |
| `tacacs profiles` | TACACS profile names |

**Medium sensitivity — contains user/device identifiers:**

| Command | Data Exposed |
|---------|-------------|
| `endpoint list/search` | MAC addresses, endpoint group membership |
| `session list/search` | Active users, MAC addresses, NAS IPs, ISE server |
| `radius auth-log` | Auth history: usernames, MACs, pass/fail, timestamps, policy matches |
| `radius troubleshoot` | Full auth detail: all of auth-log plus protocol, TLS version, VLAN, identity store |
| `radius failures` | Failed auth attempts with usernames and failure reasons |
| `internal-user list` | Usernames, descriptions, user IDs |
| `guest list` | Guest usernames and IDs |

**High sensitivity — contains secrets or enables write operations:**

| Command | Data Exposed / Risk |
|---------|-------------|
| `network-device get` | **RADIUS shared secrets in plaintext**, device IPs, CoA ports |
| `config show` | ISE hostname, admin username, masked password, sponsor username |
| `internal-user get` | User details including email, group membership, enabled status |
| `auth-profile get` | Full authorization profile config (VLANs, ACLs, attributes) |
| `endpoint add/update/delete` | **Write operation** — modifies endpoint database |
| `network-device add/update/delete` | **Write operation** — modifies NAD database, exposes shared secrets |
| `internal-user add/update/delete` | **Write operation** — creates/modifies/removes user accounts |
| `guest create/delete` | **Write operation** — creates/removes guest accounts |
| `session disconnect/reauth` | **Write operation** — disrupts active user sessions |

### Recommended Agent Scoping

**Troubleshooting-only agent (most common):**
- ISE role: ERS Operator + MNT Admin
- Safe commands: `session list/search`, `radius auth-log`, `radius troubleshoot`, `endpoint list/search`, `identity-group list`, `deployment nodes`
- Risk: exposes usernames, MACs, auth history — acceptable for helpdesk/NOC use

**Read-all agent (full visibility):**
- ISE role: ERS Operator + MNT Admin
- All read commands including `network-device get` (exposes shared secrets)
- Risk: shared secrets visible — use only if agent environment is trusted

**Full access agent (not recommended for production):**
- ISE role: ERS Admin
- All commands including writes
- Risk: agent can modify ISE configuration — use `--read-only` flag as a speed bump only

### Audit Trail

Every CLI command is logged to `~/.cisco-ise/audit.jsonl` with timestamp, command, and cluster name. Review agent activity with:

```bash
cat ~/.cisco-ise/audit.jsonl | tail -20
```
