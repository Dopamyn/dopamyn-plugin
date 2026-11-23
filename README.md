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

The plugin adds the Dopamyn connector (`https://mcp.dopamyn.ai/mcp`) and eight skills:

| Skill | Use it to |
|---|---|
| `daily-replies` | Triage replies to your DM campaigns and send approved answers |
| `competitor-leads` | Turn a competitor's audience into a list and a DM campaign |
| `keyword-leads` | Reach people posting about a keyword |
| `icp-builder` | Turn one sentence about your ideal customers into a tested ICP and an Outreach Agent that finds new ones every day |
| `outreach-agents` | Set up an always-on Outreach Agent from a recipe (for example your competitors' audience), and add, remove, pause or replace its Stream Agents |
| `weekly-report` | Write a weekly outreach report |
| `agent-dms` | Schedule a Claude job that researches each new lead of an Outreach Agent, or each new member of a live list, and writes its DM in the sending account's persona; a Claude-written Auto DM campaign sends them at your daily cap |
| `background-dms` | Set up a campaign Dopamyn researches and writes in the background: for each person it answers your questions from their profile and posts, leaves out people who do not qualify, and writes the DM in the sending account's persona. You tune the brief on sample people first; nothing to schedule in Claude. |

In a chat, and in a Claude-written campaign, Claude writes every DM itself, in the sending account's persona from your workspace profile (the workspace persona fills what it does not cover; a Claude-written campaign needs the account's own `accounts/<x_id>/dm-persona.md`). For 10 people or fewer you approve each exact message before it is sent. More people get a campaign: your own text or variants, or a Claude-written campaign that you approve once as one block (the people, the sending accounts, the daily cap, the start time and 3-5 sample DMs; ask to see every DM if you want). Dopamyn then schedules the DMs and applies the gap, daily cap, cooldown and skips. With `agent-dms`, the scheduled job adds a DM for each new lead or list member without asking per DM, up to the day's room. Every DM spends one credit (your monthly plan credits, then purchased credits), the same as a send from the Dopamyn app. When a campaign does not fit in your credits now, Claude shows you the numbers and the choices and waits for yours.

## Layout

| File | Read by |
|---|---|
| `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.mcp.json` | Claude Code |
| `plugin.json`, `mcp.json`, `assets/` | ChatGPT |
| `skills/*/SKILL.md` | Both |

## Contributing

This repository is a copy. Dopamyn publishes it from its main codebase on every release, so a change made here is overwritten by the next release. Report problems to https://dopamyn.ai.
