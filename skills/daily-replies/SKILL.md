---
name: daily-replies
description: Triage replies to Dopamyn DM campaigns that still need an answer, draft answers, and send the approved ones. Use when the user asks what replies need attention, wants to clear their DM inbox, or says "daily replies".
---

# Daily replies

Requires an active Dopamyn subscription. If a tool says one is required, stop and pass that on to the user.

Goal: no interested reply waits more than a day.

1. Call `get_dm_replies` with `unanswered_only` true and `limit` 20. If the user names a campaign or a list, pass `campaign_ids` or `list_short_id`; to see only the hot ones, pass `intents`.
2. Group the replies by `intent`, in this order: interested, unclear, acknowledged, not_interested, opt_out, auto_reply, pending. A reply with `locked` true is an encrypted X Chat reply Dopamyn cannot read yet: list it, and do not draft an answer to it.
3. For each interested or unclear reply, draft one short answer.
   - Answer what the person said. Quote nothing back.
   - Move toward one next step: a call, a link, or one question.
   - Keep it under 400 characters, plain text, no hashtags.
4. Show the user a table: recipient, campaign, their reply (short), intent, your draft.
5. Ask which drafts to send. The user can edit any draft first. To see the whole thread before drafting, call `get_conversation(account=<the reply's sender_x_id>, conversation_id=<the reply's conversation_id>)`.
6. For each approved draft:
   - When the reply carries a `conversation_id`, call `send_dm(action="reply", account=<the reply's sender_x_id>, conversation_id=<the reply's conversation_id>, message=<the approved text>)`.
   - Otherwise call `send_dm(action="new_chat", account=<the reply's sender>, to_handle=<the reply's recipient>, message=<the approved text>)`.
   - If the tool answers that the person is on the DM Block list, ask the user. Only if they say to send to them anyway, call it again with `override_block_list_for` set to that person's handle.
   - After the answer to an interested reply is sent, offer to track the person on the CRM board. Only if the user agrees: `get_contact(view="summary", person=<the reply's recipient>)`, then `update_contact(action="set_stage", contact_id=<the contact_id>, stage=<a stage from get_crm_setup>)`, or `add_to_leads(person=<the reply's recipient>, add_to_pipeline=true)` when `in_crm` is false.
7. For each opt_out reply, do not answer. Tell the user, and offer to add the person to the Block list. Only if the user says yes, call `update_block_list(action="block", handles=[<the reply's recipient>])`.
8. For not_interested replies, suggest no answer, or one short thank-you if the user wants it.
9. If `next_cursor` is not null, tell the user that more replies are waiting.

Rules:
- Never call `send_dm` without the user's approval of the exact text.
- If a send result is uncertain, do not retry. Tell the user to check the conversation on X.
- Each send uses one DM from the monthly allowance, then message credits.

End with one line: sent, skipped, blocked, and still waiting.
