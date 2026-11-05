# Will County case search link in Clerk Links

| | |
|---|---|
| **Ticket** | DDT-476 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-22 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning app utility bar) |
| **Area** | The Clerk Links panel of court search links |
| **Related** | None recorded |
| **Code** | None. Changed in the org |

---

## 1. Summary

Staff asked for a new link, **Will Case Search (login)**, in the Clerk Links panel, directly below the existing **Will Case Search** link. It points to the Will County court's login-required attorney portal, matching the existing `(login)` entries for Cook and DuPage. The link was added in the org on 2026-07-22.

## 2. What was reported

On 2026-07-21 a staff member asked for one-click access to the Will County login portal alongside the public Will Case Search link. No error; a small enhancement.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Clerk Links | Rich Text item on the Lightning app utility bar | New link `Will Case Search (login)` below `Will Case Search` |

**Out of scope**

- Any other links in the panel.

## 4. How it works (background)

Clerk Links is not a custom component. It is a **Rich Text** utility item, labelled `Clerk Links`, on the Lightning app utility bar. Its content is a list of formatted links to court and clerk search sites, grouped by county: Cook, Chicago, DuPage, Lake, Kane, McHenry, Will, and inmate and Wisconsin searches. Some counties have a second `(login)` link for the portal that needs an attorney login.

In the repo the same Clerk Links item appears in three utility bar flexipages: `DDT_Console_UtilityBar`, `Driver_Defence_Team_UtilityBar` and `HVSConsole_UtilityBar`.

## 5. Root cause

Not applicable: this was a request for a new link, not a fault.

## 6. Solution

The new link was added to the Clerk Links rich text, directly under `Will Case Search`, with the label `Will Case Search (login)` and the same styling as the other `(login)` entries (bold county name, underlined label, opens in a new tab). Confirmed done on 2026-07-22 with a screenshot of the panel.

The exact method (App Manager in Setup, or a metadata deploy) is not recorded in the ticket. **The repo's copies of the three utility bars do not yet contain the new link**, so the change was made in the org and git lags it.

## 7. Testing

Checked by viewing the panel after the change (screenshot on the ticket). No automated tests apply.

**Not covered:** whether all three utility bars were updated. Not recorded in the ticket.

## 8. Step by step

Which screen was used is not recorded in the ticket. A Rich Text utility item is edited on the app's utility bar, so:

1. **Open the app's utility bar** (Setup, App Manager, edit the app, **Utility Items**) and select the `Clerk Links` Rich Text item.
2. **Add the new line under `Will Case Search`**, labelled `Will Case Search (login)`, linking to the Will County login portal and formatted like the other `(login)` lines.
3. **Save**, reload the app, and click the link to check it opens the right site.
4. **Record the changed utility bar in git** so the repo matches production.

## 9. Rollback

Delete the line from the rich text and save. Safe: nothing else depends on it.

## 10. Known limits

- **The link list is copied per app.** Each utility bar holds its own copy of the Clerk Links text, so a link added to one app does not appear in the others.
- **Court sites change addresses.** A dead link has to be fixed by hand in each copy.

## 11. Reuse for similar issues

**How to recognise one:** any request to add, rename or fix a link in Clerk Links.

- **It's a Rich Text utility item**, not code. Edit it in App Manager.
- **Find every copy first**: search the repo's `flexipages` for a link that is already in the panel.
- **Copy an existing line's formatting** so the new link matches.
- **Record the change in git** after editing the org, so the repo does not fall further behind.
