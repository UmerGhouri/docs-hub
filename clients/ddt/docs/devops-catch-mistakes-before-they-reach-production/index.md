# DevOps: catch mistakes before they reach production

Sets the rules for how code reaches production: every change goes through a reviewed pull request, a validation against production gates the merge, and the repository is kept an exact record of what is live. The tickets fixed the gaps that let drift, untested code and false green checks through.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-596 | [Clear the failing Apex tests in production, and restore a lost Local Services Ads feature](./ddt-596-clear-production-apex-test-failures/README.md) | Live since 2026-08-12 |
| DDT-601 | [Auto-close stale pull requests and check production against git](./ddt-601-stale-pr-close-and-drift-check/README.md) | Done 2026-08-11 |
| DDT-602, DDT-603 | [Bring Apex that was live in production but missing from git into the repository](./ddt-602-record-live-apex-in-git/README.md) | Done 2026-08-17 |
| DDT-639 | [AI code reviewer blocks merges only on serious findings](./ddt-639-ai-reviewer-blocks-only-serious/README.md) | Done 2026-08-05 |
| DDT-747 | [Make the Salesforce validation a required check before merge](./ddt-747-require-salesforce-validation-check/README.md) | Done 2026-08-17 |
| DDT-836 | [Restore the missing test for the Case inbox trigger](./ddt-836-restore-case-inbox-trigger-test/README.md) | Live since 2026-08-25 |
| DDT-869 | [Review-thread janitor fails on older pull request branches](./ddt-869-janitor-pin-workflow-file/README.md) | Done 2026-10-28 |
| DDT-965 | [Validation gate posted a failure for runs that did not fail](./ddt-965-validation-gate-false-red/README.md) | Done 2026-10-28 |
| DDT-1027 | [Stop merged pull requests from marking tickets Done](./ddt-1027-stop-merge-auto-closing-tickets/README.md) | Done 2026-10-28 |
