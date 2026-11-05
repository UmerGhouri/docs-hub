# Marketing Attribution

Answers which marketing brings in paying clients and what each source costs. Every touch is stored raw, configuration-driven rules decide credit, and approved monthly spend turns that into cost per signed client. The project is still in progress; this section covers its completed tickets.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-357 | ["How did you hear about us?" blank or wrong on PI matters](./ddt-357-matter-how-heard-per-matter/README.md) | Live since 2026-07-30 |
| DDT-509 | [Record who referred a client, once, and carry it through conversion](./ddt-509-referral-source-capture/README.md) | Live since 2026-07-28 |
| DDT-555 | [One attribution answer per case: the credit hierarchy and channel dictionary](./ddt-555-attribution-credit-hierarchy/README.md) | Live since 2026-08-20 |
| DDT-556 | [Backfill attribution onto 31,000 historical cases and leads](./ddt-556-attribution-backfill-run/README.md) | Live since 2026-08-24 |
| DDT-557 | [Ask "How did you hear about us?" on every new case, with a note for "Other"](./ddt-557-how-heard-required-at-intake/README.md) | Live since 2026-09-23 |
| DDT-558 | [Which flows fire on a mass update, and a switch to pause them](./ddt-558-backfill-flow-suppression/README.md) | Live since 2026-07-29 |
| DDT-559 | [Retire the old "Repeat Client" flag in favour of one returning-client measure](./ddt-559-retire-repeat-client-flag/README.md) | Live since 2026-09-15 |
| DDT-560 | [Retire a flow that kept its own hidden campaign table](./ddt-560-retire-campaignid-updater-flow/README.md) | Live since 2026-09-22 |
| DDT-561 | [Weekly digest of marketing sources the dictionary cannot map](./ddt-561-unmapped-token-weekly-digest/README.md) | Live since 2026-09-15 |
| DDT-570 | ["Referring Party" search on the Lead returns no results](./ddt-570-referring-party-lookup-empty/README.md) | Done 2026-08-04 |
| DDT-574 | [New cases and leads were never given an attribution](./ddt-574-attribution-go-forward-sweeper/README.md) | Live since 2026-09-07 |
| DDT-745 | [Lead conversion fails on "Google Local Service Lead"](./ddt-745-lsa-value-blocks-lead-convert/README.md) | Live since 2026-08-17 |
| DDT-811 | [Keep referral credit safe once a touchpoint log shares the same object](./ddt-811-referral-credit-vs-touch-log/README.md) | Live since 2026-09-08 |
| DDT-812 | [Every touch row re-wrote the records it points at](./ddt-812-stop-redundant-parent-writes/README.md) | Live since 2026-09-08 |
| DDT-813 | [New attribution fields on Inbound Contact, visible on the page and in the permission set](./ddt-813-inbound-contact-attribution-fields/README.md) | Live since 2026-10-01 |
| DDT-814 | [Log a touch row for every new Lead](./ddt-814-lead-touch-logging-flow/README.md) | Live since 2026-09-14 |
| DDT-815 | [Check that converted records already link their touch rows](./ddt-815-conversion-linking-check/README.md) | Done 2026-09-16 |
| DDT-816 | [Clean up the phone tracking-number registry](./ddt-816-phone-registry-cleanup/README.md) | Live since 2026-09-14 |
| DDT-817 | [Call phone numbers did not match the tracking-number registry](./ddt-817-call-registry-phone-match/README.md) | Live since 2026-09-10 |
| DDT-818 | [Log a touch row for every inbound call](./ddt-818-inbound-call-touch-flow/README.md) | Live since 2026-09-15 |
| DDT-819 | [Backfill historical inbound calls into the touch log without doubling them](./ddt-819-call-history-backfill/README.md) | Live since 2026-09-22 |
| DDT-821 | [Send the website visit history with each form, one touch row per visit](./ddt-821-website-visit-history/README.md) | Live since 2026-11-02 |
| DDT-822 | [A call or ad-click touch did not move the credit numbers](./ddt-822-touch-triggers-credit-recompute/README.md) | Live since 2026-09-16 |
| DDT-823 | [Alarm when the touch log stops filling, and re-link orphaned touches nightly](./ddt-823-heartbeat-and-nightly-relink/README.md) | Live since 2026-10-09 |
| DDT-824 | [New credit rules for new records only: schema, door flag and rule version](./ddt-824-forward-only-credit-rule/README.md) | Live since 2026-09-16 |
| DDT-825 | [One stored, indexed phone key so one person matches as one person](./ddt-825-stored-phone-match-key/README.md) | Live since 2026-09-15 |
| DDT-826 | [Re-score attribution history under the new credit rules](./ddt-826-rescore-history-new-credit-rules/README.md) | Live since 2026-10-07 |
| DDT-862, DDT-1022, DDT-1083, DDT-1086 | [Capture who referred a new client, with an honest "not known" escape](./ddt-862-referrer-capture-gate/README.md) | Live since 2026-09-22 |
| DDT-868 | [Call transfer link fields were unreadable for call-logging users](./ddt-868-call-session-link-fields-unreadable/README.md) | Live since 2026-10-01 |
| DDT-902 | [Write the sales-comp case source from attribution](./ddt-902-comp-case-source-from-attribution/README.md) | Live since 2026-10-14 |
| DDT-947 | [Scheduled Apex slots: the real count and the dispatcher policy](./ddt-947-scheduled-apex-slot-ceiling/README.md) | Done 2026-09-28 |
| DDT-948, DDT-991 | [Referral recompute flows that had never fired, and referral history tracking](./ddt-948-referral-recompute-flows-decision/README.md) | Live since 2026-09-10 |
| DDT-951 | [Merge duplicate client records and clean invisible characters from phones](./ddt-951-merge-duplicate-client-records/README.md) | Done 2026-09-16 |
| DDT-952 | [Referral panel save now respects user permissions](./ddt-952-referral-panel-permission-check/README.md) | Live since 2026-09-15 |
| DDT-959 | [Monthly marketing cost ledger with confidence labels](./ddt-959-monthly-marketing-cost-ledger/README.md) | Done 2026-09-12 |
| DDT-961 | [Confidence labels for every attribution channel](./ddt-961-channel-confidence-labels/README.md) | Live since 2026-09-24 |
| DDT-967 | [Returning-client match missed two common phone formats](./ddt-967-returning-client-phone-formats/README.md) | Live since 2026-09-09 |
| DDT-992 | [Decision: require a referrer, with a "could not name them" escape](./ddt-992-referrer-capture-decision/README.md) | Done 2026-09-12 |
| DDT-1000 | [Consolidate duplicate phone normalisation code](./ddt-1000-shared-phone-normalisation/README.md) | Live since 2026-09-15 |
| DDT-1014 | [Measure what Local Services Ads returns, and say how much is assumed](./ddt-1014-lsa-return-measurement/README.md) | Done 2026-09-24 |
| DDT-1026 | [Wire the new credit rules into the engine and turn the door rule on](./ddt-1026-credit-engine-door-rule/README.md) | Live since 2026-09-30 |
| DDT-1078 | [Re-score a record when its Campaign or how-heard answer arrives late](./ddt-1078-rescore-late-campaign-howheard/README.md) | Live since 2026-09-22 |
| DDT-1079 | [Monthly count of re-scored attribution records in the Monday digest](./ddt-1079-monthly-rescore-count/README.md) | Live since 2026-09-22 |
| DDT-1080 | [Two flows with hardcoded how-heard to Campaign tables](./ddt-1080-hardcoded-campaign-flow-tables/README.md) | Done 2026-09-22 |
| DDT-1081 | [Duplicate clients that share no phone or email](./ddt-1081-duplicate-clients-name-dob/README.md) | Done 2026-09-25 |
| DDT-1082 | [Record which tracking number each inbound call came in on](./ddt-1082-call-tracking-number-lookup/README.md) | Live since 2026-09-22 |
| DDT-1084 | [Referrers and vendors were labelled as potential clients](./ddt-1084-referrer-accounts-client-type/README.md) | Live since 2026-09-22 |
| DDT-1085 | [Call touch times drift a few seconds from the final call start time](./ddt-1085-call-touch-time-drift/README.md) | Done 2026-09-22 |
| DDT-1117 | [One Legal Growth Engine label across its ten lead-source tokens](./ddt-1117-one-lge-vendor-label/README.md) | Live since 2026-09-25 |
| DDT-1119 | [Make the touch-log heartbeat alarm able to fire in production](./ddt-1119-heartbeat-alarm-non-admin-user/README.md) | Live since 2026-10-09 |
| DDT-1120 | [Stop intake from creating duplicate clients](./ddt-1120-stop-duplicate-clients-at-intake/README.md) | Live since 2026-10-09 |
| DDT-1131 | [Measure how old the first-touch cookie is when a form arrives](./ddt-1131-measure-first-touch-cookie-age/README.md) | Live since 2026-11-02 |
| DDT-1134 | [Capture Google's iPhone click ids (gbraid, wbraid) in the firm's own cookies](./ddt-1134-capture-gbraid-wbraid-click-ids/README.md) | Live since 2026-11-02 |
| DDT-1135 | [Inbound-call Leads silently defaulted to Defense](./ddt-1135-call-leads-default-to-defense/README.md) | Live since 2026-09-30 |
| DDT-1154 | [Returning clients were recorded as referrals](./ddt-1154-returning-clients-not-referrals/README.md) | Live since 2026-09-30 |
| DDT-1166 | [Marketing spend: one approved row per channel per month](./ddt-1166-marketing-spend-approval-object/README.md) | Live since 2026-10-02 |
| DDT-1167 | [Pull Google Ads and LSA spend daily into draft spend rows](./ddt-1167-daily-google-ads-spend-pull/README.md) | Live since 2026-10-08 |
| DDT-1169 | [Enter lead vendor cost by hand each month](./ddt-1169-manual-lead-vendor-cost/README.md) | Live since 2026-10-15 |
| DDT-1178 | [Ask "how did you hear about us" once, and "who?" only when needed](./ddt-1178-ask-how-heard-once/README.md) | Live since 2026-10-15 |
| DDT-1180 | [How-heard list: missing vendor options, and every option labelled by type](./ddt-1180-how-heard-list-vendor-options/README.md) | Live since 2026-10-02 |
| DDT-1225, DDT-1332, DDT-1432 | [Retire the cancelled Heeet attribution package without losing leads](./ddt-1225-retire-heeet-attribution-package/README.md) | Live since 2026-11-05 |
| DDT-1226 | [Move the Google Ads developer token out of the code](./ddt-1226-google-ads-token-out-of-code/README.md) | Live since 2026-10-15 |
| DDT-1272 | [Tie each tracking-number row to its Google Ads campaign by ID](./ddt-1272-phone-registry-google-ads-ids/README.md) | Live since 2026-10-16 |
| DDT-1294 | [Outside law offices were labelled as internal referrals](./ddt-1294-outside-law-office-referral-label/README.md) | Live since 2026-10-19 |
| DDT-1299 | [Decision: keep or cut CallRail call tracking](./ddt-1299-keep-or-cut-callrail/README.md) | Done 2026-10-22 |
| DDT-1303 | [Automated PR reviewer: a blocking finding must actually block](./ddt-1303-pr-reviewer-blocking-findings/README.md) | Live since 2026-10-20 |
| DDT-1326 | [Alert when the daily marketing spend pull fails](./ddt-1326-spend-pull-failure-alert/README.md) | Live since 2026-10-14 |
| DDT-1330 | [Lead referral panel briefly showed the full picker on a slow load](./ddt-1330-referral-panel-slow-load/README.md) | Live since 2026-10-16 |
| DDT-1353 | [One Google error lost a day of the LSA snapshot](./ddt-1353-lsa-snapshot-same-day-retry/README.md) | Live since 2026-10-19 |
| DDT-1356 | [Let each case see every touch its person made](./ddt-1356-journey-every-touch-per-case/README.md) | Live since 2026-10-20 |
| DDT-1357 | [Label every touch: kind, source system and a duplicate-proof key](./ddt-1357-label-every-touch/README.md) | Live since 2026-10-22 |
| DDT-1358 | [Turn a case's campaign, "how did you hear" and referral fields into touches](./ddt-1358-case-fields-become-touches/README.md) | Live since 2026-10-26 |
