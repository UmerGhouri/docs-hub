# Adding a user to a scheduled report email

| | |
|---|---|
| **Ticket** | DDT-1104, DDT-1503 |
| **Project** | Support tickets |
| **Status** | Live since 2026-11-04 (DDT-1104 done 2026-09-24) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-23 (DDT-1104); Firm staff, 2026-11-04 (DDT-1503) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (report and dashboard subscriptions, folder sharing) |
| **Area** | Scheduled report and dashboard emails |
| **Related** | None recorded |
| **Code** | Not applicable: done in Salesforce, no code |

---

## 1. Summary

Two requests to add people to an existing scheduled email: one to a **report** subscription, one to a **dashboard** subscription. Both were done in Salesforce by editing the subscription's recipients. For the dashboard, the new recipients were also given **View** access to its folder, so they can open the dashboard from the email.

## 2. What was reported

- **DDT-1104, 2026-09-23:** a staff member asked to receive the daily **Docket - Final Check for Today** report email (Dockets folder). Firm leadership noted she only needed adding to the report's subscription.
- **DDT-1503, 2026-11-04:** two intake team members were to be added to the daily 6:00 PM **Payment Center Dashboard** email. It already went to three people and ran as the subscription owner, a senior firm user.

## 3. Scope

**In scope:**
- The recipient list of the **Docket - Final Check for Today** report subscription.
- The recipient list of the **Payment Center Dashboard** 6:00 PM subscription.
- **View** access to the **Payment Center** folder for the two new dashboard recipients.

**Out of scope:** the schedules, the existing recipients, and the reports and dashboard themselves. None of them changed.

## 4. How it works (background)

```
Subscription (owned by one user, runs on a schedule)
  -> refreshes the report or dashboard as its running user
  -> emails the result to every listed recipient
```

- Adding **other people** to a subscription needs the right permission. The ticket notes that adding recipients to someone else's dashboard subscription may need the owner or an admin with **Subscribe Others to Dashboards**.
- A dashboard email shows data as its running user ("Viewing As"). To open the dashboard itself from the email, a recipient needs access to its folder.

## 5. Root cause

Not applicable: these were access requests, not faults.

## 6. Solution

- **DDT-1104 (2026-09-24):** the staff member was added to the **Docket - Final Check for Today** subscription.
- **DDT-1503 (2026-11-04):** both intake team members were added to the 6:00 PM **Payment Center Dashboard** subscription, and both were given **View** access to the **Payment Center** folder. The requester confirmed they are on the intake team and should have access. Existing recipients and the daily 6:00 PM schedule were left unchanged.

Why folder access too: the email carries a snapshot, but clicking through to the dashboard needs folder access. The ticket asked to confirm both users could reach the dashboard and its folder.

## 7. Testing

- **DDT-1104:** the acceptance criterion was that she receives the next scheduled email. Whether she confirmed it is not recorded in the ticket.
- **DDT-1503:** the requester asked both new recipients to confirm they received the email from 6:00 PM on 2026-11-04. Their confirmation is not recorded in the ticket.

## 8. Step by step

1. **Confirm each new recipient is an active user.**
2. **Open the report or dashboard**, then **Subscribe** (or the existing subscription) as the subscription owner or an admin who can subscribe others.
3. **Add the recipients** under the subscription's recipients. Leave the schedule and existing recipients as they are.
4. **Save.**
5. **For a dashboard:** check each new recipient has access to the dashboard's folder. If not, share the folder with them at **View** level.
6. **Ask the recipients to confirm** they got the next scheduled email.

## 9. Rollback

Remove the recipients from the subscription, and remove their folder share if it was added only for this. Safe to do at any time.

## 10. Known limits

- A subscription has one owner. If that user is deactivated, the subscription stops for everyone on it.
- A dashboard email shows the running user's data. A recipient with less access sees more in the email than they could see in Salesforce. Check that this is intended before adding someone.

## 11. Reuse for similar issues

**How to recognise one:** "please add X to the email for report Y."

- Edit the existing subscription's recipients rather than creating a new subscription, so there is one schedule to maintain.
- For dashboards, add folder access at the same time.
- Leave the schedule and existing recipients untouched, and say so when closing the ticket.
