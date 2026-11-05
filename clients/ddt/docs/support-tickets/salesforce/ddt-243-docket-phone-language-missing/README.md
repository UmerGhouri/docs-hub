# Daily Docket not showing client phone and language

| | |
|---|---|
| **Ticket** | DDT-243 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-30 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC) |
| **Area** | Daily Docket (court calendar): the appearance detail popup |
| **Related** | DDT-240 (Daily Docket failed to load the same afternoon) |
| **Code** | Commit `f79209b79` in the ddt-salesforce repo (records production's restored version in git) |

---

## 1. Summary

The Daily Docket's appearance popup showed a dash for the client's **Phone** and **Language**, although both were filled in on the client record. The feature that shows them had been live for about a week, but it existed only in production and on a feature branch. That afternoon an older copy of the Docket Deck code was deployed over it and removed it. It was back the same evening, and the next day production's version was committed to git so it could not be lost the same way again.

## 2. What was reported

- **2026-06-29:** on the Daily Docket, the detail popup for a court appearance showed:
  - Court Action, Case #, Balance Due: filled in
  - **Phone: —**
  - **Language: —**

  The client record had both values. Attorneys use these on the morning of court to reach the client and to know whether an interpreter is needed.

One appearance was given as the example. Because the cause was a code revert, every appearance on the docket was affected until the fix.

## 3. Scope

**In scope**

| Component | Type | Role |
|---|---|---|
| `DayBackCloneController` | Apex class | Its row type (`EventDTO`) carries `clientPhone` and `clientLanguage` |
| `DocketDeckEditController` | Apex class | `getAppearancesForDateLive` reads the client's phone and language and fills them in |
| `docketDeckEventModal` | LWC | The popup that shows Phone and Language |
| `docketDeckCalendar` | LWC | The Daily Docket screen |

**Out of scope**

- **No new fields, objects or permission sets.** The backfill commit says the drift was code only.
- **The data.** Phone and language were correct on the client records. Nothing needed repairing.

## 4. How it works (background)

```
docketDeckCalendar calls DocketDeckEditController.getAppearancesForDateLive(date)
  -> DayBackCloneController builds one EventDTO per Court_Appearance__c
  -> getAppearancesForDateLive queries each appearance's Court Case (Opportunity) and its client Account:
       Opportunity__r.Account.Phone
       Opportunity__r.Account.PersonMobilePhone
       Opportunity__r.Account.Language__c
  -> sets EventDTO.clientPhone   = Phone, or PersonMobilePhone if Phone is blank
  -> sets EventDTO.clientLanguage = Language__c
docketDeckEventModal shows clientPhone (formatted) and clientLanguage, or a dash when blank
```

The phone and language feature was added on 2026-06-23 and deployed straight to production. Its commit sat on a feature branch and never reached `main`.

## 5. Root cause

The ticket guessed a field-mapping or query problem. It was not: the query was right. The code that runs it had been **removed from production**.

1. The feature changed two classes: two new fields on `DayBackCloneController.EventDTO`, and the code in `DocketDeckEditController` that fills them.
2. Those changes were in production and on a feature branch, but **not on `main`**.
3. On 2026-06-29 a separate change to `DayBackCloneController` was made from the `main` copy and deployed to production. The `main` copy had no `clientPhone` or `clientLanguage`.
4. The commit that later restored git (`f79209b79`) records it directly: an older local copy of the Docket Deck components was deployed to production that afternoon and **silently reverted the phone and language enrichment**.

So the popup received rows with no phone or language and showed a dash.

**The lesson.** A feature that is live in production but not on `main` will be removed by the next person who deploys from `main`.

## 6. Solution

### 6.1 Restore the feature in production

The developer who owned the Docket Deck reported it fixed that evening. The production retrieve the next morning (6.2) confirms the enrichment was back in production by then; the ticket does not record the deploy itself. Users were asked to **fully close and reopen the Salesforce app** to pick up the change.

The restored enrichment in `DocketDeckEditController.getAppearancesForDateLive`:

```apex
// Client phone (prefer the primary Phone, fall back to Person mobile) + language.
Account acct = (opp != null) ? opp.Account : null;
if (acct != null) {
    r.clientPhone = String.isNotBlank(acct.Phone) ? acct.Phone : acct.PersonMobilePhone;
    r.clientLanguage = acct.Language__c;
}
```

### 6.2 Record production's version in git

On 2026-06-30 the current production versions of the four Docket Deck components were retrieved and committed (`f79209b79`, merged the same day). From then on, a deploy from `main` carries the feature instead of removing it.

Why retrieve rather than re-apply the feature branch: production also held other direct-to-prod Docket Deck changes made after the feature branch (client attendance badges, a drag-to-reassign fix, a Balance Due colour legend). Retrieving production captured all of them at once.

## 7. Testing

Not recorded in the ticket beyond the developer's report that it was fixed. The acceptance check was that Phone and Language show in the popup for the example appearance.

**Not covered:** no automated test checks that the popup shows phone and language, and no check stops a deploy that removes them.

## 8. Step by step

If Phone or Language show a dash on the Daily Docket:

1. **Check the client record.** Open the court case → client Account. If `Phone`, `PersonMobilePhone` and `Language__c` are blank, the dash is correct.
2. **Check one more appearance.** If every appearance shows a dash, it is the code, not the data.
3. **Check production's `DocketDeckEditController`.** It should read `Opportunity__r.Account.Phone`, `PersonMobilePhone` and `Language__c`, and set `clientPhone` and `clientLanguage`.
4. **Check production's `DayBackCloneController`.** `EventDTO` should declare `clientPhone` and `clientLanguage`.
5. **If either is missing,** find the deploy that removed it (Setup → Deployment Status), and redeploy the version from `main`. If `main` does not have it either, retrieve the last good version first.
6. **Ask users to close and reopen the app.**
7. **Commit production's version to git** the same day.

## 9. Rollback

Not needed. Rolling back would remove the phone and language again, which is the bug.

## 10. Known limits

- **Phone falls back to the person mobile.** If the Account's main `Phone` is blank, the popup shows `PersonMobilePhone`. Any other phone field is not shown.
- **The client is reached through the appearance's Court Case.** An appearance with no Court Case shows no phone or language.
- **Nothing automated protects this.** A future deploy of an old copy of either class would remove the feature again.

## 11. Reuse for similar issues

**How to recognise one:** a feature that worked last week is suddenly gone for everyone, with no error, and the data behind it is fine.

- **Suspect a revert before a bug.** Ask what was deployed since it last worked.
- **Check that the feature exists on `main`.** If it only exists in production, the next deploy from `main` removes it.
- **Retrieve production and commit it** when production is ahead of git, one component at a time, reading each diff.
- **Tell mobile users to close and reopen the app** after a fix, or they may keep seeing the old behaviour.
