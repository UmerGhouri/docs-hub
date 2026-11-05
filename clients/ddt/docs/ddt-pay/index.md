# DDT Pay

The firm's own payment and invoicing tool, built natively on Authorize.Net to replace the previous managed payments package. It covers payment plans, invoices and guided payment actions, with rules that keep the money correct. The project is still in progress; this section covers its completed tickets.

## Issues

| Tickets | Issue | Status |
|---|---|---|
| DDT-338 | [Invoice page: show status and client, add related lists](./ddt-338-invoice-page-related-lists/README.md) | Done 2026-07-20 |
| DDT-339 | [Take a Payment: sensible defaults and a page refresh after paying](./ddt-339-take-payment-defaults-refresh/README.md) | Done 2026-07-20 |
| DDT-340, DDT-347 | [Payment Plan panel: clearer names and how each installment was paid](./ddt-340-payment-plan-panel-paid-view/README.md) | Done 2026-10-29 |
| DDT-341 | [Plan-health warning fired on normal monthly plans](./ddt-341-plan-health-spacing-threshold/README.md) | Done 2026-07-20 |
| DDT-342 | [Re-entering a saved card created a duplicate card on file](./ddt-342-duplicate-card-on-file/README.md) | Done 2026-07-20 |
| DDT-343 | [Changing plan frequency now asks for the new installment amount](./ddt-343-change-frequency-new-amount/README.md) | Done 2026-07-20 |
| DDT-344 | [Re-amortizing a partly paid installment now splits it first](./ddt-344-reamortize-part-paid-installment/README.md) | Done 2026-10-28 |
| DDT-345 | [Prefilling the card entry window: checked and dropped](./ddt-345-card-form-prefill-decision/README.md) | Done 2026-10-30 |
| DDT-346 | [New-Plan Health reports and dashboard for collections](./ddt-346-new-plan-health-dashboard/README.md) | Done 2026-10-29 |
| DDT-350 | [Invoice Builder: running totals, section headings and already-billed warnings](./ddt-350-invoice-builder-upgrade/README.md) | Done 2026-10-29 |
| DDT-590 | [Preserve pre-2024 payment history before the old payments package goes](./ddt-590-preserve-pre-2024-payment-history/README.md) | Live since 2026-08-30 |
| DDT-1315 | [Route each card payment to the Defense or SOS processor](./ddt-1315-route-payments-by-case-class/README.md) | Done 2026-10-28 |
| DDT-1395 | [Money on an invoice only changes through DDT Pay actions](./ddt-1395-lock-money-fields-to-actions/README.md) | Done 2026-10-28 |
| DDT-1396 | [A guided "Waive fee" action in DDT Pay](./ddt-1396-guided-waive-fee-action/README.md) | Done 2026-10-28 |
| DDT-1397 | [Invoice total is checked against its lines and charges](./ddt-1397-invoice-total-from-lines/README.md) | Done 2026-10-28 |
| DDT-1398 | [Autopay never charges more than the client owes](./ddt-1398-cap-autopay-at-balance/README.md) | Done 2026-10-28 |
| DDT-1399 | [Trial fee and Amount sync work on DDT Pay cases](./ddt-1399-fee-paths-on-ddt-pay-cases/README.md) | Done 2026-10-28 |
| DDT-1400 | [Nightly DDT Pay check runs from the shared dispatcher](./ddt-1400-nightly-check-on-dispatcher/README.md) | Done 2026-10-28 |
| DDT-1414 | [A guided Refund action in DDT Pay](./ddt-1414-guided-refund-action/README.md) | Done 2026-10-28 |
| DDT-1436 | [Trust invoices never charge through the operating card processor](./ddt-1436-trust-invoices-block-card-charges/README.md) | Done 2026-10-28 |
