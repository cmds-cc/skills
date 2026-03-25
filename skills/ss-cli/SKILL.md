---
name: ss-cli
description: Use when managing secrets from Delinea Secret Server — fetching credentials, syncing env files, deploying with injected secrets, resolving config templates, and rotating passwords. Covers all ss-cli operations.
license: MIT
metadata:
  author: sieteunoseis
  version: "2.0.0"
---

# ss-cli — Delinea Secret Server CLI

CLI for managing secrets from Delinea Secret Server. Fetch credentials, sync env files, deploy with injected secrets, resolve config templates, and rotate passwords.

## Before Any Operation

Always validate the token first:

```bash
ss-cli token-status --json
```

| Field         | Description                                                                 |
| ------------- | --------------------------------------------------------------------------- |
| `valid`       | `true` if token is usable                                                   |
| `source`      | `"file"` (from `ss-cli login`), `"session"` (SS_TOKEN env var), or `"none"` |
| `minutesLeft` | Remaining minutes (file-based only)                                         |
| `expiresAt`   | ISO timestamp of expiry (file-based only)                                   |

If `valid` is `false`, stop and tell the user: **"Secret Server token is expired or missing. Please run: `ss-cli login`"**

If `minutesLeft` is less than 5 and the workflow will take time, warn the user before starting.

## Fetching Secrets

```bash
# Get a secret by ID
ss-cli get <id> --format json

# Get a specific field
ss-cli get <id> --field password
```

## Env File Sync

Refreshes a `.env` file by pulling latest values from Secret Server using a JSON map.

```bash
# Use configured defaults
ss-cli refresh-env

# Specify paths explicitly
ss-cli refresh-env --env-file /path/to/global.env --map-file /path/to/env-map.json
```

### One-time config setup

```bash
ss-cli config set defaultEnvFile /path/to/global.env
ss-cli config set envMapFile /path/to/env-map.json
```

### Map file format

```json
[
  {
    "secretId": 21908,
    "name": "LDAP credentials",
    "fields": {
      "username": "LDAP_USERNAME",
      "password": "LDAP_PASSWORD"
    }
  },
  {
    "secretId": 21909,
    "name": "InfluxDB",
    "fields": {
      "username": "INFLUXDB_USERNAME",
      "password": "INFLUXDB_PASSWORD",
      "url": "INFLUXDB_URL"
    },
    "transforms": {
      "url": "hostname"
    }
  }
]
```

Supported transforms: `hostname` (extracts hostname from URL), `dbname` (extracts path from URL).

### Pairing with dotenvx

```bash
ss-cli refresh-env --env-file .env --map-file env-map.json
dotenvx encrypt .env
dotenvx run -- docker-compose up -d
```

After sync, remind the user to restart services that load the env file at startup.

## Deploy with Injected Secrets

Runs a command with secrets injected as environment variables. Secrets are never written to disk.

```bash
# Deploy with a map file
ss-cli run --map-file env-map.json -- docker-compose up -d

# Deploy in a specific directory
ss-cli run --map-file env-map.json -- sh -c 'cd "$1" && docker-compose pull && docker-compose up -d' _ /opt/myservice

# Inject a single secret with prefix
ss-cli run --secret 21909 --env-prefix DB_ -- docker-compose up -d
# Injects: DB_USERNAME, DB_PASSWORD, DB_URL, etc.
```

Prefer `ss-cli run` over writing a `.env` file whenever possible — secrets never touch the filesystem.

## Resolve Config Templates

Replaces `<ss:ID:field>` placeholders in any config file with live values from Secret Server.

### Placeholder format

```
<ss:SECRET_ID:FIELD_NAME>
```

### Usage

```bash
# Preview resolved output
ss-cli resolve --input template.yml

# Write to a local file
ss-cli resolve --input template.yml --output resolved.yml

# Read from stdin
cat template.yml | ss-cli resolve

# Deploy to remote server (resolved config never stored locally)
ss-cli resolve --input nginx.conf.tpl | ssh user@server "sudo tee /etc/nginx/conf.d/app.conf > /dev/null && sudo nginx -s reload"
```

Works with any text format — nginx, docker-compose, systemd, YAML, JSON, Kubernetes manifests, shell scripts. Templates can be committed to version control safely — they contain only placeholders.

## Rotate and Sync

Updates a secret field, then re-syncs downstream env files.

```bash
# Step 1: Update the secret
# Never pass the new value directly on the command line — use a placeholder or prompt
ss-cli update <id> --field password=<new-value>
ss-cli update <id> --field password=<new-value> --field notes="Rotated 2026-03-19"

# Step 2: Verify
ss-cli get <id> --format json

# Step 3: Re-sync env files
ss-cli refresh-env --env-file /path/to/global.env --map-file /path/to/env-map.json

# Step 4: Restart affected services
docker-compose -f /path/to/docker-compose.yml up -d --force-recreate
```

Always verify the update before re-syncing. Restart all services that use the rotated credential.

## Audit Trail

```bash
ss-cli audit -n 5              # last 5 entries
ss-cli audit --verify          # check HMAC chain integrity
```

## Security Notes

- Never pass credential values directly on the command line (e.g., `--field password=<value>`). Use a shell variable or prompt instead — never hardcode credentials.
- Never store secret values in files when avoidable. Use `ss-cli run` (subprocess injection) or `ss-cli resolve` (stdout-only).
- Session tokens (`source: "session"`) are in-memory only and cannot be shared across terminals.
- For cross-process agent access, use file-based tokens (`ss-cli login`).
- Resolved config output should never be committed to version control.
