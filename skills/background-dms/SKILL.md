---
name: background-dms
description: Set up an Auto DM campaign that Dopamyn researches and writes in the background. For each person Dopamyn reads their profile and recent posts, answers the questions the user cares about (for example "are they building on Jupiter?"), leaves out people who do not qualify, and writes a DM in the sending account's voice that opens with what the person is working on. No scheduled Claude job is needed. Use when the user wants personalized DMs that keep going on their own for a big list, a live list or an Outreach Agent's leads, says "have Dopamyn write the DMs", "research each person first", "qualify and DM my leads in the background", or wants to change, check or stop such a campaign.
---

# Background DMs: Dopamyn researches and writes each DM

Requires an active Dopamyn subscription. If a tool says one is required, stop and pass that on to the user.

What this sets up: one **campaign Dopamyn writes** (`launch_dm_campaign(drafted_by_dopamyn=true, brief=...)`). About 30 minutes before each person's DM is due, Dopamyn:

1. reads the person's profile and recent posts and follows the brief's `research_instructions`: it answers the user's questions, decides to write or to leave the person out, and picks openers that quote the person's own words;
2. leaves out a person who does not meet a stated requirement. They get no DM and spend no credit;
3. writes the DM from the brief's `offer` and `goal`, in the sending account's own DM persona, opening with what research found;
4. queues it, or holds it for a person when its quality check is unsure.

You set it up with the user, show samples, and check on it. You do not write these DMs and you run no scheduled job.

## Which way to write a campaign

- The user gave the text: a fixed-message campaign (`message`).
- The user wants you to write each DM and see it before it is added, or the group is small: a Claude-written campaign (`drafted_by_claude`, playbook `agent-dms`).
- The user wants it to research, qualify and write on its own, with nothing to schedule in Claude: this playbook.

If it is not clear which the user wants, ask once: "Should I write each DM here so you see every one, or should Dopamyn research and write them in the background?"

## Setup (the user is here)

1. **Pick the people.**
   - A saved list: `list_my_lists`, then `get_list(list_short_id=<it>)` for its size and a few members.
   - An Outreach Agent's leads: `get_agents(kind="outreach", view="list")`, then `view="overview"` for the agent's lead filter list (a live saved list of its leads). The campaign watches that list (`keep_in_sync`), so pass the list as `list_short_id`. `agent_id` is for Claude-written campaigns only. If the agent has no lead filter list, `update_agent(action="add_lead_filter")` adds one after the user agrees.
   - Handles the user names (up to 500).
2. **Pick the sending accounts.** `get_inbox_status(view="accounts")`. Every sender needs X Premium and DM access. Each DM is written in the voice of the account that sends it: call `get_workspace_profile(account_handle=<sender>)` and check it has a `dm-persona.md`. If it has none, offer `generate_account_dm_persona` first; without one the DM is written in a plain voice.
3. **Write the brief with the user.** Ask one or two at a time, with an example answer each:
   - `offer`: what do you offer these people, in a line or two?
   - `goal`: the one next step (a reply, a link, a question). A first DM that asks a stranger for a call is held for review, so prefer "worth a look?" over "can we hop on a call?".
   - `research_instructions`: what should Dopamyn find out about each person, who should it leave out, and what should the DM open with? Examples:
     - "Only message people who are building in the Jupiter ecosystem. Open with what they shipped most recently."
     - "Is this person a founder or an employee? Leave out agencies and job seekers. Mention the product they are building by name."
     - "Find out if they posted about prediction markets in the last month. If not, leave them out."
   Optional: `who_you_are` (one line), `ideal_customer`, `tone`. Read `get_workspace_profile` first and do not ask for what `about.md` and `pitch.md` already say: Dopamyn reads those files too.
   Write requirements as requirements ("only message people who ..."). A question alone ("are they on Jupiter?") is answered but leaves nobody out.
4. **Preview on real people.** Call `preview_dm_drafts(handles=[3 of the people], brief=<the brief>, sender_handle=<a sender>)`. Show the user, per person: the decision (write or left out) with its reason, the answers, the opener with the person's own words, and the draft. A call takes up to a minute. Preview 3-5 people in all (two calls), and include one you expect to be left out if the brief has a requirement.
   - If a person is left out wrongly, or kept wrongly, change `research_instructions` and preview again.
   - If a draft reads wrong, change `offer`, `goal` or `tone`. The voice itself comes from the sender's persona, not the brief.
   - `status` "needs_review" means the campaign would hold that DM for a person. Fix the brief until most samples are "ready".
   - If research did not finish for someone, call again for them: what was found is kept.
5. **Check the reach.** For a list: `check_dm_reach(view="list", list_short_id=<it>, sender_handles=[...], skip_already_messaged_days=D)`. For handles: `check_dm_reach(view="handles", ...)`. Show the reachable count and the sends left (`get_dm_usage` if the user asks what is left).
6. **Get the user's yes on one block.** Show together: the people (count and a sample), the sending accounts, the start and the gap between sends, the skip window, the whole brief with its `research_instructions`, the 3-5 samples from step 4, and these two sentences: "Dopamyn researches each person and writes their DM about 30 minutes before it sends. You will not see each DM first; ones its quality check is unsure of wait for your review." For a live list or an agent's lead list with `keep_in_sync`, add: "It keeps messaging new people until it is paused." Go on only after the user agrees to that block.
7. **Launch.** `launch_dm_campaign(drafted_by_dopamyn=true, brief=<the brief>, list_short_id=<it>` or `handles=[...], sender_handles=[...], skip_already_messaged_days=D, campaign_name="<name>")`, with `keep_in_sync="new_only"` or `"current_and_new"` for a live list. If it answers that research is not available right now, nothing was launched: tell the user, and offer to launch without `research_instructions` or to try later. If it answers `does_not_fit`, show the numbers and let the user choose. If the result is uncertain, do not retry: call `list_dm_campaigns` first.
8. **Hand over.** Tell the user the campaign and its link (`url`), how many DMs are scheduled, when the first is due, and how to check on it or stop it (below).

## Check on it

- `get_dm_campaign(campaign_id=...)`: state, counts, the brief, and `research_counts` (researched, write, left_out).
- `get_dm_campaign(view="research")`: what was found per person, with the answers to the user's questions. `decision="left_out"` lists who was left out and why. This is also the answer to "which of my leads are on Jupiter?" for the people researched so far.
- `get_dm_campaign(view="recipients", status="writing")`: people whose DM is not written yet. `status="skipped"` shows who got no DM and why.
- Drafts held for review: `list_campaign_drafts_for_review(campaign_id=...)`. Show each draft, the person and why it is held. `review_campaign_draft(action="approve")` sends a real DM: get the user's yes for that one draft, with the exact final text. Never approve in a loop. A held draft that nobody reviews expires and is not sent.
- Replies arrive like any campaign's: `get_dm_replies`.

## Change or stop

- Change what it looks for or writes: `update_dm_campaign(action="edit_brief", brief={<only the fields to change>})`. Preview the change first with `preview_dm_drafts` and get the user's yes. People whose DM is not written yet follow the new brief; DMs already written or sent keep their text. Name only the fields you change: one left out keeps its value. `research_instructions=""` turns research off, so never send it empty unless the user asked for that.
- Pause: `set_dm_campaign_status(action="pause")`. Safe to do at once when the user asks. Resume sends the held DMs, so get the user's yes first.
- Stop for good: `cancel_dm_campaign` after the user confirms the campaign by name.
- Senders, the people of a campaign, or a rerun: these are changed in the Dopamyn app for a campaign Dopamyn writes (`campaign_app_only`).

## A one-off question about a few people

To answer "is @x building on Jupiter?" for one to three accounts without a campaign, call `preview_dm_drafts` with the user's `offer` (a brief needs something to write from) and `research_instructions` holding the question, and read `research.answers`. For a quick look at one account you can also read it yourself with `analyze_x_account(view="posts")`.

## Never

- Launch without the user's yes on the one block in step 6.
- Launch on `research_instructions` you have not previewed on real people.
- Promise that every DM is reviewed: only held drafts are.
- Approve held drafts in a loop, or without showing the exact text.
- Retry a launch whose result is uncertain.
