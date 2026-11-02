# Dopamyn plugin

Connects Claude Code (and ChatGPT) to your Dopamyn workspace: search X accounts, find leads from competitors, events and keywords, save lists, launch and report on DM campaigns, and triage replies.

## Requirements

- An active **Dopamyn subscription**. Without one, sign-in works but every tool that reads or changes your workspace is refused with "An active Dopamyn subscription is required to use the MCP connector". Subscribe at https://dopamyn.ai.
- A Dopamyn account to sign in with. There is no API key: the plugin signs in with OAuth the first time you use it.

## Install in Claude Code

```
/plugin marketplace add Dopamyn/dopamyn-plugin
/plugin install dopamyn@dopamyn
```

Then run `/mcp`, select the Dopamyn server that the plugin added, and sign in with your Dopamyn account.

Updates do not run automatically: run `/plugin marketplace update dopamyn`, or turn on auto-update for the marketplace in `/plugin`.

## What you get

The plugin adds the Dopamyn connector (`https://mcp.dopamyn.ai/mcp`) and seven skills:

| Skill | Use it to |
|---|---|
| `daily-replies` | Triage replies to your DM campaigns and send approved answers |
| `competitor-leads` | Turn a competitor's audience into a list and a DM campaign |
| `event-leads` | Reach hosts, sponsors and speakers of crypto events |
| `keyword-leads` | Reach people posting about a keyword |
| `icp-builder` | Turn one sentence about your ideal customers into a tested ICP and an Outreach Agent that finds new ones every day |
| `weekly-report` | Write a weekly outreach report |
| `agent-dms` | Schedule a Claude job that researches each new lead of an Outreach Agent and writes its DM; a Claude-drafted Auto DM campaign sends them at your daily cap |

Nothing is sent until you approve the exact message and the recipients. The one exception is `agent-dms`: you approve the Outreach Agent, the sending accounts, the daily cap, the skip window and the brief once, with three sample DMs, and the scheduled job then adds a DM for each new lead without asking per DM. Every DM uses the same monthly allowance and message credits as a send from the Dopamyn app.

## Layout

| File | Read by |
|---|---|
| `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.mcp.json` | Claude Code |
| `plugin.json`, `mcp.json`, `assets/` | ChatGPT |
| `skills/*/SKILL.md` | Both |

## Contributing

This repository is a copy. Dopamyn publishes it from its main codebase on every release, so a change made here is overwritten by the next release. Report problems to https://dopamyn.ai.
