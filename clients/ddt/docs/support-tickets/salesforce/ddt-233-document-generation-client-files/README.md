# Document generation failing on client files

| | |
|---|---|
| **Ticket** | DDT-233, DDT-241 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-30 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, S-Docs document templates (since replaced by Dochly) |
| **Area** | Generating court documents (Appearance and others) from a client's file |
| **Related** | DDT-230 (signature on generated documents), DDT-1439 (S-Docs uninstalled) |
| **Code** | None. Nothing was changed in the repo. |

---

## 1. Summary

A staff member clicked **Generate** on a client's file and was sent back to the record with no document produced. It first looked specific to one client, then happened again on another. The support team generated each document in turn and found that everything worked except one S-Docs template, **Appearance - Cook County (Short)**. The team only uses the long form, so the staff member switched to it and the problem stopped. Why the short template failed is not recorded.

## 2. What was reported

- **DDT-233, 2026-06-25.** Generating documents on one client's Account redirected back to the record, and nothing appeared in the **DDT Generated Documents** related list. The staff member had tried another browser and logging out and back in. At that point it affected only that client.
- **DDT-241, 2026-06-29.** The same staff member was "still having issues", now on another client's file, while trying to produce the Appearance, a notice of filing, a motion for discovery and a PTR. A screen recording was supplied. The firm marked it high priority because time-sensitive filings were blocked.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| S-Docs template **Appearance - Cook County (Short)** | Document template (data record in the S-Docs package) | Identified as the only failing template; the team agreed it could be removed |

**Out of scope**

- **The other templates.** They generated correctly and were not changed.
- **The automated Appearance flow** (`Ready_to_Create_Paper_Appeareance`). It was not changed under these tickets. See section 10.

## 4. How it works (background)

```
Staff member opens the client's file
  -> Generate (S-Docs) -> chooses templates
  -> S-Docs merges the record data into each template
  -> the finished file is listed under "DDT Generated Documents"
```

Cook County had two Appearance templates, a short form and a long form. Separately, the record-triggered flow `Ready_to_Create_Paper_Appeareance` created S-Docs jobs automatically, choosing the template by county and courthouse. In the 2026-05-22 snapshot of that flow, Cook cases at the **1st - Daley Center** courthouse used the short form and the other Cook courthouses the long form.

## 5. Root cause

**Narrowed down, not explained.** On 2026-06-30 the support team generated each of the requested documents and reported:

> Everything else is generating correctly; the issue only seems to be with the "Appearance - Cook County (Short)" S-Docs template.

The first diagnosis, that the problem was specific to one client's data, was wrong: it moved to another client, and the common factor was the template, not the record. What inside the short template failed is **not recorded in the ticket**.

## 6. Solution

1. The firm confirmed it uses the **long** Cook County Appearance, not the short one.
2. The staff member generated with the long form and confirmed the problem was gone.
3. The firm asked for the short form to be removed because it is not used, and the support team agreed. Whether the template was then deleted or deactivated is **not recorded in the ticket**.

## 7. Testing

The firm reported on 2026-06-30 that the staff member was no longer having the problem. No automated test applies: S-Docs templates are data, not code.

## 8. Step by step

1. **Get the exact list of documents** the user was generating and the record, ideally with a screen recording.
2. **Generate each template on its own** on the same record. The one that fails is the one to look at.
3. **Generate the same template on a second record** to tell a template fault from a data fault.
4. **Check whether the failing template is still needed.** If an equivalent template is in use, switch the user to it.
5. **Retire a template nobody uses**, after checking no automation points at it (search the flows for its name and Id).

## 9. Rollback

Not applicable: nothing was changed beyond advising the user to use the long form.

## 10. Known limits

- **The automated flow pointed at the short form for one courthouse.** The 2026-05-22 snapshot of `Ready_to_Create_Paper_Appeareance` selected the Cook short template for **1st - Daley Center** cases. If the short template was removed and the flow still referred to it, automatic Appearances for that courthouse would have failed. The tickets do not record whether this was checked.
- **No longer applicable as written.** S-Docs was replaced by Dochly and uninstalled under DDT-1439. The method in section 8 still applies to Dochly templates.

## 11. Reuse for similar issues

**How to recognise one:** document generation "does nothing" or bounces back to the record, with no error, and the user reports it as a problem with one client.

- **Split the batch.** Generating several templates at once hides which one fails. Generate them one at a time.
- **Test a second record before blaming the data.** Here the "one client" diagnosis was wrong.
- **Ask whether the failing template is needed at all.** Retiring a duplicate is often the fastest correct fix.
- **Before deleting a template, search the flows for it.** Automated generation may still point at it.
