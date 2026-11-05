# SFTP access for records retrieval

| | |
|---|---|
| **Ticket** | DDT-1101 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-28 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-23 |
| **Resolved by** | Support team |
| **Systems** | FileZilla (SFTP client), ChartSwap Insights (records retrieval portal) |
| **Area** | Medical records retrieval |
| **Related** | None recorded |
| **Code** | Not applicable: desktop setup and walkthrough, no code |

---

## 1. Summary

A staff member asked for help setting up SFTP access to ChartSwap Insights using FileZilla, so records could be pulled over SFTP to cut retrieval costs. This was a setup walkthrough, not a Salesforce integration. An IT helper arranged a call with her to get FileZilla working, and the ticket was closed on the day of that call.

## 2. What was reported

- **2026-09-23:** a staff member asked for a walkthrough on installing FileZilla, getting SFTP credentials from ChartSwap Insights, connecting, and pulling a test record. She did not yet have credentials.
- The business reason: retrieving records by SFTP instead of the current method costs less.

## 3. Scope

**In scope:** installing FileZilla on the staff member's computer, obtaining SFTP connection details from ChartSwap Insights, connecting, and retrieving a test record.

**Out of scope:** any Salesforce integration. The ticket says so explicitly.

## 4. How it works (background)

```
ChartSwap Insights (records vendor) offers an SFTP server
  -> staff member connects with an SFTP client (FileZilla)
     using the host, username and password or key that ChartSwap issues
  -> downloads the records to her computer
```

## 5. Root cause

Not applicable: this was a request for setup help, not a fault.

## 6. Solution

- **2026-09-24:** an IT helper offered to set up a call to get FileZilla running and shared their availability.
- The staff member booked the call for **2026-09-28**.
- The ticket was closed on 2026-09-28.

What was done on the call, and whether the credentials were obtained and a test record retrieved, is not recorded in the ticket.

## 7. Testing

The acceptance criteria were: FileZilla installed, ChartSwap SFTP credentials obtained, a successful connection, and a test record retrieved end to end. Whether each was confirmed is not recorded in the ticket.

## 8. Step by step

The steps the ticket asked for, in order. The details of how the call went are not recorded.

1. **Install FileZilla** (the client, not the server) on the staff member's computer.
2. **Request SFTP access from ChartSwap Insights:** host, port, username, and password or key.
3. **In FileZilla,** open the Site Manager, add a new site, set the protocol to **SFTP**, and enter the details from step 2.
4. **Connect.** FileZilla asks you to trust the server's key on the first connection.
5. **Download a test record** and confirm it opens.

## 9. Rollback

Not applicable: nothing in a shared system was changed. Uninstalling FileZilla removes the client.

## 10. Known limits

- Records downloaded by SFTP land on the staff member's computer. Where they should be stored afterwards is not recorded in the ticket.
- The outcome of the walkthrough is not recorded.

## 11. Reuse for similar issues

- For any new SFTP vendor, the same four facts are needed: host, port, username, and password or key. Get them from the vendor first; the client setup takes minutes once you have them.
- Book a short screen-share call rather than writing instructions back and forth. That is how this one was handled.
- Never paste SFTP credentials into a ticket, Slack or a document.
