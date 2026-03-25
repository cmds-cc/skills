---
name: audiocodes-cli
description: Use when troubleshooting VoIP on AudioCodes Mediant VE SBCs — call statistics, active alarms, device health checks, and SIP trace capture via the REST API. Covers device configuration management, multi-format output, and connectivity diagnostics.
license: MIT
metadata:
  author: sieteunoseis
  version: "1.0.0"
---

# AudioCodes CLI

A CLI for troubleshooting VoIP on AudioCodes Mediant VE SBCs via the REST API.

## Setup

The CLI must be available. Either:

```bash
# Option 1: Use npx (no install needed, works immediately)
npx audiocodes-cli --help

# Option 2: Install globally for faster repeated use
npm install -g audiocodes-cli
```

If using npx, prefix all commands with `npx`: `npx audiocodes-cli calls list`
If installed globally, use directly: `audiocodes-cli calls list`

### Configuration

Configure a device (interactive prompt for password — never pass credentials on the command line):

```bash
audiocodes-cli config add <name> --host <hostname> --username <user> --password <pass> --insecure
```

Use `--insecure` for self-signed certificates (common on AudioCodes devices).

Or use environment variables (set via your shell profile, a `.env` file, or a secrets manager — never hardcode credentials):

```bash
# These should be set securely, e.g. via dotenv, vault, or shell profile
# AUDIOCODES_HOST, AUDIOCODES_USERNAME, AUDIOCODES_PASSWORD
```

Test the connection:

```bash
audiocodes-cli config test
```

## Common Workflows

### Check device health

```bash
audiocodes-cli doctor
```

Shows configuration status, REST API connectivity, device model/firmware/uptime, and security checks (config file permissions, audit trail).

### List active alarms

```bash
audiocodes-cli alarms list
audiocodes-cli alarms list --severity critical
audiocodes-cli alarms list --format json
```

### View call statistics

```bash
# Global call stats (default)
audiocodes-cli calls list

# Per-IP-group stats
audiocodes-cli calls list --scope ipGroup

# Per-SRD stats
audiocodes-cli calls list --scope srd
```

Shows active calls, established calls, attempted calls, failed calls, ASR, ACD, and more.

### SIP trace capture (pending lab validation)

```bash
audiocodes-cli sip-trace start --output trace.pcap --filter 10.0.0.1
audiocodes-cli sip-trace stop
```

Note: SIP trace endpoints are pending validation against the AudioCodes REST API.

### Device configuration management

```bash
audiocodes-cli config add lab-ve --host 10.0.0.50 --username Admin --password "$AUDIOCODES_PASSWORD" --insecure
audiocodes-cli config add prod-sbc --host 10.0.0.100 --username Admin --password "$AUDIOCODES_PASSWORD"
audiocodes-cli config use prod-sbc
audiocodes-cli config list
audiocodes-cli config show           # masks passwords
audiocodes-cli config remove lab-ve
audiocodes-cli config test           # test connectivity to active device
```

### Multiple Devices

```bash
audiocodes-cli config add lab --host 10.0.0.50 --username Admin --password "$AUDIOCODES_PASSWORD" --insecure
audiocodes-cli config add prod --host 10.0.0.100 --username Admin --password "$AUDIOCODES_PASSWORD"
audiocodes-cli config use prod
audiocodes-cli calls list --device lab    # override per-command
```

## Output Formats

Use `--format` to control output:

- `--format table` — human-readable tables (default)
- `--format json` — structured JSON for parsing
- `--format toon` — token-efficient format (recommended for AI agents, ~40% fewer tokens than JSON)
- `--format csv` — CSV for spreadsheet export

**For AI agents:** Use `--format toon` for queries to reduce token usage. Use `--format json` when you need to parse structured data.

## Global Flags

- Use `--insecure` for self-signed certificates (common on AudioCodes devices).
- The `--clean` flag removes empty/null values from results.
- All operations are audit-logged to `~/.audiocodes-cli/audit.jsonl`. Use `--no-audit` to skip.
- Use `--debug` to see HTTP request/response details.

## API Endpoints

The AudioCodes Mediant VE REST API (`/api/v1/`) provides:

| Endpoint                             | Description                                      |
| ------------------------------------ | ------------------------------------------------ |
| `/status`                            | Device status (model, firmware, uptime, serial)  |
| `/alarms/active`                     | Active device alarms                             |
| `/alarms/history`                    | Historical alarms                                |
| `/kpi/current/sbc/callStats/{scope}` | Real-time call statistics (global, ipGroup, srd) |
| `/kpi/current/media`                 | Media statistics (codecs, DSP)                   |
| `/kpi/current/network`               | Network statistics (DDoS, HA, ports)             |
| `/kpi/current/system`                | System statistics (CPU, license, storage)        |
| `/kpi/history`                       | Historical KPIs (15-minute intervals)            |
| `/sip/sendRequest`                   | Send SIP requests                                |
| `/sip/sipRecording`                  | SIP recording control                            |
| `/actions/reset`                     | Device reset                                     |
| `/actions/saveConfiguration`         | Save config to NVRAM                             |
| `/files`                             | Upload/download configuration files              |
| `/license`                           | License management                               |
