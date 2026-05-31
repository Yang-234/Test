# MFQ Test Plan - Strategy Summary

---

## M - Model Module Strategy

The core goal of the M module is to **ensure the correctness and completeness of the model definition itself**. The testing strategy focuses on state machine coverage and consistency boundaries:

1. **100% State Machine Coverage** — Every state × event combination must have a clearly defined transition path; undefined transitions are not allowed.
2. **Strong Consistency Priority Verification** — Atomicity of inventory reservation/deduction is the foundation of oversell prevention and has the highest priority.
3. **Idempotency Must Be Tested** — All inventory operations and external callbacks must define idempotent behavior at the model level.
4. **Reconciliation Mechanism as Fallback** — Eventual consistency scenarios rely on reconciliation tasks for repair; coverage and execution of reconciliation are the quality baseline.
5. **External Interactions Focus on Contract Verification** — Ensure that timeouts, retries, and fallbacks for each external interface are defined in the model.

---

## F - Functional Module Strategy

The core goal of the F module is to **ensure system functional behavior meets expectations, covering all normal and abnormal paths**. The testing strategy focuses on inventory synchronization correctness and message processing robustness:

1. **Happy Path First** — Ensure the main chain is打通, and all core interfaces can be called normally.
2. **Inventory Synchronization is the Top Priority** — The idempotency and consistency of the four operations (reservation, deduction, release, restore) directly relate to funds and user experience.
3. **Out-of-Order Messages Cannot Be Ignored** — In distributed systems, out-of-order messages are the norm; the combination of payment → shipping → receipt must be covered.
4. **High-Concurrency Flash Sales are Typical Boundaries** — Not overselling is the bottom line of e-commerce systems; concurrent testing is needed for verification.
5. **API Contracts Focus on Field Validation and Error Codes** — Illegal input interception happens at the gateway layer; unified error code standards facilitate troubleshooting.

---

## Q - Quality Module Strategy

The core goal of the Q module is to **ensure the operational quality of the system in the production environment, including performance standards, fault self-healing, monitoring, and security reliability**. The testing strategy is built on two pillars: fault drills and observability:

1. **Performance Baselines Must Be Quantified** — Do not rely on empirical judgment; each core interface must have clear P99/error rate/TPS targets.
2. **Circuit Breaker + Compensation + Reconciliation is the Reliability Iron Triangle** — Circuit breaker prevents avalanches, compensation guarantees eventual consistency, and reconciliation provides fallback repair.
3. **Observability is Not Just an Operations Matter** — Logs, metrics, and alerts must be verified during the testing phase to ensure that "blind flight" becomes "instrument flight" after going live.
4. **Security Focuses on Authorization Bypass and Data Masking** — Horizontal authorization bypass (user A viewing user B) and sensitive data leakage are the two most common security issues in e-commerce.
5. **Fault Drills are Not Just for Show** — Database switching, dependency unavailability, and network partitions should be fully演练ed in the testing environment to ensure orderly recovery during production failures.

---

## Overall Strategy Priorities

| Dimension | Strategy Points | Corresponding Module |
|---|---|---|
| First Priority | State Machine Completeness + Inventory Atomicity + Oversell Prevention | M + F |
| Second Priority | Happy Path Full Chain + API Contract Validation | F |
| Third Priority | Idempotency + Out-of-Order Messages + Reconciliation Fallback | M + F + Q |
| Fourth Priority | Circuit Breaker/Degradation/Compensation + Performance Baselines | Q |
| Fifth Priority | Observability + Security + Fault Recovery | Q |

## Key Trade-offs

1. **Strong Consistency vs High Availability**: Order payment chain chooses strong consistency (prefer failure over overselling), non-core display chain chooses eventual consistency (allows delays).
2. **Full Coverage vs Time Cost**: P0 first test (121/core paths), P1 supplement (70/boundary and exceptions), P2 optional.
3. **Automation vs Manual**: Happy Path + API field validation is suitable for automated regression; fault drills and reconciliation repair require manual judgment.
4. **Real-time Reconciliation vs Scheduled Reconciliation**: Inventory differences use scheduled reconciliation (5min), payment inconsistencies use active query + retry (more real-time).