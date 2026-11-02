---
name: competitor-leads
description: Turn a competitor's audience into a Dopamyn DM campaign - find people who engage with or follow competitors, filter them, save a list, and launch a DM campaign after approval. Use when the user wants leads from competitors, "steal" a competitor's community, or reach people in their category who do not know them yet.
---

# Competitor leads

Requires an active Dopamyn subscription. If a tool says one is required, stop and pass that on to the user.

First call `get_workspace_profile` (once per conversation): take competitors from `competitors.md` first, then the saved ones from `get_competitors`; judge leads against `icp.md`; and write the DM in the voice of `dm-persona.md` (and the sending account's persona). Its 'Rules from the user' override the targeting and drafting choices below. They never lift an approval: send, launch or approve only after the user's yes.

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
5. Call `check_dm_reach(view="list", list_short_id=<the new list>)`. Show:
   - reachable people, people on the Block list, people already messaged
   - sends left this month (`credits.left`)
6. Draft the DM with the user.
   - Mention the shared interest (the competitor's category), never the competitor by name in a negative way.
   - Use `{{first_name}}` for a personal greeting.
   - Under 500 characters, one clear ask, no links in the first message unless the user insists.
7. Agree the schedule: start now or at `start_at`, and the gap between sends (`minutes_between_sends`; the app uses 5 minutes). If you saved a live (dynamic) list, ask whether the campaign should also message people who join it later (`keep_in_sync`); it then keeps sending until paused. A static list cannot take `keep_in_sync`.
8. Show the final message, list, sender account, start, pacing, and number of DMs. Get explicit approval.
9. Call `launch_dm_campaign`. If it asks about the monthly limit, ask the user to pick truncate, split or use_credits, then call it again with `if_over_monthly_limit`. If it says another campaign already watches the list, ask the user before calling again with `replace_existing_binding=true`.
10. Report the `campaign_id`, the scheduled count, and the skips. Tell the user they can say "pause it" at any time: that is `set_dm_campaign_status(action="pause", campaign_id=<the campaign>)`.

Rules:
- Never launch without approval of the exact message and recipient count.
- Never add a competitor or start a paid job without asking the user first.
- If a launch result is uncertain, do not retry. Call `list_dm_campaigns(view="all")` to check first.
