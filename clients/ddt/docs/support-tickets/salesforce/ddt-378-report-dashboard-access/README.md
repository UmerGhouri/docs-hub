# Granting access to a report or dashboard

| | |
|---|---|
| **Ticket** | DDT-378, DDT-1469 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-30 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-17 and 2026-10-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (reports, dashboards, report folder sharing) |
| **Area** | Reporting: intake and sales-compensation dashboards |
| **Related** | None recorded |
| **Code** | None. Both were handled in the Salesforce UI; no pull request |

---

## 1. Summary

Two requests for access to a Salesforce dashboard or report. The first (DDT-378) turned out not to be an access problem at all: the user could already open the dashboard and had expected to see her own name on its charts. The second (DDT-1469) was a real access gap: the user could see a dashboard chart but got "You can't view report" when drilling into the source report. She was given access to the report, it was checked while logged in as her, and a Court Case column was then added to the report's detail rows at her request.

## 2. What was reported

- **DDT-378 (2026-07-17).** A staff member asked for access to the *Intake Scorecard* dashboard so she could follow intake metrics. Within two minutes she replied that she did have access, could open the report and see all intake, and simply did not see her name on the charts. She asked for the request to be disregarded.
- **DDT-1469 (2026-10-28).** A staff member could see the *Points by Seller* chart on a sales dashboard, but opening the underlying report *Sales Points by Intake Owner: This Month* gave "You can't view report. Looks like you don't have access to the report." She needed to see which cases each seller had signed up. After access was granted she asked for the detail rows, which showed only "board entries", to include the Court Case with a link to it.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| *Sales Comp Dash* | Report folder | The user was given access to the report (DDT-1469). The exact sharing entry is not recorded in the ticket |
| *Sales Points by Intake Owner: This Month* | Matrix report on `Board_Entry__c` | Court Case added as a detail column, linking to the case (DDT-1469) |
| *Intake Scorecard* | Dashboard | No change (DDT-378) |

**Out of scope**

- **Changing who appears on the Intake Scorecard charts** (DDT-378). Not requested once the user confirmed she had access.
- **Points that had not appeared yet for a reassigned case** (raised in the DDT-1469 thread). Not a fault; see 10.

## 4. How it works (background)

```
Dashboard component (chart)
  -> reads a source report
       -> the report lives in a report folder
            -> the folder is shared with users, roles or groups (Viewer / Editor / Manager)
       -> the report returns only rows the user can see (object, field and record access)
```

Seeing a dashboard chart does **not** prove the viewer can open its source report. A dashboard can run as a fixed user, so its charts render even for someone who has no access to the report folder. Clicking through to the report then checks the viewer's own folder access.

The *Sales Comp Dash* folder is not shared with everyone. Read from production on 2026-11-05:

```sql
SELECT Name, Type, AccessType FROM Folder WHERE Name = 'Sales Comp Dash'
-- Sales Comp Dash | Report | Hidden
```

## 5. Root cause

- **DDT-378:** no access problem. The user already had access; her name was simply not one of the values on the charts.
- **DDT-1469:** the user was not among those the *Sales Comp Dash* report folder was shared with. The dashboard chart rendered for her, but the report behind it did not. Exactly how the dashboard was set to run is not recorded in the ticket.

## 6. Solution

### 6.1 DDT-378

Nothing was changed. The user confirmed in the thread that she could open the dashboard and its report.

### 6.2 DDT-1469: grant access to the report

The user was given access to the report. The ticket records it as "full access to the report" and does not record whether this was a folder share to her user, a role or a group, or at which access level. The ticket's requirement was Viewer access to the report or its folder, as appropriate to the folder's existing sharing.

Prefer sharing the **folder** over moving the report: moving a report to another folder changes who else can see it.

### 6.3 DDT-1469: add the Court Case to the detail rows

The report is built on `Board_Entry__c`, so its detail rows showed board entry names. The Court Case was added as a column, and it links straight to the case. On `Board_Entry__c` the court case is the `Opportunity__c` lookup (Opportunity is labelled *Court Case* in this org). The report shows a last-modified date of 2026-10-30.

## 7. Testing

- **DDT-1469 access:** verified while logged in as the user. The report opened and showed all sellers' points and cases. The user then confirmed in the thread: "I can view it now."
- **DDT-1469 column:** the user was asked to refresh the report and confirm. Her confirmation is not recorded in the ticket.
- **DDT-378:** confirmed by the user herself; nothing to test.

**Not covered:** whether the user can see every `Board_Entry__c` row through her own object and record access was not separately recorded; the login-as check showed data returned.

## 8. Step by step

1. **Ask what exactly fails.** Get the exact error text and whether it is the dashboard or the report behind a chart. "I can't see it" sometimes means "I can see it but not my own row" (DDT-378).
2. **Find the folder.** Open the report, or query it:

   ```sql
   SELECT Name, FolderName, Format FROM Report WHERE Name = '<report name>'
   ```

3. **Check the folder's sharing.** Reports tab, the folder, Share. See who it is shared with and at what level.
4. **Share the folder** with the user (or the role or group they belong to) at **Viewer**, unless they need to edit. Sharing the dashboard's folder does not share the source reports' folders; share each one.
5. **Check data access.** The report returns only rows the user can see. If the report opens but is empty, check object access, field-level security and record sharing on the report's base object.
6. **Log in as the user** (Setup, Users, Login) and open both the dashboard and the report. Confirm data appears.
7. **Ask the user to confirm**, and record the answer on the ticket.

## 9. Rollback

Remove the user from the folder's sharing (Reports, the folder, Share). Remove the added column in the report builder. Neither affects any other user or any data.

## 10. Known limits

- **Folder sharing is all or nothing for that folder.** If the access was given as a folder share, every report in *Sales Comp Dash* is visible to the user, not only this one.
- **A sales point appears only after the client's first payment**, not when the case is marked Hired. A case whose intake owner was changed shows the point under the new owner once the first payment comes in, with nothing to do manually. This came up in the DDT-1469 thread and is how the board works, not a fault.

## 11. Reuse for similar issues

**How to recognise one:** "You can't view report" or "Insufficient privileges" when opening a report or dashboard, often after clicking through from a chart that did render.

- **Confirm the problem first.** In DDT-378 there was no access problem at all.
- **Dashboard visible, report not:** the dashboard runs as someone else; share the report's folder with the viewer.
- **Report opens but is empty:** folder access is fine; look at object, field and record access on the report's base object.
- **Grant at folder level, at the lowest level that works** (Viewer), and to a group or role when several people need the same thing.
- **Verify by logging in as the user**, not by checking the sharing screen.
- **Follow-up report changes** (an extra column, a link) are small and can be done in the same ticket when the requester asks in the thread.
