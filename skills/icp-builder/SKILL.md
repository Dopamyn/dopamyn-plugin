---
name: icp-builder
description: Turn one plain sentence, for example "we want RWA traders", into an ICP (ideal customer profile) and an Outreach Agent that finds new ICP people every day - test the ICP first with one-off searches (who they are, whom they follow, whom they engage with), merge them and rank people by how many searches found them, then build the agent from the searches that worked, with a lead filter an Auto DM campaign can watch. Use when the user describes the people they want to reach and has no ICP yet, or when get_icp has nothing.
---

# ICP builder

The goal is an Outreach Agent that keeps finding the ICP. Steps 1-8 test the ICP on today's data. Step 10 builds the agent from the searches that worked.

First call `get_workspace_profile` (once per conversation): start the ICP card from `icp.md` and its example accounts, and follow its 'Rules from the user' for whom to target (they never lift the user's yes before a paid search or any launch). If `icp.md` is missing, build from the user's sentence as below.

One search alone is noisy. A person who is in two or three searches is a strong match. So: run each search, save it as a static list, merge the lists, and keep the people in the most lists.

1. Fill the ICP card from the user's sentence. Show it to the user and get a yes before you search.
   - Who (role): `ai_roles` (for example `trader`, `founder`, `developer`), or `tags`, `categories`, `professions`.
   - Stricter who: `ai_traits` (for example `whale`, `technical-analyst`), `ranges.trading_posts` (posts that name a ticker: the best proof that someone trades).
   - Topic: `ai_domains` (for example `rwa`, `defi`, `perps-derivatives`).
   - Topic words: 3-5 words for `text.summary_word` (for example "RWA", "tokenized", "tokenization").
   - Seed accounts: 10-30 projects that only this ICP follows. Leave out big chains, exchanges and TradFi names (for example binance, arbitrum, blackrock): everyone follows them.
   - Size: `min_followers`, `min_smart_followers`.
   - Not these: `entity_types=["person"]`, `exclude.ai_roles=["bot-account"]`, `max_following=3000` (drops mass-followers), `only=["active"]`.
   - Do not use `asset_classes` for people. It describes projects.
2. Check the seeds. Call `search_x_accounts(handles=[<the seeds>])`. A seed that is not in the Directory cannot go in `follows` or `engaged_with`: drop it from those searches and tell the user.
3. Run the free searches. For each one, show the count and the top 10, then ask whether to keep it.
   - **A. Who + topic:** `search_x_accounts(ai_roles=[<role>], ai_domains=[<topic>], entity_types=["person"], exclude={"ai_roles": ["bot-account"]})`. This is usually the most precise search.
   - **A2. Who + topic words (wider):** the same, with `text={"summary_word": [<the topic words>]}` in place of `ai_domains`, and `"ai_domains": [<topic>]` added to `exclude`. A2 then holds only the people A does not have. Without the exclude, most of A is also in A2, and the merge counts one signal twice.
   - **B. Who they follow:** with a saved Common Followers list of the seeds (`find_competitor_audience(view="watchlists")` lists them), call `find_competitor_audience(view="followers", watchlist=<its value>, tags=["traders"], entity_types=["person"], min_competitors_followed=5, ranges={"smart_followers": {"min": 100}, "following": {"max": 3000}}, sort_by="smart_followers", limit=50)`. A named list gives only its top 42 people and no `list_filters`, so save B from the handles shown. Without such a list, use `search_x_accounts(follows=[<the seeds>], ai_roles=[<role>])`, but `follows` matches people who follow any one seed, so it is wide.
   - **C. Who they engage with:** `search_x_accounts(engaged_with=[<the seeds>], engaged_with_days=90, engaged_with_min_count=2, ai_roles=[<role>])`. This shows active interest.
   - **D. Events (optional):** `find_events(search=<the topic>)`, then `search_x_accounts(event_ids=[<the ids>], ai_roles=[<role>])`.
   - If a search finds more than 10,000 people, make it stricter before you save it. If it finds fewer than 30, drop it or make it wider.
4. Paid search (optional). Ask the user first. `start_x_search(kind="keywords", keywords=<the topic words or tickers>, relevance_brief=<one sentence about who counts>)` saves the authors of matching posts as a list. Follow it with `get_x_search(view="job", job_id=<the job_id>)` until `done`. Run one search only.
5. Save each kept search as a **static** list: `create_list(name="ICP <topic> - A", source="directory", filters=<the result's list_filters>)`. Save B with `create_list(name="ICP <topic> - B", handles=[<the handles shown>])`. Only static lists can be merged, so do not pass `list_type="dynamic"` here.
6. Merge: `create_list(name="ICP <topic> - merged", source="merge", merge_short_ids=[<the saved lists>])`.
7. Rank. Call `get_list(list_short_id=<the merged list>, view="members", sort_by="in_lists", min_in_lists=2, filters={"ranges": {"smart_followers": {"min": 100}}}, limit=50)`. Each member's `in_lists` is the number of saved lists that contain the person. Always pass a size filter: inside one `in_lists` value the order is not by quality, and small accounts come first. `unresolved_count` is the number of members the filter could not read.
   - **Tier 1:** `min_in_lists=3`, or in both A and B. To find who is in both A and B, call `search_x_accounts(handles=[<the B handles>])` with the filters of A. These are the core ICP.
   - **Tier 2:** `min_in_lists=2`.
   - **Tier 3:** in one list only. Use them for content, not for DMs.
   - To narrow further, pass `filters` with `search_x_accounts`' names, for example `{"ranges": {"followers": {"min": 1000}}, "only": ["active"]}`. `relationship`, `follows` and `engaged_with` do not work here: put them in the searches of step 3.
8. Check the top 10 of tier 1 with `analyze_x_account(view="posts", handle=<the handle>)`. Drop people whose posts do not fit. Tell the user what you dropped and why.
9. Save the final tier as its own list: `create_list(name="ICP <topic> - tier 1", source="saved_list", from_list_short_id=<the merged list>, filters=<the same filters>)`, or `create_list(name=..., handles=[<the kept handles>])` after manual drops. This is the first batch to reach out to.
10. Build the Outreach Agent. This is the main result of the playbook: the searches of step 3 run again on a schedule, and new ICP people arrive every day.
    - Each kept search becomes one Stream Agent:
      - A and A2 → `directory_match` (free) with the same filters.
      - B → `follows_n_of` (free) on the seed watchlist, with the same N as `min_competitors_followed`.
      - C → `engagers_of` on the seed watchlist (paid).
      - D → `event_attendees` (free) on the same events.
      - The paid search of step 4 → `query_authors` (paid) with the same words.
      - Add `new_followers_of` on the seed watchlist (paid) to catch people who start to follow the seeds.
    - Lead filter = the "who" of the ICP card: `filters={"roles": [<role>], "domains": [<topic>], "traits": [...], "entity_type": "person", "min_smart_followers": <size>}`. Name it "ICP <topic> - leads". Leave `min_actors` at 1: it counts the different accounts in a lead's signals (for example two seeds that the lead engaged with), and a `directory_match` lead has none.
    - Call `create_agent(kind="outreach", action="plan", prompt=<the ICP card in one paragraph, with the seeds and the kept searches>)`. Compare the plan with the list above, and add or drop agents to match. Show the user the plan, with the paid Stream Agents marked, and ask its questions.
    - After the user's yes: `create_agent(kind="outreach", action="create", name="ICP <topic>", plan=<the edited plan>)`. The agent is built inactive. Turn it on with `set_agent_status(kind="outreach", action="activate")` only after the user agrees. If it answers `needs_confirmation`, show `charges.terms` first.
11. Offer the outreach. Do not start one without the user's yes.
    - Warm intros: `find_warm_intros(list_short_id=<the tier 1 list>)`.
    - First batch: `check_dm_reach(view="list", list_short_id=<the tier 1 list>)`, then the DM steps of the competitor-leads playbook.
    - Every new lead: `launch_dm_campaign` on the agent's lead filter list with `keep_in_sync`, after the same reach check and approval. The campaign then messages each new ICP person the agent finds.

Report at the end: the ICP card, the count of each search, the tier counts, the `list_url` of the merged list and the tier 1 list, and the `agent_url` of the Outreach Agent with its Stream Agents and whether it is on.

Rules:
- Get the user's yes on the ICP card before the first search.
- Never start a paid search (`start_x_search`, `find_people_tweeting_about`, `update_competitors(action="add")`) without asking the user first.
- Never turn on the Outreach Agent or launch a campaign without the user's yes. A paid Stream Agent spends provider calls on every run.
- Name every saved list with the "ICP <topic>" prefix, so the user can find and delete them later.
- If a list save is uncertain, do not retry. Call `list_my_lists` to check first.
