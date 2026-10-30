---
name: weekly-report
description: Write a short weekly report on Dopamyn DM outreach - what was sent, who replied, what works best, and what to change next week. Use when the user asks how outreach is going, for a weekly or monthly report, or which campaign, account or message works best.
---

# Weekly outreach report

Requires an active Dopamyn subscription. If a tool says one is required, stop and pass that on to the user.

1. Call `get_outreach_results` with `days` 7. Call it again with `days` 30 for the trend. For a calendar week or month, pass `date_from` and `date_to` instead.
2. Call `get_outreach_results` with `days` 30 and `group_by` job, then `group_by` sender, then `group_by` variant. Add `group_by` list for the saved lists that answer best, and `group_by` weekday_hour with the user's `tz_offset_minutes` for the best time to send, when the user asks.
3. Call `list_dm_campaigns(view="all")` for running campaigns, and `get_dm_usage(view="summary")` for the sends left (the campaign quota counts campaign DMs only).
   - For the best and the weakest campaign, call `get_dm_campaign(campaign_id=<the campaign>)`: its one-line health says what it waits on, and `rerun_counts` says how many people a rerun of each group (`no_reply`, `not_contacted`, ...) would reach.
   - Call `list_pipeline_contacts(view="summary")` for the contacts and deal value per CRM stage.
4. Write the report in this order. Keep it under 250 words.
   - **This week:** DMs sent, replies, reply rate, interested replies. Compare the reply rate with the 30-day rate.
   - **Needs you:** interested replies not answered (`needs_answer`). If it is above zero, offer the daily-replies playbook.
   - **What works:** the best campaign, sender and message by reply rate. Ignore groups with `low_sample` true and say so.
   - **What does not:** the weakest group and the top `not_sent_reasons`.
   - **Capacity:** sends left this cycle (`left`: monthly allowance plus message credits) and running campaigns.
   - **Pipeline:** contacts and deal value per stage, when the workspace uses the CRM board.
   - **Next week:** 2-3 concrete actions, for example pause a weak campaign, reuse the best message, or run competitor-leads. A rerun of a campaign's `no_reply` people (`launch_dm_campaign` with `rerun_campaign_id`) sends real DMs: offer it with its `rerun_counts` size and wait for the user.
5. Rates are fractions of DMs sent. Show them as percentages.

Do not pause, launch or send anything from this report. Offer the action and wait for the user.
