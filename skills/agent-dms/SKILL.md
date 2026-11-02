---
name: agent-dms
description: Set up a scheduled Claude job that writes a DM, after research, for each new lead of a Dopamyn Outreach Agent and adds it to a Claude-drafted Auto DM campaign, which sends them on its own schedule. Use when the user wants Claude (not a template) to message an Outreach Agent's leads, says "Claude DMs for my agent", "auto DM with context", or wants to change, pause or check such a job.
---

# Agent DMs: Claude writes each first DM, Auto DM sends it

Requires an active Dopamyn subscription. If a tool says one is required, stop and pass that on to the user.

What this sets up:
- One **Claude-drafted Auto DM campaign** for one Outreach Agent (`launch_dm_campaign(drafted_by_claude=true, agent_id=...)`). It starts empty. It holds what the user approves once: the sending accounts, the daily cap and the skip window. Dopamyn sends its DMs 3 minutes apart, at most the daily cap per UTC day, with the same cooldown, rate-limit, Block-list, contact and quota checks as every Auto DM campaign.
- One **scheduled Claude job** that runs the "Run prompt" below. Each run reads the agent's new leads, researches each person, writes one DM and adds it to the campaign with `update_dm_campaign(action="add_people", drafts=[...])`. Claude never sends the DM itself and never picks the sender: Dopamyn places each DM on one of the campaign's accounts.

Dopamyn checks every add: only leads of the agent are added (`skipped_not_agent_lead`), people already in the campaign are skipped, and people any workspace campaign messaged inside the skip window are skipped (`skipped_already_messaged`). Template Auto DM and AI-brief campaigns stay as they are; this is a separate kind of campaign.

## Setup (the user is here)

1. **Pick the agent.** Call `get_agents(kind="outreach", view="list")`. If the user did not name one, ask. Then call `list_dm_campaigns` and, for each campaign with `drafted_by` "claude" that is not cancelled or finished, `get_dm_campaign(campaign_id=...)`: its `agent` says which Outreach Agent it writes for. If this agent already has one, show it (state, daily_cap, room_today, sender_pool, counts) and go to "Change or stop".
2. **Look at the leads.** Call `get_agents(kind="outreach", view="leads", agent_id=<it>, limit=10)`. Show a few leads and `latest_signal` (why the agent found them). If the user wants only the leads of one lead filter list, read it with `get_agents(kind="outreach", view="list_leads", list_id=<it>)` (`view="overview"` lists them) and keep its `list_id`. If there are no leads, say so: the job adds nothing until the agent finds some.
3. **Pick the sending accounts.** Call `get_inbox_status(view="accounts")`. Offer the accounts that can send DMs and do not need a reconnect. Every sender needs X Premium. Keep their handles.
4. **Pick the daily cap and the skip window.**
   - Daily cap: the most DMs the campaign sends per UTC day, across all its senders (suggest 20, at most 500). Each sent DM uses one DM from the monthly allowance, then message credits. Show `get_dm_usage` if the user asks what is left.
   - Skip window: do not DM a person that any workspace campaign messaged in the last 7 days, 30 days (default) or ever (-1).
5. **Write the brief with the user.** Ask, one or two at a time, with an example answer each:
   - What do you offer these leads, in one line? What is the one next step (a link, a call, a question)?
   - Who is the message from (which person or brand voice)?
   - Tone, length, and anything to never say (prices, claims, competitor names).
   - If the skill `dopamyn-inbox-personas` is installed, load it and use the sending account's voice card. If not, the brief is the voice.
   Keep the brief under 4,000 characters: offer, next step, voice, never-say. Dopamyn does not store the brief: it goes into the scheduled job's prompt.
6. **Show three sample DMs.** Pick three leads from step 2 and follow "Research" and "Write" below for each. Show each draft with the lead and the one-line need. Change the brief until the user is happy with the style. Add nothing and send nothing.
7. **Get the user's yes on one block.** Show, together: the agent (and the lead filter list, if one), the sending accounts, the daily cap, the skip window, the gap (3 minutes), the start (now), the brief, and this sentence with the cap filled in: "A scheduled Claude job adds DMs for new leads; Dopamyn sends them 3 minutes apart, at most N a day." Go on only after the user agrees to that block.
8. **Launch the campaign.** Call `launch_dm_campaign(drafted_by_claude=true, agent_id=<it>, sender_handles=[...], daily_cap=N, skip_already_messaged_days=D, campaign_name="Claude DMs: <agent name>")`. Leave out every other argument: the campaign starts empty, starts now, and the gap is 3 minutes. Keep `campaign_id`. If the result is uncertain, do not retry: call `list_dm_campaigns` first.
9. **Create or update the schedule.** Ask how often (every hour is the default; leads arrive on the agent's own cadence) and the time zone (ask, never assume). Use the scheduling tool this client has:
   - Claude Code: a cloud routine (`/schedule`, or the `RemoteTrigger` tool).
   - Claude desktop or claude.ai: a scheduled task (`create_scheduled_task`, or the client's equivalent).
   - If there is none, give the user the Run prompt to schedule by hand.
   Name it `Dopamyn agent DMs: <agent name>`. The prompt is the "Run prompt" block below with every placeholder filled in. Before you create one, list the existing ones: if a job with this name exists, update its prompt instead of making a second one.
10. **Hand over.** Tell the user: the agent, the campaign and its link (`url`), the schedule and the first run time, the daily cap, and how to stop it: say "pause the Claude DMs for <agent>" (see "Change or stop"), or pause the job in Claude.

## Change or stop

- Pause: `set_dm_campaign_status(action="pause", campaign_id=...)`. Queued DMs are held, and each scheduled run stops at once because the campaign reads `paused`. Safe to do at once when the user asks.
- Resume: `set_dm_campaign_status(action="resume", campaign_id=...)`. It sends the held DMs, so show the campaign and get the user's yes first.
- Stop for good: `cancel_dm_campaign(campaign_id=..., confirm_campaign=<its name>)` after the user agrees. Every unsent DM is cancelled. Then delete or pause the scheduled job.
- Senders: `update_dm_campaign(action="set_senders", campaign_id=..., sender_handles=[...])` after the user agrees. Queued DMs move to the new accounts, and later adds use them. The scheduled job reads the senders from the campaign on each run (`sender_pool`), so its prompt does not change.
- Brief, schedule or lead filter list: update the scheduled job's prompt. Dopamyn holds no copy.
- Daily cap, skip window or gap: these are fixed at launch. To change one, get the user's yes, cancel this campaign, launch a new one (setup steps 7-8), and put the new `campaign_id` in the scheduled job's prompt.
- Results: `get_dm_campaign(campaign_id=...)` gives the state, `room_today`, counts and per-sender counts; `view="recipients"` lists who got which DM. Replies arrive in the DM inbox like any campaign's (`get_dm_replies`).
- A Claude-drafted campaign has no shared message: edits of its message or variants, a rerun, and a DM sequence that follows it up are refused (`claude_campaign_not_editable`).

## Research (each lead)

1. `latest_signal` on the lead says why the agent found them (who they follow, mention, engage with, which event). This is the first reason to write.
2. `search_x_accounts(handle_contains=<handle>)`: bio, followers, location, Dope score, AI tags. `analyze_x_account(view="posts", handle=<handle>, sort_by="recent", limit=10)` for what they post this week.
3. Write one line, the need: "They are a [builder / creator / trader / thought leader / networker], right now [doing X], so the hook is [Y]." Name the product, event or people; "grow your audience" is not a need.
4. If you cannot write that line with confidence, skip the lead and say why. A skipped lead costs nothing; a vague DM is gone for good.
5. Skip a lead whose tags or posts show a bot, a giveaway or engagement farm, or a scam pattern.

## Write (each DM)

- Open with the reason from research (the signal or their current work), not with who we are.
- One plain line on what the brief offers, tied to their need.
- One next step from the brief: a question, a link or a call. Never two.
- Under 400 characters, plain text, no hashtags, no "hope you are well", no fake familiarity.
- Never claim what you could not check (their role, a launch, a meeting).
- Follow the brief's never-say list exactly.

## Run prompt

Fill the placeholders and use this block as the scheduled job's prompt. Leave out the `<LIST_ID>` sentence when the user did not pick a lead filter list.

```
You write the DMs of the Dopamyn Outreach Agent "<AGENT_NAME>" (agent_id <AGENT_ID>) for its Claude-drafted Auto DM campaign (campaign_id <CAMPAIGN_ID>, skip window <SKIP_DAYS> days). Load the skill `agent-dms` and follow its "Research" and "Write" sections. If the skill is missing, use the brief alone.

Brief:
<BRIEF>

The user approved this campaign's agent, senders, daily cap, skip window and brief at setup, so add drafts without asking per add, but only as below. Dopamyn sends the DMs on the campaign's schedule; never call send_dm for these leads. Never add more than room_today allows, never add anyone who is not a lead of this agent, and never retry an add whose result is uncertain.

1. Call get_dm_campaign(campaign_id="<CAMPAIGN_ID>"). Go on only when its state is "running", "queued", "on_cooldown" or "waiting_for_claude"; on any other state (paused, cancelled, finished, failed, unknown) report the state and stop. Room = room_today; if it is 0 or null, report and stop. Senders = sender_pool; if it is empty or null, report "no connected sender" and stop.
2. Read the people already queued: get_dm_campaign(campaign_id="<CAMPAIGN_ID>", view="recipients", status="queued", limit=100), then the same with status="held", each page by offset=next_offset until next_offset is null. Keep their handles.
3. Read leads: get_agents(kind="outreach", view="leads", agent_id="<AGENT_ID>", limit=50), newest first. Only the leads of lead filter list <LIST_ID>: get_agents(kind="outreach", view="list_leads", list_id="<LIST_ID>", limit=50) instead. Drop a lead whose handle is in step 2's list or whose contact.last_job_id is <CAMPAIGN_ID> (already in this campaign), whose contact.state is "opted_out", or whose contact.last_contacted_at is inside the skip window (any time, when the window is -1). Read the next page (cursor=next_cursor) until you have room leads or 3 pages.
4. Check reach for all kept leads in one call: check_dm_reach(view="handles", handles=[their handles], sender_handles=<Senders>, skip_already_messaged_days=<SKIP_DAYS>). If reachable_after_skip is 0, report "no sender reaches them" and stop. If it equals handle_count, every lead is reachable: skip step 5a. If the counts are null (status "computing"), or lower, do step 5a.
5. For each kept lead, in order, until room is used:
   a. Only when step 4 says so: call check_dm_reach(view="person", account="all", person=<lead handle>). If reachable_by has none of Senders, skip the lead ("no sender reaches them").
   b. Research and write the DM per the skill and the brief. If you cannot name the need, skip the lead.
   c. Collect {handle, message}. When you have 5, or as many as room is left, call update_dm_campaign(action="add_people", campaign_id="<CAMPAIGN_ID>", drafts=[...]). Set room to the answer's room_today; if it is 0 or null, stop adding. Count the answer's added and every skipped_* field. If the answer has held_until_resume true, the campaign was paused: stop adding.
   d. If the add is refused with campaign_senders_changed, nothing was added: call get_dm_campaign(campaign_id="<CAMPAIGN_ID>") again, check the state and room as in step 1, and retry that batch once; if it is refused again, report it and stop. On any other refusal, report the reason and stop. If the result is uncertain, do not retry: stop and report it.
6. Report (under 25 lines). First line: "Added A | skipped K | room left R". Then one line per lead: @handle, the need line, added / skipped and why.
```

## Never

- Launch the campaign without the user's yes on the one block in setup step 7.
- Add a DM during setup.
- Call `send_dm` for an agent's lead: the campaign sends every DM.
- Add more DMs in a run than `room_today` allows.
- Add a person who is not a lead of the agent.
- Retry an add or a launch whose result is uncertain.
- Create a second scheduled job for the same agent; update the one that exists.
