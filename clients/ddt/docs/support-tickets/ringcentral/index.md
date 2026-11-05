# RingCentral

Phones, call logging, recordings and texting through RingCentral, and how they reach Salesforce.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-95, DDT-205, DDT-581 | [RingCentral contact sync filling the address book](./ddt-95-ringcentral-contact-sync-runaway/README.md) | Live since 2026-09-30 |
| DDT-131, DDT-606 | [Outbound calls from the PI line fail](./ddt-131-pi-line-outbound-fails/README.md) | Done 2026-08-03 |
| DDT-150 | [Extension not receiving inbound or transferred calls](./ddt-150-extension-not-receiving-calls/README.md) | Done 2026-06-29 |
| DDT-219, DDT-238, DDT-495, DDT-566, DDT-577, DDT-619, DDT-634 | [Setting or fixing a user's outbound caller ID](./ddt-219-outbound-caller-id-settings/README.md) | Done (DDT-219 2026-06-29, DDT-238 2026-06-30, DDT-495 2026-07-29, DDT-566 2026-07-29, DDT-577 2026-07-30, DDT-634 2026-08-04, DDT-619 2026-08-21) |
| DDT-227 | [Adding a user to intake analytics and performance reports](./ddt-227-intake-analytics-report-access/README.md) | Done 2026-06-24 |
| DDT-228 | [Main line rings one person first, then overflows to Intake](./ddt-228-main-line-ring-first-overflow/README.md) | Done 2026-06-29 |
| DDT-262 | [Cannot sign in to Shadow Agent](./ddt-262-shadow-agent-sign-in/README.md) | Done 2026-07-02 |
| DDT-277 | [AI receptionist cannot find a staff member and drops the overflow call](./ddt-277-ai-receptionist-name-lookup/README.md) | Done 2026-07-22 |
| DDT-298, DDT-696, DDT-1037, DDT-1042 | [Calls not ringing: IP Phone Offline / No Digital Line](./ddt-298-ip-phone-offline-calls/README.md) | Done 2026-09-18 |
| DDT-326 | [Outbound calls cut off mid-voicemail](./ddt-326-outbound-voicemail-cut-off/README.md) | Done 2026-07-14 |
| DDT-498, DDT-743, DDT-784, DDT-841, DDT-929 | [Existing clients routed to Intake instead of their case manager](./ddt-498-existing-clients-route-owner/README.md) | Live since 2026-09-04 (last of the five closed) |
| DDT-505 | [RingCentral app stuck connecting, transfers missed](./ddt-505-rc-app-stuck-connecting/README.md) | Done 2026-07-27 |
| DDT-609, DDT-934 | [Available rep never rings on a queue](./ddt-609-rep-never-rings-queue/README.md) | Done 2026-09-07 |
| DDT-617, DDT-618 | [Warm and cold transfers failing for a user](./ddt-617-transfers-failing-user/README.md) | Live since 2026-08-04 |
| DDT-622, DDT-804 | [Transfer fails with extension not recognized](./ddt-622-transfer-extension-not-recognized/README.md) | Done 2026-08-24 |
| DDT-626, DDT-849 | [Moving a rep from the SOS group to the PI group](./ddt-626-move-rep-sos-to-pi/README.md) | Live since 2026-09-01 |
| DDT-630 | [Unintended transfer during a live call, mislogged as outbound](./ddt-630-unintended-transfer-mislogged/README.md) | Done 2026-08-05 |
| DDT-636, DDT-757 | [New case managers receiving intake calls](./ddt-636-new-cm-intake-calls/README.md) | Live since 2026-08-05 |
| DDT-694, DDT-806 | [Reps receiving new calls while on an active call](./ddt-694-busy-rep-new-calls/README.md) | Done 2026-08-24 |
| DDT-741, DDT-883, DDT-945, DDT-946, DDT-1257 | [Emergency contacts of clients routed to Intake](./ddt-741-emergency-contacts-routing/README.md) | Live since 2026-09-28 (DDT-945); last ticket in the group closed 2026-10-07 |
| DDT-828 | [Incoming ringtone won't stop during an active call](./ddt-828-ringtone-during-active-call/README.md) | Done 2026-09-01 |
| DDT-845 | [PI fax confirmation emails routed to the injury inbox](./ddt-845-pi-fax-confirmations-inbox/README.md) | Live since 2026-08-28 |
| DDT-859 | [Routing a hired client's calls to the Spanish CMT queue](./ddt-859-hired-client-spanish-cmt/README.md) | Done 2026-09-01 |
| DDT-923 | [Removing the Intake Specialist step from main line routing](./ddt-923-remove-intake-specialist-step/README.md) | Done 2026-09-04 (request withdrawn, no change made) |
| DDT-1020 | [Shadow Agent stops ringing and fails outbound dial](./ddt-1020-shadow-agent-ringing-issues/README.md) | Done 2026-09-18 |
| DDT-1033, DDT-1046 | [Call monitoring setup and silent listen-in](./ddt-1033-call-monitoring-listen-in/README.md) | Done 2026-09-24 (DDT-1033 done 2026-09-18) |
| DDT-1035, DDT-1050 | [Outbound calls fail: cannot connect due to internal errors](./ddt-1035-outbound-internal-errors/README.md) | Done 2026-10-01 (DDT-1035 done 2026-09-28) |
| DDT-1045, DDT-1271, DDT-1403 | [Granting access to call recordings and ACE](./ddt-1045-call-recording-access/README.md) | Done 2026-10-26 |
| DDT-1055 | [Adding a rep to the Overflow queues](./ddt-1055-add-rep-overflow-queues/README.md) | Done 2026-09-21 |
| DDT-1096 | [Warm transfers to an attorney land on case managers](./ddt-1096-warm-transfer-wrong-phone/README.md) | Live since 2026-09-24 |
| DDT-1108 | [PI after-hours calls reach no one](./ddt-1108-pi-after-hours-dnd/README.md) | Done 2026-09-25 |
| DDT-1233, DDT-1234 | [Queue goes to voicemail with members available](./ddt-1233-queue-straight-to-voicemail/README.md) | Done 2026-10-06 |
| DDT-1337, DDT-1409, DDT-1468 | [Router rollover sends unanswered calls to the wrong queue](./ddt-1337-router-rollover-wrong-queue/README.md) | Live since 2026-10-30 (DDT-1337 live since 2026-10-21, DDT-1468 done 2026-10-29, DDT-1409 done 2026-10-30) |
| DDT-1340 | [Outbound calls missing from the RingCentral call log](./ddt-1340-outbound-missing-call-log/README.md) | Live since 2026-10-16 |
| DDT-1368 | [Intake queue not ringing all available members](./ddt-1368-intake-queue-skips-members/README.md) | Live since 2026-10-19 |
| DDT-1406 | [RingCentral October 2026 developer update impact](./ddt-1406-rc-developer-update-impact/README.md) | Done 2026-10-23 |
| DDT-1464 | [Inbound faxes landing on the wrong extension](./ddt-1464-inbound-faxes-wrong-extension/README.md) | Done 2026-10-28 |
