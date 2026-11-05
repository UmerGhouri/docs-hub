# Federal case status path

| | |
|---|---|
| **Ticket** | DDT-92 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-04 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (path component, Lightning App Builder) |
| **Area** | Case management: showing where a Federal matter stands |
| **Related** | Not recorded in the ticket |
| **Code** | Not applicable: configured in the org, no pull request |

---

## 1. Summary

Federal matters had no visual stage path, unlike the other practice areas. We added a chevron path, driven by the **Federal Case Status** field, that shows only when the Matter Type is "Federal". It has six stages, in the order a federal case moves.

## 2. What was reported

On 2026-06-02 the ticket asked for a chevron (path) component on Federal case records, matching the path already used for other practice areas, so users can see the current Federal stage and move to the next one from the page.

The ticket proposed a nine-stage starting list and asked for the real list to be confirmed with the Federal practice lead. The list that was built (section 6) is different, which shows the confirmation step mattered.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Federal Case Status | Picklist field | Drives the path; values ordered as the stages |
| Chevron / path component | Record page component | Added, visible only when Matter Type is "Federal" |

**Out of scope**

- **Task gating on stage changes**, as some Criminal paths have. Raised as an open question; no decision is recorded.
- **A separate "Closed" stage.** The ticket raised whether closure should be handled outside the path, as for Criminal paths. The six stages built include no closure stage.

## 4. How it works (background)

A Salesforce path reads one picklist field and draws one chevron per value, **in the picklist's value order**. The current value is highlighted, and users can mark a stage complete or jump to another stage from the page.

```
Record page loads
  -> visibility rule: is Matter Type = "Federal"?
       no  -> path hidden
       yes -> path drawn from the Federal Case Status picklist, in value order
```

Because the chevrons follow the picklist's order, the order of values in the field **is** the order of the path.

## 5. Root cause

Not applicable: this was a request for a new component, not a defect.

## 6. Solution

From the completion note (2026-06-04): a chevron path was created on the **Federal Case Status** field, shown when **Matter Type is "Federal"**, with these stages in order:

1. Investigation (Target/Subject Letter)
2. Initial Appearance
3. Pre-Indictment
4. Indicted/Filed
5. Trial
6. Supervised Release Violation

Why it was done this way:

- **Driven by one picklist**, the same pattern as the other practice-area paths, so it looks and behaves the same.
- **A visibility rule on Matter Type**, so only Federal matters show it and no other matter gets an empty path.

The ticket asked for the metadata to be committed to the `ddt-salesforce` repo. A search of the repo on 2026-11-05 found no field or path named for Federal case status, so it was not committed. The field's exact object and API name are not recorded in the ticket.

## 7. Testing

Not recorded in the ticket.

**Not covered:** the ticket's acceptance criteria (correct order, "Mark Stage as Complete" working, rendering across screen sizes and profiles) have no recorded check.

## 8. Step by step

1. **Confirm the stage list and its order** with the practice lead before building.
2. **Create or edit the status picklist field** and put its values in stage order.
3. **Add the path component** to the record page in Lightning App Builder and bind it to that field.
4. **Add a visibility rule** so the path shows only for the right Matter Type.
5. **Save and activate** the page, then open a Federal matter and a non-Federal matter to confirm it shows on one and not the other.
6. **Retrieve the changed field and page** into the `ddt-salesforce` repo.

## 9. Rollback

Remove the path component from the record page in Lightning App Builder, or delete its visibility rule's target. The picklist field and its values can stay; removing the component does not change any record's data.

## 10. Known limits

- **Reordering the picklist reorders the path.** Anyone editing the field's values changes what users see.
- **No closure stage.** A finished Federal matter stays on its last stage in the path.
- **Not in the repo**, so the next deploy that touches the same record page from git could drop the component. Retrieve the page before editing it.

## 11. Reuse for similar issues

**How to recognise one:** a practice area or matter type needs its own visual stage tracker.

- **Get the real stage list from the people who run the cases.** The proposed list here had nine stages; the one built has six.
- **Order the picklist values in stage order** before adding the path.
- **Use a visibility rule on the type field** so other records don't get an empty path. An empty path is what DDT-94 had to remove.
- **Retrieve the page and field into the repo** the same day, so a later deploy doesn't overwrite them.
