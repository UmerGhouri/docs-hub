# #Phone-Bot Tracker

The firm's own phone monitoring bot, built to replace an earlier monitoring bot and go further. It watches RingCentral for lost calls, staffing changes and callback recovery, and posts alerts, hourly stats and daily recaps to Slack. It also adds Salesforce client details to alerts, callback reminders and a lookup of past rep presence.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-380, DDT-386, DDT-387 | [Near-real-time phone alerts from RingCentral push events](./ddt-380-push-events-near-real-time-alerts/README.md) | Live since 2026-07-20 |
| DDT-381, DDT-383, DDT-384 | [Slack alerts when a phone queue drops below its staffing floor](./ddt-381-queue-staffing-floor-alerts/README.md) | Live since 2026-07-20 |
| DDT-382, DDT-388 | [Lost-call alerts with repeat-caller and paid-lead badges](./ddt-382-lost-call-alerts-repeat-callers/README.md) | Live since 2026-07-20 |
| DDT-385 | ["Recovered" notice when a missed caller is reached](./ddt-385-callback-recovery-detection/README.md) | Live since 2026-07-20 |
| DDT-389 | [Callout when an available rep lets a call ring out](./ddt-389-rep-rang-out-callouts/README.md) | Live since 2026-07-20 |
| DDT-390, DDT-392 | [Hourly and daily call reports in Slack](./ddt-390-hourly-daily-call-reports/README.md) | Live since 2026-07-20 |
| DDT-391 | [Salesforce caller lookup on every phone alert](./ddt-391-salesforce-caller-lookup-on-alerts/README.md) | Live since 2026-07-20 |
| DDT-393 | [Event time in Central Time on every phone alert](./ddt-393-central-time-on-alerts/README.md) | Live since 2026-07-20 |
| DDT-394 | [Readable queue names and routing chains in phone alerts](./ddt-394-readable-queue-names-routing/README.md) | Live since 2026-07-20 |
| DDT-395 | [Phone Bot alerts were hard to read in Slack](./ddt-395-slack-alert-formatting-overhaul/README.md) | Live since 2026-07-20 |
| DDT-396 | [Stats "By team" list showed 15+ fragments instead of teams](./ddt-396-canonical-team-bucketing-stats/README.md) | Live since 2026-07-20 |
| DDT-397 | [Rep and channel mentions in phone alerts did not ping anyone](./ddt-397-real-slack-mentions-for-reps/README.md) | Live since 2026-07-20 |
| DDT-398 | [Apply the previous phone bot's production rules for presence and call logs](./ddt-398-presence-and-call-log-rules/README.md) | Live since 2026-07-20 |
| DDT-399 | [Staffing alerts overnight, and repeat misses left as Slack noise](./ddt-399-business-hours-and-auto-tickets/README.md) | Live since 2026-07-20 |
| DDT-400 | [Missed callers forgotten: remind after 15 minutes without a callback](./ddt-400-callback-reminder-unrecovered-misses/README.md) | Live since 2026-07-20 |
| DDT-401 | [Answer "was a rep on Do Not Disturb at a given time?"](./ddt-401-historical-presence-lookup/README.md) | Live since 2026-07-20 |
| DDT-402, DDT-404, DDT-586 | [Answered calls reported as lost: classify the whole call session](./ddt-402-answered-calls-reported-as-lost/README.md) | Live since 2026-07-20 |
| DDT-403 | [Caller lookup showed "unavailable" after the Salesforce session expired](./ddt-403-salesforce-lookup-session-retry/README.md) | Live since 2026-07-20 |
| DDT-562 | ["Who is offline?" had no answer, and staffing alerts were too noisy](./ddt-562-roster-snapshot-staffing-toggle/README.md) | Live since 2026-07-28 |
| DDT-583 | [LOST CALL alerts for callers who hung up during the greeting](./ddt-583-filter-greeting-hangups/README.md) | Live since 2026-07-30 |
| DDT-584 | [Routing line showed phone-system internals instead of "Mainline"](./ddt-584-mainline-routing-alias/README.md) | Live since 2026-07-30 |
| DDT-585 | [Rang-out callouts missed "Timeout" legs and blamed "DDT" as a rep](./ddt-585-rang-out-timeout-detection/README.md) | Live since 2026-07-30 |
| DDT-599 | [Phone alerts were hard to tell apart at a glance](./ddt-599-emoji-headers-on-phone-alerts/README.md) | Live since 2026-07-31 |
| DDT-615, DDT-620 | [Reminder and recovered posts should thread under the lost call](./ddt-615-thread-followups-under-lost-call/README.md) | Live since 2026-08-03 |
| DDT-616 | [Hourly call recap wording was ambiguous](./ddt-616-hourly-recap-wording/README.md) | Live since 2026-08-03 |
| DDT-646 | [Phone alerts lagged by minutes because real-time events never arrived](./ddt-646-realtime-ringcentral-webhook-events/README.md) | Live since 2026-08-05 |
| DDT-647 | [Hired clients flagged high-value, and reps blamed for offline phones](./ddt-647-hired-client-badge-offline-device-strikes/README.md) | Live since 2026-08-05 |
| DDT-648 | [False misses on transferred calls, and reps shown by internal id](./ddt-648-transfer-suppression-rep-registry/README.md) | Live since 2026-08-05 |
| DDT-649 | [Reminder fired after a relative of the caller was already reached](./ddt-649-recovery-across-related-callers/README.md) | Live since 2026-10-05 |
| DDT-650 | [One missed call posted four times](./ddt-650-duplicate-alerts-from-concurrent-ticks/README.md) | Live since 2026-08-05 |
| DDT-691 | [Phone Bot logs only covered 13 minutes of history](./ddt-691-reduce-phone-bot-log-volume/README.md) | Live since 2026-08-12 |
| DDT-692 | [Reminder fired for a caller who had already been called back](./ddt-692-reminder-after-recovery-freshness-check/README.md) | Live since 2026-08-12 |
| DDT-703 | [Lost-call alerts took one to two minutes to post](./ddt-703-fast-lost-call-on-disconnect/README.md) | Live since 2026-08-12 |
| DDT-704 | [Reminder understated how long the caller had waited](./ddt-704-reminder-wait-time-undercount/README.md) | Live since 2026-10-05 |
| DDT-714 | [Short missed calls stalled, and two queues were not monitored](./ddt-714-out-of-order-events-and-queue-names/README.md) | Live since 2026-08-12 |
| DDT-1010 | [Label every inbound call by the team it was for](./ddt-1010-label-inbound-calls-by-team/README.md) | Live since 2026-09-15 |
