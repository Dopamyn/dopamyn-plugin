---
name: event-leads
description: Find the attendees of a crypto event or its side events, pick the right people, save them as a Dopamyn list, and DM them before or after the event. Use when the user mentions an event (Token2049, KBW, ETHDenver, a side event) and wants to meet or reach its attendees.
---

# Event leads

Requires an active Dopamyn subscription. If a tool says one is required, stop and pass that on to the user.

1. Call `find_events` with the event name as `search`.
   - If the user means the whole main event, use its `main_event_id`.
   - If they mean one side event, use its `event_id`.
   - If more than one event matches, ask the user which one.
   - For the conferences themselves, call `find_events(view="main_events")`. To pick side events by city, date, category or size, call `find_events(view="directory", search=<the event name>, ...)` with those filters; its rows add hosts, sponsors, speakers and apply links.
   - If the user wants the people who run events rather than attend them: `find_events(view="hosts", search=<the event name>)` gives the X accounts hosting the matching events, and `find_event_people(role="host")` or `find_event_people(role="sponsor_or_speaker", main_event_ids=[<the main_event_id>])` the host, sponsor and speaker directories, with their event counts. `find_event_people(view="handles", ...)` gives up to 500 handles to save. Go to step 4 with those handles.
2. Call `search_x_accounts` with `main_event_ids` set to the `main_event_id`, or `event_ids` set to the `event_id`, plus the user's filters. Good defaults:
   - `entity_types` person, when the user wants people
   - `professions` or `tags` for the roles they want (for example founder/ceo, investor/vc)
   - `sort_by` quality
   - `limit` 20
3. Show the attendees with their profession and followers, and `available_count`, how many match in all. To check a few people closely, call `search_x_accounts(view="profiles", handles=[<their handles>])`.
4. After the user agrees, save the list, for example "Token2049 founders":
   - the attendees you showed: `create_list(name=<the name>, handles=[<their handles>])`;
   - or every attendee the search matches: `create_list(name=<the name>, source="directory", filters=<the search's list_filters>)`. A live list (`list_type="dynamic"`) re-runs the search each time it is used, so it picks up attendees Dopamyn learns about later. Tell the user which one you save and how many people it holds.
5. Call `check_dm_reach(view="list", list_short_id=<the new list>)` and show the reach and the sends left.
6. Draft the DM with the user.
   - Before the event: suggest meeting there. Name the event.
   - After the event: refer to the event and offer a follow-up.
   - Use `{{first_name}}`. Under 500 characters.
7. Agree when it starts: before the event, `start_at` can time the first DMs, and `minutes_between_sends` sets the pace. Get explicit approval of the message, list, sender, start and pace, then call `launch_dm_campaign`.
8. Report the `campaign_id` and the scheduled count.

Rules:
- Never launch without approval.
- If a launch result is uncertain, do not retry. Call `list_dm_campaigns(view="all")` first.
