---
name: outreach-agents
description: Set up or change an always-on Dopamyn Outreach Agent - pick a recipe (for example reach the competitors' audience), plan its Stream Agents, lead filters and lead prompt, build it, and turn it on after approval; or add, remove, pause and replace the Stream Agents of an agent that runs. Use when the user wants leads to keep coming in every day, names a playbook like "reach my competitors' audience", or asks to change what an Outreach Agent watches.
---

# Outreach Agents

Requires an active Dopamyn subscription. If a tool says one is required, stop and pass that on to the user.

An Outreach Agent finds leads every day. Each Stream Agent runs one signal on one input (an account, a list, the competitors list, keywords). Its Lead filters (filter fields and a lead prompt, which an AI checks on each new lead) keep only the people the user wants. Auto DM on its lead list messages them.

First call `get_workspace_profile` (once per conversation): `icp.md` says who fits, and its 'Rules from the user' override the targeting below.

## Recipes

Use a recipe when the user's words match it, then adjust it to what they say.

- **Reach the competitors' audience** ("my competitors' audience", "people around my competitors"): Stream Agents on the competitors list: `new_followers_of` (paid, people who start following a competitor), `engagers_of` (paid, people who reply to, quote or repost a competitor), and `followed_by` (free, accounts a competitor starts following) when the user wants partners, hires or investors. The Lead filters and the lead prompt say who of them to keep. Check the competitors first with `get_competitors(view="list")`; with none saved, ask the user for the accounts and add them with `update_competitors(action="add", handles=[...])` after their yes.
- **Who they are** (a role, a sector, a place): see the `icp-builder` playbook.

## Set up a new agent

1. Call `create_agent(kind="outreach", action="plan", prompt=<the user's words>)`. It creates nothing. The plan has `agents`, `lead_filter`, `lead_prompt`, `questions` and `warnings`.
2. Ask the plan's questions. When an answer changes the plan, plan again with the answer in the prompt.
3. Show the user the plan in plain words: each Stream Agent with its `why` and whether it is paid, the Lead filters and the lead prompt. Change what the user asks.
4. After the user's yes, call `create_agent(kind="outreach", action="create", name=<a short name>, plan=<the plan's data>)`. The agent is built inactive. Its Lead filters and lead prompt are saved, and its lead list is made.
5. Call `set_agent_status(kind="outreach", action="activate", agent_id=<it>)`. On a plan that charges, show `charges.terms`, and pass `accept_charges=charges.confirm_code` only after the user agrees.
6. To DM the leads, use the `agent-dms` or `background-dms` playbook on the agent's lead list (`lead_filter.short_id`).

## Change an agent

1. Call `get_agents(kind="outreach", view="overview", agent_id=<it>)` to see its Stream Agents (with their `id`) and its Lead filters.
2. Call `create_agent(kind="outreach", action="plan", agent_id=<it>, prompt=<what the user wants changed>)`. The planner sees what runs now. Its plan holds new `agents` to add and `changes` to the running ones: `{source_id, action: remove | pause | resume, name, why}`.
3. Show the user each change and each new Stream Agent. Remove deletes a Stream Agent (the leads it found stay); pause keeps it to start later.
4. After the user's yes:
   - remove: `update_agent(kind="outreach", action="remove_source", agent_id=<it>, source_id=<id>)`;
   - pause: `update_agent(kind="outreach", action="update_source", agent_id=<it>, source_id=<id>, spec={"bank_status": "paused"})`;
   - add: `create_agent(kind="outreach", action="create", agent_id=<it>, plan=<the plan's data>)`. It adds them paused and saves a new lead prompt.
   - resume, and start what was added: `set_agent_status(kind="outreach", action="activate", agent_id=<it>)`, with the charges rule of step 5 above.
5. A change to the Lead filters of a list that a live Auto DM campaign watches asks a question: pass it to the user (`update_agent(action="update_lead_filter")`).

Rules:
- Never add, remove or turn on a Stream Agent without the user's yes on the exact change.
- Paid Stream Agents spend credits on each run. Say which ones are paid before the user agrees.
- If a result is uncertain, do not retry. Read the agent with `view="overview"` first.
