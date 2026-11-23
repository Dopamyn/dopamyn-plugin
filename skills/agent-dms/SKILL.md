---
name: agent-dms
description: Set up a Claude-written Auto DM campaign on a Dopamyn Outreach Agent or a live list, and a scheduled Claude job that writes a DM, after research, for each new lead or new list member in the sending account's persona. Dopamyn sends them on the campaign's schedule. Use when the user wants Claude (not a template) to keep messaging an Outreach Agent's leads or a live list's new members, says "Claude DMs for my agent", "auto DM with context", or wants to change, pause or check such a job.
---

# Agent DMs: Claude writes each DM, Dopamyn sends it

Requires an active Dopamyn subscription. If a tool says one is required, stop and pass that on to the user.

What this sets up:
- One **Claude-written Auto DM campaign** on one source: an Outreach Agent (`launch_dm_campaign(drafted_by_claude=true, agent_id=...)`) or a live list (`launch_dm_campaign(drafted_by_claude=true, list_short_id=...)`). It starts empty. It holds what the user approves once: the sending accounts, the daily cap and the skip window. Dopamyn sends its DMs 3 minutes apart, at most the daily cap per UTC day, with the same cooldown, rate-limit, Block-list, contact and quota checks as every Auto DM campaign.
- One **scheduled Claude job** that runs the "Run prompt" below. Each run reads the people the campaign has not messaged yet (`get_dm_campaign(view="pending")`), asks which account sends to each (`get_dm_campaign(view="sender_preview")`), writes each DM in that account's persona, and adds it with `update_dm_campaign(action="add_people", drafts=[{handle, sender_handle, message}])`. People Claude chooses not to message go in `skips`. Claude never sends a DM itself.

This skill is for sources that keep growing. For a fixed group (a static list, or people the user names), write every DM in the chat instead and add them all at once: one-off campaigns place every draft across days at the daily cap.

Dopamyn checks every add: only people of the source are added (`skipped_not_in_source`), people already in the campaign or skipped before are left out, and people any workspace campaign messaged inside the skip window are skipped (`skipped_already_messaged`). A draft whose sender has no `accounts/<x_id>/dm-persona.md` is refused (`refused_persona_missing`).

## Setup (the user is here)

1. **Pick the source.**
   - Outreach Agent: `get_agents(kind="outreach", view="list")`. If the user did not name one, ask.
   - Live list: `list_my_lists`, and only a live (dynamic) list. A static list is a one-off campaign, not this skill.
   Then call `list_dm_campaigns` and, for each campaign with `drafted_by` "claude" that is not cancelled or finished, `get_dm_campaign(campaign_id=...)`: its `agent` or its list says what it writes for. If this source already has one, show it (state, daily_cap, room_today, sender_pool, counts) and go to "Change or stop".
2. **Look at the people.** Agent: `get_agents(kind="outreach", view="leads", agent_id=<it>, limit=10)`; show a few leads and `latest_signal` (why the agent found them). List: `get_list(list_short_id=<it>)`; show its size and a few members. If there is nobody yet, say so: the job adds nothing until people arrive.
3. **Pick the sending accounts.** Call `get_inbox_status(view="accounts")`. Offer the accounts that can send DMs and do not need a reconnect. Every sender needs X Premium. Keep their handles.
4. **Read each sender's persona.** Call `get_workspace_profile(account_handle=<sender>)` for each. Each DM is written in the voice of the account that sends it, from its own `dm-persona.md`; the workspace persona never stands in. If an account has none, offer `generate_account_dm_persona` (or `propose_profile_update` for `accounts/<x_id>/dm-persona.md` when it has too few typed DMs) with the user (show the before and after, and `decide_profile_update` only after their yes), or leave that account out. Write nothing for an account until its persona exists.
5. **Pick the daily cap and the skip window.**
   - Daily cap: the most DMs the campaign sends per UTC day, across all its senders (suggest 20, at most 500). Each sent DM spends one credit: the month's plan credits first, then purchased credits. Show `get_dm_usage` (its `credits` block) if the user asks what is left. If an add answers `does_not_fit`, nobody was added: show the user its numbers and options and never pick `if_not_fit` without their explicit choice.
   - Skip window: do not DM a person that any workspace campaign messaged ever (-1, the default), in the last 30 days or in the last 7 days.
   - For a list, now run `check_dm_reach(view="list", list_short_id=<it>, sender_handles=[...], skip_already_messaged_days=D)` with these senders and this window, and show the reach.
6. **Write the brief with the user.** Ask, one or two at a time, with an example answer each:
   - What do you offer these people, in one line? What is the one next step (a link, a call, a question)?
   - Anything to never say (prices, claims, competitor names)?
   The voice comes from each sender's persona, not from the brief. Keep the brief under 4,000 characters. Dopamyn does not store it: it goes into the scheduled job's prompt.
7. **Write 3-5 sample DMs.** Pick 3-5 people from step 2, follow "Research" and "Write" below for each, each in the persona of a sender from step 4 (spread the samples across the senders). Show each draft with the person, the sender and the one-line need. Change the brief until the user is happy. Add nothing and send nothing.
8. **Get the user's yes on one block.** Show, together: the source (agent or list) with the people count and a sample of them, the sending accounts, the daily cap, the start (now), the skip window, the gap (3 minutes), the brief, the 3-5 sample drafts, and this sentence with the cap filled in: "A scheduled Claude job writes and adds DMs for new people each run; Dopamyn sends them 3 minutes apart, at most N a day, until the campaign is paused." Show all drafts if the user asks. Go on only after the user agrees to that block.
9. **Launch the campaign.** Call `launch_dm_campaign(drafted_by_claude=true, agent_id=<it>` or `list_short_id=<it>, sender_handles=[...], daily_cap=N, skip_already_messaged_days=D, campaign_name="Claude DMs: <source name>")`. Leave out every other argument: the campaign starts empty, starts now, and the gap is 3 minutes. Add `priority=true` only when the user asked for this campaign to go before their other campaigns; tell them it pushes their other campaigns on the same accounts back. If it answers that another campaign watches the list, ask the user before calling again with `replace_existing_binding=true`. Keep `campaign_id`. If the result is uncertain, do not retry: call `list_dm_campaigns` first.
10. **Run it once now.** Follow the Run prompt below for this campaign, so today's room is used. The user approved the block in step 8, so add without asking per add. When the preview gives a sample person the sender of their approved sample, add the approved text unchanged.
11. **Create or update the schedule.** Ask how often (every hour is the default) and the time zone (ask, never assume). Use the scheduling tool this client has:
    - Claude Code: a cloud routine (`/schedule`, or the `RemoteTrigger` tool).
    - Claude desktop or claude.ai: a scheduled task (`create_scheduled_task`, or the client's equivalent).
    - If there is none, give the user the Run prompt to schedule by hand.
    Name it `Dopamyn Claude DMs: <source name>`. The prompt is the "Run prompt" block below with every placeholder filled in. Before you create one, list the existing ones: if a job with this name exists, update its prompt instead of making a second one.
12. **Hand over.** Tell the user: the source, the campaign and its link (`url`), what the first run added, the schedule and the next run time, the daily cap, and how to stop it: say "pause the Claude DMs for <source>" (see "Change or stop"), or pause the job in Claude.

## Change or stop

- Pause: `set_dm_campaign_status(action="pause", campaign_id=...)`. Queued DMs are held, and each scheduled run stops at once because the campaign reads `paused`. Safe to do at once when the user asks.
- Resume: `set_dm_campaign_status(action="resume", campaign_id=...)`. It sends the held DMs, so show the campaign and get the user's yes first.
- Stop for good: `cancel_dm_campaign(campaign_id=..., confirm_campaign=<its name>)` after the user agrees. Every unsent DM is cancelled. Then delete or pause the scheduled job.
- Senders: `update_dm_campaign(action="set_senders", campaign_id=..., sender_handles=[...])` after the user agrees. Nothing moves: unsent DMs on a removed account are cancelled (`cancelled_sender_removed`) and the next runs write those people again for a kept account. A new sender needs its own `dm-persona.md` first (setup step 4). The scheduled job reads the senders from the campaign on each run, so its prompt does not change.
- Brief or schedule: update the scheduled job's prompt. Dopamyn holds no copy.
- Daily cap, skip window or gap: these are fixed at launch. To change one, get the user's yes, cancel this campaign, launch a new one (setup steps 8-9), and put the new `campaign_id` in the scheduled job's prompt.
- Results: `get_dm_campaign(campaign_id=...)` gives the state, `room_today`, counts and per-sender counts; `view="recipients"` lists who got which DM. Replies arrive in the DM inbox like any campaign's (`get_dm_replies`).
- A Claude-written campaign has no shared message: edits of its message or variants and a rerun are refused (`claude_campaign_not_editable`). A DM sequence can follow it up; its follow-ups are fixed texts, and their Step 1 names are only `{{first_name}}`, `{{name}}` and `{{handle}}`. Once such a sequence is published, every person a run adds later gets its follow-ups too, with nobody present, and those DMs are not counted in the campaign's daily cap (they count in each account's daily limit): say so when the user approves the sequence.
- A live-list campaign whose list another campaign took over (`replace_existing_binding`) stops getting new people: `view="pending"` is empty with a note, and adds are refused (`list_watch_ended`). Do not retry. Tell the user; to keep messaging the list, they can launch a new campaign on it.

## Research (each person)

1. `latest_signal` on an agent's lead says why the agent found them (who they follow, mention, engage with, which event). This is the first reason to write. A list member has none: start from the next step.
2. `search_x_accounts(handle_contains=<handle>)`: bio, followers, location, Dope score, tags. `analyze_x_account(view="posts", handle=<handle>, sort_by="recent", limit=10)` for what they post this week. Both are Enterprise only. On a plan without them (Trial, Pro), do not call them: write from step 1 alone (the lead's `latest_signal` and name), and skip the person (`no_clear_need`) when the signal does not give a need.
3. Write one line, the need: "They are a [builder / creator / trader / thought leader / networker], right now [doing X], so the hook is [Y]." Name the product, event or people; "grow your audience" is not a need.
4. If you cannot write that line with confidence, skip the person (`no_clear_need`) and say why. A skipped person costs nothing; a vague DM is gone for good.
5. Skip a person whose tags or posts show a bot (`bot`), a giveaway or engagement farm (`giveaway`), or a scam pattern (`scam`).

## Write (each DM)

- Write in the sender's persona: its `dm-persona.md` sets the voice, and its 'Rules from the user' override everything below.
- Open with the reason from research (the signal or their current work), not with who we are.
- One plain line on what the brief offers, tied to their need.
- One next step from the brief: a question, a link or a call. Never two.
- Under 400 characters, plain text, no hashtags, no "hope you are well", no fake familiarity.
- Never claim what you could not check (their role, a launch, a meeting).
- Follow the brief's never-say list exactly.

## Run prompt

Fill the placeholders and use this block as the scheduled job's prompt. `<SOURCE>` is "the Outreach Agent <name>" or "the live list <name>".

```
You write the DMs of the Claude-written Dopamyn campaign <CAMPAIGN_ID> for <SOURCE> (skip window <SKIP_DAYS> days). Load the skill `agent-dms` and follow its "Research" and "Write" sections. If the skill is missing, use the brief and each sender's persona alone.

Brief:
<BRIEF>

The user approved this campaign's source, senders, daily cap, skip window and brief at setup, so add drafts without asking per add, but only as below. Dopamyn sends the DMs on the campaign's schedule; never call send_dm for these people. Add only people from view="pending", never more than room_today allows, and never retry an add whose result is uncertain.

1. Call get_dm_campaign(campaign_id="<CAMPAIGN_ID>"). Go on only when its state is "running", "queued", "on_cooldown" or "waiting_for_claude"; on any other state (paused, cancelled, finished, failed, unknown) report the state and stop. Room = room_today; if it is 0 or null, report and stop. If sender_pool is empty or null, report "no connected sender" and stop.
2. Find reachable people. Read get_dm_campaign(campaign_id="<CAMPAIGN_ID>", view="pending", limit=50, offset=0). If its note says the campaign no longer watches its list (list_watch_ended), report that and stop: do not retry. Otherwise call get_dm_campaign(campaign_id="<CAMPAIGN_ID>", view="sender_preview", handles=[that page's handles]); the preview is the reach check, so no check_dm_reach. Keep a person whose sender_handle is set and not in persona_missing. Leave out a person with an unreachable_reason or whose sender is in persona_missing, and never record them as skips: a skip blocks a person for good, and a reason like already_messaged can expire. Read the next page (offset 50, 100, ...) and preview it the same way, until you hold room kept people or a pending page comes back empty. Read at most 5 pending pages per run (about 250 people previewed): after the fifth, stop paging, write for the reachable people you hold and leave the rest to the next run. If persona_missing named a sender, report "@<sender> has no DM persona: add accounts/<x_id>/dm-persona.md in the workspace profile".
3. For each sender the preview named, call get_workspace_profile(account_handle=<sender>) once and use its dm-persona.md.
4. For each kept person, in order, until room is used:
   a. Research and write the DM per the skill, the brief and the persona of the sender the preview named. If you cannot name the need, or the person is a bot, giveaway or scam, make a skip {handle, reason: no_clear_need, bot, giveaway, scam or other, note} instead.
   b. Collect {handle, sender_handle, message}. When you have 5 drafts, or as many as room is left, call update_dm_campaign(action="add_people", campaign_id="<CAMPAIGN_ID>", drafts=[...], skips=[...]). Set room to the answer's room_today; if it is 0 or null, stop adding. Count the answer's added, every skipped_* field and every refused_* field. If the answer has held_until_resume true, the campaign was paused: stop adding. If refused_persona_missing names a sender, write nothing more for it.
   c. If the add is refused with campaign_senders_changed, nothing was added: call get_dm_campaign(campaign_id="<CAMPAIGN_ID>") again, check the state and room as in step 1, preview that batch again (step 2), rewrite a draft whose sender changed, and retry once; if it is refused again, report it and stop. On any other refusal, report the reason and stop. If the result is uncertain, do not retry: stop and report it.
5. Report (under 25 lines). First line: "Added A | skipped K | refused F | room left R". Then one line per person: @handle, sender, the need line, added / skipped / refused and why.
```

## Never

- Launch the campaign without the user's yes on the one block in setup step 8.
- Write a DM for a sender without its own `dm-persona.md`, or in the workspace persona instead.
- Call `send_dm` for these people: the campaign sends every DM.
- Add a person who is not in `view="pending"`, or more DMs in a run than `room_today` allows.
- Retry an add or a launch whose result is uncertain.
- Create a second scheduled job for the same source; update the one that exists.
