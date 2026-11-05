# CloudFiles integration moved to the Client Credentials flow

| | |
|---|---|
| **Ticket** | DDT-582 |
| **Project** | Support tickets (Salesforce) |
| **Status** | Done 2026-07-30 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-07-30 (passing on a request from the vendor) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (External Client App for the CloudFiles managed package), CloudFiles admin settings |
| **Area** | The connection CloudFiles uses to reach Salesforce, which carries file uploads from Matters and Court Cases to external storage |
| **Related** | DDT-109 (a CloudFiles upload error in the same managed package) |
| **Code** | Not applicable: no PR. The change was made in Salesforce Setup and in CloudFiles' admin settings. |

---

## 1. Summary

CloudFiles, the managed package DDT used to move Salesforce files to external storage, asked the firm to switch its Salesforce connection from **user-based OAuth** to the **Client Credentials flow**. The vendor said the old method would stay supported but "may become more prone to errors going forward", following Salesforce's changes to OAuth settings. The firm approved the switch on 2026-07-29 and the Support team recorded it as done on 2026-07-30. This was preventive work, not a fix for an outage.

## 2. What was reported

The vendor's product team emailed twice (2026-07-20 and 2026-07-24) recommending the change. In their words, Client Credentials "is a direct server-to-server connection that's more reliable". The firm approved it by email on 2026-07-29, and the ticket was opened on 2026-07-30.

Nothing was broken at the time. No failed uploads or error counts are recorded against this ticket.

## 3. Scope

**In scope (as planned in the ticket)**

- The **External Client App** for the CloudFiles package in Salesforce, with the Client Credentials grant enabled.
- A **run-as user** for the connection, which this flow requires.
- The **CloudFiles connection settings**, switched from user-based OAuth to Client Credentials.

**Out of scope**

- **DDT-109**, an upload error inside the CloudFiles package. The ticket asked only that the new connection not make it worse.
- **CloudFiles' own Salesforce automation** (its flows and permission sets). They were not changed by this ticket.

## 4. How it works (background)

```
Before:  CloudFiles  --(each user's own OAuth authorisation)-->  Salesforce
After:   CloudFiles  --(app's client id + secret, Client Credentials)-->  Salesforce
                         runs as one designated Salesforce user
```

- **User-based OAuth:** each person authorises the app, and CloudFiles acts as that person. Every user's authorisation is a separate thing that can expire or be revoked.
- **Client Credentials:** CloudFiles authenticates as the app itself, server to server. Salesforce runs every call as **one designated user**, so that user's access decides what CloudFiles can do.

This is the same family of setup as other vendor integrations at the firm that use an External Client App with a dedicated integration user; only the grant type differs.

## 5. Root cause

Not applicable as a fault. The reason for the change was the vendor's advice that user-based OAuth would become less reliable as Salesforce changes its OAuth settings. The ticket notes the firm's history of catching vendor deprecations only after they break in production, and treats this as a chance to migrate on its own schedule instead.

## 6. Solution

The ticket's plan, in order:

1. Get the vendor's "Client Credentials Flow Setup" guide and read it end to end first.
2. In Salesforce, enable the **Client Credentials** grant on the CloudFiles External Client App, and set the **run-as user**. The plan recommended a dedicated integration user with a least-privilege permission set covering only what CloudFiles needs.
3. In CloudFiles' admin settings, switch the connection to Client Credentials, enter the app's client id and secret, and test the connection.
4. Cut over in parallel if CloudFiles allows it, otherwise in a quiet window, with a rollback plan ready.
5. Test uploads from Matters and Court Cases, the upload path behind DDT-109, and any scheduled syncs, as more than one user.
6. Store the client secret in the password vault, document the setup in the repo, and confirm to the vendor.

**What is recorded as done:** the Support team marked the ticket done on 2026-07-30 with the comment "its done". Which run-as user was chosen, which permissions it holds, where the secret is stored, and which of steps 4 to 6 were carried out are **not recorded in the ticket**.

## 7. Testing

Not recorded in the ticket. No test results, test users or upload checks are written down.

**Not covered, as far as the record shows:** the DDT-109 upload path, scheduled syncs, and a written rollback.

## 8. Step by step

To repeat this for CloudFiles or a similar vendor integration:

1. **Get the vendor's setup guide** and confirm which grant type and which Salesforce app they expect.
2. **Decide the run-as user.** Prefer a dedicated integration user over a person. Grant it only what the vendor needs, and ask the vendor for their minimum-access list.
3. **In Setup, open the vendor's External Client App** and enable the Client Credentials flow in its OAuth policies. Set the run-as user.
4. **Put the client id and secret into the vendor's admin settings** and test the connection there.
5. **Test the real work** the integration does (here, file uploads from Matters and Court Cases), not just the connection test.
6. **Store the secret in the vault**, and write down where it is and how to rotate it.
7. **Retrieve the changed app policy into the repo** and open a PR, so git records the change.
8. **Tell the vendor** the migration is done.

## 9. Rollback

Not recorded in the ticket. The plan asked for a way to return to user-based OAuth quickly. In principle that means switching the connection back in CloudFiles' admin settings and turning the Client Credentials flow off on the app, but that procedure was not written down or tested.

## 10. Known limits

- **The repo has no record of this change.** No commit after 2026-07-30 touches the CloudFiles External Client App policy, so the repo copy predates the switch. Anyone checking the setup should read it in Setup, not from git.
- **Everything CloudFiles does now runs as one user.** If that user is deactivated, frozen or loses a permission, CloudFiles stops working for everyone at once.
- **The client secret needs rotating**, and no rotation schedule is recorded.
- **CloudFiles was later being retired.** From 2026-08-05 the firm began replacing CloudFiles with its own Google Drive integration and preparing to cancel the licence, so this connection may no longer be in use.

## 11. Reuse for similar issues

**How to recognise one:** a vendor writes that your integration "uses the default user-based authentication" and recommends a server-to-server method.

- **Treat it as preventive work with a deadline you set**, before the old method fails in production.
- **Choose the run-as user deliberately.** It becomes the integration's entire set of permissions.
- **Test the integration's real jobs**, not only the vendor's connection check.
- **Write the record as you go:** the run-as user, its permissions, where the secret lives, the rotation plan, and a rollback. A ticket closed with "done" leaves the next person to rediscover all of it.
- **Commit the changed app policy.** A change made only in Setup is invisible to anyone reading the repo.
