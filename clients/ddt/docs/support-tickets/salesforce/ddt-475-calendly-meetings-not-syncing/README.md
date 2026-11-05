# Calendly meetings not syncing to Salesforce

| | |
|---|---|
| **Ticket** | DDT-475, DDT-761 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-21 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, custom notification), Calendly connector |
| **Area** | Meetings booked through Calendly appearing on Salesforce records and calendars |
| **Related** | None recorded |
| **Code** | PR #220 in the `ddt-salesforce` repo (DDT-761 notification only) |

---

## 1. Summary

Two reports that Calendly meetings were not reaching Salesforce, with different causes. In July the Calendly to Salesforce connection had dropped firm-wide and was reconnected. In August one user's meetings were reaching Salesforce but landing on an admin account's calendar, because the email on his Salesforce user did not match his Calendly email; the email was corrected. The August ticket also asked for a bell notification when a meeting is booked, which did not exist, so a new flow and custom notification type were added.

## 2. What was reported

- **DDT-475 (2026-07-21).** Calendly-booked meetings no longer appeared on the Activity tab or the Calendly tab of client records. Several Case Managers' bookings were affected.
- **DDT-761 (2026-08-18).** A sales staff member reported that none of his Calendly meetings appeared in Salesforce, and that he got no Salesforce bell notification when a meeting was booked.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Calendly to Salesforce connection | Connector setup | Reconnected (DDT-475) |
| The affected user's Salesforce email | User record | Changed to match his Calendly email (DDT-761) |
| `Calendly_Meeting_Booked` | Custom notification type | New: desktop and mobile (PR #220) |
| `Calendly_Meeting_Booked_Notification` | Record-triggered flow on Event | New: notifies the host when a Calendly Event is created (PR #220) |
| `scripts/post-validation.py` | Repo script | Recognises the `notificationTypes` directory (PR #220) |

**Out of scope**

- **Backfilling meetings missed during the July outage.** Not recorded in the ticket.

## 4. How it works (background)

```
Client books through Calendly
  -> the Calendly connector creates an Event in Salesforce
       Subject contains " - Calendly - "
       Owner = owner of the matched Lead/Contact,
               else the user whose User.Email equals the Calendly host email,
               else the integration user
  -> the Event shows on the record's Activity tab, the Calendly tab, and the owner's calendar
```

The host is matched **by email**. If no Salesforce user has the host's Calendly email, the Event is owned by the integration user, so it sits on that account's calendar instead of the host's.

## 5. Root cause

### DDT-475

The Calendly to Salesforce connection **dropped on June 25**, so no bookings reached Salesforce firm-wide until it was reconnected on **July 22**. Why it dropped is not recorded in the ticket.

### DDT-761

The meetings **were** reaching Salesforce. They were landing on an admin account's calendar, not the user's. His Salesforce user carried a different email address from the one his Calendly account used, so the connector could not resolve him as the host and fell back to the integration user.

**The missing notification was a missing feature, not a regression.** The org had no Calendly notification of any kind.

## 6. Solution

### 6.1 Reconnect Calendly (DDT-475)

The connection was reconnected on 2026-07-22 and live records were confirmed flowing again. The exact reconnection steps are not recorded in the ticket.

### 6.2 Fix the user's email (DDT-761)

The user's email in Salesforce was changed to the address his Calendly account uses. Salesforce sends a verification email for an email change, and the change takes effect only after the user clicks the link in it. The user confirmed on 2026-08-20.

### 6.3 Bell notification on booking (PR #220)

- **`Calendly_Meeting_Booked`** custom notification type, desktop and mobile.
- **`[Event] Calendly Meeting Booked Notification`** flow: after-save, on create of an Event whose Subject contains ` - Calendly - `. A decision skips cancellations, so only new bookings alert. It notifies the **Event owner** (the host) with the title "New Calendly meeting booked" and the meeting name taken from the Subject.
- The sender is read from the `Flow_Metadata__mdt` custom metadata, not a hardcoded user Id.

Why it's done this way:

- **No owner is excluded.** An early version skipped Events owned by the integration user. Review found that a real host is also that user, so the filter would have silenced their own bookings. Gating on a blank `WhoId` was worse: of 814 connector Events, 451 had a correctly resolved host and no matched record, and would all have been silenced.
- **If a host fails to resolve again, the integration user gets the alert.** That surfaces the misconfiguration instead of hiding it.
- **Shipped as Draft and activated after deploy**, so the notification type was confirmed present before the flow could fire.

## 7. Testing

- PR #220 was validated against production as a dry run: both components created cleanly, 1 of 1 test passing. There is no flow-specific test.
- The DDT-761 user posted a screenshot on 2026-08-20 after clicking the verification link.

**Live check (2026-11-05):**

| Query | Result |
|---|---|
| `SELECT ApiName, IsActive FROM FlowDefinitionView WHERE ApiName = 'Calendly_Meeting_Booked_Notification'` | Active, last modified 2026-08-21 |

**Not covered:** a test of the July reconnection beyond confirming records were flowing; whether missed July bookings were backfilled.

## 8. Step by step

1. **Check whether anything reaches Salesforce.** Query recent Events whose Subject contains ` - Calendly - `. If there are none firm-wide, the connection is down: reconnect it from the Calendly connector setup.
2. **If Events exist but not on the user's calendar,** check their owner. Events owned by the integration user mean the host could not be matched.
3. **Compare the user's Salesforce email with the email on their Calendly account.** Change the Salesforce one to match.
4. **Ask the user to click the verification link** Salesforce sends for the email change.
5. **For the notification,** create the `Calendly_Meeting_Booked` notification type and the Event flow as in 6.3, deploy the flow as Draft, then activate it.

## 9. Rollback

- **Notification:** deactivate `Calendly_Meeting_Booked_Notification` in Setup, Flows. The notification type is harmless on its own.
- **Email change:** change it back on the user record. That would break host matching again.

## 10. Known limits

- **Host matching depends on email.** Any user whose Salesforce and Calendly emails differ will have meetings land on the integration user's calendar.
- **The notification relies on the Subject marker** ` - Calendly - `. If the connector changes its Subject format, the flow stops firing.
- **Events created before the fix stay where they landed.** Moving them is not recorded in the ticket.

## 11. Reuse for similar issues

**How to recognise one:** "nothing is syncing" from one user, while other users' records arrive normally.

- **Check if records arrive at all** before assuming the integration is down. A firm-wide gap is a connection problem; a one-user gap is usually identity matching.
- **Look at who owns the records.** Records owned by an integration user mean the integration could not match a person.
- **Compare identities across systems** (email, username) for the affected user.
- **Treat "notification not firing" as a question**: check whether the notification was ever built.
- **For a notification flow, don't silence the fallback owner.** Let a misconfiguration alert someone.
