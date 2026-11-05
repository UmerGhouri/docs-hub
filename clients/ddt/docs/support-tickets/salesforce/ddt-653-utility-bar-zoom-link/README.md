# Zoom links site in the utility bar

| | |
|---|---|
| **Ticket** | DDT-653 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-05 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-08-05 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning app utility bar) |
| **Area** | Staff tools: quick links in the utility bar |
| **Related** | None recorded |
| **Code** | None. Configured in production |

---

## 1. Summary

Firm operations asked for a link to the firm's "DDT Zooms" directory site to be added to the Salesforce utility bar, next to the other helpful links, so staff can open it from anywhere in Salesforce. The support team added it the same day.

## 2. What was reported

A request, not a fault. On 2026-08-05 firm operations asked for a new entry, labelled something like "DDT Zooms", in the utility bar's helpful-links list, opening the Zoom directory site in a new tab, visible to the same people as the other links.

## 3. Scope

**In scope:** one new link in the utility bar's helpful-links item.

**Out of scope:** the other links, and the Zoom directory site itself, which is hosted outside Salesforce.

## 4. How it works (background)

In the repo copy of the `DDT Console` app's utility bar (`DDT_Console_UtilityBar` FlexiPage), the helpful links are a **Rich Text** utility item labelled **"Other Links"**: a list of hyperlinks, each opening in a new tab. Adding a link means editing that rich text.

## 5. Root cause

Not applicable: this was a request for a new link.

## 6. Solution

The support team added the link and closed the ticket the same day. The ticket does not record which app or utility item was edited, or the final label.

The change was made directly in production. No file in the repo's `force-app` folder contains the new link (searched 2026-11-05), so the change has not been retrieved into git.

## 7. Testing

Not recorded in the ticket. The acceptance criteria were: the link appears in the helpful-links list, opens the site in a new tab, and is visible to the same audience as the other links.

## 8. Step by step

1. In Setup, open **App Manager** and edit the Lightning app that shows the helpful links (in the repo, the `DDT Console` app).
2. Open **Utility Items** and select the helpful-links item (a Rich Text item, "Other Links" in the repo copy).
3. Add a new line in the same style as the others. Set the link to open in a new tab.
4. Save, then reload Salesforce and check the link from the utility bar.
5. Retrieve the utility bar FlexiPage into git and open a PR as the record of the change.

## 9. Rollback

Remove the line from the rich text item and save. Safe.

## 10. Known limits

- **Git does not have this change.** If the link was added to `DDT_Console_UtilityBar`, deploying the repo's copy of that page would remove it. Retrieve the page before anyone deploys it.
- **Rich text links are hand-maintained.** If the site's address changes, the link must be edited by hand.

## 11. Reuse for similar issues

**How to recognise one:** a request to add, rename or remove a quick link for staff.

- **Find the existing helpful-links item** and match its style, rather than adding a new utility item.
- **Record the change in git** by retrieving the utility bar FlexiPage after editing it in Setup.
