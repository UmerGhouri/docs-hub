# Trust Ledger

Trust (IOLTA) accounting inside Salesforce. The bank leg and the client ledgers are reconciled in Salesforce, and this work adds the third leg, the firm's books, plus the monthly signed reconciliation the trust-account rules require. The project is still in progress; this section covers its completed tickets.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-835 | [Settlement statement showed all zeros when its data failed to load](./ddt-835-settlement-statement-silent-zero-load/README.md) | Live since 2026-08-31 |
| DDT-838 | [Settlement statement access follows matter access again](./ddt-838-settlement-statement-record-sharing/README.md) | Live since 2026-09-28 |
| DDT-842 | [Trust tie-out tests passed or failed depending on who ran them](./ddt-842-tie-out-tests-own-permissions/README.md) | Live since 2026-09-24 |
| DDT-843 | [A good recovery save was reported as a failed save](./ddt-843-recovery-save-reload-failure/README.md) | Live since 2026-09-24 |
| DDT-847 | [Keep rejected liens off the client settlement statement](./ddt-847-exclude-rejected-liens-statement/README.md) | Live since 2026-08-27 |
| DDT-863 | [A green pipeline on a Lightning component change proves nothing](./ddt-863-green-pipeline-lwc-rule/README.md) | Done 2026-09-03 |
| DDT-884 | [Trust tie-out columns ran together and the upload summary was unreadable](./ddt-884-tie-out-columns-and-summary/README.md) | Live since 2026-09-01 |
| DDT-894, DDT-895 | [Trust tie-out sign-off locked behind five numbered steps](./ddt-894-tie-out-numbered-sign-off/README.md) | Live since 2026-09-24 |
| DDT-896 | [Weekly trust worksheet emailed to the bookkeeper](./ddt-896-weekly-bookkeeper-trust-worksheet/README.md) | Live since 2026-10-20 |
| DDT-897, DDT-1314 | [Signed trust reconciliation statement with a ledger per client](./ddt-897-signed-reconciliation-statement-pdf/README.md) | Live since 2026-10-26 |
| DDT-898 | [Bank file deposits now fill the landed date in reconciliation step 3](./ddt-898-match-bank-file-deposits/README.md) | Live since 2026-10-12 |
| DDT-899 | [Trust bank feed to QuickBooks stopped and the books fell behind](./ddt-899-bank-feed-stopped-in-quickbooks/README.md) | Done 2026-10-13 |
| DDT-900 | [Put the trust feature in the development sandbox](./ddt-900-trust-stack-in-sandbox/README.md) | Done 2026-09-25 |
| DDT-901 | [Trust tie-out totals would understate past the row limit](./ddt-901-tie-out-totals-past-row-cap/README.md) | Live since 2026-09-25 |
| DDT-1150 | [Trust reconciliation stopped being run after 1 September](./ddt-1150-reconciliation-not-being-run/README.md) | Done 2026-10-12 |
| DDT-1312 | [Trust Tie-Out: seven fixes from the first live sign-off](./ddt-1312-tie-out-seven-fixes/README.md) | Live since 2026-10-26 |
| DDT-1313, DDT-1378 | [Trust balance per client report for the bookkeeper](./ddt-1313-trust-balance-per-client/README.md) | Live since 2026-10-27 |
| DDT-1416 | [A deposit not at the bank after one business day turns red](./ddt-1416-late-deposit-turns-red/README.md) | Live since 2026-10-28 |
