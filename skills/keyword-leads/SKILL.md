---
name: keyword-leads
description: Find people on X who are posting right now about a topic, token or problem the user's project solves, save them as a Dopamyn list, and DM them with a reference to what they said. Use when the user wants intent-based leads, people talking about a keyword, ticker or pain point.
---

# Keyword leads

Requires an active Dopamyn subscription. If a tool says one is required, stop and pass that on to the user.

First call `get_workspace_profile` (once per conversation): take keywords and the relevance brief from `icp.md` and `pitch.md`, and write the DM in the voice of `dm-persona.md` and the sending account's persona (`get_workspace_profile(account_handle=<the sender>)`), the workspace persona filling what the account's does not cover. A Claude-written campaign is stricter: each of its drafts needs the sender's own `dm-persona.md` (step 6). Its 'Rules from the user' override the targeting and drafting choices below. They never lift an approval: send, launch or approve only after the user's yes.

1. Agree on the search with the user.
   - `keywords`: 1-4 strong words or a ticker (for example "$ETH restaking" or "bridge fees").
   - `exact_phrase` true only for a fixed phrase.
   - `relevance_brief`: one sentence about who counts as a good lead. It filters out posts that only mention the words.
   - `lookback_days`: 7 unless the user says otherwise.
   - If the user wants the people who talk about an account rather than a topic (their own, or a competitor's), the deep search does that: `start_x_search(kind="mentions_of", mention_handles=[<the account>])` saves the authors of posts naming it, and `start_x_search(kind="mentioned_by", from_handles=[<the account>])` saves the accounts it names. Skip the quick look and go to step 4.
   - If the user wants the followers or the following of one account instead, ask the user first: `start_x_search(kind="followers", handle=<the account>)` (or `kind="following"`) is a paid fetch, reused free when one ran in the last 7 days. When `get_x_search(view="job", kind="followers", handle=<the account>)` is `done`, read the people with `get_x_search(view="accounts", kind="followers", handle=<the account>)` and save them with `create_list(name=<the name>, source="followers", handle=<the account>)`, adding `filters=<the result's list_filters>` when the read had filters. Then go to step 5.
2. Pick the depth with the user.
   - Quick look: `find_people_tweeting_about` reads about the 20 newest matching posts in one paid call and returns the authors with their posts.
   - Reach: `start_x_search` keeps up to 3,000 posts (`max_posts`) over a few minutes and saves the authors as a list. With a `relevance_brief` it reads more posts than it keeps, up to 3,000. Use it when the user wants more than a handful of people.
   Either way, run one search. Do not loop over many keywords. Ask the user before running a second search.
3. Quick look: call `find_people_tweeting_about` once. Show the authors with the post text and `post_url`. Drop accounts that are projects or bots if the user wants people. After the user agrees, call `create_list` with the kept handles.
4. Reach: first call `get_x_search(view="recent", kind="all")` to see whether the same search already ran. If not, call `start_x_search` once, tell the user it takes a few minutes, then call `get_x_search(view="job", job_id=<the job_id>)` every minute or two until `done`. The result's `list_short_id` is the saved list; there is nothing to `create_list`. Report `posts_scanned`, `authors_found` (`accounts_mentioned` for mentioned_by) and `stopped_reason`, show the handles in `author_preview` with their `post_url`, and give `list_url` for the full list.
5. Agree the sending accounts and the skip window with the user, then call `check_dm_reach(view="list", list_short_id=<the list>, sender_handles=[...], skip_already_messaged_days=D)` and show the reach and the sends left.
6. Pick how to DM them. One pipeline:
   - **10 people or fewer**: write each DM yourself, referring to the post they wrote, in the sending account's persona (the workspace persona fills what it does not cover). Show each exact message; after the user's yes on it, call `send_dm(action="new_chat", account=<the sender>, to_handle=<the person>, message=<the approved text>)`, one person at a time.
   - **More than 10, with fixed text, a template or variants from the user**: a fixed-message campaign, `launch_dm_campaign(list_short_id=<the list>, message=<the text>, message_variants=[...])`. A campaign sends the same text to everyone, so refer to the topic, not to one person's post.
   - **More than 10 otherwise**: a Claude-written campaign. For a live list that should keep messaging people who join it later, use the `agent-dms` playbook. For a static list:
     - The senders and the skip window are the ones of step 5. Agree the daily cap (20 when the user has no view) and the start time.
     - Read each sender's persona with `get_workspace_profile(account_handle=<the sender>)`. A campaign draft needs the sender's own `dm-persona.md`, with no workspace fallback: for a sender without one, offer `generate_account_dm_persona` (or `propose_profile_update` for `accounts/<x_id>/dm-persona.md` when it has too few typed DMs) with the user, or leave that sender out, and write nothing for it until it exists.
     - Write 3-5 sample drafts for people on the list, each in a sender's persona, and get the user's yes on one block: the people count and a sample of them, the senders, the daily cap, the start time and the sample drafts. Show all drafts if the user asks. That yes covers the launch and the adds below of the list's people, with drafts like the samples shown; a person who is not on the list needs a new yes.
     - Call `launch_dm_campaign(drafted_by_claude=true, list_short_id=<the list>, sender_handles=[...], daily_cap=N, start_at=<the start>, skip_already_messaged_days=D, campaign_name=<the name>)`. It starts empty.
     - Read a page of people with `get_dm_campaign(campaign_id=<it>, view="pending", limit=100, offset=<the people left out so far>)` and call `get_dm_campaign(campaign_id=<it>, view="sender_preview", handles=[...])` for them. The preview is the reach check for drafts: no separate `check_dm_reach(view="handles")`. Leave out people with an `unreachable_reason` and people whose sender is in `persona_missing` (offer `generate_account_dm_persona` as above). Never record them as skips: a skip blocks a person for good, and a reason like `already_messaged` can expire.
     - Write one DM per kept person in the persona of the sender the preview named. When the preview gives a sample person the sender of their approved sample, use the approved text unchanged. Add them with `update_dm_campaign(action="add_people", campaign_id=<it>, drafts=[{handle, sender_handle, message}], skips=[...])`, up to 100 per call. Put people you choose not to message (no clear need, a bot, a giveaway, a scam) in `skips`. Added and skipped people leave the pending read, so the next page's offset is the number of people you left out so far, counting people whose draft was refused.
     - Repeat until a pending page comes back empty, reading at most 5 pending pages: after the fifth, stop and ask the user before reading more. A run without the user present (a scheduled job) reads at most 5 pending pages, writes for the reachable people it holds and stops. Dopamyn spreads the DMs across days at the daily cap; `last_send_day` says when the last one sends. Report any `refused_*` answer to the user.
   - For a short set of handles from the quick look that the user does not want saved as a list, a fixed-message campaign takes `handles` instead of a list (check them first with `check_dm_reach(view="handles", handles=[<the handles>])`). A Claude-written campaign takes no list at all: after the one block, `launch_dm_campaign(drafted_by_claude=true, sender_handles=[...], daily_cap=N, ...)`, then `get_dm_campaign(campaign_id=<it>, view="sender_preview", handles=[<the handles>])` as the reach check, a draft per reachable person in the named sender's persona, and `update_dm_campaign(action="add_people", drafts=[{handle, sender_handle, message}])`. Never create a list the user did not ask for.
7. Before each `send_dm` or `launch_dm_campaign`, get the user's explicit approval: of each exact message for `send_dm`; of the text, the people, the senders, the start and the pace for a fixed-message campaign; of the one block for a Claude-written campaign.
8. Report what was sent or scheduled.

Rules:
- Never send or launch without the user's approval (step 7).
- Never send in a loop: more than 10 people get a campaign.
- If a result is uncertain, do not retry.
