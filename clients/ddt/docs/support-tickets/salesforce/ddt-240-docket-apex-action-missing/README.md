# Daily Docket error: no Apex action available

| | |
|---|---|
| **Ticket** | DDT-240 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-30 |
| **Priority** | High |
| **Reported by** | Attorney, 2026-06-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC) |
| **Area** | Daily Docket (court calendar) |
| **Related** | DDT-243 (phone and language missing from the same screen, same afternoon), DDT-237 (Apex class access for another controller, same user, same day) |
| **Code** | Not recorded in the ticket. Related repo history: the Docket Deck backfill commit `f79209b79` in the ddt-salesforce repo |

---

## 1. Summary

The Daily Docket stopped loading and showed `No apex action available for DocketDeckEditController.getAppearancesForDateLive`. It worked that morning and failed for several users, on mobile and desktop. The ticket does not record the cause or the fix. The repo history shows that the Docket Deck components were being changed directly in production that day, and that an older copy was deployed over them that afternoon.

## 2. What was reported

- **2026-06-29:** an attorney opened the Daily Docket on mobile and got:

  `apex://DocketDeckEditController: No apex action available for DocketDeckEditController.getAppearancesForDateLive`

  The docket did not load at all, and **Retry** did not help.
- **Same day, a few minutes later:** a second staff member reported the same error **on desktop**, and said it had been working for everyone that morning.

Affected: everyone who opened the Daily Docket that afternoon. The ticket gives no count.

## 3. Scope

**In scope**

| Component | Type | Role |
|---|---|---|
| `DocketDeckEditController` | Apex class | Serves `getAppearancesForDateLive`, the call the docket makes to load a day |
| `DayBackCloneController` | Apex class | Builds the appearance rows that `getAppearancesForDateLive` enriches |
| `docketDeckCalendar` | LWC | The Daily Docket screen that calls the method |

**Out of scope**

- **Permission sets.** The ticket suspected missing Apex class access, because the same user had a class-access ticket (DDT-237) earlier that day. The second report rules that out as the main cause (see 5).

## 4. How it works (background)

```
User opens the Daily Docket and picks a date
  -> docketDeckCalendar (LWC) calls DocketDeckEditController.getAppearancesForDateLive
  -> DayBackCloneController.getCourtAppearancesForDate builds one row per court appearance
  -> getAppearancesForDateLive adds balance due, client phone and client language to each row
  -> the calendar renders the rows
```

`getAppearancesForDateLive` was added with Docket Deck v2 on 2026-06-22. At the time, the Docket Deck was being developed **directly in production**, so production was ahead of git.

**What the error means.** "No apex action available for `Class.method`" is what Lightning returns when it cannot find a usable `@AuraEnabled` method of that name on the class as it is deployed in the org. The method may be missing, may have lost `@AuraEnabled`, or the class may not be usable at that moment. A user who simply lacks access to an Apex class normally gets a different message, naming the class they have no access to.

## 5. Root cause

**Not recorded in the ticket.** The ticket offered two guesses: the method had been removed or renamed, or the user lacked access to the class.

What the sources do establish:

1. **It was not one user's permissions.** A second user hit the same error on desktop, and the page had worked for everyone that morning. A permission gap on one user does not do that.
2. **The method existed in git.** It is present in every repo copy of `DocketDeckEditController` from 2026-06-22 onward.
3. **Production was changed that afternoon.** The commit that later brought the Docket Deck back into git (`f79209b79`, 2026-06-30) records that on 2026-06-29 an **older local copy of the Docket Deck components was deployed to production**, which silently reverted the phone and language feature (DDT-243). It names the four components involved: `DayBackCloneController`, `DocketDeckEditController`, `docketDeckCalendar` and `docketDeckEventModal`.
4. **The phone and language feature lived on a feature branch only.** The commit that added it (2026-06-23) added two fields to the row class in `DayBackCloneController` and code in `DocketDeckEditController` that fills them. Neither change was on `main`. A same-day commit on `main` changed `DayBackCloneController` from the `main` copy, which did not have those two fields, and was deployed to production.

The ticket does not say whether that deploy is what broke `getAppearancesForDateLive`. It is the only production change to these components recorded for that afternoon.

**The lesson.** When a screen fails for everyone at once after working that morning, look for a deploy before looking at permissions.

## 6. Solution

**Not recorded in the ticket.** On the related ticket DDT-243, raised the same afternoon about the same screen, the developer who owned the Docket Deck reported it fixed that evening and asked users to **fully close and reopen the Salesforce app**. On 2026-06-30 the current production versions of all four Docket Deck components were retrieved and committed to git (`f79209b79`), so the next deploy from git would not revert them again.

## 7. Testing

Not recorded in the ticket. The ticket's acceptance check was that the Daily Docket loads on mobile without the error.

**Not covered:** no test checks that `getAppearancesForDateLive` is still callable after a deploy of the Docket Deck classes.

## 8. Step by step

How to work this error if it comes back:

1. **Find out who is affected.** Ask a second user, ideally on desktop. If everyone gets it, skip to step 3.
2. **If only one user gets it**, check their Apex class access to `DocketDeckEditController` (profile and permission sets).
3. **Check the class in production.** Setup → Apex Classes → `DocketDeckEditController`. Confirm it is valid and that `getAppearancesForDateLive` is present and `@AuraEnabled`.
4. **Check recent deploys.** Setup → Deployment Status. Look for a deploy that touched `DocketDeckEditController` or `DayBackCloneController` since the screen last worked.
5. **Compare production with git** for those classes before redeploying anything. Production may be ahead of git; redeploying a stale copy is how this kind of failure starts.
6. **Redeploy the correct version**, then ask users to fully close and reopen the app (mobile) or hard-refresh (desktop).
7. **Record production's version in git** the same day.

## 9. Rollback

Not applicable: the ticket records no change to roll back.

## 10. Known limits

- The Docket Deck was being edited directly in production at the time, so git could lag production. Any deploy of these components from a local copy can revert someone else's live work.
- The Salesforce mobile app may keep the failing state until it is fully closed and reopened.

## 11. Reuse for similar issues

**How to recognise one:** "No apex action available" on a screen that worked earlier the same day, for more than one user.

- **Ask a second user before touching permissions.** One user failing points to access; everyone failing points to a deploy.
- **Check Deployment Status first.** The answer is usually a deploy in the last few hours.
- **Check classes that share data structures.** A class that fills fields on another class's row type depends on that other class. Deploying an old copy of one can break the other.
- **Never deploy a component from a local copy without comparing it to production first.**
