# Phone bot

The AI phone receptionist: call handling, intake, transfers and what it writes to Salesforce.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-99 | [Error handling and failure alerts for Flowyte routing callouts](./ddt-99-flowyte-routing-error-alerts/README.md) | Live since 2026-06-03 |
| DDT-261 | [Flowyte phone routing failing after the API moved](./ddt-261-flowyte-endpoint-moved/README.md) | Live since 2026-07-02 |
| DDT-372, DDT-373, DDT-374, DDT-600, DDT-628, DDT-676, DDT-1053 | [Lost-call alert fires on calls that were answered](./ddt-372-lost-call-false-positives/README.md) | Live since 2026-07-20 (last change live 2026-09-22) |
| DDT-571 | [Router backup uses the owner's fallback extension](./ddt-571-routing-fallback-extension/README.md) | Live since 2026-07-29 |
| DDT-572, DDT-595 | [Routing past clients by case recency](./ddt-572-past-client-recency-routing/README.md) | Live since 2026-07-30 (DDT-595 follow-up completed 2026-08-17) |
| DDT-573, DDT-580 | [Router record priority: Contact above Lead](./ddt-573-router-contact-over-lead/README.md) | Live since 2026-07-30 |
| DDT-594 | [Routing a share of unmatched callers to the AI receptionist](./ddt-594-no-match-callers-ai-receptionist/README.md) | Live since 2026-08-20 |
| DDT-660, DDT-742 | [High-value label on lost-call alerts](./ddt-660-high-value-lost-call/README.md) | Live since 2026-08-17 |
| DDT-661, DDT-754 | [Marking recovered lost calls in Slack](./ddt-661-recovered-call-marker/README.md) | Live since 2026-08-21 |
| DDT-709 | [Lost-call analysis confuses presence with queue membership](./ddt-709-presence-vs-queue-membership/README.md) | Live since 2026-08-12 |
| DDT-710, DDT-746, DDT-1336 | [Missed calls not reported by the phone bot](./ddt-710-missed-calls-not-reported/README.md) | Live since 2026-08-12 (DDT-710) and 2026-08-17 (DDT-746); DDT-1336 Done 2026-10-15, no counting defect |
| DDT-727 | [Hourly call summary counts inbound only and uses answer rate](./ddt-727-hourly-summary-answer-rate/README.md) | Done 2026-08-19 (inbound-only change reported done 2026-08-13) |
| DDT-755, DDT-840, DDT-1054 | [Lost-call alert calls an existing client a new lead](./ddt-755-lost-call-new-lead-label/README.md) | Live since 2026-08-19 |
| DDT-770, DDT-808 | [Hourly and daily recap team breakdown](./ddt-770-recap-team-breakdown/README.md) | Live since 2026-08-24 |
| DDT-783, DDT-834, DDT-942, DDT-1258 | [Recap credits pickups to the wrong rep or team](./ddt-783-recap-pickup-attribution/README.md) | Live since 2026-10-06 |
| DDT-917 | [Hourly staffing report miscounts inbound and answered calls](./ddt-917-hourly-report-miscounts-inbound/README.md) | Live since 2026-09-04 |
| DDT-921 | [Lost-call alert says Case Management instead of PI Case Management](./ddt-921-relationship-label-pi-cm/README.md) | Live since 2026-09-04 |
| DDT-922, DDT-1129 | [Rep presence state on lost-call alerts](./ddt-922-presence-on-alerts/README.md) | Live since 2026-09-30 (DDT-922); DDT-1129 done 2026-09-28 |
| DDT-1029, DDT-1099, DDT-1260 | [Short abandoned calls counted and alerted wrongly](./ddt-1029-short-abandoned-calls/README.md) | Live since 2026-10-05 (DDT-1029 live since 2026-09-21; DDT-1260 done 2026-10-23 with no change) |
| DDT-1039, DDT-1041 | [Team and door labels follow where a call arrived](./ddt-1039-call-arrival-team-label/README.md) | Live since 2026-09-17 |
| DDT-1040 | [Recap tables misaligned and printed twice](./ddt-1040-recap-table-layout/README.md) | Live since 2026-09-17 |
| DDT-1075 | [Missed-call alert blames the rep when the phone is offline](./ddt-1075-alert-blames-offline-phone/README.md) | Done 2026-09-24 |
| DDT-1335 | [Phone bot alert shows the queue name for the router destination](./ddt-1335-router-destination-names/README.md) | Live since 2026-10-19 |
