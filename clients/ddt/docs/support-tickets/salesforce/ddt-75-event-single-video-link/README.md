# Single video link field on events

| | |
|---|---|
| **Ticket** | DDT-75 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-01 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Flow, page layouts) |
| **Area** | Scheduling: the video meeting link on calendar Events |
| **Related** | None recorded |
| **Code** | Not recorded in the ticket. No PR is linked, and the repo history has no commit for this change. |

---

## 1. Summary

Events had two separate link fields, **Zoom Link** and **Google Meet Link**. Staff had to guess which one to fill in, and anything that showed the link needed logic for both. The two were consolidated into a single **Video Link** field, and the Apex class, flow and page layouts that used them were updated to the one field.

## 2. What was reported

- **2026-06-01, DDT-75:** two parallel link fields on Event caused confusion at scheduling time, needed duplicate logic in templates and reports, and sometimes meant the calendar showed the wrong link when the wrong field was filled in.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Video Link | Event field | The single field for the meeting link |
| `BeginActionController` | Apex class | References the Video Link field |
| Case Evaluation Flow | Flow | Maps to the Video Link field |
| Event page layouts | Layouts | Updated: All Staff Meeting, Case Evaluation Layout, Depositions, Event Layout, the mobile layout, General, Generic Layout, Staff Meeting |

**Out of scope, or not recorded**

- **Whether the field was new or a renamed existing field**, and its API name. The ticket proposed `Video_Link__c` but the resolution note gives only the label.
- **Copying existing Zoom and Google Meet values into Video Link**, and the precedence rule when both were filled. The ticket asked for both; the resolution does not record either.
- **Integrations** (Zoom, Google Meet, Calendly, Fathom). Not recorded as changed.
- **Deleting the old fields.** The ticket planned to hide them first and delete later. Not recorded.

## 4. How it works (background)

```
Before:  Event.Zoom Link          <- filled for Zoom meetings
         Event.Google Meet Link   <- filled for Google Meet meetings
         consumers (Apex, flows, layouts) read one or the other

After:   Event.Video Link         <- filled for any video meeting
         consumers read the one field
```

## 5. Root cause

Not a defect: a data model change. Two fields holding the same kind of value for different vendors meant every writer and reader had to pick one.

## 6. Solution

All consumers recorded in the resolution note were pointed at the single Video Link field:

- `BeginActionController` (Apex) reads and writes the Video Link field.
- The Case Evaluation Flow maps to it.
- The eight Event page layouts in section 3 show it.

Why one field: the value is the same thing whatever the vendor. A single field means one place to fill in, one place to read, and no conditional logic in every template and report.

## 7. Testing

Not recorded in the ticket.

## 8. Step by step

1. Create the single link field on Event (Activity) as a URL field, or rename one of the existing fields.
2. Grant field-level security for it to the profiles or permission sets that schedule events.
3. Copy existing values: Zoom Link into it where set, Google Meet Link where set, with an agreed rule when both are set.
4. Search Apex, flows, layouts, email templates and reports for the two old fields. At minimum, update `BeginActionController`, the Case Evaluation Flow and the eight layouts in section 3.
5. Deploy and check a new Event and an existing Event both show the link.
6. Remove the old fields from layouts and field-level security. Delete them only after a waiting period.

## 9. Rollback

Point the consumers back at the old fields and put them back on the layouts. Not recorded whether the old fields still hold their values.

## 10. Known limits

- **Current state not confirmed.** A read of Event's field list on 2026-11-05 returned no field with "Video", "Zoom" or "Meet" in its label apart from the standard Meeting URL field. Whether the Video Link field was later renamed or removed is not recorded. Check the org before relying on this page.
- External tools that wrote to the old field names would keep writing there unless changed.

## 11. Reuse for similar issues

**How to recognise one:** several fields holding the same kind of value, split by vendor or source, with consumers that each read only one.

- Consolidate into one field, then move every writer and every reader. List them first by searching the metadata.
- Copy existing data before removing anything, and agree a precedence rule when both old fields are filled.
- Hide old fields before deleting them. Deleted fields still count toward the object's field limit until purged.
