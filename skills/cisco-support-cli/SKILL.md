---
name: cisco-support-cli
description: Use when querying Cisco Support APIs — bug search/lookup, support cases, end-of-life dates, PSIRT security advisories, product info, software suggestions, serial number coverage/warranty, and RMA tracking.
license: MIT
metadata:
  author: sieteunoseis
  version: "1.0.0"
---

# cisco-support CLI

CLI for querying Cisco Support APIs. Covers bugs, cases, EoX (end-of-life), PSIRT security advisories, product info, software suggestions, serial coverage/warranty, and RMA tracking.

## Setup

```bash
# Install globally
npm install -g cisco-support

# Configure credentials (from https://apidocs-prod.cisco.com/)
cisco-support config add --client-id <your-id> --client-secret <your-secret>

# Or with Secret Server
cisco-support config add --client-id '<ss:22001:client-id>' --client-secret '<ss:22001:client-secret>'

# Verify
cisco-support config test
cisco-support doctor
```

## Bug Search & Lookup

```bash
# Get specific bugs
cisco-support bug get CSCvx12345
cisco-support bug get CSCvx12345,CSCvx67890

# Search by keyword
cisco-support bug search --keyword "memory leak"
cisco-support bug search --keyword "crash" --product "Cisco Unified Communications Manager"

# Search by product and release
cisco-support bug search --pid CSR1000V --release 17.3

# Filter by severity and status
cisco-support bug search --keyword "voice" --severity 1,2 --status open

# Progressive search (auto-fallback across strategies)
cisco-support bug search --keyword "onemedia gateway" --progressive
```

## Support Cases

```bash
cisco-support case get 123456789
cisco-support case get 123456789,987654321
cisco-support case search --contract 12345
cisco-support case search --user userID123
```

## End-of-Life (EoX)

```bash
cisco-support eox --pid WS-C3850-24T
cisco-support eox --serial FJC12345678
cisco-support eox --software "15.0(2)SE"
cisco-support eox --date-range 2026-01-01 2026-06-30
```

## Security Advisories (PSIRT)

```bash
cisco-support psirt latest --count 10
cisco-support psirt get cisco-sa-20260101-example
cisco-support psirt --cve CVE-2026-1234
cisco-support psirt --bug CSCvx12345
cisco-support psirt --severity critical
cisco-support psirt --year 2026
cisco-support psirt --published-after 2026-01-01 --published-before 2026-03-01
```

## Product Information

```bash
cisco-support product --serial FJC12345678
cisco-support product --pid WS-C3850-24T
cisco-support product mdf --id 1234
```

## Software Suggestions

```bash
cisco-support software suggest --pid WS-C3850-24T
cisco-support software releases --pid WS-C3850-24T
cisco-support software compare --pid WS-C3850-24T --from 16.9 --to 17.3
```

## Serial Coverage & Warranty

```bash
cisco-support serial coverage FJC12345678
cisco-support serial coverage FJC12345678,FJC98765432
cisco-support serial warranty FJC12345678
```

## RMA Tracking

```bash
cisco-support rma get 12345678
cisco-support rma --user userID123
cisco-support rma --serial FJC12345678
```

## Common Workflows

### Check if hardware is end-of-life
```bash
cisco-support eox --pid WS-C3850-24T --format json
cisco-support eox --serial FJC12345678
```

### Find known bugs for a software version
```bash
cisco-support bug search --pid C9800-CL-K9 --release 17.9 --severity 1,2,3
```

### Security audit for a product
```bash
cisco-support psirt --severity critical --year 2026
cisco-support psirt --cve CVE-2026-1234
```

### Pre-upgrade research
```bash
cisco-support software suggest --pid ISR4321
cisco-support software compare --pid ISR4321 --from 17.6 --to 17.9
cisco-support bug search --pid ISR4321 --release 17.9 --status open
```

### Check warranty before RMA
```bash
cisco-support serial warranty FJC12345678
cisco-support rma --serial FJC12345678
```

## Output Formats

All commands support `--format table|json|toon|csv`:

```bash
cisco-support bug search --keyword "crash" --format json
cisco-support eox --pid WS-C3850-24T --format csv > eox-report.csv
```

## Configuration

```bash
cisco-support config show          # display config (secret masked)
cisco-support config test          # verify OAuth token works
cisco-support config enable-api bug,psirt   # enable specific APIs
cisco-support config disable-api rma        # disable specific APIs
cisco-support doctor               # full health check
```

## API Access

Credentials are obtained from https://apidocs-prod.cisco.com/. You need a Cisco CCO account and must register an application to get a client ID and secret. Each API may require separate enablement on the portal.
