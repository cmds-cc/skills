# cisco-uc-engineer

[![Skills.sh](https://img.shields.io/badge/skills.sh-cisco--uc--engineer-blue)](https://skills.sh/sieteunoseis/cisco-uc-engineer/cisco-uc-engineer)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-support-orange?logo=buy-me-a-coffee)](https://buymeacoffee.com/automatebldrs)

Unified Communications engineer skill for AI agents. Orchestrates [cisco-axl](https://github.com/sieteunoseis/cisco-axl), [cisco-dime](https://github.com/sieteunoseis/cisco-dime), [cisco-perfmon](https://github.com/sieteunoseis/cisco-perfmon), [cisco-risport](https://github.com/sieteunoseis/cisco-risport), [cisco-support](https://github.com/sieteunoseis/cisco-support), and [audiocodes-cli](https://github.com/sieteunoseis/audiocodes-cli) for UC troubleshooting, provisioning, monitoring, and lifecycle management — including AudioCodes SBCs.

## What is this?

This is a **meta-skill** — it doesn't have its own CLI. Instead, it teaches AI agents how to use multiple Cisco UC CLI tools together to solve cross-domain problems like:

- Phone not registering? Check config (axl) → check registration (risport) → pull traces (dime)
- Call quality issues? Check performance counters (perfmon) → pull SIP traces (dime)
- Audit investigation? Pull audit logs (dime) → check current config (axl)
- Pre-upgrade research? Check bugs (support) → check EoX (support) → compare versions (support)
- Syslog errors? Pull syslogs (dime) → search known bugs (support)

## Installation

### 1. Install the CLI tools you need

```bash
npm install -g cisco-axl      # CUCM configuration
npm install -g cisco-dime      # CUCM log collection
npm install -g cisco-perfmon   # Performance counters
npm install -g cisco-risport   # Device registration
npm install -g cisco-support   # Bugs, EoX, PSIRT, software, warranty
```

### 2. Install the skill

```bash
npx skills add sieteunoseis/cisco-uc-engineer --skill cisco-uc-engineer
```

The skill auto-detects which CLI tools are installed and adapts workflows accordingly. You don't need all five — install what you need.

## Available Tools

| Tool                                                             | Purpose                                                             | npm                                                                                                 |
| ---------------------------------------------------------------- | ------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| [cisco-axl](https://github.com/sieteunoseis/cisco-axl)           | CUCM configuration — phones, lines, route patterns, CSS, SIP trunks | [![npm](https://img.shields.io/npm/v/cisco-axl)](https://www.npmjs.com/package/cisco-axl)           |
| [cisco-dime](https://github.com/sieteunoseis/cisco-dime)         | CUCM log collection — SIP traces, SDL, audit logs                   | [![npm](https://img.shields.io/npm/v/cisco-dime)](https://www.npmjs.com/package/cisco-dime)         |
| [cisco-perfmon](https://github.com/sieteunoseis/cisco-perfmon)   | Real-time performance counters — CPU, memory, call stats            | [![npm](https://img.shields.io/npm/v/cisco-perfmon)](https://www.npmjs.com/package/cisco-perfmon)   |
| [cisco-risport](https://github.com/sieteunoseis/cisco-risport)   | Device registration status — phone reg, CTI, trunk status           | [![npm](https://img.shields.io/npm/v/cisco-risport)](https://www.npmjs.com/package/cisco-risport)   |
| [cisco-support](https://github.com/sieteunoseis/cisco-support)   | Cisco Support APIs — bugs, EoX, PSIRT, software, warranty           | [![npm](https://img.shields.io/npm/v/cisco-support)](https://www.npmjs.com/package/cisco-support)   |
| [audiocodes-cli](https://github.com/sieteunoseis/audiocodes-cli) | AudioCodes Mediant VE SBC — call stats, alarms, KPIs, health checks | [![npm](https://img.shields.io/npm/v/audiocodes-cli)](https://www.npmjs.com/package/audiocodes-cli) |

### Community Tools

| Tool                                                  | Purpose                                                    | npm                                                                                                                 |
| ----------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| [webex-cli](https://github.com/Cloverhound/webex-cli) | Webex Calling — users, locations, numbers, auto-attendants | [![npm](https://img.shields.io/npm/v/@cloverhound/webex-cli)](https://www.npmjs.com/package/@cloverhound/webex-cli) |

## Troubleshooting Workflows

### Phone Not Registering

```bash
cisco-axl get Phone --name SEP001122334455 --format json
cisco-risport query --mac 001122334455
cisco-dime select sip-traces --last 30m --download
```

### Call Routing Issue

```bash
cisco-axl sql query "SELECT dnorpattern, routePartitionName FROM numplan WHERE dnorpattern LIKE '%1001%'"
cisco-axl get CallingSearchSpace --name CSS_INTERNAL --format json
cisco-dime select sip-traces --last 1h --download
```

### Call Quality / One-Way Audio

```bash
cisco-perfmon collect "Cisco CallManager" --counter "CallsActive,RegisteredHardwarePhones"
cisco-risport query --mac 001122334455
cisco-dime select sip-traces --last 30m --download
```

### Performance Investigation

```bash
cisco-perfmon collect "Cisco CallManager" --counter "RegisteredHardwarePhones,CallsActive,CallsAttempted,CallsCompleted"
cisco-perfmon collect "Processor" --counter "% CPU Time"
cisco-risport summary
cisco-dime select syslog --last 1h
```

### Audit / Change Investigation

```bash
cisco-dime select audit --last 1d --download
cisco-axl get Phone --name SEP001122334455 --format json
```

## Diagnostic Decision Tree

When a problem is reported, follow this order:

1. **Is it registered to CUCM?** → `cisco-risport query --mac <mac>`
2. **Is the config correct?** → `cisco-axl get Phone --name <name>`
3. **What do the logs show?** → `cisco-dime select sip-traces --last 30m`
4. **Is the system healthy?** → `cisco-perfmon collect "Cisco CallManager" --counter "CallsActive"`
5. **Are there known bugs?** → `cisco-support bug search --keyword "<error>"`
6. **Is the hardware end-of-life?** → `cisco-support eox --pid <model>`

Start broad, narrow down. Don't pull traces until you've checked the basics.

## Related Skills

| Skill                                                                               | Description                                 |
| ----------------------------------------------------------------------------------- | ------------------------------------------- |
| [cisco-axl-cli](https://skills.sh/sieteunoseis/cisco-axl/cisco-axl-cli)             | Detailed cisco-axl command reference        |
| [cisco-dime-cli](https://skills.sh/sieteunoseis/cisco-dime/cisco-dime-cli)          | Detailed cisco-dime command reference       |
| [cisco-perfmon-cli](https://skills.sh/sieteunoseis/cisco-perfmon/cisco-perfmon-cli) | Detailed cisco-perfmon command reference    |
| [cisco-risport-cli](https://skills.sh/sieteunoseis/cisco-risport/cisco-risport-cli) | Detailed cisco-risport command reference    |
| [cisco-support-cli](https://skills.sh/sieteunoseis/cisco-support/cisco-support-cli) | Cisco Support APIs — bugs, EoX, PSIRT       |
| [cisco-ise-cli](https://skills.sh/sieteunoseis/cisco-ise/cisco-ise-cli)             | Cisco ISE endpoint & session management     |
| [ss-cli](https://skills.sh/sieteunoseis/ss-cli/ss-cli)                              | Delinea Secret Server credential management |
| [audiocodes-cli](https://skills.sh/sieteunoseis/audiocodes-cli/audiocodes-cli)      | AudioCodes Mediant VE SBC command reference |

## Funding

If you find this useful, consider supporting the project:

[![Buy Me A Coffee](https://img.buymeacoffee.com/button-api/?text=Buy%20me%20a%20coffee&emoji=☕&slug=automatebldrs&button_colour=FFDD00&font_colour=000000&font_family=Cookie&outline_colour=000000&coffee_colour=ffffff)](https://buymeacoffee.com/automatebldrs)

## License

MIT
