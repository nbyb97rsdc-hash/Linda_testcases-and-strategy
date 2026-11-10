# Strategy Note

This test plan treats inventory consistency and oversell prevention as the top priority (P0), because inventory inconsistency leads directly to financial loss.

## Priorities

| Priority | Scope | Notes |
| --- | --- | --- |
| P0 | Inventory reserve/deduct, cancel/refund/return restock, idempotency, reconciliation, oversell prevention, payment-deduction consistency | Involves financial and oversell risk; all scenarios fully covered |
| P1 | Normal order lifecycle, WMS shipment callback, partial/split shipment, refund flow, SC2P payment sync | Core flows, fully covered |
| P2 | Cart experience, page UI, minor format display, low-frequency edge cases | Covered if time permits; deferrable under resource constraints |
| Non-functional | Performance and reliability focus on the flash-sale / inventory path | Low-traffic paths skip high-concurrency testing to save cycle time |

## Key Trade-offs

1. **P0 focuses on financial risk**: inventory correctness (oversell prevention) outranks experience and display. Cart and UI — which have no direct financial impact — are downgraded to P2, prioritizing test depth on the inventory path.
2. **Separate "oversell correctness" from "high-concurrency load testing"**: oversell prevention (no over-deduction under concurrency) is P0 functional correctness; the 2000-concurrency performance test is a non-functional item run only on the flash-sale/inventory core path. Other low-traffic interfaces skip load testing to shorten the test cycle.
3. **Split return/refund flow priority**: the "restock correctness" part of returns is P0; the full return interaction and display flow is P1, so secondary flows don't slow P0 progress.
4. **Dual-path split**: the WMS inventory path gets full P0 coverage; the SC2P payment path gets P1 coverage on payment-event sync and three-way reconciliation, with emphasis on the consistency hand-off between payment and deduction.
5. **Built-in quality over firefighting**: observability (inventory difference rate, queue backlog, stuck orders, restock retention) and reconciliation are designed up front, so issues don't depend on after-the-fact manual triage.

Beyond the happy path, all exception and compensation logic is verified in depth. Consistency uses dual protection: "runtime idempotency control + offline reconciliation".
