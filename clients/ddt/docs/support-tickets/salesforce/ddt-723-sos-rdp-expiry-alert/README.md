# SOS RDP expiry alert and its failing test

| | |
|---|---|
| **Ticket** | DDT-723, DDT-1224 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-29 (DDT-723). Test fix DDT-1224 deployed, done 2026-10-06. |
| **Priority** | Medium (DDT-1224: High) |
| **Reported by** | Firm staff (SOS team), 2026-08-12. DDT-1224 raised by the Support team, 2026-10-05. |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Opportunity fields, Lightning page, Apex, public group, permission set, Custom Label) |
| **Area** | SOS hearing cases: tracking a client's Restricted Driving Permit (RDP) and warning the SOS team before it expires |
| **Related** | DDT-1449 (SOS field map and hearings report) |
| **Code** | PR #535 (DDT-723) and PR #617 (DDT-1224) in the ddt-salesforce repo |

---

## 1. Summary

The SOS team asked for a place on the SOS Blueback to record a client's Restricted Driving Permit, and for an email before the permit expires. PR #535 added four RDP fields, an "RDP" section on the Hired Opportunity page, and a daily job that emails a new public group at 6, 3 and 1 calendar months before expiry, on open and closed cases. A week later one of the job's tests started failing in production depending on who ran it; PR #617 fixed the test so it no longer depends on the real group's members.

## 2. What was reported

- **DDT-723, 2026-08-12.** The SOS team asked for an RDP yes/no on the SOS Blueback that, when ticked, shows Issuance Date, Expiry Date and Duration (One-year / Five-year), plus an automated email to the SOS team six months before expiry, on both open and closed cases. A follow-up asked that the box sit above or below the SOS Documents section, not inside it. The ticket left three questions open: which object and record type, who receives the email, and whether it fires once or repeatedly. They were answered on 2026-09-25: confirm the object against production, send to an SOS public group, and send at 6, 3 and 1 months, with no new scheduled job.
- **DDT-1224, 2026-10-05.** `RdpExpiryAlertTest.theEmailNamesTheClientTheDatesAndLinksTheCase` failed in production with "addressed to the user, not an address list", while the other 18 tests passed and the class was unchanged. It passed when the deploy ran on 2026-09-29 and failed from 2026-10-02 onward. While it stood, every `RunLocalTests` validation or deploy to production failed; it blocked the full-suite gate on PR #608. Production behaviour was correct.

## 3. Scope

**In scope (PR #535)**

| Component | Type | Change |
|---|---|---|
| `Opportunity.RDP__c` | Checkbox | New, default false |
| `Opportunity.RDP_Issuance_Date__c`, `RDP_Expiry_Date__c` | Date | New |
| `Opportunity.RDP_Duration__c` | Restricted picklist | New: One-year, Five-year |
| `Opportunity.RDP_Last_Alert_Months__c`, `RDP_Alerted_Expiry_Date__c` | Number, Date | New, written only by the job, not on any page |
| `Opportunity_Record_BB_Default_Hired` | Lightning page | New "RDP" section after SOS Documents |
| `SOS_RDP_Alerts` | Public group | New; deploys empty, seeded by script |
| `DDT_RDP_Fields` | Permission set | Read/edit on the four visible fields |
| `RDP_Expiry_Alert_Enabled` | Custom Label | On/off switch |
| `RdpExpiryAlert`, `RdpExpiryAlertTest` | Apex | The job and its tests |
| `GlobalMinuteScheduler`, `GlobalMinuteSchedulerTest` | Apex | New rider `dispatchRdpExpiryAlert` |
| `scripts/apex/ddt-723-assign-rdp-fields.apex`, `scripts/apex/ddt-723-seed-sos-rdp-alerts-group.apex` | Anonymous Apex | Post-deploy steps |

**In scope (PR #617):** `RdpExpiryAlertTest` only.

**Out of scope:** the mobile-only Hired page and the mobile Blueback screen flows (desktop is what was asked for; mobile is a follow-up if wanted). Abstract Research and RRR case types, which have no hearing and so no SOS Documents section. `RdpExpiryAlert.cls` was not changed by DDT-1224.

## 4. How it works (background)

Checked against production before building:

- An "SOS case" is an **Opportunity** whose `Case_Type__c` is one of the six SOS values. The ticket guessed an "SOS record type" on Opportunity; there is none.
- The Hired Opportunity page is a Lightning page, so field-level conditional visibility works. The SOS Documents section shows when `BB_Blocks__c` contains `SOS Documents`, which is true for the four hearing case types.
- Existing SOS groups did not fit: one is a queue (queues were ruled out) and the other is a large dashboard-sharing group.
- Production was near its custom object limit, so the alert state lives in two Opportunity fields rather than a log object.

The alert, once built:

```
GlobalMinuteScheduler runs every minute
  -> at 08:37 America/Chicago, dispatchRdpExpiryAlert queues RdpExpiryAlert (first rider, wrapped)
  -> query: RDP__c = true AND RDP_Expiry_Date__c between today and today + 6 months (no stage filter)
  -> pick the most urgent stage due (1, else 3, else 6 months)
  -> skip if the stamp already shows that stage, or a more urgent one, for this expiry date
  -> email each active user in SOS_RDP_Alerts, then stamp the case
```

## 5. Root cause

### DDT-723

Not applicable: a new feature, not a defect.

### DDT-1224

`RdpExpiryAlert.recipientIds()` reads the real `SOS_RDP_Alerts` group and returns its active users **ordered by Name**. The test's `@TestSetup` added the person running the test to the existing group but left the group's real members in place, then asserted that the first message sent (`lastSent[0]`) was addressed to the runner.

When the class was deployed on 2026-09-29 the group was empty, so the runner was the only member and the test passed. The six SOS team members were added to the group within the hour, after the live email test. From then on, the first message went to whichever member sorts first by name, so the test passed only for runners whose names sort before that member, and failed for everyone else.

**The lesson:** a test that reads real configuration data (group members, custom settings, users) passes or fails depending on what that data holds when it runs. It can pass at deploy time and fail days later with no code change.

## 6. Solution

### DDT-723 (PR #535)

- **Ride the dispatcher, no new scheduled job.** `RdpExpiryAlert` runs from `GlobalMinuteScheduler` at 08:37 Chicago time, first in `execute()`, wrapped so a failure cannot stop the riders below it. It only enqueues a Queueable, so it cannot throw an uncatchable batch limit exception. "Today" is computed in Chicago time, not the running user's timezone.
- **A standing query plus a stamp, not "exactly 180 days out".** A dispatcher branch gets one attempt at one minute. If the job only fired on the exact day, missing that minute would lose the alert. Instead it looks at every permit expiring within six months and stamps the last stage sent. A missed day is sent the next day; a permit entered late gets only the alert due now; changing the expiry date (a renewal) re-arms all three.
- **Selective query.** Bounded on `RDP_Expiry_Date__c` on both sides, so it does not scan all Opportunities. At most 100 cases a run; the rest go the next day.
- **Recipients are a public group,** so membership is changed in Setup with no code change. Emails go one per user from the SOS team's org-wide address, with no Task logged.
- **Failure handling.** A case where every email failed is not stamped, so it is retried tomorrow. A failed stamp is logged at ERROR and the case is emailed again the next day: a repeat email is noise, a lapsed permit is worse.
- **Off switch.** Set the Custom Label `RDP_Expiry_Alert_Enabled` to `false`. An empty group also sends nothing and stamps nothing.

### DDT-1224 (PR #617)

In `@TestSetup`, when the group already exists, delete its existing members before adding the runner. Test DML is rolled back when the test ends, so the real group is untouched.

```apex
} else {
    groupId = g[0].Id;
    delete [SELECT Id FROM GroupMember WHERE GroupId = :groupId];
}
insert new GroupMember(GroupId = groupId, UserOrGroupId = UserInfo.getUserId());
```

A smaller alternative was considered: find the runner's message in `lastSent` instead of taking `[0]`. Clearing the group was chosen because it makes the runner the only recipient, so every assertion in the suite sees the same data regardless of who runs it.

## 7. Testing

**DDT-723.** Production check-only validation ran `RdpExpiryAlertTest` and `GlobalMinuteSchedulerTest`, 42 of 42 passing, 15 of 15 components; coverage 92.4% on `RdpExpiryAlert` and 76.5% on `GlobalMinuteScheduler`. The deploy itself reported 44 of 44 tests passing. That validation compiled the Apex and ran its tests; it sent no email and rendered no page.

Live checks after the deploy (2026-09-29):

- **Emails received** for one open and one closed test case, both with expiry six months out. During the test the group held only one developer, so the SOS team got no test emails; it was restored to the six members right after.
- **No repeats:** a second run the same day sent nothing.
- **Section position** confirmed on a live Hired SOS hearing case.
- **Cleanup:** the test case was deleted and RDP cleared on the closed test case; 0 production cases had RDP ticked afterwards.

**Not verified live:** the three detail fields appearing when RDP is ticked, because that needs RDP saved on a real client case. The page rule is the same one the Blueback already uses.

**DDT-1224.** Validated check-only in production by a runner whose name sorts after the group's first member, 18 of 18 passing. The done-when condition on the ticket was the suite passing for runners on both sides of that name.

## 8. Step by step

For DDT-723, in this order:

1. **Confirm the object and page against production:** which field identifies an SOS case, which Lightning page the Hired case uses, and whether a suitable group already exists.
2. **Make the edits on production's current copy** of `GlobalMinuteScheduler`, its test and the flexipage (retrieve and diff first).
3. **Deploy any other change the dispatcher now depends on first.** The deploy doc records that `GlobalMinuteScheduler` had gained a call to another unreleased class, which would fail to compile without it.
4. **Validate at deploy time as the deployer, then quick deploy,** using only this change's files and a labels file containing only the new label (deploying the repo's whole labels file would overwrite every production label).
5. **Run `scripts/apex/ddt-723-assign-rdp-fields.apex`** to assign `DDT_RDP_Fields`. It is idempotent.
6. **Check that One-year and Five-year are offered on the Hired record type;** enable them in Setup if not.
7. **Run `scripts/apex/ddt-723-seed-sos-rdp-alerts-group.apex`** to add the SOS members. It refuses to run if any member is missing or inactive.
8. **Open an SOS hearing case** and check the RDP section's position and visibility.
9. **Prove the alert:** on one open and one closed test case, tick RDP and set expiry to today plus six calendar months, then wait for the 08:37 run (or call `RdpExpiryAlert.run(Date.today());` in anonymous Apex) and confirm the emails arrived.

For DDT-1224:

1. Add the `delete` line shown in section 6 to `RdpExpiryAlertTest`'s setup.
2. Validate check-only in production with `--tests RdpExpiryAlertTest`, run by someone whose name sorts after the group's first member.
3. Merge and deploy.

## 9. Rollback

- **Stop the emails:** set the Custom Label `RDP_Expiry_Alert_Enabled` to `false`; nothing is queued from the next minute.
- **Remove the rider:** redeploy the previous `GlobalMinuteScheduler`.
- **Fields:** additive, and can stay.
- **DDT-1224:** reverting the test change brings back the runner-dependent failure; there is no reason to.

## 10. Known limits

- Every Hired SOS case is closed in Salesforce (Closed Won means hired), so real RDPs sit on closed cases. That is why the alert does not filter on stage; adding a stage filter later would silently stop it.
- Mobile pages do not show the RDP section.
- Group nesting is followed five levels deep; roles inside the group are not expanded.
- `RdpExpiryAlertTest` still uses the real group record when it exists; it is now independent of the members, but not of the group's developer name.

## 11. Reuse for similar issues

**How to recognise the DDT-1224 kind of failure:** a test passes for some people and fails for others, or passed at deploy time and fails days later, with no code change. Look for a test that reads real configuration data and asserts on order or count.

- In `@TestSetup`, put the configuration into a known state (clear it, then add exactly what the test needs). Test DML rolls back.
- Avoid asserting on list position when the order comes from data, such as names.
- Run a validation as more than one user before calling a test fixed.

**For a new date-based alert:** ride the shared dispatcher rather than adding a scheduled job, use a standing window query plus a stamp so a missed day recovers, compute "today" in a fixed timezone, send to a public group, and prove it with a received email on both an open and a closed record.
