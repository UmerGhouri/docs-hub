# RingCentral SMS tasks not auto-logging in the sandbox

| | |
|---|---|
| **Status** | Resolved 2026-10-16 |
| **Found by** | Support team, during sandbox testing |
| **Systems** | RingCentral for Salesforce (Open CTI package `rcsfl`), Salesforce |
| **Area** | "SMS/MMS between X and Y" tasks on leads |
| **Change log rows** | 13–15 |

---

## 1. Summary

Texts sent and received in the sandbox were not being saved as "(MM/DD/YYYY) SMS/MMS between X and Y" tasks, while production logged them fine. Two causes: the sandbox call center still pointed at production, and the tester's browser used a day/month date format, which the RingCentral panel can't handle. We fixed the call center URL and set Chrome to English (United States).

## 2. How SMS logging actually works (background)

This is the important part, because it isn't obvious:
- SMS tasks are written by the **RingCentral for Salesforce panel** (Open CTI, package `rcsfl`) **running in the number owner's open Salesforce browser tab**. There is **no server-side sync**: the "RingCentral for Salesforce" connected app has 0 logins.
- Admin setting `autoLogSms = true`. While the panel is open, a text is logged about **15–20 seconds** later. Texts sent while it's closed are caught up in one burst when it loads.
- **One task per conversation per day**; later texts that day are appended.
- If that day's task (or its lead) is deleted, auto-log for that conversation **stops for the rest of the day**. A manual **Log** in the panel brings it back.
- Flow `Lead_Link` attaches unlinked SMS tasks to a lead by `Normalized_Phone__c`.

## 3. Root cause

1. **Call center pointed at production.** The sandbox copy of the RingCentral call center had the production adapter URL.
2. **Browser date format.** The panel builds the day's task key (`rcsfl__CALL_UNIQUE_ID__c`) and the subject from the browser-formatted date. With a day/month format (UTC+5 locale) it produced `(NaN/NaN/NaN)` titles, and auto-log silently skipped.

## 4. Solution

| Fix | Where |
|---|---|
| Call center adapter URL → `https://[sandbox-org]--rcsfl.sandbox.vf.force.com/apex/OpenCTIIndex` | Sandbox Setup → Call Centers → RingCentral |
| User added to the RingCentral call center | Sandbox user record |
| Chrome language → **English (United States)** | Tester's browser (not an org change) |

## 5. Testing

- After the browser change, a text at 07:12:34 UTC was auto-logged at 07:12:54.
- Deleting and recreating a lead with the same number broke auto-log for that conversation; re-linking in the panel fixed it.

## 6. Known limits

- Logging depends on someone having the panel open. Texts are only logged by the **owner of the line** being logged in.
- That's why we later built the [Text Messages panel](../2026-10-20-text-messages-panel/README.md), which reads texts straight from RingCentral instead of relying on logged tasks.

## 7. Reuse for similar issues

- **SMS tasks missing for one user only?** Check their browser language/date format first.
- **SMS tasks missing in a sandbox?** Check the call center adapter URL points at the sandbox.
- **Auto-log stopped mid-day for one conversation?** Someone deleted that day's task or lead. Use **Log** in the panel.
