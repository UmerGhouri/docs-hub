# Access to the shared PI Gmail inbox and leads

| | |
|---|---|
| **Ticket** | DDT-126 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-09 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-06-09 |
| **Resolved by** | Support team |
| **Systems** | Google Workspace (Gmail shared inbox) |
| **Area** | Staff access to the PI intake inbox |
| **Related** | None recorded |
| **Code** | Not applicable: access change in Google Workspace, no code |

---

## 1. Summary

A staff member could not see the shared PI Gmail inbox or the leads arriving in it, which stopped them doing intake work. The support team granted access. The first grant went to the wrong shared inbox (the defense inbox), and was corrected to the PI inbox two days later.

## 2. What was reported

- **2026-06-09:** one intake staff member reported that the shared PI inbox and its incoming leads were not visible to them, and sent a screenshot of what they saw.
- **2026-06-09:** the support team granted access and closed the ticket.
- **2026-06-10:** the staff member replied that they had been given access to the **defense** shared inbox, not the PI one.
- **2026-06-11:** the support team updated the access. The staff member confirmed the same day that they were all set.

One person was affected.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Shared PI inbox | Google Workspace mailbox | Access granted to one staff member |

**Out of scope**

- Any change to who else can see the PI inbox, or to the defense inbox beyond the mistaken grant. Not recorded in the ticket whether the mistaken defense-inbox access was removed.

## 4. How it works (background)

The firm has more than one shared Gmail inbox, at least one for PI and one for defense. Leads for PI arrive in the PI inbox. A staff member sees a shared inbox only once they have been given access to it, either as a mailbox delegate or as a member of a Google Group, depending on how the inbox is set up. The ticket asked the support team to confirm which of the two the PI inbox is; the answer is not recorded in the ticket.

## 5. Root cause

The staff member had never been given access to the PI inbox. The first attempt to fix it granted access to the wrong shared inbox, because the request did not name the inbox precisely enough to tell the two apart.

## 6. Solution

Access to the shared PI inbox was granted to the staff member in Google Workspace on 2026-06-11, replacing the mistaken grant to the defense inbox. Whether this was done as mailbox delegation or Google Group membership is not recorded in the ticket.

## 7. Testing

- **Live check:** the staff member confirmed in the request thread on 2026-06-11 that they could use the inbox.
- **Not covered:** no separate check by the support team that incoming leads were visible.

## 8. Step by step

1. **Confirm which inbox the person needs**, by its exact name, before changing anything. The firm has a PI inbox and a defense inbox.
2. **Find out how that inbox is shared:** mailbox delegation or a Google Group.
3. **Grant access** in the Google Workspace admin console (delegate or group member, as found in step 2).
4. **Ask the person to open the inbox and confirm they can see new leads.** Do not close the ticket until they confirm.
5. **If access went to the wrong inbox first, remove that grant** so access stays limited to what the person needs.

## 9. Rollback

Remove the person as a delegate or group member of the PI inbox. Safe to do at any time; it only removes visibility.

## 10. Known limits

- The ticket does not record whether the mistaken grant to the defense inbox was removed. Anyone auditing shared-inbox access should check for it.
- Access is per person. A new intake hire needs the same grant; nothing in this fix adds it automatically.

## 11. Reuse for similar issues

- **Name the inbox exactly** in the request and in the confirmation. "The shared inbox" is ambiguous when there are several.
- **Close only after the user confirms.** This ticket was closed on the first grant and reopened in practice the next day because it was the wrong inbox.
- When granting access to one inbox by mistake, remove it again; shared inboxes carry client correspondence.
