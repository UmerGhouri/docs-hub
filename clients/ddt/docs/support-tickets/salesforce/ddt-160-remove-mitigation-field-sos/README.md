# Removing Is Mitigation Needed from SOS layouts

| | |
|---|---|
| **Ticket** | DDT-160 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-11 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-11 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (page layouts) |
| **Area** | Hearing section of the Court Case (Opportunity) record page for SOS case types |
| **Related** | DDT-158 (another SOS layout clean-up the same day) |
| **Code** | None. Page layout change in Setup |

---

## 1. Summary

SOS hearing cases do not use mitigation, but the **Is Mitigation Needed?** field still appeared in the Hearing section of their Court Case layouts. It was removed from the SOS layouts and left on the object and on every other layout.

## 2. What was reported

On 2026-06-11 staff reported that "Is Mitigation Needed?" (`Is_Mitigation_Needed__c`) showed in the Hearing section on SOS Court Cases. SOS Formal, Informal and Out of State hearings do not use mitigation, so the field was irrelevant and confusing there.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunity.Is_Mitigation_Needed__c` | Field placement | Removed from the Hearing section of the SOS Court Case layouts |

Which layouts were edited by name is not recorded in the ticket. The request named the layouts used for SOS Formal Hearings, SOS Informal Hearings and SOS Out of State Hearings.

**Out of scope**

- **Deleting the field.** It stays on the object; the ticket said explicitly not to delete it.
- **Non-SOS layouts.** The field stays where mitigation is still used.

## 4. How it works (background)

Court Cases show a different page layout per record type. A field can be on some layouts and not others, so removing it from the SOS layouts hides it on SOS cases only.

## 5. Root cause

Not applicable: a layout clean-up request, not a defect.

## 6. Solution

The field was removed from the Hearing section of the SOS layouts. The support team confirmed it as fixed on 2026-06-11.

Why a layout change and not a deletion: other case types still use mitigation, and deleting a field deletes its data on every record.

## 7. Testing

The requester acknowledged the change the same day. No other check was recorded.

**Not covered:** a recorded check that the field still shows on non-SOS layouts.

## 8. Step by step

1. **Find which layouts SOS cases use:** Setup → Object Manager → Opportunity → Page Layouts → Page Layout Assignment, and note the layout for each SOS record type and profile.
2. **Edit each of those layouts** and drag `Is Mitigation Needed?` out of the Hearing section. Save.
3. **Open an SOS case** and confirm the field is gone.
4. **Open a non-SOS case** and confirm it is still there.
5. If the record page is a Lightning page using Dynamic Forms rather than the layout, remove it in App Builder with a visibility rule instead.

## 9. Rollback

Add the field back to the same section of each SOS layout. Safe: no data was touched.

## 10. Known limits

- **The field still exists on SOS records**, so it can still appear in reports, list views and any automation that reads it.

## 11. Reuse for similar issues

**How to recognise one:** "this field doesn't apply to our case type."

- **Remove it from that record type's layout, not from the object.**
- **Check the layout assignment** for every profile, since the same record type can map to different layouts per profile.
