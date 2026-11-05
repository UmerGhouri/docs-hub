# Claim Lead in Slack blocked by the Salesforce app

| | |
|---|---|
| **Ticket** | DDT-1018, DDT-1091 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-23 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-09-15 |
| **Resolved by** | Support team |
| **Systems** | Slack (workspace app management), Salesforce for Slack |
| **Area** | Intake: claiming new leads from Slack lead alerts |
| **Related** | DDT-1241 (single claim channel for the Slack Claim button) |
| **Code** | Not applicable: Slack admin setting |

---

## 1. Summary

An intake team member could not use the "Claim Lead" button on Slack lead alerts. Clicking it asked them to connect the Salesforce app, and Slack then said the app was restricted on the workspace. The app behind the button, Salesforce Automations (Legacy), had been set to Denied in the workspace's app settings. Users who connected before that still worked, but nobody new could connect. The workspace owner approved the app and the button worked again.

## 2. What was reported

- **DDT-1018, 2026-09-15:** an intake team member clicked "Claim Lead" in Slack, was taken to a Connect page, clicked Connect, and landed on a Slack error: "You are not authorized to install Salesforce Automations (Legacy) on [the workspace]. Salesforce Automations (Legacy) is restricted for this workspace..." They could not claim leads from Slack.
- **DDT-1091, 2026-09-22:** the same person reported the same error on the "Claim this Lead!" button in the new-lead channel, while the first ticket was still waiting on an admin.

Only one user was known to be affected, because everyone else had connected before the restriction.

## 3. Scope

**In scope:** the approval status of the Salesforce Automations (Legacy) app in the Slack workspace's app management settings.

**Out of scope**

- **Moving the Claim button to the newer Salesforce for Slack app.** The ticket raised it as an option; it was not needed to fix this. The claim channel itself was reworked later under DDT-1241.
- **Any Salesforce-side change.** The claim action itself was fine.

## 4. How it works (background)

```
Lead alert posted in Slack with a Claim button
  -> user clicks Claim
     -> first time only: Slack asks the user to connect (authorize)
        the Salesforce Automations (Legacy) app
     -> the app claims the lead in Salesforce
```

Connecting is a per-user authorization, which Slack treats as an app install. If the workspace has the app set to restricted or denied, a user who has not connected yet cannot finish it. Users who connected earlier keep working.

## 5. Root cause

The Salesforce Automations (Legacy) app was set to **Denied** in the workspace's Slack app settings (restricted on 13 August, per the ticket). That blocked every new connection. It only showed up for one person because everyone else had already connected before the restriction.

Two things made this slow to fix:

- **Two Salesforce apps in Slack have similar names.** The full address of the error page carried the exact app, which is how the right one was identified.
- **The Slack app manager could not change it.** An app manager in the workspace found the app but had no way to change it from Denied to Approved. A second admin tried and also could not. Only the workspace owner could approve it.

## 6. Solution

The workspace owner approved Salesforce Automations (Legacy) for the workspace on 2026-09-22. Both tickets were closed on 2026-09-23.

**Workaround while waiting:** open the lead from the alert's link and set the Intake Owner to yourself in Salesforce. The result is the same; the Slack button is only a shortcut.

## 7. Testing

The workspace owner confirmed on the ticket that the approval was done. Not recorded in the ticket: a test claim by the affected user after the approval.

## 8. Step by step

1. Ask the affected user to click Claim again and copy the full address of the error page. It names the exact app, which matters because two Salesforce apps have similar names.
2. Open the Slack admin console and go to app management. Slack names this section differently between versions ("Apps", "Apps and workflows", "Manage apps").
3. Find Salesforce Automations (Legacy) under restricted or denied apps.
4. Approve it for the workspace. If Approve is missing or greyed out, the setting is above app manager level; ask the workspace owner (or org owner) to do it.
5. Ask the user to click Claim, connect, and confirm the lead is claimed in Salesforce.

## 9. Rollback

Set the app back to restricted or denied in app management. Users who have not connected will be blocked again; those already connected are not affected.

## 10. Known limits

- The button still depends on the legacy app. If it is restricted again, new users will hit the same error.
- Only the workspace owner could change this setting here, so a fix depends on that one person.

## 11. Reuse for similar issues

**How to recognise one:** a Slack button works for most people but one newer user gets "not authorized to install ... restricted for this workspace."

- It is an app approval setting in Slack, not a Salesforce problem.
- Get the exact app from the error page's address before searching app management.
- Check who can actually approve apps. App managers may not be able to override a denial; the workspace owner can.
- Give the user a manual workaround in Salesforce while the approval waits.
