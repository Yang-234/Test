# M - Models Module: Test Cases and Plan

---

## 1. End-to-End Business Flow Model

### 1.1 Flow Definition

```
User Browse → Add to Cart → Place Order → Payment → Fulfillment (Shipping) → Sign → [Return & Refund]
```

### 1.2 Flow Model Test Cases

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| M-E2E-001 | Positive End-to-End Flow Verification | Sufficient inventory, user logged in | Browse product → Add to cart → Place order → Payment → Shipping → Sign | Each node status transitions correctly in sequence, no skipping | P0 |
| M-E2E-002 | End-to-End Interruption - Payment Failure | Sufficient inventory, order placed successfully | Place order → Payment failure | Order remains "Pending Payment", inventory remains "Reserved", auto-cancel after timeout and release inventory | P0 |
| M-E2E-003 | End-to-End Interruption - Cancellation Before Shipping | Paid, not shipped | Payment success → User cancellation | Order "Cancelled", inventory rolls back from "Sold" to "Available", triggers refund | P0 |
| M-E2E-004 | End-to-End - Complete Return & Refund Flow | Signed | Sign → Apply for return → Approval → Return logistics → Warehouse sign → Refund → Inventory restoration | Order "Returned", inventory "Restored", refund received | P1 |
| M-E2E-005 | End-to-End - Partial Return | Order contains multiple products, signed | Apply for return for 1 item | Only returned product inventory restored, remaining product status unchanged, partial refund | P1 |

---

## 2. Order State Machine

### 2.1 State Definition

| State | English | Description |
|---|---|---|
| 待支付 | PENDING_PAYMENT | Order created, waiting for user payment |
| 已支付 | PAID | Payment successful, waiting for shipping |
| 已发货 | SHIPPED | Shipped out, logistics in transit |
| 已签收 | SIGNED | User confirmed receipt |
| 已取消 | CANCELLED | Order cancelled (before/after payment) |
| 已退货 | RETURNED | Return & refund completed |
| 部分退货 | PARTIAL_RETURN | Partial product return completed |

### 2.2 Event Definition

| Event | Triggered By | Description |
|---|---|---|
| place_order | User | Submit order |
| pay_success | Payment Gateway | Payment success callback |
| pay_failure | Payment Gateway | Payment failure callback |
| pay_timeout | System Timer | Payment timeout |
| ship | Warehouse System | Ship out |
| sign | User/Logistics | Sign confirmation |
| cancel_by_user | User | User-initiated cancellation |
| cancel_by_system | System | System auto-cancellation (e.g., timeout) |
| request_return | User | Apply for return |
| approve_return | Customer Service | Return approval |
| reject_return | Customer Service | Return rejection |
| return_received | Warehouse | Return received in warehouse |
| refund_done | Finance System | Refund completed |

### 2.3 State Transition Matrix

| Current State | Event | Target State | Condition | Test Case ID |
|---|---|---|---|---|
| (Initial) | place_order | PENDING_PAYMENT | Inventory reservation successful | M-OSM-001 |
| PENDING_PAYMENT | pay_success | PAID | Payment amount matches, order not expired | M-OSM-002 |
| PENDING_PAYMENT | pay_failure | PENDING_PAYMENT | Allow retry, not exceeding max attempts | M-OSM-003 |
| PENDING_PAYMENT | pay_timeout | CANCELLED | Exceeds payment time limit (e.g., 30min) | M-OSM-004 |
| PENDING_PAYMENT | cancel_by_user | CANCELLED | - | M-OSM-005 |
| PAID | cancel_by_user | CANCELLED | Not shipped | M-OSM-006 |
| PAID | ship | SHIPPED | Inventory deduction successful, WMS confirmation | M-OSM-007 |
| PAID | cancel_by_system | CANCELLED | Timeout not shipped (e.g., 48h), system auto-cancel | M-OSM-008 |
| SHIPPED | sign | SIGNED | Logistics confirms receipt | M-OSM-009 |
| SIGNED | request_return | SIGNED | Return application in progress (sub-state) | M-OSM-010 |
| SIGNED | approve_return + return_received + refund_done | RETURNED | Return approved and warehouse signed | M-OSM-011 |
| SIGNED | approve_return + return_received + refund_done | PARTIAL_RETURN | Partial product return | M-OSM-012 |
| SIGNED | reject_return | SIGNED | Return application rejected | M-OSM-013 |

### 2.4 State Machine Test Cases

| ID | Test Case Name | Test Steps | Expected Result | Priority |
|---|---|---|---|---|
| M-OSM-001 | Place Order - Normal Creation | Trigger place_order, inventory sufficient | Order created, status=PENDING_PAYMENT, inventory reserved | P0 |
| M-OSM-002 | Payment - Success | PENDING_PAYMENT → Trigger pay_success | Status→PAID, inventory from reserved→sold | P0 |
| M-OSM-003 | Payment - Failure Retryable | PENDING_PAYMENT → Trigger pay_failure | Status remains PENDING_PAYMENT, inventory still reserved, user can retry | P0 |
| M-OSM-004 | Payment - Timeout Auto-Cancel | PENDING_PAYMENT exceeds 30min | System auto-triggers pay_timeout, status→CANCELLED, inventory released | P0 |
| M-OSM-005 | Pending Payment - User Cancel | PENDING_PAYMENT → cancel_by_user | Status→CANCELLED, inventory released to available | P0 |
| M-OSM-006 | Paid - User Cancel (Not Shipped) | PAID → cancel_by_user | Status→CANCELLED, inventory restored from sold to available, triggers refund process | P0 |
| M-OSM-007 | Shipping - Normal | PAID → ship | Status→SHIPPED, record logistics tracking number | P0 |
| M-OSM-008 | Timeout Not Shipped - System Cancel | PAID exceeds 48h not shipped | cancel_by_system, status→CANCELLED, refund + inventory restoration | P1 |
| M-OSM-009 | Sign - Normal | SHIPPED → sign | Status→SIGNED, order completed | P0 |
| M-OSM-010 | Return - Application | SIGNED → request_return | Status remains SIGNED, generates return order, sub-status=under review | P1 |
| M-OSM-011 | Return - Completion | Approval → Warehouse sign → Refund | Status→RETURNED, inventory restored, refund received | P1 |
| M-OSM-012 | Partial Return | Multiple product order, return 1 item | Status→PARTIAL_RETURN, only 1 item inventory restored | P1 |
| M-OSM-013 | Return - Approval Rejected | SIGNED → request_return → reject | Status back to SIGNED, return order closed | P1 |
| M-OSM-014 | Illegal State Transition - Duplicate Payment | PAID → pay_success | Idempotent processing, status unchanged, returns existing payment result | P0 |
| M-OSM-015 | Illegal State Transition - Ship Cancelled Order | CANCELLED → ship | Reject operation, log exception | P0 |

---

## 3. Inventory State Model

### 3.1 State Definition

| State | English | Description |
|---|---|---|
| 可用 | AVAILABLE | Quantity available for purchase |
| 预留 | RESERVED | Locked after order placement, cannot oversell |
| 已售/已扣减 | DEDUCTED/SOLD | Deducted after payment success |
| 退货中 | RETURNING | Return application approved, awaiting warehouse confirmation |

### 3.2 Inventory Operation Events

| Event | Trigger Condition | State Change | Idempotency Requirement |
|---|---|---|---|
| reserve | Place order | AVAILABLE → RESERVED | Duplicate reserve returns existing reservation result |
| confirm_deduct | Payment success | RESERVED → DEDUCTED | Duplicate deduct is idempotent |
| release | Cancel/Timeout | RESERVED → AVAILABLE | Duplicate release is idempotent |
| restore | Return completed | DEDUCTED → AVAILABLE | Duplicate restore requires validation |
| deduct_direct | Direct deduction (partial scenarios) | AVAILABLE → DEDUCTED | Idempotent |

### 3.3 Inventory State Model Test Cases

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| M-IVT-001 | Inventory Reservation - Success | Available inventory=10 | reserve(3) | Available=7, reserved=3, returns reservation success | P0 |
| M-IVT-002 | Inventory Reservation - Failure (Insufficient Stock) | Available inventory=2 | reserve(5) | Reservation fails, available=2, reserved=0, returns insufficient stock | P0 |
| M-IVT-003 | Inventory Deduction - Success | Reserved=3 | confirm_deduct(3) | Sold=3, reserved=0, total inventory unchanged | P0 |
| M-IVT-004 | Inventory Release - Success | Reserved=3 | release(3) | Available restored +3, reserved=0 | P0 |
| M-IVT-005 | Inventory Restoration - Return Completed | Sold=3 | restore(1) | Available+1, sold=2 | P1 |
| M-IVT-006 | Duplicate Reservation - Idempotent | Already reserved 3, available=7 | Reserve(3) again | Returns original reservation result, available=7, reserved=3 (no duplicate deduction) | P0 |
| M-IVT-007 | Duplicate Deduction - Idempotent | Reserved=3, sold=0 | confirm_deduct(3) executed twice | First deduction successful, second idempotent return already deducted | P0 |
| M-IVT-008 | Duplicate Release - Idempotent | Reserved=3 | release(3) executed twice | First release successful, second idempotent return already released, available not doubled | P0 |
| M-IVT-009 | Inventory Reservation - Zero Inventory | Available=0 | reserve(1) | Reservation fails, returns insufficient stock | P0 |
| M-IVT-010 | Inventory Reservation - Concurrent (Flash Sale) | Available=1, 100 concurrent requests each reserve(1) | Trigger simultaneously | Only 1 request succeeds in reservation, remaining 99 fail, available=0, reserved=1 | P0 |
| M-IVT-011 | Inventory Deduction - Deduction After Reservation Expired | Reserved=3, but reservation expired and released | confirm_deduct(3) | Deduction fails because reservation no longer exists | P1 |
| M-IVT-012 | Inventory Restoration - Over-Restoration Protection | Sold=2 | restore(5) | Restoration fails/alert, sold minimum is 0 | P1 |

---

## 4. External Interaction Model

### 4.1 Interaction Relationships

```
Storefront <──> Order Service <──> WMS (Warehouse Management System)
                         │
                         ├──> Payment Gateway
                         │
                         └──> SCMP (Supply Chain Management Platform)
```

### 4.2 Interface Interaction Model

| Interaction Party | Direction | Interface | Description |
|---|---|---|---|
| Storefront → Order Service | Inbound | Create Order | User order request |
| Order Service → WMS | Outbound | Shipping Instruction | Notify warehouse to pick and ship |
| WMS → Order Service | Inbound (Callback) | Shipping Confirmation | Return logistics tracking number and status |
| Order Service → Payment Gateway | Outbound | Initiate Payment | Redirect/request payment |
| Payment Gateway → Order Service | Inbound (Callback) | Payment Result Notification | Payment success/failure callback |
| Order Service → SCMP | Outbound | Inventory Sync | Push inventory changes |
| Order Service → SCMP | Outbound | Order Status Sync | Push order status changes |

### 4.3 External Interaction Model Test Cases

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| M-EXT-001 | WMS Shipping Instruction - Success | Order paid, inventory deducted | Send shipping instruction to WMS | WMS returns successful acceptance, order records shipping request time | P0 |
| M-EXT-002 | WMS Shipping Instruction - Timeout | Order paid | Send shipping instruction, WMS 5s no response | Triggers timeout retry mechanism, order remains PAID | P1 |
| M-EXT-003 | WMS Shipping Confirmation Callback - Success | Shipping instruction sent | WMS callback shipping confirmation (with tracking number) | Order status→SHIPPED, record tracking number | P0 |
| M-EXT-004 | WMS Shipping Confirmation Callback - Out of Order | Shipping instruction not yet sent (or delayed due to network) | Receive shipping confirmation callback first, then shipping request response | Callback暂存/重试等待, eventual consistency | P1 |
| M-EXT-005 | WMS Shipping Confirmation Callback - Duplicate | Order already SHIPPED | Receive same shipping confirmation callback again | Idempotent processing, no duplicate update | P0 |
| M-EXT-006 | Payment Gateway - Payment Success Callback | Initiate payment | Payment gateway async callback pay_success | Order→PAID, inventory deduction | P0 |
| M-EXT-007 | Payment Gateway - Callback vs Query Inconsistency | Payment gateway callback pay_success, but active query returns processing | Query为准 + Alert | Triggers manual/auto reconciliation | P1 |
| M-EXT-008 | Payment Gateway - Callback Lost | Initiate payment, callback not received | Timed active query of payment result | Update order status based on query result | P0 |
| M-EXT-009 | SCMP Inventory Sync - Normal | Inventory change | Push inventory change to SCMP | SCMP sync successful | P1 |
| M-EXT-010 | SCMP Inventory Sync - Failure Retry | Inventory change, SCMP unavailable | Push fails | Exponential backoff retry, record failure queue | P1 |

---

## 5. Consistency Boundaries Model

### 5.1 Consistency Classification

| Scenario | Consistency Level | Description |
|---|---|---|
| Order Creation & Inventory Reservation | Strong Consistency | Order placement must reserve inventory simultaneously, no overselling |
| Payment Confirmation & Inventory Deduction | Strong Consistency | Payment success must deduct inventory |
| Order Status & WMS Status | Eventual Consistency | Allow brief inconsistency, repair via callback/reconciliation |
| Storefront Display Inventory vs Real Inventory | Eventual Consistency | Allow brief delay (e.g., cache), timed reconciliation |
| Refund & Inventory Restoration | Eventual Consistency | Inventory restored asynchronously after refund success |

### 5.2 Reconciliation Points

| Reconciliation Point | Reconciliation Content | Frequency | Repair Strategy |
|---|---|---|---|
| Order vs Inventory | Whether inventory status matches order status | Every 5 minutes | Auto repair + alert |
| Order vs WMS | Whether order shipping status matches WMS outbound status | Every hour | Difference report + manual intervention |
| Order vs Payment | Whether order payment status matches payment gateway record | Every 10 minutes | Repair based on payment gateway为准 |
| Inventory vs SCMP | Whether local inventory matches SCMP inventory | Every 15 minutes | Full/incremental sync |

### 5.3 Consistency Boundary Test Cases

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| M-CON-001 | Strong Consistency - Order Placement Inventory Reservation Atomicity | Available=1 | Concurrent two order requests | Only 1 succeeds, other fails (insufficient stock), no overselling | P0 |
| M-CON-002 | Strong Consistency - Payment Timeout Inventory Release | Inventory reserved, payment timeout | Timer triggers cancellation | Order cancellation and inventory release in same transaction, status consistent | P0 |
| M-CON-003 | Eventual Consistency - WMS Shipping Status Sync | Order PAID, WMS shipped but callback not received | Timed reconciliation task executes | Reconciliation finds difference, update order to SHIPPED based on WMS status | P1 |
| M-CON-004 | Eventual Consistency - Inventory Reconciliation | Local inventory differs from SCMP by 5 pieces | Timed reconciliation task executes | Generate difference report, trigger incremental sync repair | P1 |
| M-CON-005 | Reconciliation - Order vs Inventory Status Inconsistency | Order CANCELLED but inventory still RESERVED | Reconciliation task finds | Auto release inventory, record repair log | P0 |
| M-CON-006 | Reconciliation - Order vs Payment Status Inconsistency | Order PAID but payment gateway shows failure | Reconciliation task finds | Order cancelled + inventory restored + alert based on payment gateway为准 | P0 |
| M-CON-007 | Eventual Consistency - Shipping Status Sync Delay | WMS shipped, callback delayed 30s | Callback arrives after 30s | Order finally updated to SHIPPED, user receives shipping notification | P1 |
| M-CON-008 | Reconciliation - Concurrent Reconciliation Task Mutual Exclusion | Two reconciliation tasks execute simultaneously | Distributed lock | Only one task executes, other skipped | P1 |

---

## 6. Model Integrity Validation Cases

| ID | Test Case Name | Test Content | Expected Result | Priority |
|---|---|---|---|---|
| M-CHK-001 | State Machine Completeness Validation | Traverse all states × all events, confirm each combination has clear definition (transition or rejection) | No undefined state transitions, no dead corners | P0 |
| M-CHK-002 | State Machine Deadlock Check | Check for unreachable states or states that cannot exit | No deadlock states, all end states reachable | P1 |
| M-CHK-003 | Inventory Model Numerical Consistency | At any moment: Available + Reserved + Sold + Returning = Total inventory | Equation holds恒成立, validate after each operation | P0 |
| M-CHK-004 | Flow Model Coverage | Check model for each business process step having corresponding state | 100% coverage, no missing links | P1 |
| M-CHK-005 | External Interaction Contract Integrity | Check each external interface defines: request/response format, timeout, retry strategy, fallback | All interfaces defined completely | P0 |

---

## 7. Priority Summary

| Priority | Quantity | Description |
|---|---|---|
| P0 | 28 | Core processes, state machine critical transitions, strong consistency, idempotency, inventory atomic operations |
| P1 | 19 | Boundary scenarios, eventual consistency, reconciliation, return process, reconciliation mutual exclusion |

> **Total: 47 Model Module Test Cases**

---

## 8. Strategy Explanation (M Module Focus)

The core goal of the M module is to **ensure the correctness and completeness of the model definition itself**. The testing strategy focuses on state machine coverage and consistency boundaries:

1. **100% State Machine Coverage** — Every state × event combination must have a clearly defined transition path; undefined transitions are not allowed.
2. **Strong Consistency Priority Verification** — Atomicity of inventory reservation/deduction is the foundation of oversell prevention and has the highest priority.
3. **Idempotency Must Be Tested** — All inventory operations and external callbacks must define idempotent behavior at the model level.
4. **Reconciliation Mechanism as Fallback** — Eventual consistency scenarios rely on reconciliation tasks for repair; coverage and execution of reconciliation are the quality baseline.
5. **External Interactions Focus on Contract Verification** — Ensure that timeouts, retries, and fallbacks for each external interface are defined in the model.