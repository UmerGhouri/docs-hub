# Broken link in the Cook County payment instructions template

| | |
|---|---|
| **Ticket** | DDT-1429 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-29 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (document template attached to the disposition letter email) |
| **Area** | Court cases: disposition letter email for Cook County cases |
| **Related** | None recorded |
| **Code** | Not applicable: content change made in the org, no PR |

---

## 1. Summary

The "Payment Instructions: Cook County" template, attached automatically to the disposition letter email on Cook County court cases, ended with a link to the county clerk's traffic case search page that no longer worked. The link was replaced with the clerk's current case search page. The ticket first asked for a payment-site link instead; the team clarified that the firm deliberately does not send clients a Cook County payment link, so the case search link was the right replacement.

## 2. What was reported

On 2026-10-26 a staff member reported that the link near the end of the Cook County payment instructions was dead, so clients reading the disposition letter could not follow the instructions. The ticket proposed replacing it with the county's online payment site.

## 3. Scope

**In scope:** the one link at the end of the "Payment Instructions: Cook County" template.

**Out of scope:**

- **The automation that attaches the template** to the disposition letter email when the court case's `County__c` is Cook. Unchanged.
- **`County__c`.** Not modified.

## 4. How it works (background)

```
Disposition letter email sent on a Court Case (Opportunity)
  -> County__c = Cook
  -> "Payment Instructions: Cook County" is attached automatically
  -> the client follows the instructions, which end with a link to the clerk's case search
```

The link is static text in the template. When the county moves a page, the template keeps pointing at the old address until someone edits it.

## 5. Root cause

The old address in the template no longer worked, and the clerk's traffic case search is now at a different address. Nothing in Salesforce changed.

**The ticket's first proposed fix was wrong for the firm.** It asked for the county's online payment site. In the support thread a staff member explained that the firm does not add the payment link for Cook County: the payment site is too confusing for clients, who often enter the wrong information, so the firm attaches the payment instructions instead. The link that broke was the **case search** link, and its replacement was the clerk's new case search address.

## 6. Solution

1. **2026-10-26:** the link was first changed to the payment site, as the ticket asked.
2. **Same day:** staff clarified that the firm does not send a Cook County payment link and supplied the clerk's new case search address.
3. **By 2026-10-28:** the template's link was updated to the clerk's current traffic case search page, and the requester was asked to click it from the template and confirm.

Where the template is edited is not recorded in the ticket. It is not a file in the `ddt-salesforce` repo.

## 7. Testing

**Live check:** on 2026-10-29 the requester clicked the link and confirmed it opens the case search page, with a screenshot.

**Not covered:** no check is recorded that a newly sent Cook County disposition letter email carries the updated template, or that no other template still holds the old address.

## 8. Step by step

1. **Find the template** named "Payment Instructions: Cook County" and the link near its end.
2. **Ask what the link is for** before replacing it. Here it was the case search, not a payment page.
3. **Get the current address** from the staff who use it, and open it yourself to check it works.
4. **Replace both the link text and the link target**, if they differ.
5. **Search other templates** for the old address.
6. **Ask the requester to click the link from the template** and confirm the right page opens.

## 9. Rollback

Edit the link back to its previous value. There is no reason to: the old address no longer works.

## 10. Known limits

- **External addresses change without notice.** The template will break again the next time the county moves the page.
- **Other county templates** may hold links that have also moved. They were not checked as part of this ticket.

## 11. Reuse for similar issues

**How to recognise one:** a client-facing template or email has a dead external link.

- **Confirm the purpose of the link with the people who send it** before replacing it. The replacement a ticket suggests may not be what the firm wants clients to see.
- **Check the replacement live** by clicking it from the template itself, not just from a browser.
- **Search every template for the old address** so one fix covers every copy.
