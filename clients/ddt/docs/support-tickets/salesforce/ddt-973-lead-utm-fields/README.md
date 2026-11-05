# UTM fields on Lead

| | |
|---|---|
| **Ticket** | DDT-973 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-14 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (page layouts, Lightning record page), website lead integration |
| **Area** | Lead attribution: GCLID and UTM values from the website |
| **Related** | None recorded |
| **Code** | PR #343 in the ddt-salesforce repo |

---

## 1. Summary

The ticket asked for five UTM fields on Lead and field-level security for the website integration user, believing website attribution had nowhere to land. All of that already existed and had been working for three weeks. The real problem was that no Lead page displayed the fields. The fix added them, read-only, to the Attribution section of the Lead record pages. A last snag on one converted Lead was a missing user permission, not the page.

## 2. What was reported

- **2026-09-09:** the request was to create `UTM_Source__c`, `UTM_Medium__c`, `UTM_Campaign__c`, `UTM_Term__c` and `UTM_Content__c` on Lead (Text 255, like `GCLID__c`), give the website integration user edit access and the requester read access, and ship both together. The ticket said that without them the website's lead insert would be rejected and the lead never created.
- **2026-09-10, after the deploy:** the requester opened one Lead in a DDT app and saw no Attribution section and no UTM values, although the API showed the record held them.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Lead_3_sections` | Lightning record page (Dynamic Forms) | Added `GCLID__c` and the five UTM fields, read-only, to the existing Attribution section (11 to 17 fields) |
| `Lead-PI Layout`, `Lead-Defense Layout`, `Lead-SOS Layout` | Page layouts | New read-only Attribution section with the same six fields |

**Out of scope**

- **The fields and FLS the ticket asked for.** They already existed (see section 5). Nothing was created or granted.
- **The Opportunity UTM fields.** The ticket said "Lead only, do NOT create on Opportunity". Opportunity has carried the five fields since 2026-07-17 and the defense convert path fills them. Reading that line as an instruction to remove them would break live attribution, so they were left alone.
- **The website's integration setting** that turns on writing click ids to Salesforce. Not touched.

## 4. How it works (background)

```
Website form
  -> website integration user inserts the Lead with GCLID__c and UTM_*__c
  -> Lead record page draws the fields the viewer can read
```

**Lead has three record pages, and they do not draw fields the same way:**

| Page | How it draws fields | Used by |
|---|---|---|
| `Lead_3_sections` | Dynamic Forms: the page itself lists the fields | Org default, so every DDT app |
| `Lead_Record_Page1` | The page layout | One standard console app |
| `Lead_Record_Page3` | The page layout | Another standard console app |

On a Dynamic Forms page the page layout decides nothing about which fields appear. On the other two, the layout decides.

**A page or layout never grants access.** A field appears only if the viewer has read FLS on it, and Salesforce hides a section when the viewer can read none of its fields.

## 5. Root cause

**The ticket's stated cause was wrong.** Checked in production on 2026-09-10:

| Checked | Found |
|---|---|
| The five UTM fields on Lead, Text(255) | Already existed, created 2026-07-16 |
| `Website_API_Integration` permission set | Already granted edit on the five UTM fields and `GCLID__c`, assigned to the website integration user |
| `UTM_Attribution_Access` permission set | Already granted access to the six fields, assigned to the requester |
| Leads carrying UTM data | 49, all written by the website integration user, continuously from 2026-08-20 to 2026-09-10 |

So the website pipeline was working. **The actual defect was display:** the six fields were on none of the three Lead layouts and not in the Attribution section of `Lead_3_sections`. The requester had read access and opened Leads that held values, but nothing drew them.

**A first fix that would not have worked.** The first version of the PR changed only the three page layouts. It would have deployed and validated cleanly and changed nothing for the requester, because DDT apps use `Lead_3_sections`, where the layout has no effect. That mistake came from listing the repo's flexipages folder instead of querying the org: `Lead_3_sections` existed in production but not in the repo, so a repo-side check could not see it. A reviewer caught it before merge.

**The follow-up report (2026-09-10)** had a different cause again. The Lead the requester opened had been converted. A converted Lead redirects to the client record unless the user holds the "View and edit converted lead records" permission, and admin profiles do not include it by default. The fields and the Attribution section were in place throughout.

## 6. Solution

1. **`Lead_3_sections`:** added `GCLID__c`, `UTM_Source__c`, `UTM_Medium__c`, `UTM_Campaign__c`, `UTM_Term__c` and `UTM_Content__c` to the existing Attribution section as `readonly`, matching `Referral_Source__c`, which was already there. Retrieved from production before editing, because the file was not in git. The diff against production was a single insertion: 60 lines added, none changed or removed.
2. **The three Lead layouts:** a read-only Attribution section with the same six fields, placed before Description Information, for the two console apps. Each diff was a single insertion of 35 lines.
3. **For the converted-Lead report:** the requester was assigned the "View and edit converted lead records" permission, so converted Leads open normally and show the Attribution section.

**Why read-only:** these values are written by the integration and used to tie ad spend to leads. A hand edit would silently corrupt that.

**Who sees what:** the five UTM fields are readable only through permission sets, so only the users holding them see the values. For everyone else the section looks as before. `GCLID__c` is readable on many profiles, so most users now see it on the page; that grants nothing they could not already query.

## 7. Testing

- **Check-only deploy against production**, posted as the `salesforce/validation` status on PR #343. That proves the metadata deploys. **It renders nothing**, so it cannot prove a field appears on a page. This is exactly why the layout-only version looked correct and was not.
- **Live check:** after the deploy, the requester was asked to open a recent Lead in a DDT app, not a console app. The first attempt hit the converted-Lead redirect described in section 5. After the permission was assigned, the Attribution section showed `GCLID__c` and all five UTM values (2026-09-11).
- **Not covered:** no automated test exists for page layouts or record pages.

## 8. Step by step

1. **Check the premise in the org before building anything.** Query `FieldDefinition` for the fields, check which permission sets grant them and to whom, and count records that already hold values. Here everything the ticket asked for already existed.
2. **Find which Lightning page the user actually sees.** Check the org default and every app and profile assignment for the object. Query the org; do not rely on the repo's flexipages folder.
3. **Check how that page draws fields.** Dynamic Forms: edit the page. `force:detailPanel`: edit the page layout.
4. **Retrieve each page and layout from production** before editing, and add the fields read-only to the right section.
5. **Confirm each diff against production** is only your insertion.
6. **Validate check-only against production**, get the PR approved, deploy.
7. **Open a record as the requester would**, in their app, and confirm the values show.
8. **If a record still will not show,** check whether it is a converted Lead and whether the user has "View and edit converted lead records".

## 9. Rollback

Redeploy the previous versions of `Lead_3_sections` and the three layouts. That only hides the fields again; no data or access changes. Safe. The converted-lead permission can be removed from the user in Setup.

## 10. Known limits

- Users without read FLS on the UTM fields still see nothing, by design. Granting it is a permission set change, not a page change.
- A new Lead record page assigned to an app or profile would need the fields added again.
- Converted Leads stay unreachable for any user without the converted-lead permission.

## 11. Reuse for similar issues

**How to recognise one:** "the data is there, I can see it in a report or the API, but not on the record."

- **Verify the ticket's diagnosis against the org first.** Here the requested fields and access had existed for weeks.
- **Work out which page renders the record** for that user and app, and whether it uses Dynamic Forms or the page layout. Editing the wrong one deploys cleanly and changes nothing.
- **Query the org for pages, not the repo.** A page that exists only in production is invisible to a repo check.
- **A green check-only deploy does not show a field on screen.** Open the record.
- **If one specific record will not open,** check whether it is converted before changing anything else.
