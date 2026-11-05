# Locked out of Salesforce: passkey and verification codes

| | |
|---|---|
| **Ticket** | DDT-575, DDT-587, DDT-769, DDT-829, DDT-855, DDT-856, DDT-1130, DDT-1229, DDT-1262 |
| **Project** | Support tickets (IT) |
| **Status** | Done 2026-10-08 (last of the nine closed) |
| **Priority** | Urgent (DDT-575, DDT-587, DDT-856, DDT-1229), High (the other five) |
| **Reported by** | Firm staff, including attorneys, 2026-07-29 to 2026-10-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce login and identity verification (passkeys, Salesforce Authenticator, temporary verification codes), the Salesforce mobile app, the device's own password manager |
| **Area** | Getting a user back into Salesforce when the sign-in or identity-verification step fails |
| **Related** | None recorded |
| **Code** | Not applicable: no code or metadata changed. Every fix was done in Setup or by walking the user through their device. |

---

## 1. Summary

Over ten weeks, nine staff lockouts came in with the same shape: the user could not get past the Salesforce sign-in page, either because the identity-verification step wanted a passkey the user could not produce, or because Salesforce answered "Please check your username and password" to a password the user was sure of. In every case the Support team got the user back in, in most cases the same day, by working on the user record in Setup (a temporary verification code, a password reset, or clearing a broken verification method) or by walking the user through the device side of the passkey. One case had a different cause altogether: Salesforce had frozen the user automatically after seeing a sign-in through an anonymising VPN.

## 2. What was reported

Nine tickets, from seven different users. One user raised three of them.

- **DDT-575 (2026-07-29).** Clicking "Create Passkey" asked for a 6-digit PIN, but the user's PIN screen only accepted 4 digits. The user had no phone PIN at all and unlocked with a fingerprint. Resolved the same day.
- **DDT-587 (2026-07-30).** Signing in with a phone (the QR-code option) returned "no passkey found", although it had always worked before. The sign-in page told the user to ask an admin for a temporary verification code. Fixed that morning, then **recurred on 2026-07-31** and was fixed again the same way.
- **DDT-769 (2026-08-19).** After buying a new phone, the user pressed "verify identity" and nothing happened: no code, no error. The next morning the user could sign in on a computer with a passkey, but still not on the new phone.
- **DDT-829 (2026-08-24).** Signed out for inactivity mid-task, then "Please check your username and password" on every attempt, on two devices, and the reset-password link led back to the login page. The same user was locked out of their Google account at the same time.
- **DDT-855 (2026-08-27).** Salesforce had started asking for a passkey instead of the Windows PIN the user was used to, and creating a passkey on the phone kept failing. Locked out again on 2026-08-28 and on 2026-08-31, on the laptop as well as the phone.
- **DDT-856 (2026-08-27).** The password-reset email arrived and the reset completed, but signing in still returned "Please check your username and password". Fixed within about fifteen minutes.
- **DDT-1130 (2026-09-25).** The Salesforce mobile app returned "Please check your username and password". The user had already tried Forgot Password and had connected Salesforce Authenticator with its two-word phrase. The first person to pick it up did not have the permission to reset authentication methods and passed it on.
- **DDT-1229 (2026-10-05).** The "Verify Your Identity" page asked for a passkey and told the user to request a temporary verification code from an admin. Locked out from early morning until the code was issued.
- **DDT-1262 (2026-10-07).** The same "contact your admin for a temporary verification code" page, reported by an attorney who was due in court that morning.

**Impact.** Each one was a user fully unable to work in Salesforce. Two of the reports came from attorneys with a court appearance that morning.

## 3. Scope

**In scope**

| Where | What was used |
|---|---|
| Setup → Users → the user's record | Generate a temporary verification code; reset the password; check whether the user is locked or frozen |
| Setup → Identity Verification History | Named in DDT-855 as the place to see which verification methods are registered and to clear stale ones |
| The user's device | The passkey itself, held by the device's password manager (Google Password Manager in DDT-575 and DDT-587), Bluetooth for phone-based sign-in, and the Salesforce Authenticator app |

**Out of scope**

- **Changing the org's identity-verification policy.** None of the nine tickets changed which methods Salesforce requires. Each was fixed one user at a time.
- **The user's Google account** (DDT-829). Google Workspace access is a separate system.
- **The Salesforce mobile app defect** raised with Salesforce support during DDT-855. Its outcome is not recorded in the ticket.

## 4. How it works (background)

```
User enters username + password
  -> Salesforce checks the password                 (fails here -> "Please check your username and password")
  -> Salesforce asks the user to verify identity    (passkey, Salesforce Authenticator, or other registered method)
       passkey  -> the browser asks the DEVICE's password manager for it
                   (Windows Hello, Google Password Manager, a phone via QR code + Bluetooth)
  -> if the user has nothing usable, the page says
     "Contact your admin and request a temporary verification code"
```

Three things about this flow explain most of these tickets:

1. **A passkey is not stored in Salesforce.** It lives in a password manager on a device. Salesforce only asks for it. Any PIN or fingerprint prompt the user sees while creating or using a passkey comes from the device, not from Salesforce.
2. **"Please check your username and password" is a generic message.** It appears for a wrong password, but also when the user is locked or frozen. The user cannot tell which, so an admin has to look at the user record.
3. **An admin can always get a user past the verification step** with a temporary verification code, generated on the user's record in Setup. The user then signs in and can register a working method.

## 5. Root cause

There is no single root cause. The tickets fall into three groups.

### 5.1 The passkey was on a device the user could not use (DDT-575, DDT-587, DDT-769, DDT-855)

- **DDT-575:** the 6-digit PIN request came from Google Password Manager on the device, not from Salesforce. The ticket's first guess, that the operating system's PIN policy needed changing, was wrong: the user's own unlock PIN was irrelevant.
- **DDT-587:** phone sign-in uses the phone's passkey through a QR code. A passkey saved to Google Password Manager only shows up while the user's work Google account is signed in on the phone, and the QR handshake needs Bluetooth on both the computer and the phone. The user confirmed both were in order, so the thread concluded an admin had to clear the passkey on the user's account. The exact admin action is not recorded in the ticket.
- **DDT-769:** a new phone has none of the old phone's registrations. Not recorded in the ticket how the phone side was finally resolved.
- **DDT-855:** passkey creation on the phone kept timing out. During the thread, Support attributed the phone failures to the Salesforce mobile app and raised a case with Salesforce. The laptop lockouts were cleared by an admin each time. The outcome of the Salesforce case is not recorded.

### 5.2 The user was frozen by Salesforce's own security (DDT-829)

Salesforce froze the user automatically. Its notice said it had "identified and contained suspicious activity", detected "originating from a high-risk anonymizing proxy", which is "auto-contained in accordance with Salesforce's security policies". The user confirmed they had been connected through a consumer VPN service. The password was never the problem, which is why resetting it led nowhere.

**The lesson.** "Please check your username and password" after a correct password is a reason to open the user record, not to reset the password again.

### 5.3 The password or verification step needed an admin reset (DDT-856, DDT-1130, DDT-1229, DDT-1262)

- **DDT-856 and DDT-1229:** fixed by an admin on the user record within minutes. The exact action is not recorded in the ticket.
- **DDT-1130:** fixed with an admin password reset. The note to the user was to type the new password into the mobile app by hand, without autofill and watching for automatic capitals, then approve the Salesforce Authenticator prompt. That points to the saved or autofilled password in the app as the likely cause, but the ticket does not confirm it.
- **DDT-1262:** a temporary verification code was generated and sent to the user, who then signed in.

## 6. Solution

Every ticket was resolved by hand, one user at a time. Nothing was deployed.

| Ticket | What fixed it |
|---|---|
| DDT-575 | Explained that the PIN prompt belongs to Google Password Manager. Use the device PIN set up for it, or, if forgotten, reset Google Password Manager's passkey data in Chrome (Settings → Google Password Manager → Settings) and create a new passkey. The user reported it resolved. |
| DDT-587 | Checked that the work Google account was signed in on the phone and Bluetooth was on for both devices. An admin then reset the user's verification, twice, on consecutive days. |
| DDT-769 | Closed 2026-09-07. The phone-side fix is not recorded. |
| DDT-829 | The administrator contacted the user privately about the automatic freeze and the VPN. The steps to restore access are not recorded. |
| DDT-855 | An admin reset the user's verification on 2026-08-27 and again on 2026-08-31. A case was raised with Salesforce about the mobile app. |
| DDT-856 | Admin fix on the user record, about fifteen minutes. |
| DDT-1130 | Admin password reset, then the new password typed by hand in the mobile app and the Authenticator prompt approved. |
| DDT-1229 | Admin fix on the user record. The user signed in under two hours after the ticket was opened. |
| DDT-1262 | Temporary verification code generated in Setup and given to the user. |

**Why a temporary verification code and not removing verification.** The code gets the user past the verification step once, without weakening the requirement for anyone else. Once in, the user can register a method that works.

## 7. Testing

**How each fix was checked.** In every ticket, the check was the user confirming they could sign in ("I'm in", "It's working now"). Nothing else was measured.

**Not covered.**

- Whether each user registered a working, durable verification method afterwards. DDT-587 and DDT-855 both recurred within days, which suggests at least those two did not.
- The phone side of DDT-769 and DDT-855.
- What happened to the Salesforce case raised during DDT-855.

## 8. Step by step

For the next "I can't get into Salesforce" report:

1. **Ask what the screen says**, and for a screenshot. "Verify your identity", "no passkey found" and "check your username and password" are different problems.
2. **If it is a verification screen asking for a passkey:**
   1. Ask whether a PIN or fingerprint prompt is involved. If so, it comes from the device's password manager (Windows Hello, Google Password Manager), not Salesforce. Have the user use that manager's PIN, or reset that manager's passkey data and create a new passkey.
   2. If they are using a phone to sign in by QR code, check that their work Google account is signed in on the phone and that Bluetooth is on for both devices.
   3. If that does not work, an admin opens **Setup → Users → the user → Generate temporary verification code** and gives the user the code **privately**.
3. **If it says "Please check your username and password" and the user is sure of the password:**
   1. Open the user record in Setup and check whether the user is **locked or frozen** before resetting anything.
   2. If frozen, check the user's login history and any Salesforce security notice. A sign-in through an anonymising VPN or proxy is frozen automatically. Ask the user to disconnect from it before they sign in again.
   3. Otherwise reset the password. Tell the user to type the new password by hand (no autofill, watch for automatic capitals), especially in the mobile app.
4. **Check the person working the ticket can do this.** Resetting a user's authentication methods needs admin permissions in Salesforce. In DDT-1130 the first person to pick it up could not, which cost a weekend.
5. **Once the user is in**, have them register a verification method that will work next time, and check it in **Setup → Identity Verification History**.
6. **Confirm with the user** that they can sign in, on every device they need.

## 9. Rollback

Not applicable: nothing was changed that would need undoing. A temporary verification code expires on its own, and a password reset is replaced by the password the user sets.

## 10. Known limits

- **These fixes are per user and they recur.** One user raised three of these tickets and was locked out on at least five separate days between 2026-07-30 and 2026-08-31. A temporary code gets someone in today; it does not stop the next lockout.
- **A new phone starts from nothing.** Anyone who replaces their phone will lose phone-based passkeys and Authenticator until they register again.
- **Deliver codes privately.** A temporary verification code is a sign-in credential. Send it to the user directly, never in a shared channel or a ticket comment.
- **A frozen user cannot be fixed with a password reset**, and is easy to misread as a password problem because the error message is the same.
- **Out-of-hours lockouts wait.** DDT-1130 was reported on a Friday evening and fixed on Monday.

## 11. Reuse for similar issues

**How to recognise one:** a user says they cannot get into Salesforce, mentions a passkey, a PIN, a verification code, a new phone or "check your username and password".

- **Separate the two halves of sign-in:** the password check and the identity-verification step fail for different reasons and have different fixes.
- **A PIN or fingerprint prompt during passkey setup belongs to the device**, not Salesforce. Changing the operating system's PIN policy is almost never the fix.
- **Open the user record before resetting a password.** Locked and frozen users look exactly like wrong passwords from the login page.
- **Keep the temporary verification code as the standard way in**, and follow it with registering a working method so the same user does not come back next week.
- **A user who keeps coming back** (one user raised three of these nine tickets) needs their registered methods reviewed, not another code.
