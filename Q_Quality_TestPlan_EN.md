# Q - Quality Module: Test Cases and Plan

---

## 1. Performance

### 1.1 Performance Metrics Definition

| Metric | Target Value | Measurement Method |
|---|---|---|
| Order Interface P99 Latency | < 500ms | Pressure test tool statistics |
| Payment Callback Processing P99 Latency | < 200ms | APM Monitoring |
| Product Detail Page P99 Latency | < 300ms | Includes cache hit scenarios |
| Concurrent Order TPS | ≥ 1000/s (single node) | Pressure test |
| Inventory Query QPS | ≥ 5000/s (single node) | Pressure test (via cache) |
| Error Rate | < 0.1% | Error requests / total requests |
| Payment Callback Success Rate | ≥ 99.95% | Successful callbacks / total callbacks |

### 1.2 Performance Test Cases

| ID | Test Case Name | Test Scenario | Test Conditions | Pass Criteria | Priority |
|---|---|---|---|---|---|
| Q-PERF-001 | Order Interface - Baseline Performance | POST /orders | 1 concurrent,持续60s | P99 < 500ms, no errors | P0 |
| Q-PERF-002 | Order Interface - Target Load | POST /orders | 500 concurrent,持续5min | P99 < 1s, error rate < 0.1%, TPS ≥ 1000/s | P0 |
| Q-PERF-003 | Order Interface - Pressure Limit | POST /orders | Gradually increase concurrent until system crash | Record limit TPS and crash point, confirm graceful degradation instead of avalanche | P1 |
| Q-PERF-004 | Order Interface - Long-term Stability | POST /orders | 500 concurrent,持续30min | No memory leak, no continuous TPS decline, no continuous error rate increase | P0 |
| Q-PERF-005 | Flash Sale Scenario - Instant Peak | POST /orders (flash sale) | 10000 concurrent instant burst | No inventory overselling, P99 < 3s, queue mechanism effective, no system crash | P0 |
| Q-PERF-006 | Payment Callback - High Throughput | /callback/payment | 2000 TPS,持续5min | P99 < 200ms, success processing rate ≥ 99.95% | P0 |
| Q-PERF-007 | Product Query - Cache Hit | GET /products | 5000 QPS | P99 < 50ms, cache hit rate > 95% | P1 |
| Q-PERF-008 | Product Query - Cache Breakdown (Hotspot) | GET /products/{hotSku} | Single hot SKU 5000 QPS | Cache not broken, database connections not surge | P0 |
| Q-PERF-009 | Inventory Query - High Concurrency | GET /inventory/{skuId} | 3000 QPS | P99 < 100ms | P1 |
| Q-PERF-010 | Mixed Scenario - End-to-End Pressure Test | Browse:Query:Order:Payment = 100:30:10:5 | Simulate online traffic distribution,持续15min | Each interface P99 meets target, error rate < 0.1% | P0 |
| Q-PERF-011 | Cold Start - Cache Warm-up | Start pressure test after cache cleared | 500 concurrent orders | First request may be slow (warming up), reach normal P99 within 1min | P1 |

---

## 2. Reliability

### 2.1 Retry Strategy

| ID | Test Case Name | Test Scenario | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| Q-REL-001 | WMS Shipping - Call Timeout Retry | WMS interface response timeout | Simulate WMS 3s timeout | Auto retry 3 times (exponential backoff: 1s, 2s, 4s), finally return success or failure | P0 |
| Q-REL-002 | WMS Shipping - Post-Retry Exhaustion Handling | WMS continuous 3 retries all timeout | WMS持续不可用 | Order marked as retry exhausted, enters compensation queue, triggers alert | P0 |
| Q-REL-003 | Payment Callback - Consumer Failure Retry | MQ consumer payment callback throws exception | Consumer processing fails | MQ auto redelivery, retry 3 times then enters dead letter queue (DLQ) | P0 |
| Q-REL-004 | Retry - Idempotency Guarantee | Retry causes same message processed multiple times | Observe retry process | Business results not duplicated, idempotency key effective | P0 |
| Q-REL-005 | Retry - Backoff Time Correctness | Observe retry interval | Record retry timestamps | Interval matches configuration (e.g., 1s, 2s, 4s exponential backoff) | P1 |

### 2.2 Fallback Degradation

| ID | Test Case Name | Test Scenario | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| Q-REL-006 | WMS Unavailable - Order Process Degradation | WMS service completely unavailable | User places order and pays | Order/payment unaffected, shipping enters待处理 queue, user sees "Preparing" | P0 |
| Q-REL-007 | Payment Gateway Unavailable - Degradation Handling | Payment gateway completely unavailable | User initiates payment | Prompts "Payment service busy, please retry later", order remains pending payment | P0 |
| Q-REL-008 | Recommendation Service Unavailable -不影响 Main Flow | Product recommendation service down | Browse product detail page | Recommendation area shows default/cached data or no recommendation, core product info displays normally | P1 |
| Q-REL-009 | SCMP Unavailable - Degradation | SCMP sync interface unavailable | Inventory changes sync | Local inventory flows normally, sync messages enter retry queue, not blocking main flow | P0 |
| Q-REL-010 | Cache (Redis) Unavailable - Degradation | Redis cluster down | Query product/inventory | Degrade to direct database read, system can continue serving (performance degraded but available) | P0 |

### 2.3 Timeout Control

| ID | Test Case Name | Test Scenario | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| Q-REL-011 | Payment Gateway - Connection Timeout | Payment gateway response exceeds configured connection timeout (e.g., 1s) | Initiate payment | Return timeout error after timeout, does not block thread, supports user retry | P0 |
| Q-REL-012 | WMS Interface - Read Timeout | WMS exceeds read timeout (e.g., 5s) without returning complete response | Send shipping instruction | Trigger timeout → retry → finally fail or succeed, does not hang thread | P0 |
| Q-REL-013 | Database Connection Timeout | Database connection pool exhausted | Concurrent requests | New requests fail fast after waiting queue timeout, not无限等待 | P1 |
| Q-REL-014 | Global Timeout Propagation | Upstream request sets 30s timeout | Downstream各服务调用 | 各服务 timeout cumulative does not exceed upstream timeout, avoid upstream已超时 downstream仍在执行 | P1 |

### 2.4 Circuit Breaker

| ID | Test Case Name | Test Scenario | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| Q-REL-015 | Circuit Breaker - Open | WMS continuous failure rate reaches threshold (e.g., 50%/10 times) | Continuously send failed requests | Circuit breaker OPEN, subsequent requests fail fast directly, do not call WMS | P0 |
| Q-REL-016 | Circuit Breaker - Half-Open Probe | Circuit breaker OPEN after cooling period (e.g., 30s) | Send少量 probe requests | Circuit breaker HALF_OPEN, allows limited requests through for probing | P0 |
| Q-REL-017 | Circuit Breaker - Close Recovery | HALF_OPEN state, probe requests successful | Probe requests连续 successful | Circuit breaker CLOSE, resume normal calls | P0 |
| Q-REL-018 | Circuit Breaker - Re-Open | HALF_OPEN state, probe request fails | Probe request fails | Circuit breaker re-OPEN, reset cooling period | P1 |
| Q-REL-019 | Circuit Breaker - Does Not Block Core Flow | During WMS circuit breaker | User places order and pays | Order/payment unaffected, shipping enters待处理 (compensation queue) | P0 |

### 2.5 Compensation Mechanism

| ID | Test Case Name | Test Scenario | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| Q-REL-020 | Compensation - Timeout Unpaid Orders | Orders in PENDING_PAYMENT over 30min | Compensation定时任务 scan | Order auto-cancelled, inventory released, status→CANCELLED | P0 |
| Q-REL-021 | Compensation - Shipping Failure Retry Queue | WMS shipping连续失败 enters compensation queue | Compensation task consumes queue | Retry with backoff strategy, manual intervention alert after max retries exceeded | P0 |
| Q-REL-022 | Compensation - Payment Callback Lost Active Query | Order PENDING_PAYMENT over 5min | Active query payment gateway | Get final payment status and update order (payment success→PAID, failure→remain pending or cancel) | P0 |
| Q-REL-023 | Compensation - Refund Failure Retry | Refund request fails (finance system unavailable) | Compensation task retries refund | Exponential backoff retry, manual intervention工单 after 1h未成功 | P1 |
| Q-REL-024 | Compensation - Idempotency | Compensation task concurrent with normal process | Normal payment callback and compensation query execute simultaneously | Only once effective, order status correct | P0 |

---

## 3. Observability

### 3.1 Logs

| ID | Test Case Name | Validation Content | Expected Result | Priority |
|---|---|---|---|---|
| Q-OBS-001 | Key Operation Log - Order Placement | Log output when placing order | Contains traceId, userId, orderId, skuIds, amount, timestamp, duration | P0 |
| Q-OBS-002 | Key Operation Log - Payment | Payment callback processing log | Contains traceId, orderId, transactionId, status, amount, duration | P0 |
| Q-OBS-003 | Key Operation Log - Inventory Change | Inventory reserve/deduct/release/restore | Contains traceId, skuId, operation, beforeQty, afterQty, delta, source(order number) | P0 |
| Q-OBS-004 | Exception Log - Complete Stack Trace | System throws exception | Contains traceId, userId, request params, stacktrace, error code | P0 |
| Q-OBS-005 | External Call Log | Call WMS/payment gateway/SCMP | Contains traceId, target, method, request(desensitized), response, statusCode, duration | P0 |
| Q-OBS-006 | Log Level Correctness | Various scenarios | INFO: normal process; WARN: retry/degradation/business exception; ERROR: system exception/data inconsistency | P1 |
| Q-OBS-007 | Log Does Not Leak Sensitive Information | Check各日志 output | Password/token/ID card/bank card number not appearing in logs (desensitized or not recorded) | P0 |

### 3.2 Metrics

| ID | Test Case Name | Metric Name | Description | Alert Condition | Priority |
|---|---|---|---|---|---|
| Q-OBS-008 | Order TPS Monitoring | order_create_tps | Real-time order rate | TPS deviates from normal value ±30% | P0 |
| Q-OBS-009 | Payment Success Rate | payment_success_rate | Payment success / total payment requests | < 95% alert | P0 |
| Q-OBS-010 | Interface P99 Latency | http_request_duration_p99 | Each interface P99 duration | Exceeds target value 2x | P0 |
| Q-OBS-011 | Inventory Difference Ratio | inventory_diff_ratio | Local inventory vs SCMP difference pieces / total pieces | > 0.1% alert | P0 |
| Q-OBS-012 | Message Backlog | mq_consumer_lag | MQ consumer backlog message count | > 1000 pieces or abnormal growth rate | P0 |
| Q-OBS-013 | Stuck Order Count | stuck_order_count | Active orders over 24h without flow | > 50 orders alert | P0 |
| Q-OBS-014 | Circuit Breaker State | circuit_breaker_state | Each circuit breaker current state | OPEN immediate alert | P0 |
| Q-OBS-015 | Compensation Queue Backlog | compensation_queue_depth | Compensation task queue depth | > 100 pieces or continuous growth | P1 |
| Q-OBS-016 | Error Rate | error_rate | 5xx errors / total requests | > 0.5% alert | P0 |
| Q-OBS-017 | Database Connection Pool Utilization | db_pool_utilization | Active connections / total connections | > 80% alert | P1 |

### 3.3 Alerts

| ID | Test Case Name | Alert Rule | Notification Method | Priority | Validation Method |
|---|---|---|---|---|---|
| Q-OBS-018 | Inventory Difference Alert | inventory_diff_ratio > 0.1%持续5min | DingTalk/Feishu + PagerDuty | P0 | Simulate SCMP sync failure, observe alert trigger |
| Q-OBS-019 | Message Backlog Alert | mq_lag > 1000持续3min | DingTalk/Feishu | P0 | Stop consumer, observe backlog and alert |
| Q-OBS-020 | Stuck Order Alert | stuck_order > 50 | DingTalk + PagerDuty | P0 | Simulate compensation task停摆, order滞留 |
| Q-OBS-021 | Payment Success Rate Sudden Drop Alert | success_rate < 95%持续3min | PagerDuty | P0 | Simulate payment gateway continuous return failure |
| Q-OBS-022 | Error Rate Surge Alert | error_rate > 1%持续2min | PagerDuty | P0 | Simulate dependent services all timeout |
| Q-OBS-023 | Circuit Breaker Open Alert | circuit_breaker = OPEN | DingTalk/Feishu | P0 | Simulate WMS continuous failure triggers circuit breaker |
| Q-OBS-024 | P99 Latency Surge Alert | P99 > target value 200%持续5min | DingTalk/Feishu | P1 | Simulate DB slow query |
| Q-OBS-025 | Alert Storm Suppression | Same alert rule triggered repeatedly in short time | - | P1 | 5min内 same rule only send 1 time, aggregate notification |
| Q-OBS-026 | Alert Auto-Recovery Notification | Alert condition resolved | DingTalk/Feishu | P1 | Receive recovery notification after fault recovery |

---

## 4. Security

### 4.1 Authentication & Authorization

| ID | Test Case Name | Test Scenario | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| Q-SEC-001 | Unauthenticated Access to Authenticated Interface | Access order/payment/personal orders without token | Without Authorization header | 401 Unauthorized | P0 |
| Q-SEC-002 | Expired Token Access | Token已过期 | Use expired token request | 401 + TOKEN_EXPIRED error code | P0 |
| Q-SEC-003 | User A Accesses User B's Order | User A logged in | GET /orders/{userB_orderId} | 403 Forbidden, does not expose others' data | P0 |
| Q-SEC-004 | User A Cancels User B's Order | User A logged in | POST /orders/{userB_orderId}/cancel | 403 Forbidden | P0 |
| Q-SEC-005 | Regular User Accesses Admin Interface | Regular user token | GET /admin/orders | 403 Forbidden | P0 |
| Q-SEC-006 | Unauthorized Modification of Product Price | User tampers with price in request when placing order | POST /orders with tampered price | Backend uses server-calculated price, ignores client price | P0 |

### 4.2 Permission Boundaries

| ID | Test Case Name | Test Scenario | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| Q-SEC-007 | Customer Service Role Permissions | Customer service login | Can check orders/refunds/notes, cannot modify products/prices/system config | Permission isolation correct | P0 |
| Q-SEC-008 | Warehouse Role Permissions | Warehouse personnel login | Can view shipping orders/confirm shipping/confirm return receipt, cannot view complete user info | Permission isolation correct | P1 |
| Q-SEC-009 | Permission Elevation Test | Regular user modifies role field in request | Pass role=admin | Ignore client role, use server token parsing为准 | P0 |

### 4.3 Sensitive Data Protection

| ID | Test Case Name | Test Scenario | Expected Result | Priority |
|---|---|---|---|---|
| Q-SEC-010 | Phone Number Desensitization Display | Receiving phone number in order details | Display as 138****1234 | P0 |
| Q-SEC-011 | Payment Information Not Stored | Bank card number/payment password | Not stored明文 in any system table | P0 |
| Q-SEC-012 | Log Desensitization | Phone number/ID card/bank card in logs | Desensitized or replaced with placeholders | P0 |
| Q-SEC-013 | API Response Desensitization | User information in interface response | Phone number/ID card/bank card desensitized in response | P0 |
| Q-SEC-014 | Address Information Access Control | Logistics/warehouse personnel view address | Can only see receiving address, cannot export or batch query | P1 |
| Q-SEC-015 | HTTPS Enforcement | All API requests | HTTP requests redirected to HTTPS or rejected | P0 |

---

## 5. Recoverability

### 5.1 Fault Drills

| ID | Test Case Name | Fault Scenario | Drill Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| Q-REC-001 | Database Primary Failure Switch | MySQL primary down | Simulate kill primary process | Auto switch to replica within 30s, business恢复, no data loss | P0 |
| Q-REC-002 | Redis Cluster Failure | Redis primary node down | Simulate kill Redis primary node | Sentinel auto switch, business degrades to DB or returns cached data, no full崩溃 | P0 |
| Q-REC-003 | MQ Broker Failure | Kafka/RabbitMQ Broker down | Simulate kill Broker | Producers auto switch to other Brokers, consumers resume from last offset | P0 |
| Q-REC-004 | Payment Gateway Completely Unavailable | Payment gateway network断开 | Simulate network断开 | Payment requests fail fast, order remains pending payment, active query compensation after 5min | P0 |
| Q-REC-005 | WMS Completely Unavailable | WMS service down | Simulate kill WMS | Order payment unaffected, shipping enters compensation queue, circuit breaker effective | P0 |
| Q-REC-006 | Single Service Node Down | Order service某节点 down | Simulate kill single node | Load balancer auto removes, traffic distributed to other nodes, user无感知 | P0 |
| Q-REC-007 | Full Network Partition | Order service and database network中断 | Simulate iptables block | Application layer fails fast, no dirty data, auto reconciliation after network恢复 | P1 |
| Q-REC-008 | Disk Full | Log/data disk usage > 95% | Simulate disk写满 | Trigger alert, log auto rotation cleanup, key business not中断 | P1 |

### 5.2 Data Repair

| ID | Test Case Name | Repair Scenario | Repair Steps | Validation Method | Priority |
|---|---|---|---|---|---|
| Q-REC-009 | Order Status Repair | Order status inconsistent with inventory status | Execute reconciliation script → find difference → auto repair | After repair, order and inventory status consistent, repair log complete | P0 |
| Q-REC-010 | Inventory Data Repair | Available + reserved + sold ≠ total inventory | Reconciliation finds → calculate difference → recalculate based on inventory流水 → update inventory table | Numerical恒等式 restored, record difference reason | P0 |
| Q-REC-011 | Duplicate Deduction Repair | Payment gateway duplicate deduction | Reconciliation finds → manual confirmation → initiate refund → record repair order | User receives refund, order status correct | P0 |
| Q-REC-012 | Message Dead Letter Queue Processing | Dead letter queue backlog | View dead letter reason → fix code/data → redeliver or manual process | Dead letter consumed or confirmed discarded, queue cleared | P1 |
| Q-REC-013 | Stuck Order Batch Repair | Batch orders stuck in PAID over 48h not shipped | Script scan → trigger shipping or auto refund →二次确认 | Stuck orders cleared, user receives result notification | P1 |
| Q-REC-014 | Data Repair - Auditable | Execute任意 data repair operation | View repair audit log | Record operator, time, repair content(before/after), reason | P0 |

### 5.3 Backup & Recovery

| ID | Test Case Name | Test Content | Expected Result | Priority |
|---|---|---|---|---|
| Q-REC-015 | Database Full Backup | Daily full backup | Backup file complete and recoverable, backup process does not affect business | P0 |
| Q-REC-016 | Database Incremental Backup | Binlog real-time backup | Incremental backup normal, delay < 5s | P0 |
| Q-REC-017 | Backup Recovery Drill | Restore database from backup file | Recovery time < 2h(RTO), data loss < 5min(RPO) | P0 |
| Q-REC-018 | Order Data Recovery Validation | Verify after backup restore | Random sample order data consistent with production (before recovery time point) | P1 |

---

## 6. Priority Summary

| Priority | Quantity | Description |
|---|---|---|
| P0 | 58 | Performance baseline/pressure test/flash sale, retry/circuit breaker/compensation core mechanisms, log/metric/alert core coverage, security authentication authorization desensitization, fault switching/recovery |
| P1 | 25 | Pressure limit, cold start, backoff validation, half-open probe, log level, alert suppression/recovery, disk full, dead letter processing |

> **Total: 83 Quality Module Test Cases**

---

## 7. Strategy Explanation (Q Module Focus)

The core goal of the Q module is to **ensure the operational quality of the system in the production environment, including performance standards, fault self-healing, monitoring, and security reliability**. The testing strategy is built on two pillars: fault drills and observability:

1. **Performance Baselines Must Be Quantified** — Do not rely on empirical judgment; each core interface must have clear P99/error rate/TPS targets.
2. **Circuit Breaker + Compensation + Reconciliation is the Reliability Iron Triangle** — Circuit breaker prevents avalanches, compensation guarantees eventual consistency, and reconciliation provides fallback repair.
3. **Observability is Not Just an Operations Matter** — Logs, metrics, and alerts must be verified during the testing phase to ensure that "blind flight" becomes "instrument flight" after going live.
4. **Security Focuses on Authorization Bypass and Data Masking** — Horizontal authorization bypass (user A viewing user B) and sensitive data leakage are the two most common security issues in e-commerce.
5. **Fault Drills are Not Just for Show** — Database switching, dependency unavailability, and network partitions should be fully演练ed in the testing environment to ensure orderly recovery during production failures.