---
name: competitor-leads
description: Turn a competitor's audience into a Dopamyn DM campaign - find people who engage with or follow competitors, filter them, save a list, and launch a DM campaign after approval. Use when the user wants leads from competitors, "steal" a competitor's community, or reach people in their category who do not know them yet.
---

# Competitor leads

Requires an active Dopamyn subscription. If a tool says one is required, stop and pass that on to the user.

First call `get_workspace_profile` (once per conversation): take competitors from `competitors.md` first, then the saved ones from `get_competitors`; judge leads against `icp.md`; and write the DM in the voice of `dm-persona.md` and the sending account's persona (`get_workspace_profile(account_handle=<the sender>)`), the workspace persona filling what the account's does not cover. A Claude-written campaign is stricter: each of its drafts needs the sender's own `dm-persona.md` (step 6). Its 'Rules from the user' override the targeting and drafting choices below. They never lift an approval: send, launch or approve only after the user's yes.

1. Find out which competitor to target.
   - If the user named one, use it.
   - If not, call `get_competitors(view="list")` for the saved competitors. `get_competitors(view="compare")` sets their reach and posting next to the user's own.
   - If none are saved, or the user names one that is not, ask the user first, then call `update_competitors(action="add", handles=[<the accounts>])`. Each added account starts a paid follower fetch, so its followers can take a while to appear.
   - To read a saved Common Followers list instead of the competitors, call `find_competitor_audience(view="watchlists")` and pass its `watchlist` value to the `followers`, `mentioners` and `presets` views only. The `engagers` view and `start_x_search` do not take it.
   - To see which audience is worth it, call `find_competitor_audience(view="presets")` once. It counts each audience card of the Competitors page (investors, creators, people who work in crypto, ...).
2. Collect the leads. Use one or more:
   - `find_competitor_audience(view="engagers", competitor_handle=<the competitor>, days=30, exclude_my_followers=true)`. These people talk to the competitor, so they are the warmest.
   - `find_competitor_audience(view="mentioners", days=30, exclude_my_followers=true)`. These people mention several competitors.
   - `find_competitor_audience(view="followers", exclude_my_followers=true)`. These people follow several competitors. Pass `preset` for one audience card.
   - For the people who reposted, quoted or replied to a few of the competitor's posts: ask the user first, because it is a paid job, then call `start_x_search(kind="engagement", post_urls=[<the post links>])` and follow it with `get_x_search(view="job", kind="engagement", job_id=<the job_id>)` until `done`. The job saves the people as a list: go to step 5 with its `list_short_id`.
3. Filter the leads with the user's goal in mind. Filter in the call where you can: the same filters as `search_x_accounts` (`entity_types`, `professions`, `ranges`, `exclude`, ...) keep the count honest.
   - Drop accounts with fewer than 100 followers unless the user wants small accounts.
   - Drop obvious projects, bots and news accounts if the user wants people.
   - Show the user the top 20 with the `why` line, and the count you kept.
4. After the user agrees, save the list with a clear name, for example "Rival engagers - Oct 2026":
   - the kept handles: `create_list(name=<the name>, handles=[<the kept handles>])`;
   - or, from a `followers` search over the saved competitors (no `watchlist`, or `watchlist="competitors"`), everyone it matches, not only the rows shown: `create_list(name=<the name>, source="competitor_followers", filters=<the result's list_filters>)`. `list_filters` is null for a named list and for `mentioners`: save those results from the kept handles. Add `list_type="dynamic"` for a live list that picks up new matches. Tell the user which one you save.
5. Agree the sending accounts and the skip window with the user, then call `check_dm_reach(view="list", list_short_id=<the new list>, sender_handles=[...], skip_already_messaged_days=D)`. Show:
   - reachable people, people on the Block list, people already messaged
   - sends left this month (`credits.left`)
6. Pick how to DM them. Mention the shared interest (the competitor's category), never the competitor by name in a negative way; under 500 characters, one clear ask, no links in the first message unless the user insists. One pipeline:
   - **10 people or fewer**: write each DM yourself, referring to what they engage with, in the sending account's persona (the workspace persona fills what it does not cover). Show each exact message; after the user's yes on it, call `send_dm(action="new_chat", account=<the sender>, to_handle=<the person>, message=<the approved text>)`, one person at a time.
   - **More than 10, with fixed text, a template or variants from the user**: a fixed-message campaign, `launch_dm_campaign(list_short_id=<the list>, message=<the text>, message_variants=[...])`. Use `{{first_name}}` for a personal greeting.
   - **More than 10 otherwise**: a Claude-written campaign. For a live list that should keep messaging people who join it later, use the `agent-dms` playbook. For a static list:
     - The senders and the skip window are the ones of step 5. Agree the daily cap (20 when the user has no view) and the start time.
     - Read each sender's persona with `get_workspace_profile(account_handle=<the sender>)`. A campaign draft needs the sender's own `dm-persona.md`, with no workspace fallback: for a sender without one, offer `generate_account_dm_persona` (or `propose_profile_update` for `accounts/<x_id>/dm-persona.md` when it has too few typed DMs) with the user, or leave that sender out, and write nothing for it until it exists.
     - Write 3-5 sample drafts for people on the list, each in a sender's persona, and get the user's yes on one block: the people count and a sample of them, the senders, the daily cap, the start time and the sample drafts. Show all drafts if the user asks. That yes covers the launch and the adds below of the list's people, with drafts like the samples shown; a person who is not on the list needs a new yes.
     - Call `launch_dm_campaign(drafted_by_claude=true, list_short_id=<the list>, sender_handles=[...], daily_cap=N, start_at=<the start>, skip_already_messaged_days=D, campaign_name=<the name>)`. It starts empty.
     - Read a page of people with `get_dm_campaign(campaign_id=<it>, view="pending", limit=100, offset=<the people left out so far>)` and call `get_dm_campaign(campaign_id=<it>, view="sender_preview", handles=[...])` for them. The preview is the reach check for drafts: no separate `check_dm_reach(view="handles")`. Leave out people with an `unreachable_reason` and people whose sender is in `persona_missing` (offer `generate_account_dm_persona` as above). Never record them as skips: a skip blocks a person for good, and a reason like `already_messaged` can expire.
     - Write one DM per kept person in the persona of the sender the preview named. When the preview gives a sample person the sender of their approved sample, use the approved text unchanged. Add them with `update_dm_campaign(action="add_people", campaign_id=<it>, drafts=[{handle, sender_handle, message}], skips=[...])`, up to 100 per call. Put people you choose not to message (no clear need, a bot, a giveaway, a scam) in `skips`. Added and skipped people leave the pending read, so the next page's offset is the number of people you left out so far, counting people whose draft was refused.
     - Repeat until a pending page comes back empty, reading at most 5 pending pages: after the fifth, stop and ask the user before reading more. A run without the user present (a scheduled job) reads at most 5 pending pages, writes for the reachable people it holds and stops. Dopamyn spreads the DMs across days at the daily cap; `last_send_day` says when the last one sends. Report any `refused_*` answer to the user.
7. Agree the schedule: start now or at `start_at`, and the gap between sends (`minutes_between_sends`; the app uses 5 minutes). If you saved a live (dynamic) list, ask whether the campaign should also message people who join it later (`keep_in_sync`); it then keeps sending until paused. A static list cannot take `keep_in_sync`.
8. For a fixed-message campaign, show the final message, list, sender account, start, pacing, and number of DMs; for a Claude-written campaign, the one block of step 6. Get explicit approval.
9. Call `launch_dm_campaign` (then, for a Claude-written campaign, write and add the drafts as in step 6). Each DM spends one credit. If the result carries `does_not_fit`, nothing was sent: show the user how many fit now (`fit_now`), how many fit before the plan ends, the credits left and the options (send only to those that fit now, spread across months, launch anyway with the rest waiting for credits, or buy credits at `buy_credits_url`). Never pick one yourself: call again with `if_not_fit` set to the option the user chose. If it says another campaign already watches the list, ask the user before calling again with `replace_existing_binding=true`.
10. Report the `campaign_id`, the scheduled count, and the skips. Tell the user they can say "pause it" at any time: that is `set_dm_campaign_status(action="pause", campaign_id=<the campaign>)`.

Rules:
- Never send or launch without approval of the exact message (or the one block) and the recipient count.
- Never send in a loop: more than 10 people get a campaign.
- Never add a competitor or start a paid job without asking the user first.
- If a launch result is uncertain, do not retry. Call `list_dm_campaigns(view="all")` to check first.
