# Call monitoring setup and silent listen-in

| | |
|---|---|
| **Ticket** | DDT-1033, DDT-1046 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-24 (DDT-1033 done 2026-09-18) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-16 and 2026-09-17 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (Call Monitoring groups, desktop app) |
| **Area** | Supervisors listening in on live calls for coaching and QA |
| **Related** | None recorded |
| **Code** | Not applicable: nothing was changed in Salesforce or the repo |

---

## 1. Summary

Two supervisors could not listen in on live calls in RingCentral. In the first case the listen (headphones) icon was missing for one team member, because that person was not in the supervisor's Call Monitoring group; the firm's RingCentral administrator added them. In the second case the supervisor's setup was correct and the desktop app was the right device, so the answer was to check that the target was inside her monitoring rights and then work through three app-side checks.

## 2. What was reported

- **DDT-1033, 2026-09-16.** A supervisor needed to listen in on one team member's calls for coaching and QA. In the RingCentral Extensions view, the headphones icon showed beside two other team members but was missing from this person's row.
- **DDT-1046, 2026-09-17.** A different staff member could start a listen-in on active calls but heard no audio. She used the RingCentral desktop app, was trying to listen to intake calls, and said it had never worked for her on any call, day or time.

One supervisor and one target user in DDT-1033; one user in DDT-1046.

## 3. Scope

**In scope**

| Component | Where | Change |
|---|---|---|
| The supervisor's Call Monitoring group | RingCentral admin portal | DDT-1033: the missing team member added as a monitored user, with Listen allowed |
| The second user's monitoring rights and app | RingCentral admin portal and desktop app | DDT-1046: checked, no change recorded |

**Out of scope**

- Salesforce. Call monitoring is configured entirely in RingCentral.
- Widening the second user's monitoring rights beyond the teams she already covers. Nobody asked for that.

## 4. How it works (background)

```
RingCentral admin portal
  -> Call Monitoring group
       Monitoring users  (the supervisors who may listen in)
       Monitored users   (the extensions they may listen to)
       Permissions       (Listen, and optionally other actions)
  -> supervisor's app shows a headphones icon beside each monitored user
  -> clicking it during a live call joins the call silently
```

A supervisor can only listen to extensions that sit in a group where they are a monitoring user. Anyone outside those groups shows no icon.

## 5. Root cause

### DDT-1033: target not in the group

The team member was not a monitored user in the supervisor's Call Monitoring group, while the two people whose icons did show were. That is exactly what a missing headphones icon means.

### DDT-1046: setup correct, cause on the target or app side

RingCentral confirmed call monitoring was enabled for the user, and the desktop app is a supported device for listen-in. Her monitor rights covered two teams, **CMT** and **Defense: Intake**. That left two possibilities:

1. The call she tried to monitor belonged to someone outside those two teams, which cannot work.
2. The call belonged to someone she can monitor, in which case the problem is the app, not the setup.

She replied that she had been trying to listen to intake calls, which are inside her rights. Which of the app-side checks fixed it is not recorded in the ticket.

## 6. Solution

### DDT-1033

The firm's RingCentral administrator opened the supervisor's Call Monitoring group and added the team member as a monitored user, with the Listen permission.

### DDT-1046

The user was given these checks, in order:

1. **Settings → Audio.** Confirm the speaker is the right device, then use "Test speaker" and confirm the tone is heard.
2. **Settings → About → Check for updates.** Monitoring breaks on older builds.
3. **Sign out fully and sign back in.** The app session re-registers, which clears a stale connection.

If all three failed on a user she is allowed to monitor, the next step was to raise it with RingCentral support. Whether that was needed is not recorded in the ticket.

## 7. Testing

- **DDT-1033:** the acceptance check was that the headphones icon appears beside the team member in the supervisor's Extensions view and that Listen works on a live call. The confirmation itself is not recorded in the ticket.
- **DDT-1046:** not recorded in the ticket.

**Not covered:** no test call was recorded for either ticket.

## 8. Step by step

**Icon missing for one person (DDT-1033 shape)**

1. Ask the supervisor which person is missing the headphones icon, and confirm the icon shows for others.
2. In the RingCentral admin portal, open the Call Monitoring group the supervisor belongs to.
3. Add the person as a monitored user and make sure Listen is allowed.
4. Ask the supervisor to check the Extensions view for the icon, and try Listen on a live call.

**Listen-in connects but has no audio (DDT-1046 shape)**

1. In the admin portal, confirm the user is a monitoring user in a Call Monitoring group, and note which teams the group covers.
2. Ask which person or queue they were trying to monitor. If it is outside those groups, that is the answer.
3. Ask which device they use. The desktop app is supported.
4. Have them check Settings → Audio and run the speaker test.
5. Have them update the app from Settings → About.
6. Have them sign out fully and back in.
7. If it still fails on a person they are allowed to monitor, raise a case with RingCentral support with the user's details.

## 9. Rollback

Remove the person from the Call Monitoring group. Safe at any time; it only removes the listen option.

## 10. Known limits

- A new team member is not added to any monitoring group automatically. Each new hire has to be added to their supervisor's group, or the icon will be missing again.
- Monitoring rights are per group. A supervisor who moves to cover another team needs that team's members added.

## 11. Reuse for similar issues

**How to recognise one:** a supervisor can listen to some people but not others, or can start a listen-in but hears nothing.

- **Icon missing for one person:** that person is not a monitored user in any group the supervisor monitors. Compare against someone whose icon does show.
- **Icon present, no audio:** check the target is inside the supervisor's groups first, then work through audio device, app version and a full sign-out.
- **Monitoring is RingCentral configuration, not Salesforce.** Nothing in the repo or the org needs changing.
