# E-Commerce System Test Plan & Test Cases (MFQ Framework)

## M: Models

### 1. End-to-End Business Flow Model
Browse products (SPU/SKU) → Add to cart → Place order → Reserve inventory → Pay → Deduct inventory → WMS outbound/ship → User signs → (optional) Request return → Restock

### 2. Order State Machine Model
State set:
- Created (pending payment)
- Paid
- Shipped
- Signed
- Cancelled
- Refunding → Refunded
- Returning → Return complete

Transition rules:
- Created + payment success → Paid
- Paid + WMS shipment callback → Shipped
- Shipped + user sign-off → Signed
- Created (unpaid) + cancel order → Cancelled
- Paid + cancel order (requires refund) → Refunding → Refunded
- Signed + return approval → Returning → Return complete

Trigger events: place order, pay, cancel, refund, shipment callback, sign, request return, return review, return complete

### 3. Inventory State Model
Inventory is divided into four states: available, reserved, deducted, restored
Transition rules:
- Place order: available → reserved
- Payment success: reserved → deducted
- Cancel before payment: reserved → available
- Cancel/refund after payment: deducted → restored → available
- Return complete: deducted → restored → available
- Partial shipment: only the shipped quantity moves from reserved to deducted, the rest stays reserved
- Restored inventory must pass reconciliation before becoming available, to prevent double restock

### 4. External Interaction Model
Storefront ↔ WMS warehouse system (reserve/deduct/release requests, shipment callback)
Storefront ↔ SC2P supply-chain payment system (payment events, refund event sync)
Communication: cross-system asynchronous message queues (every operation carries a unique business idempotency key)

### 5. Consistency Boundaries
- Strong consistency: the order-placement inventory-reservation step must be strongly consistent to prevent overselling
- Eventual consistency: WMS shipment status, return restock, and restored inventory rely on scheduled reconciliation
- Reconciliation points: hourly inventory reconciliation, daily order-settlement reconciliation, daily payment/order/inventory three-way reconciliation

### Dedicated Section: Inventory Synchronization Consistency
Inventory sync is the highest-risk point of this system.
- Sync direction: the e-commerce system initiates reserve/deduct/release requests; WMS returns result callbacks
- Consistency guarantee: every inventory operation carries a unique business idempotency key
- Reconciliation mechanism: scheduled jobs compare the e-commerce local inventory snapshot against WMS inventory and raise alerts on any difference
- On detected inconsistency: automatically trigger the compensation flow (release / re-deduct / restock) and keep an audit record

## F: Functional (Test Cases)

> Case tree: F-x are top-level cases, F-x.y are sub-cases; priority P0 = financial loss / oversell risk, P1 = core flow, P2 = minor / experience.

### 1. Core Happy-Path Cases
| Case ID | Case Name | Priority | Expected Result |
| --- | --- | --- | --- |
| F-1.1 | Place order with valid SKU and reserve inventory | P0 | Available -1, reserved +1, order status = Created |
| F-1.2 | Payment success deducts inventory | P0 | Reserved -1, deducted +1, order status = Paid |
| F-1.3 | WMS pushes shipment callback | P1 | Order transitions to Shipped, WMS outbound triggered |
| F-1.4 | User signs off, order closed | P1 | Order status = Signed, lifecycle closed |

### 2. Inventory Sync Cases
| Case ID | Case Name | Priority | Expected Result |
| --- | --- | --- | --- |
| F-2.1 | Reservation success | P0 | Available decreases, reserved increases |
| F-2.2 | Reservation failure (no stock) | P0 | Order rejected directly, inventory unchanged |
| F-2.3 | Deduction success | P0 | Reserved decreases, deducted increases |
| F-2.4 | Deduction failure | P0 | Compensation triggered, reserved released |
| F-2.5 | Order cancel returns reserved inventory | P0 | Reserved returned to available |
| F-2.6 | Duplicate-callback idempotency | P0 | Duplicate message executes once only, no double deduction |
| F-2.7 | Out-of-order message handling | P0 | Late deduction message for a cancelled order is discarded, not executed |

### 3. Cancel / Refund / Return Cases
| Case ID | Case Name | Priority | Expected Result |
| --- | --- | --- | --- |
| F-3.1 | Cancel unpaid order | P0 | Order = Cancelled, reserved returned to available |
| F-3.2 | Cancel paid order (refund) | P1 | Order = Refunding → Refunded, deducted → restored → available, refund initiated |
| F-3.3 | Restock after return | P0 | Return approved, deducted → restored → available, inventory restored |
| F-3.4 | Return-restock reconciliation | P1 | Restored inventory becomes available after reconciliation, no double restock |

### 4. Boundary Scenario Cases
| Case ID | Case Name | Priority | Expected Result |
| --- | --- | --- | --- |
| F-4.1 | Zero-stock order | P1 | Reservation fails, order rejected |
| F-4.2 | High-concurrency flash sale, prevent oversell | P0 | Concurrent orders on the same SKU do not oversell |
| F-4.3 | Partial shipment | P1 | Only the shipped quantity is deducted, the rest stays reserved |
| F-4.4 | Split shipment (multiple WMS callbacks) | P1 | Each callback deducts only its quantity, total deduction ≤ ordered quantity, no duplication |
| F-4.5 | Restock after return | P0 | Inventory restored to available with correct quantity |

### 5. SC2P Payment Sync Cases
| Case ID | Case Name | Priority | Expected Result |
| --- | --- | --- | --- |
| F-5.1 | Payment event sync success | P1 | Order = Paid, reserved deducted to deducted |
| F-5.2 | Duplicate/out-of-order payment events | P1 | Idempotent handling, deducted once only |
| F-5.3 | Payment success but inventory deduction fails | P0 | Compensation releases reserved inventory, refund notification sent to user |
| F-5.4 | Payment/order/inventory three-way reconciliation | P1 | Reconciliation difference = 0; any difference raises alert and compensation |

### 6. Cart Cases
| Case ID | Case Name | Priority | Expected Result |
| --- | --- | --- | --- |
| F-6.1 | Add / update / merge cart | P2 | Item quantity and SKU updated correctly |
| F-6.2 | Checkout reserves inventory from cart | P1 | Inventory is reserved only at order placement; staying in cart holds no stock |

### 7. Critical API Validation
- Request parameters: required fields, data types, value ranges, idempotency-key validation
- Response results: business error codes, error messages
- Callback/writeback interfaces: validate order status, inventory quantity, completeness of returned fields
- Payment/refund writeback interfaces: validate payment serial number, amount, idempotency key

### 8. Exception Handling & Compensation Strategy
- Inventory-reservation timeout: retry up to 3 times, then mark as pending for manual compensation
- Lost WMS callback: scheduled task actively polls WMS for status
- Payment success but inventory deduction fails: compensation releases reserved inventory and initiates refund to the user
- Duplicate/out-of-order messages: validate current order status before processing; skip expired/invalid messages
- Restock failure: restored inventory stays stuck, raises an alert and manual compensation

## Q: Quality

### 1. Performance
- Concurrency target: flash-sale scenario supports 2000 concurrent order requests
- Latency: inventory-reservation response < 200ms
- Error rate: normal-path error rate < 0.1%
- Throughput: up to 500 orders/second

### 2. Reliability
- Retry: external sync interfaces limited to 3 retries
- Fallback: on WMS failure, requests queue first and retry after recovery
- Timeout: external system call timeout is 5 seconds
- Circuit breaker: WMS error rate > 20% trips the breaker, stopping continuous calls
- Compensation jobs: hourly inventory reconciliation, stuck-order scan, restored-inventory scan

### 3. Observability
- Logs: every inventory change, message send/receive, and callback event is fully logged
- Metrics: inventory difference rate, message-queue backlog, stuck-order count, reservation-failure rate, restored-inventory retention
- Alerts: inventory difference ≠ 0, queue backlog over threshold, stuck orders present, restored inventory over time limit

### 4. Security
- Authentication: all external APIs carry token validation
- Permission boundaries: merchants can only view their own products and inventory
- Sensitive-data masking: phone numbers and payment info are masked in logs

### 5. Recoverability
- Failure drills: simulate WMS downtime, message loss, and duplicate-message scenarios
- Data-repair process: reconciliation report → confirm difference → script/manual inventory fix → keep audit record

### 6. Release Entry & Exit Criteria
Entry criteria (can start testing):
1. Requirement and API documents finalized and reviewed
2. Development build deployed to the test environment
3. Smoke test passes; order and inventory core flows run end-to-end
4. WMS and SC2P mock services ready

Exit criteria (can release):
1. All P0 and P1 defects fixed and verified
2. All core MFQ test scenarios covered
3. Inventory reconciliation tested with no oversell risk
4. Performance and reliability tests pass
5. Test report reviewed and approved
