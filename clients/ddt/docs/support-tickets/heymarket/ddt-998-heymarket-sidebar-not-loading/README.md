# Heymarket sidebar not loading in Salesforce

| | |
|---|---|
| **Ticket** | DDT-998 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-11 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-10 |
| **Resolved by** | Support team |
| **Systems** | Heymarket, Salesforce (Lightning record page) |
| **Area** | Texting clients from the Salesforce client record |
| **Related** | None |
| **Code** | None. Nothing was changed. |

---

## 1. Summary

A staff member reported that the Heymarket texting panel was not loading on a client record in Salesforce. It was there: the panel sits lower down the record page, and the screenshot showed only the top of the page. Scrolling down showed the panel, and it was confirmed working.

## 2. What was reported

On a client (Person Account) record, the staff member saw no Heymarket panel at all and could not text the client from Salesforce. The screenshot showed the standard record layout with no Heymarket widget. One user was affected.

## 3. Scope

**In scope:** confirming whether the Heymarket panel was present and working for the user.

**Out of scope:** any change to the page, the Heymarket package, licences or permissions. None was needed.

## 4. How it works (background)

The Heymarket panel is a component placed on the client record page, below the fields shown at the top. On a normal screen it is out of view until you scroll down the page.

## 5. Root cause

Not a defect. The panel was on the page but below the visible area. The screenshot did not show anything below the top of the page.

The ticket's suggested checks (managed package licence, permission set, app assignment, Heymarket account link) were not the cause.

## 6. Solution

No change. The user was told to scroll further down the record page to reach the Heymarket panel. The support team then confirmed with a screenshot that it was there and working.

## 7. Testing

The panel was opened on a client record and shown to be working, with a screenshot posted on the ticket. Texting end to end from that user's session is not recorded separately in the ticket.

## 8. Step by step

When someone reports "the Heymarket panel is missing":

1. Open the same record and **scroll all the way down** the page. Check whether the panel is there.
2. Ask for a screenshot of the full page, scrolled to the bottom, not just the top.
3. If it is there, show the user where it sits.
4. Only if it is genuinely missing, move on to the real checks: the user's Heymarket licence and permission set, whether the component is on the page for that app and record type, and whether the user's Heymarket account is linked.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The panel's position low on the page makes it easy to miss. Moving it higher was not requested or done.

## 11. Reuse for similar issues

**How to recognise one:** "a panel or component is missing" with a screenshot that shows only the top of the page.

- **Reproduce on the same record and scroll** before checking licences or permissions.
- **Ask for a full-page screenshot.** A partial one can't show that something is missing.
- **Check placement before access.** The cheapest explanation is that it's on the page but out of view.
