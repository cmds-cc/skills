---
name: cisco-yang-cli
description: Use when managing IOS-XE devices via the cisco-yang CLI — RESTCONF queries, voice troubleshooting, dial-peer management, SIP trace, device health checks, and any YANG/RESTCONF operation on voice gateways (CUBE, ISR, CSR, Cat8k).
---

# cisco-yang CLI

CLI and SDK for IOS-XE devices via RESTCONF (HTTP/JSON). Focused on voice troubleshooting.

## Setup

The user must first configure a device (interactive prompt for password — never pass credentials on the command line):

```bash
cisco-yang config add <name> --host <host> --username <user> [--insecure]
# You will be prompted securely for the password
cisco-yang doctor   # verify connectivity
```

Or use environment variables (set via your shell profile, a `.env` file, or a secrets manager — never hardcode credentials):

```bash
# These should be set securely, e.g. via dotenv, vault, or shell profile
# YANG_HOST, YANG_USERNAME, YANG_PASSWORD
```

## Common Operations

### Query any YANG path

```bash
cisco-yang get "Cisco-IOS-XE-native:native/dial-peer"
cisco-yang get "Cisco-IOS-XE-native:native/voice" --format json
```

### Execute IOS-XE CLI commands

```bash
cisco-yang exec "show dial-peer voice summary"
cisco-yang exec "show sip-ua calls"
cisco-yang exec "show voice call history detail"
cisco-yang exec "show voip rtp connections"
cisco-yang exec "show voice port summary"
```

### Voice troubleshooting shortcuts

```bash
cisco-yang voice dial-peers       # dial-peer config
cisco-yang voice registrations    # SIP phone registrations
cisco-yang voice sip              # voice service/SIP config
cisco-yang voice calls            # active calls
cisco-yang voice dsp              # DSP utilization
cisco-yang voice stats            # call history
cisco-yang voice trace sip        # exec: show sip-ua calls
cisco-yang voice trace history    # exec: show voice call history detail
cisco-yang voice trace rtp        # exec: show voip rtp connections
```

### Discover YANG models

```bash
cisco-yang models --filter voice
cisco-yang describe Cisco-IOS-XE-voice-register-oper
```

### Modify configuration

```bash
cisco-yang set "Cisco-IOS-XE-native:native/voice" --data '{"voice":{"service":{"voip":{"sip":{}}}}}'
cisco-yang delete "Cisco-IOS-XE-native:native/dial-peer/dial-peer-voip/100"
```

## Troubleshooting Workflows

| Scenario          | Commands                                                        |
| ----------------- | --------------------------------------------------------------- |
| Calls not routing | `voice dial-peers` then `exec "show dial-peer voice summary"`   |
| One-way audio     | `exec "show voip rtp connections"` — check codec/IP/port        |
| SIP trunk down    | `voice registrations` then `exec "show sip-ua register status"` |
| Quality issues    | `exec "show voice call history detail"` — check MOS/jitter/loss |
| Gateway health    | `doctor` then `voice dsp` then `exec "show voice port summary"` |

## Output Formats

`--format table` (default), `--format json`, `--format csv`, `--format toon`

## Tips

- Use `--clean` to strip null/empty values from output
- Use `--insecure` for devices with self-signed certificates
- Use `--debug` to see raw HTTP requests/responses
- `voice trace *` commands return CLI text; other voice commands return structured JSON
- The `exec` command can run any IOS-XE CLI command via RESTCONF RPC
- Write operations (`set`, `delete`, non-show `exec`) are blocked in `--read-only` mode
