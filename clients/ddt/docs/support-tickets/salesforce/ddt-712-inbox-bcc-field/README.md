# BCC field in the shared inbox composer

| | |
|---|---|
| **Ticket** | DDT-712 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-13 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-11 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC) |
| **Area** | Shared Inbox: replying to and composing emails |
| **Related** | DDT-663 (attachment size guard in the same composer, left untouched) |
| **Code** | PR #158 and PR #159 in the `ddt-salesforce` repo |

---

## 1. Summary

The Shared Inbox email composers had **To** and **CC** but no **BCC**, so staff had to expose a recipient in CC or send a second email. An optional BCC field was added to both composers. The BCC list is stored on the `EmailMessage` as an internal record, but is deliberately never shown in the thread view, never pre-filled on Reply All, and never printed, and tests now enforce that.

## 2. What was reported

On 2026-08-11 a staff member asked for BCC in the Shared Inbox composer: the screenshot showed a pre-filled To, a CC field marked optional, and Subject straight after, with no BCC. The need was to blind-copy recipients (for example an internal colleague) without disclosing them.

The screenshot showed the **reply composer** (reached by Reply, Reply All or Forward). The New Email modal had the same gap, so both were changed.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `emailReplyComposer` | LWC | BCC input in both the mobile and desktop template branches; passes `bccAddress` to Apex |
| `emailComposeNewModal` | LWC | BCC input with its own recipient typeahead |
| `EmailReplyService` | Apex class | `bccAddress` parameter, `setBccAddresses`, stores `EmailMessage.BccAddress`; `sendFailureMessage` |
| `EmailComposeService` | Apex class | Same as above for new emails |
| `EmailReplyServiceTest`, `EmailComposeServiceTest` | Apex tests | Call sites updated; BCC tests; failure-message tests |
| `EmailInboxControllerTest`, `EmailThreadPrintControllerTest` | Apex tests | Guards: the thread view and the printable thread must never carry BCC |
| Both composers' `__tests__` | Jest | 8 new cases |

**Out of scope**

- **No new field, permission set or layout.** `EmailMessage.BccAddress` is a standard field.
- **Showing BCC anywhere in the inbox UI or on printouts.** Deliberately excluded (see section 6).
- **The long-recipient-list defect** described in section 10. It predates this change and needs its own ticket.
- **Extracting the shared address-parsing code** into one utility. Left duplicated to keep a same-day production deploy small.

## 4. How it works (background)

```
User fills To / CC / (now) BCC in a composer LWC
  -> Apex service (EmailReplyService or EmailComposeService)
  -> splitAddresses() parses each field (handles , and ; and "Name <email>")
  -> Messaging.sendEmail sends it
  -> an EmailMessage record is inserted on the inbox case
  -> thread view and printable thread read EmailMessage through their own DTOs
```

The printable thread can be handed to clients, which is why anything shown there matters.

## 5. Root cause

Not applicable: this was a feature request. The composers simply never had a BCC input or parameter.

## 6. Solution

**BCC is stored but never displayed.** `BccAddress` is saved so the firm keeps a record of who was blind-copied. It is not added to the thread view's message DTO, not shown next to the "CC:" line, and not added to the printable thread. Putting it on a printout would disclose blind recipients and break the ticket's own requirement. That guarantee used to rest on an *absence* (no BCC field on the DTO), which nothing enforces, so two tests now create a message with a real BCC, serialise the thread and the print payload, and **fail if either carries a BCC field or the blind address**.

**BCC never pre-fills.** There is no default BCC on the reply composer and the Reply All logic that folds the original To and CC into CC is untouched. If an inbound message's BCC were ever folded in, a blind recipient would be broadcast to the whole thread. The absence is commented in both files so it does not look like an oversight.

**The stored value is what was actually sent.** `splitAddresses()` drops anything without an `@`. Storing the raw entry would record recipients who were never copied (typing `Jane Doe, blind@x.com` copies one address and would have logged two). `BccAddress` stores the parsed list joined with `; `, or null when nothing valid was entered:

```apex
// shape of the stored value
String.join(splitAddresses(bccAddress), '; ')
```

**Failure messages name a possibility, never a cause.** If a send fails, a BCC was entered and the platform error mentions BCC, the user is told the BCC recipients *may* be the cause (an org-wide compliance BCC setting, or an invalid BCC address), and the raw platform error is always appended. It does not assert the compliance setting, because a per-user automatic-BCC setting and a simple typo in an address produce matching error text. This logic lives in a `@TestVisible` `sendFailureMessage(...)` in each service so it can be tested at all.

**Reused parsing.** `splitAddresses()` already handled To and CC, so BCC gets the same parsing.

## 7. Testing

- **Apex:** BCC tests in both service test classes, the two never-display guards, and `testSendFailureMessage_OffersBccAsPossibilityNeverAsserts` covering all four branches.
- **Jest:** 15 of 15 in the two composers (7 existing plus 8 new); eslint and prettier clean on changed files.
- **Validated against production, scoped, 2026-08-12:** 8 of 8 components, 67 of 67 tests, no coverage warnings.
- **A platform behaviour found by a failing test:** Salesforce normalises address-list fields on save, so commas come back as semicolons. The first test asserted the raw string round-tripped and failed in the sandbox. This applies to `ToAddress` and `CcAddress` too.

**Live check (queried 2026-11-05):**

| Query | Result |
|---|---|
| `SELECT COUNT() FROM EmailMessage WHERE BccAddress != null AND CreatedDate >= 2026-08-13T00:00:00Z` | **554** |

**Not covered by automation:** that a BCC recipient actually receives the mail and that To and CC recipients see no trace of it. That needs a real send, and the deploy notes left it to the requester in the sandbox.

## 8. Step by step

1. **Identify which composer the user means** from the screenshot (the reply composer has no "From inbox" dropdown). Change every composer with the same gap.
2. **Add the input** to every template branch (this one has separate mobile and desktop branches).
3. **Add the `bccAddress` parameter** to the Apex service, parse it with the existing `splitAddresses()`, call `setBccAddresses` only when the list is non-empty, and store the parsed list on `EmailMessage.BccAddress`.
4. **Add tests that fail if BCC ever appears** in the thread DTO or the print payload.
5. **Check the org's compliance BCC setting is off**, or `setBccAddresses` throws at runtime: `SELECT ComplianceBccEmail FROM Organization LIMIT 1` must return null (it did on 2026-08-12).
6. **Deploy Apex, LWC and tests together.** If the LWC lands first it sends a parameter the old Apex does not declare and every reply fails.
7. **Validate against production with the four changed test classes**, get the PR approved, then deploy. No post-deploy script, backfill or permission step.
8. **Have the requester send a real BCC** and print a thread that had one.

## 9. Rollback

Redeploy the previous versions of the two services, the two LWCs and their tests **together**. An empty BCC was already a no-op, so existing rows are unaffected either way; `BccAddress` values already stored stay on their records.

## 10. Known limits

- **A recipient list over 4,000 characters** sends the email and then fails to save the `EmailMessage`, so the user sees "Send failed" for a message that went out. This already applied to To and CC; BCC widens it slightly. Needs a pre-send length check on all three fields.
- **The BCC block and failure-message logic are copied in both services.** The next change has to be made twice, and the audit guarantee depends on both copies staying in step.
- **To and CC still store the raw entry**, so they can disagree with what was sent; BCC is stricter.

## 11. Reuse for similar issues

- **When adding data that must stay private, test for its absence** on every outward path (DTOs, printouts, exports). An absence that is only a convention will eventually be "fixed".
- **Store what was actually done, not what was typed**, when the record is an audit trail nobody sees in the UI.
- **Never assert one cause from a substring of an error message.** Offer the possibilities and append the raw error.
- **Deploy a changed LWC-to-Apex signature in one deployment** with its callers and tests.
- **Address fields come back with semicolons.** Do not string-match them against the raw input.
