# cmds-cc/skills

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-support-orange?logo=buy-me-a-coffee)](https://buymeacoffee.com/automatebldrs)

AI agent skills for voice, UC, and telecom infrastructure. Each skill teaches AI agents how to use CLI tools for troubleshooting, provisioning, and monitoring.

## Orchestrators

Orchestrators combine multiple CLI tools into cross-domain workflows.

| Orchestrator                                                            | Description                                                                                                                              | Docs                                                   |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| [cisco-uc-engineer](https://skills.sh/cmds-cc/skills/cisco-uc-engineer) | Cisco Unified Communications — orchestrates axl, dime, perfmon, risport, support, ucce, yang, audiocodes, genesys CLIs and cisco-cdr MCP | [docs/cisco-uc-engineer.md](docs/cisco-uc-engineer.md) |

## Skills

Individual CLI tool skills with detailed command reference.

### AudioCodes

| Skill                                                             | CLI Tool                                                         | Description                                                         | npm                                                                                                 |
| ----------------------------------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| [audiocodes-cli](https://skills.sh/cmds-cc/skills/audiocodes-cli) | [audiocodes-cli](https://github.com/sieteunoseis/audiocodes-cli) | AudioCodes Mediant VE SBC — call stats, alarms, KPIs, health checks | [![npm](https://img.shields.io/npm/v/audiocodes-cli)](https://www.npmjs.com/package/audiocodes-cli) |

### Cisco UC

| Skill                                                                               | CLI Tool                                                                   | Description                                                             | npm                                                                                               |
| ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| [cisco-axl-cli](https://skills.sh/sieteunoseis/cisco-axl/cisco-axl-cli)             | [cisco-axl](https://github.com/sieteunoseis/cisco-axl)                     | CUCM configuration — phones, lines, route patterns, CSS, SIP trunks     | [![npm](https://img.shields.io/npm/v/cisco-axl)](https://www.npmjs.com/package/cisco-axl)         |
| [cisco-dime-cli](https://skills.sh/sieteunoseis/cisco-dime/cisco-dime-cli)          | [cisco-dime](https://github.com/sieteunoseis/cisco-dime)                   | CUCM log collection — SIP traces, SDL, audit logs                       | [![npm](https://img.shields.io/npm/v/cisco-dime)](https://www.npmjs.com/package/cisco-dime)       |
| [cisco-perfmon-cli](https://skills.sh/sieteunoseis/cisco-perfmon/cisco-perfmon-cli) | [cisco-perfmon](https://github.com/sieteunoseis/cisco-perfmon)             | Real-time performance counters — CPU, memory, call stats                | [![npm](https://img.shields.io/npm/v/cisco-perfmon)](https://www.npmjs.com/package/cisco-perfmon) |
| [cisco-risport-cli](https://skills.sh/sieteunoseis/cisco-risport/cisco-risport-cli) | [cisco-risport](https://github.com/sieteunoseis/cisco-risport)             | Device registration status — phone reg, CTI, trunk status               | [![npm](https://img.shields.io/npm/v/cisco-risport)](https://www.npmjs.com/package/cisco-risport) |
| [cisco-support-cli](https://skills.sh/sieteunoseis/cisco-support/cisco-support-cli) | [cisco-support](https://github.com/sieteunoseis/cisco-support)             | Cisco Support APIs — bugs, EoX, PSIRT, software, warranty               | [![npm](https://img.shields.io/npm/v/cisco-support)](https://www.npmjs.com/package/cisco-support) |
| [cisco-ucce-cli](https://skills.sh/sieteunoseis/cisco-ucce/cisco-ucce-cli)          | [cisco-ucce](https://github.com/sieteunoseis/cisco-ucce)                   | UCCE contact center — agent states, queue stats, diagnostics            | [![npm](https://img.shields.io/npm/v/cisco-ucce)](https://www.npmjs.com/package/cisco-ucce)       |
| [cisco-yang-cli](https://skills.sh/sieteunoseis/cisco-yang/cisco-yang-cli)          | [cisco-yang](https://github.com/sieteunoseis/cisco-yang)                   | IOS-XE voice gateways — RESTCONF, dial-peers, SIP trace                 | [![npm](https://img.shields.io/npm/v/cisco-yang)](https://www.npmjs.com/package/cisco-yang)       |
| [cisco-ise-cli](https://skills.sh/sieteunoseis/cisco-ise/cisco-ise-cli)             | [cisco-ise](https://github.com/sieteunoseis/cisco-ise)                     | Cisco ISE — endpoints, guests, RADIUS/TACACS monitoring                 | [![npm](https://img.shields.io/npm/v/cisco-ise)](https://www.npmjs.com/package/cisco-ise)         |
| [cisco-cdr-mcp](https://skills.sh/cmds-cc/skills/cisco-cdr-mcp)                     | [cisco-cdr-processor](https://github.com/sieteunoseis/cisco-cdr-processor) | CDR/CMR analysis — call search, trace, quality, statistics (MCP server) | —                                                                                                 |

### Genesys Cloud

| Skill                                                       | CLI Tool                                                   | Description                                                   | npm                                                                                                                       |
| ----------------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| [genesys-cli](https://skills.sh/cmds-cc/skills/genesys-cli) | [genesys-cli](https://github.com/sieteunoseis/genesys-cli) | Genesys Cloud CX — conversations, BYOC trunks, queues, agents | [![npm](https://img.shields.io/npm/v/@sieteunoseis/genesys-cli)](https://www.npmjs.com/package/@sieteunoseis/genesys-cli) |

### Infrastructure

| Skill                                                  | CLI Tool                                         | Description                                             | npm                                                                                 |
| ------------------------------------------------------ | ------------------------------------------------ | ------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| [ss-cli](https://skills.sh/sieteunoseis/ss-cli/ss-cli) | [ss-cli](https://github.com/sieteunoseis/ss-cli) | Delinea Secret Server — credential management, env sync | [![npm](https://img.shields.io/npm/v/ss-cli)](https://www.npmjs.com/package/ss-cli) |

### Windmill

| Skill                                                         | CLI Tool                                            | Description                                                                                     | Hooks                                                                                   |
| ------------------------------------------------------------- | --------------------------------------------------- | ----------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| [wmill-safety](https://skills.sh/cmds-cc/skills/wmill-safety) | [wmill](https://www.windmill.dev/docs/advanced/cli) | Windmill CLI safety guard — risk tiers, pre-flight checklists, destructive operation protection | [windmill-safety](https://github.com/cmds-cc/cmds.cc/tree/master/hooks/windmill-safety) |

### Community

| Skill | CLI Tool                                              | Description                                                | npm                                                                                                                 |
| ----- | ----------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| —     | [webex-cli](https://github.com/Cloverhound/webex-cli) | Webex Calling — users, locations, numbers, auto-attendants | [![npm](https://img.shields.io/npm/v/@cloverhound/webex-cli)](https://www.npmjs.com/package/@cloverhound/webex-cli) |

## Installation

Install the orchestrator (includes all skills in this repo):

```bash
npx @cmds-cc/skills add cmds-cc/skills
```

Or install individual skills from their source repos:

```bash
npx @cmds-cc/skills add sieteunoseis/cisco-axl          # cisco-axl-cli skill
npx @cmds-cc/skills add sieteunoseis/audiocodes-cli     # audiocodes-cli skill
npx @cmds-cc/skills add sieteunoseis/cisco-support      # cisco-support-cli skill
```

Install hooks (optional safety enforcement):

```bash
npx @cmds-cc/hooks add cmds-cc/cmds.cc/hooks/windmill-safety
```

## Skill Sync

Skills that live in this repo are the **canonical versions**. Individual CLI repos (like `cisco-axl`, `audiocodes-cli`) also ship their own skill copy for standalone use. To keep them in sync:

- **This repo** is the source of truth for orchestrator skills (`cisco-uc-engineer`)
- **Individual repos** are the source of truth for their CLI-specific skills (`cisco-axl-cli`, `audiocodes-cli`)
- Skills in this repo that mirror individual repos are synced manually when CLI tools get new features

## Giving Back

If you find these tools useful, consider supporting the project:

[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/automatebldrs)

## License

MIT
