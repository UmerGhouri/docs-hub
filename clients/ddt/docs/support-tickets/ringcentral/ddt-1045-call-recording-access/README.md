# Granting access to call recordings and ACE

| | |
|---|---|
| **Ticket** | DDT-1045, DDT-1271, DDT-1403 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-26 |
| **Priority** | Medium (DDT-1403 High) |
| **Reported by** | Firm staff, 2026-09-17, 2026-10-08 and 2026-10-22 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (user permissions, ACE licence, desktop app, admin web portal) |
| **Area** | Staff listening to past call recordings |
| **Related** | None recorded |
| **Code** | Not applicable: nothing was changed in Salesforce or the repo |

---

## 1. Summary

Three requests from staff who could not search or play back past call recordings in RingCentral. In the first, the user already had the team-recordings permission and needed to be shown where recordings live. In the second, the ACE screen returned "Something went wrong"; what fixed it is not recorded. In the third, two newly created users had never been given the **ACE** licence (RingCentral's AI Conversation Expert); it was assigned, and it appeared only after they signed out and back in. The standing rule from that ticket: assign ACE to every new RingCentral user when the account is created.

## 2. What was reported

- **DDT-1045, 2026-09-17.** A staff member asked to listen to past call recordings, for the whole team rather than just her own calls. After being pointed at the call history, she reported seeing a transcript but no play icon.
- **DDT-1271, 2026-10-08.** A staff member who used to search the app by phone number, find a call and listen to it now got an "ACE — Something went wrong" page instead. Logging out and back in twice did not clear it.
- **DDT-1403, 2026-10-22.** A supervisor reported that two newly created users had no access to ACE.

## 3. Scope

**In scope**

| Component | Where | Change |
|---|---|---|
| The users' recording permissions | RingCentral admin portal | DDT-1045: checked; the team-recordings permission was already in place |
| ACE licence | RingCentral admin portal | DDT-1403: assigned to the two new users |
| Outbound caller ID for new users | RingCentral admin portal | DDT-1403: confirmed set to the firm's standard outbound number |
| ACE error page | RingCentral | DDT-1271: fix not recorded |

**Out of scope**

- Call recordings shown inside Salesforce. One reply asked whether the user meant Salesforce; she was working in RingCentral.
- Changing who may hear which recordings. Each user was given access matching what they asked for; the scope rules themselves were not changed.

## 4. How it works (background)

RingCentral offers recordings in more than one place, with different reach:

```
RingCentral desktop app
  -> Phone tab -> History        the user's own calls; recorded ones show a play icon

RingCentral web admin portal
  -> Analytics / Call Log        the whole team's calls and recordings,
                                 if the user has that permission

ACE (AI Conversation Expert)
  -> search past calls, read transcripts, play recordings
     needs the ACE licence on the user
```

Having permission for team recordings does not by itself put a play button in the desktop app's History, which shows the user's own calls.

## 5. Root cause

### DDT-1045: permission present, wrong place to look

The user's account already had the permission for the team's call recordings. The question was where to find them: the desktop app's History shows only the user's own calls, and the team's recordings are in the web admin portal's call log. Her later report of a transcript with no play icon was not followed up in the ticket; the final outcome is not recorded.

### DDT-1271: ACE "Something went wrong"

Access had worked before and stopped. Logging out and in did not help. The root cause and fix are not recorded in the ticket.

### DDT-1403: ACE licence never assigned

The two users had been created in RingCentral without the ACE licence. ACE is a separate licence, not part of the standard user setup, so a new user has no ACE until someone assigns it. Even after it was assigned, the users did not see it until they **signed out and back in**.

## 6. Solution

### DDT-1045

The user was given both locations:

1. Desktop app → Phone tab → History: recorded calls show a play icon (own calls).
2. RingCentral web admin portal, signed in with her work account → Analytics / Call Log: the whole team's recordings.

### DDT-1271

Not recorded in the ticket.

### DDT-1403

1. The ACE licence was assigned to both users in the RingCentral admin portal.
2. Both users were asked to sign out of RingCentral fully and sign back in.
3. Their outbound caller ID was confirmed as the firm's standard outbound number.

The firm's standing instruction from this ticket: **always assign the ACE licence when creating a RingCentral user**, and set the outbound caller ID at the same time.

## 7. Testing

- **DDT-1403:** one of the two users confirmed ACE worked after signing out and back in. Confirmation from the second user is not recorded.
- **DDT-1045 and DDT-1271:** user confirmation is not recorded in the tickets.

**Not covered:** there is no check that catches a new RingCentral user created without ACE. It depends on the person creating the account.

## 8. Step by step

**A user cannot find recordings (DDT-1045 shape)**

1. Ask whether they need their own calls or the whole team's.
2. In the RingCentral admin portal, check the user's role includes call recording access at that scope.
3. For own calls, point them at the desktop app → Phone → History.
4. For the team's calls, point them at the web admin portal → Analytics / Call Log.
5. If they see a transcript but no play icon, check whether that call was recorded at all, and follow up.

**A user has no ACE, or ACE shows an error (DDT-1403 and DDT-1271 shape)**

1. In the RingCentral admin portal, check whether the ACE licence is assigned to the user.
2. If not, assign it.
3. Ask the user to sign out of RingCentral completely and sign back in. **The licence does not appear in an existing session.**
4. Confirm the user can open ACE, search for a call and play it.
5. If the licence is assigned and the error persists after a fresh sign-in, raise it with RingCentral support.

**Creating any new RingCentral user**

1. Assign the ACE licence.
2. Set the outbound caller ID to the firm's standard outbound number.

## 9. Rollback

Remove the ACE licence or the recording permission in the RingCentral admin portal. Safe; it only removes access.

## 10. Known limits

- A user whose access changes must sign out and back in before it takes effect. Until they do, they will report that nothing changed.
- The DDT-1271 error page has no recorded cause, so if it recurs there is no known fix to apply.

## 11. Reuse for similar issues

**How to recognise one:** a staff member cannot search, see or play past calls in RingCentral, or ACE shows "Something went wrong".

- **Check the licence before the permission.** ACE is separate from the user's role.
- **Always ask for a full sign-out and sign-in** after any licence or permission change. In DDT-1403 the change was correct and still looked broken until the users did this.
- **Ask what scope they need:** own calls and the team's calls are found in different places.
- **Build ACE into new-user setup**, so the request never has to be made.
