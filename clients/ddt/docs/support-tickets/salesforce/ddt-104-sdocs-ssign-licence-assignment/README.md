# S-Docs and S-Sign licence assignment and access errors

| | |
|---|---|
| **Ticket** | DDT-104, DDT-105, DDT-127, DDT-199, DDT-200, DDT-236, DDT-259, DDT-283, DDT-305, DDT-765 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-08 (first ticket closed); last ticket closed 2026-08-19 |
| **Priority** | High |
| **Reported by** | Intake team, Attorney, Firm staff and Firm operations, 2026-06-03 to 2026-08-18 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (S-Docs and S-Sign managed packages, user licences, permission sets), Dochly |
| **Area** | Generating documents and sending them for e-signature from Salesforce records |
| **Related** | DDT-197 (Payment Center package licence for one of the same users) |
| **Code** | None. Every fix was made in Setup. |

---

## 1. Summary

Over about ten weeks, individual staff were blocked from generating documents with S-Docs or sending them for signature with S-Sign. Each time the cause was access, not a bug: the user had no seat on the S-Sign or S-Docs **package licence**, or lacked the S-Docs permission needed on the record. The fix was to assign the missing licence or permission in Setup and have the user retry.

One ticket asked for an audit of every active user's S-Docs and S-Sign licences, and the last ticket turned out not to be a fault at all: S-Docs had been removed on purpose because the firm moved document generation and signing to Dochly.

## 2. What was reported

| Ticket | Date | How it showed up |
|---|---|---|
| DDT-104 | 2026-06-03 | An intake user could generate the immigration consultation and signature documents, but emailing them for signature failed with the S-Sign "not licensed" error. |
| DDT-105 | 2026-06-03 | A second intake user hit the same S-Sign "not licensed" error on the same documents. |
| DDT-127 | 2026-06-09 | A staff member saw an "application error" when trying to generate and send documents (reported as "DocuSign / SDocs"). The ticket asked to check the S-Docs permission sets. |
| DDT-199 | 2026-06-18 | A staff member clicking the S-Docs action got **"License Required — The Visualforce Page SDCreate1 is part of the AppExchange Package SDocs, and requires a license to use."** |
| DDT-200 | 2026-06-18 | The same user, later that day, hit the S-Sign "not licensed" error at the last step of sending documents. |
| DDT-236 | 2026-06-26 | The same user hit the S-Sign "not licensed" error again, eight days after DDT-200 was closed, while sending a document to a client who had already paid. |
| DDT-259 | 2026-07-01 | An attorney got "insufficient privileges" immediately on clicking the S-Docs button on a Court Case, before any template was shown. |
| DDT-283 | 2026-07-07 | An attorney hit the S-Sign "not licensed" error while emailing a representation agreement to a client. |
| DDT-305 | 2026-07-10 | Firm operations asked for every active user to be audited for S-Docs and S-Sign licences, so missing seats could be bought. |
| DDT-765 | 2026-08-18 | An attorney reported that S-Docs no longer appeared on his Salesforce toolbar. |

The S-Sign error was identical in every case except for the user named in it:

```
[SSException] Line: 5342. Message: Insert failed. First exception on row 0; first error:
CANNOT_INSERT_UPDATE_ACTIVATE_ENTITY, SSign.SSEnvelopeDocumentTrg: execution of AfterInsert caused by:
SSign.SSUtil.SSException: User (<user>) is not licensed to use S-Sign.
Please add the user to the list of licensed users at the S-Sign user licenses page. (SSign)
Error is in expression '{!initCreateEnvelope}' in component <apex:page> in page ssign:sscreateenvelope
```

Every ticket except the audit (DDT-305) affected one user. Six different users appear across those nine tickets.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| S-Sign package (`SSign` namespace) | Managed package licence | User added to the licensed users list (DDT-104, DDT-105, DDT-200, DDT-236, DDT-283) |
| S-Docs package | Managed package licence | User added to the licensed users list (DDT-199) |
| S-Docs permission sets (the ticket names `SDocs_User`, `SDOCS_SSIGN`, `Sdocs_App`, `SDocs_Administrator`) | Permission sets | Checked and assigned where missing (DDT-127, DDT-259). Which set was assigned is not recorded. |

**Out of scope**

- **Buying more seats.** DDT-305 asked for an audit to size a purchase. The purchase itself was a firm decision, not a Setup change.
- **Restoring S-Docs to the toolbar (DDT-765).** It had been removed on purpose; nothing was changed.
- **An onboarding checklist.** DDT-305 noted that "who has licence X" kept recurring and that a new-user checklist would prevent it. Building one was not part of these tickets.

## 4. How it works (background)

S-Docs and S-Sign are AppExchange managed packages. Using them needs **two separate things**, and missing either one gives a different error:

```
User clicks the S-Docs action on a record
  -> Salesforce checks the user holds an S-Docs PACKAGE LICENCE
       missing -> "License Required ... Visualforce Page SDCreate1 ... requires a license"
  -> S-Docs checks the user's PERMISSIONS on the S-Docs objects and the record
       missing -> "insufficient privileges" before any template appears
  -> document generates
User emails it for signature
  -> S-Sign creates an envelope; its trigger SSign.SSEnvelopeDocumentTrg checks
     the user is on the S-SIGN licensed users list
       missing -> SSException "... is not licensed to use S-Sign"
```

The S-Docs licence and the S-Sign licence are **separate seats**. A user can hold one and not the other, which is why the same user could be fixed for S-Docs (DDT-199) and still fail on S-Sign an hour later (DDT-200).

In this org the **Court Case** is the standard `Opportunity` object relabelled. There is no `Court_Case__c` object, although DDT-259 named one.

## 5. Root cause

### Licence tickets (DDT-104, DDT-105, DDT-199, DDT-200, DDT-236, DDT-283)

The user was not on the package's licensed users list. The error text says so directly. Why each user lacked the seat is not recorded in the tickets. DDT-305 gives the likely pattern: recently created users had each been set up with a different set of licences, because no onboarding checklist existed.

**DDT-236 is a recurrence.** The same user's S-Sign licence was assigned under DDT-200 on 2026-06-18, and the same error came back on 2026-06-26. Why the seat was lost is not recorded in the ticket.

### Permission tickets (DDT-127, DDT-259)

- **DDT-127:** the ticket's first guess was a missing S-Docs permission set. A fix was applied on 2026-06-10 and the user still got the same error afterwards; the support team then asked which action produced it. What finally resolved it is not recorded in the ticket.
- **DDT-259:** "insufficient privileges" on opening S-Docs is a permission problem, not a licence one (a missing licence gives the "License Required" page instead). Which permission was assigned is not recorded in the ticket.

### Audit (DDT-305)

Not a fault. The outcome of the audit is not recorded in the ticket, which was closed three minutes after it was created.

### Toolbar (DDT-765)

Not a fault. A staff member replied in the thread that S-Docs had been removed **on purpose because the firm now uses Dochly**. The ticket's guesses (permission set, layout change, package problem) were all wrong.

## 6. Solution

### 6.1 Assign the S-Sign licence

Setup → Installed Packages → S-Sign → **Manage Licenses** → Add Users → select the user. This is the "S-Sign user licenses page" the error refers to. Used for DDT-104, DDT-105, DDT-200, DDT-236 and DDT-283.

In DDT-104 the user still saw the error after the first assignment, even after logging out and back in and hard-refreshing. The support team made a second change later that day and the user was asked to retry; what that second change was is not recorded in the ticket.

### 6.2 Assign the S-Docs licence

Setup → Installed Packages → S-Docs → **Manage Licenses** → Add Users. Used for DDT-199.

### 6.3 Check the S-Docs permission sets

Setup → Users → the user → Permission Set Assignments. Compare with a user for whom S-Docs works, and assign the missing S-Docs set. Used for DDT-127 and DDT-259. After the change, ask the user to **log out and back in** before retrying.

### 6.4 Answer, don't fix

For DDT-765 nothing was changed. S-Docs had been retired from the toolbar in favour of Dochly.

## 7. Testing

- **DDT-259:** the support team confirmed it working, with a screenshot, on 2026-07-02.
- **DDT-104, DDT-105, DDT-199, DDT-200, DDT-236, DDT-283:** the user was asked to retry. Thanks were recorded from the user on DDT-199, DDT-200 and DDT-236; an explicit "it works now" is not recorded in the ticket.
- **DDT-127:** the user reported the same error after the first fix. The final check is not recorded in the ticket.
- **Not covered:** no check was recorded that these users still hold their seats. DDT-236 shows a seat can disappear.

## 8. Step by step

1. **Read the exact error.** "not licensed to use S-Sign" → S-Sign licence. "License Required ... SDocs" → S-Docs licence. "insufficient privileges" on opening S-Docs → permission set. A missing button → check whether the tool has been retired (DDT-765) before changing anything.
2. **Check the seat count.** Setup → Installed Packages → the package → the licence count. If all seats are used, the firm has to buy or free one first.
3. **Assign the licence** (6.1 or 6.2), or **the permission set** (6.3).
4. **Ask the user to log out and back in**, then regenerate the document and send it again.
5. **Confirm with the user** that it went through. If the same error returns, recheck the licensed users list: the seat may have been removed (DDT-236).

## 9. Rollback

Remove the user from the package's licensed users list, or remove the permission set assignment. Both are safe and immediate, and free the seat for someone else.

## 10. Known limits

- **Seats are finite.** Each licence assignment uses a paid seat. DDT-305 exists because the firm wanted every user covered and needed a count to buy against.
- **Licences can be lost.** DDT-236 was the same user and the same error eight days after DDT-200 was fixed. The cause is not recorded.
- **S-Docs has been replaced by Dochly.** By 2026-08-18 S-Docs was removed from the toolbar on purpose. New document-generation access requests should be checked against Dochly first.

## 11. Reuse for similar issues

**How to recognise one:** a single user is blocked in a managed package that works for everyone else, and the error names the user or says "license".

- **Licence and permission are different layers.** A package licence lets the user into the package at all; permission sets decide what they can do once inside. The error wording tells you which.
- **Packages with add-ons often need separate seats** (S-Docs and S-Sign here). Fixing one does not fix the other.
- **Compare with a working user** of the same role before assigning anything.
- **Have the user log out and back in** after any licence or permission change.
- **For "the button disappeared", ask before fixing.** It may have been removed deliberately.
- **Make licences part of onboarding.** DDT-305 counted four or five "who has licence X" questions in six weeks. A new-user checklist that lists each package seat removes most of these tickets.
