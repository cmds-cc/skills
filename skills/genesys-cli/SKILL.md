---
name: genesys-cli
description: Use when managing Genesys Cloud CX via the genesys-cli CLI — conversations, BYOC trunks, queues, agents, Edge servers, external contacts, audit history, and connectivity health checks. Covers OAuth configuration, multi-org support, and all Platform API operations.
license: MIT
metadata:
  author: sieteunoseis
  version: "1.0.0"
---

# Genesys CLI

A CLI for Genesys Cloud CX via the Platform API.

## Setup

The CLI must be available. Either:

```bash
# Option 1: Use npx (no install needed, works immediately)
npx genesys-cli --help

# Option 2: Install globally for faster repeated use
npm install -g genesys-cli
```

If using npx, prefix all commands with `npx`: `npx genesys-cli conversations list ...`
If installed globally, use directly: `genesys-cli conversations list ...`

### Configuration

Configure a Genesys Cloud org (interactive prompt for client secret — never pass credentials on the command line):

```bash
genesys-cli config add <name> --client-id <id> --client-secret <secret> --region <region>
```

Regions: `mypurecloud.com` (US East), `usw2.pure.cloud` (US West), `cac1.pure.cloud` (Canada), `mypurecloud.ie` (EMEA), `mypurecloud.de` (Frankfurt), `mypurecloud.com.au` (APAC), `mypurecloud.jp` (Japan), etc.

Or use environment variables:

```bash
# These should be set securely, e.g. via dotenv, vault, or shell profile
# GENESYS_CLIENT_ID, GENESYS_CLIENT_SECRET, GENESYS_REGION
```

Test the connection:

```bash
genesys-cli config test
```

### Multi-Org Support

```bash
genesys-cli config add prod --client-id <id> --client-secret <secret> --region mypurecloud.com
genesys-cli config add staging --client-id <id> --client-secret <secret> --region mypurecloud.com
genesys-cli config use prod
genesys-cli conversations list --org staging    # override per-command
```

## Common Workflows

### Health Check

```bash
# Check connectivity, OAuth, config, and security
genesys-cli doctor
```

The doctor command verifies active org configuration, OAuth token acquisition, org details, config file permissions, and audit trail size.

### Conversations

```bash
# List recent conversations
genesys-cli conversations list --last 1h

# Filter by caller or callee
genesys-cli conversations list --caller 5551234 --last 2h
genesys-cli conversations list --callee 5559999 --last 1d

# Filter by queue
genesys-cli conversations list --queue "Customer Support" --last 4h

# Filter by disconnect reason
genesys-cli conversations list --disconnect-reason error --last 1d

# Get full detail for a specific conversation
genesys-cli conversations detail <conversationId>
```

### BYOC Trunks

```bash
# List trunks and their connection status
genesys-cli trunks list

# Show trunk call metrics
genesys-cli trunks metrics
```

**What to look for:**

- `connected` + `inService=true` means the trunk is healthy
- Trunks not connected or `inService=false` indicate SBC-to-Genesys SIP connectivity issues

### Queues

```bash
# List routing queues
genesys-cli queues list

# Include observation stats (agents online, calls waiting)
genesys-cli queues list --detail
```

### Agents

```bash
# List agents with presence status
genesys-cli agents list

# Filter agents by queue
genesys-cli agents list --queue "Customer Support"

# Limit results
genesys-cli agents list --limit 50
```

### Edge Servers

```bash
# List Edge servers and their status
genesys-cli edges list

# List telephony sites
genesys-cli edges sites
```

**What to look for:**

- Edge servers OFFLINE indicate media processing issues
- All edges should show ACTIVE status for healthy call handling

### External Contacts

```bash
# Search contacts by name, email, or phone
genesys-cli external-contacts list --query "John Doe"
genesys-cli external-contacts list --query "5551234"

# Get a specific contact by ID
genesys-cli external-contacts get <contactId>
```

### Audit History

```bash
# Recent config changes
genesys-cli audit list --last 1d

# Filter by user
genesys-cli audit list --user "admin@company.com" --last 7d

# Filter by action type
genesys-cli audit list --action Create --last 1d
genesys-cli audit list --action Delete --last 7d

# Filter by entity type
genesys-cli audit list --entity Queue --last 1d
```

## Output Formats

Use `--format` to control output:

- `--format table` — human-readable tables (default)
- `--format json` — structured JSON for parsing
- `--format toon` — token-efficient format (recommended for AI agents, ~40% fewer tokens than JSON)
- `--format csv` — CSV for spreadsheet export

**For AI agents:** Use `--format toon` for list queries to reduce token usage. Use `--format json` when you need to parse nested structures.

## Global Flags

- `--org <name>` — target a specific org configuration
- `--client-id <id>` — override OAuth client ID
- `--client-secret <secret>` — override OAuth client secret
- `--region <region>` — override Genesys Cloud region
- `--clean` — remove empty/null values from results
- `--no-audit` — disable audit logging for this command
- `--debug` — enable debug logging
- Config is stored at `~/.genesys-cli/config.json`
- All operations are audit-logged to `~/.genesys-cli/audit.jsonl`. Use `--no-audit` to skip.
