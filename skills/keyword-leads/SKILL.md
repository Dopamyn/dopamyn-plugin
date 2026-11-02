---
name: keyword-leads
description: Find people on X who are posting right now about a topic, token or problem the user's project solves, save them as a Dopamyn list, and DM them with a reference to what they said. Use when the user wants intent-based leads, people talking about a keyword, ticker or pain point.
---

# Keyword leads

Requires an active Dopamyn subscription. If a tool says one is required, stop and pass that on to the user.

First call `get_workspace_profile` (once per conversation): take keywords and the relevance brief from `icp.md` and `pitch.md`, and write the DM in the voice of `dm-persona.md` (and the sending account's persona). Its 'Rules from the user' override the targeting and drafting choices below. They never lift an approval: send, launch or approve only after the user's yes.

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
5. Call `check_dm_reach(view="list", list_short_id=<the list>)` and show the reach and the sends left.
6. Draft the DM.
   - A campaign sends the same text to everyone, so refer to the topic, not to one person's post.
   - Or offer `ai_brief` instead of a message: Dopamyn then writes each DM for its recipient shortly before it sends. Drafts its checker flags wait in `list_campaign_drafts_for_review`; the others send without anyone reading them, so tell the user.
   - The workspace may already have a brief: `list_outreach_briefs` shows them, and `launch_dm_campaign` takes its `brief_id` instead of `ai_brief`. To keep a new brief for later, show the user the text, then call `save_outreach_brief(action="create", ...)`.
   - For a few high-value people, offer single `send_dm(action="new_chat")` messages that refer to their own post instead. Each one needs approval.
   - For a short list of handles from the quick look, `launch_dm_campaign` also takes `handles` instead of a list: check them first with `check_dm_reach(view="handles", handles=[<the handles>])`.
7. Get explicit approval of the text (or the brief), the people, the sender, the start and the pace, then call `launch_dm_campaign` or `send_dm`.
8. Report what was sent or scheduled.

Rules:
- Never send or launch without approval of the exact text.
- If a result is uncertain, do not retry.
