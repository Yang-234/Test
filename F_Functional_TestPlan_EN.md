# F - Functional Module: Test Cases and Plan

---

## 1. Core Happy Path Coverage

### 1.1 Positive Main Flow

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| F-HAP-001 | User Browses Product List | User logged in, product data available | GET /products?page=1&size=20 | Returns product list with name/price/inventory status/image, pagination correct | P0 |
| F-HAP-002 | User Views Product Details | Product exists and has inventory | GET /products/{skuId} | Returns complete product info (name/description/price/specs/inventory/reviews) | P0 |
| F-HAP-003 | Add Product to Cart | User logged in, product inventory > 0 | POST /cart/items {skuId, quantity:2} | Cart adds product, quantity=2, total price calculated correctly | P0 |
| F-HAP-004 | Modify Cart Quantity | Product already in cart | PUT /cart/items/{itemId} {quantity:5} | Quantity updated to 5, total price recalculated, inventory上限 validated | P0 |
| F-HAP-005 | Remove Product from Cart | Product in cart | DELETE /cart/items/{itemId} | Product removed, total price updated | P1 |
| F-HAP-006 | Create Order | Cart has products, address filled | POST /orders {cartId, addressId, couponCode?} | Order created successfully, returns orderId, inventory reserved, cart cleared | P0 |
| F-HAP-007 | Initiate Payment | Order created, status=PENDING_PAYMENT | POST /orders/{orderId}/pay {method:"WECHAT"} | Returns payment parameters/redirect link, order status unchanged (waiting for callback) | P0 |
| F-HAP-008 | Payment Success Callback Processing | Payment initiated | Payment gateway callback /callback/payment {orderId, status:SUCCESS, transactionId} | Order→PAID, inventory→sold, send payment success notification | P0 |
| F-HAP-009 | Query Order List | User has multiple orders | GET /orders?status=PAID&page=1 | Returns user's PAID orders, sorted by time descending | P0 |
| F-HAP-010 | Query Order Details | Order exists | GET /orders/{orderId} | Returns complete order info (products/amount/status/logistics/timeline) | P0 |
| F-HAP-011 | Shipping Notification to Consumer | Order shipped | Query logistics or push notification | User sees logistics tracking number and轨迹 | P0 |
| F-HAP-012 | Confirm Receipt (Sign) | Order SHIPPED | POST /orders/{orderId}/sign | Order→SIGNED, triggers settlement/evaluation permission | P0 |
| F-HAP-013 | Apply for Return | Order SIGNED, within return period | POST /returns {orderId, skuIds[], reason} | Creates return order, status=under review | P1 |
| F-HAP-014 | View Return Progress | Return order created | GET /returns/{returnId} | Returns return order details and current status | P1 |

---

## 2. Inventory Sync Scenarios

### 2.1 Reservation Scenarios

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| F-IVT-001 | Inventory Reservation Success | Available inventory=10 | Place order reserve(sku_001, 3) | Available=7, reserved=3, returns reservation success code (e.g., RESERVE_OK) | P0 |
| F-IVT-002 | Inventory Reservation Failure - Insufficient Stock | Available inventory=2 | Place order reserve(sku_001, 5) | Returns STOCK_INSUFFICIENT, no reservation record created | P0 |
| F-IVT-003 | Inventory Reservation Failure - SKU Not Found | SKU_999 not in product database | Place order reserve(sku_999, 1) | Returns SKU_NOT_FOUND | P1 |
| F-IVT-004 | Inventory Reservation Failure - Product Offline | SKU status=OFFLINE | Place order reserve(offline_sku, 1) | Returns PRODUCT_OFFLINE, prompts product is offline | P1 |
| F-IVT-005 | Inventory Reservation Timeout - Lock Release | Reservation exceeds 30min without payment | Scheduled task check | Reservation auto-released, available inventory restored, order auto-cancelled | P0 |

### 2.2 Deduction Scenarios

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| F-IVT-006 | Inventory Deduction Success | Reserved=3, available=7, sold=0 | Payment success confirm_deduct(sku_001, 3) | Reserved=0, sold=3, total inventory unchanged, returns DEDUCT_OK | P0 |
| F-IVT-007 | Inventory Deduction Failure - Reservation Not Found | No reservation record for this SKU | confirm_deduct(sku_001, 3) | Returns RESERVE_NOT_FOUND, logs exception, triggers manual investigation | P0 |
| F-IVT-008 | Inventory Deduction Failure - Insufficient Reserved Quantity | Reserved=2, request deduction of 3 | confirm_deduct(sku_001, 3) | Returns QUANTITY_MISMATCH, deduction fails, reservation remains unchanged | P1 |
| F-IVT-009 | Inventory Deduction Failure - Reservation Expired | Reservation timeout released | confirm_deduct(sku_001, 3) | Returns RESERVE_EXPIRED, prompts to place order again | P1 |

### 2.3 Cancellation and Restoration

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| F-IVT-010 | Cancel Order - Inventory Release (Pending Payment) | Order PENDING_PAYMENT, inventory reserved=3 | cancel_by_user | Order→CANCELLED, reserved=0, available+3, returns release success | P0 |
| F-IVT-011 | Cancel Order - Inventory Restoration (Paid, Not Shipped) | Order PAID, sold=3 | cancel_by_user | Order→CANCELLED, sold-3, available+3, triggers refund | P0 |
| F-IVT-012 | Cancel Order - Refund Process Linkage | Order PAID, cancellation successful | Auto-trigger refund after cancellation | Refund amount correct (including coupon return), refund status trackable | P1 |
| F-IVT-013 | Return Completed - Inventory Restoration | Return order approved, warehouse signed | restore(sku_001, 1) | Sold-1, available+1, records restoration source (return order number) | P1 |
| F-IVT-014 | Return - Inventory Restoration Idempotency | Same return order already restored | Execute restore again | Idempotent return, inventory not duplicated | P1 |

### 2.4 Message Idempotency

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| F-IVT-015 | Duplicate Order Message - Idempotent | Order successfully created (orderId=X) | Replay same order message (same idempotencyKey) | Returns existing order info, no new order created, no duplicate inventory reservation | P0 |
| F-IVT-016 | Duplicate Payment Callback - Idempotent | Order already PAID | Payment gateway resends pay_success callback | Order status unchanged, inventory not deducted again, returns existing payment record | P0 |
| F-IVT-017 | Duplicate Shipping Confirmation - Idempotent | Order already SHIPPED | WMS resends ship_confirm callback | Status unchanged, tracking number unchanged, no duplicate notification to user | P0 |
| F-IVT-018 | Duplicate Cancellation Message - Idempotent | Order already CANCELLED | Resend cancel message | Idempotent return, inventory not released again | P0 |
| F-IVT-019 | Duplicate Return Callback - Idempotent | Return completed and in warehouse | WMS resends return_received | Idempotent return, inventory not restored again | P1 |

### 2.5 Out-of-Order Message Processing

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| F-IVT-020 | Payment Callback Arrives Before Order Creation | Order not yet created (message out of order) | Receive pay_success callback first, then order creation | Callback暂存/重试, order created后匹配成功, normal flow | P1 |
| F-IVT-021 | Shipping Confirmation Arrives Before Payment Success | Order PAID, WMS message out of order | Receive ship_confirm first, then pay_success | ship_confirm暂存/等待, pay success后正常处理 shipping | P1 |
| F-IVT-022 | Sign Arrives Before Shipping Confirmation | Order PAID | Receive sign first, then ship_confirm | sign暂存/忽略, ship_confirm处理后自动完成 sign | P1 |
| F-IVT-023 | Cancellation Message Arrives After Payment Success | Order PENDING_PAYMENT, cancellation sent | Payment success processed first, cancellation arrives later | Cancellation rejected (refund requested instead), order remains PAID | P1 |

---

## 3. Boundary Scenarios

### 3.1 Inventory Boundaries

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| F-BDY-001 | Zero Inventory - Order Fails | SKU available inventory=0 | User attempts to order | Prompts "Product sold out", cannot create order | P0 |
| F-BDY-002 | Zero Inventory - Product Sold Out After Adding to Cart | Inventory > 0 when adding to cart, inventory=0 when ordering | Submit order | Order fails, prompts insufficient inventory, cart product retained | P0 |
| F-BDY-003 | Zero Inventory - Stock Snatched During Payment | Order successful (inventory=1), someone else pays first | Initiate payment | Inventory deduction fails during payment callback, order enters exception-refund process | P1 |
| F-BDY-004 | Negative Inventory Protection | Available=0 | Send illegal request reserve(-1) or operation causing negative | Parameter validation rejects, inventory cannot be negative | P0 |

### 3.2 High Concurrency Scenarios

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| F-BDY-005 | Flash Sale - No Overselling | Flash sale product inventory=100 | 10000 concurrent order requests | Only 100 orders created successfully (reservation successful), 9900 prompt sold out, total reserved+sold ≤ 100 | P0 |
| F-BDY-006 | Flash Sale - Same User Duplicate Orders | Flash sale limit 1 per user | Same user concurrently initiates 5 order requests | Only 1 successful, others prompt "limit reached" | P1 |
| F-BDY-007 | Flash Sale - Deduction Consistency | 100 successful reservations | Payment success callbacks arrive concurrently | 100 deductions all successful, reserved→0, sold→100, no loss no duplication | P0 |
| F-BDY-008 | High Concurrency - Cart Inventory Validation | Inventory=5, 50 people attempt to order simultaneously | Concurrent ordering | Only first 5 create orders successfully, others insufficient inventory, cart product quantity auto-marked as purchasable quantity | P1 |

### 3.3 Partial Shipping / Order Splitting

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| F-BDY-009 | Partial Shipping - One Order, Multiple SKUs | Order contains SKU_A (in stock) + SKU_B (out of stock) | WMS shipping confirmation includes only SKU_A | Order status=PARTIAL_SHIPPED, shipped products have tracking number, unshipped products pending补发 | P1 |
| F-BDY-010 | Partial Shipping - Split Logistics Order | Order contains SKU_A×5 shipped from different warehouses | Warehouse 1 ships 3 pieces, warehouse 2 ships 2 pieces | Generate 2 tracking numbers, order status=PARTIAL_SHIPPED/SHIPPED (after all shipped) | P1 |
| F-BDY-011 | Partial Shipping - User Visible | Order partially shipped | User views order | Shows shipped/unshipped product details, each package has independent logistics info | P1 |
| F-BDY-012 | Partial Shipping - Partial Receipt | Order split into 2 packages, only 1 received | User signs for 1 package | Only corresponding product signed, remaining product status unchanged, order remains PARTIAL_SHIPPED | P2 |
| F-BDY-013 | Partial Shipping - Inventory Deduction | Order partially shipped |某SKU only shipped partial quantity | Shipped quantity deducted from sold, unshipped quantity remains sold status | P1 |

### 3.4 Return Boundaries

| ID | Test Case Name | Precondition | Test Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| F-BDY-014 | Return - Exceeds After-Sales Period | Signed over 30 days | Apply for return | Prompts "Return period exceeded", rejects application | P1 |
| F-BDY-015 | Return - Already Returned Order Applies Again | Order fully returned | Apply for return again | Rejects, prompts "Order return completed" | P1 |
| F-BDY-016 | Return - Partial Return, Then Return Remaining | Order 2 products, 1 returned | Apply for return for the other 1 | Allowed, order status→RETURNED (all returned) | P1 |
| F-BDY-017 | Return - Virtual Products Not Returnable | Order contains virtual products (e.g., e-coupon) | Apply for return of virtual product | Rejects, prompts "Virtual products not returnable" | P1 |
| F-BDY-018 | Return - Amount Calculation (With Coupon) | Order used 100-20 coupon, return 1 item | Approval for refund | Refund amount = actual paid amount after proportional coupon distribution | P1 |

---

## 4. Critical API Validations

### 4.1 Request Field Validation

| ID | Test Case Name | Interface | Validation Content | Expected Result | Priority |
|---|---|---|---|---|---|
| F-API-001 | Order - Required Field Missing | POST /orders | Missing addressId | 400 + error code ADDRESS_REQUIRED | P0 |
| F-API-002 | Order - Invalid Quantity | POST /orders | quantity=0 or -1 | 400 + error code INVALID_QUANTITY | P0 |
| F-API-003 | Order - SKU Not Found | POST /orders | skuId=non-existent value | 400 + error code SKU_NOT_FOUND | P0 |
| F-API-004 | Order - Amount Validation | POST /orders | Client totalAmount differs from backend calculation | 400 + error code AMOUNT_MISMATCH | P0 |
| F-API-005 | Payment - Order Not Found | POST /orders/{id}/pay | orderId does not exist | 404 + error code ORDER_NOT_FOUND | P0 |
| F-API-006 | Payment - Invalid Order Status | POST /orders/{id}/pay | Order already PAID or CANCELLED | 409 + error code ORDER_STATUS_CONFLICT | P0 |
| F-API-007 | Payment - Timeout Limit | POST /orders/{id}/pay | Order created over 30min ago | 410 + error code ORDER_EXPIRED | P1 |
| F-API-008 | Cancel - Invalid Order Status | POST /orders/{id}/cancel | Order already SHIPPED | 409 + error code CANNOT_CANCEL_SHIPPED | P0 |
| F-API-009 | Confirm Receipt - Order Not Shipped | POST /orders/{id}/sign | Order is PAID (not SHIPPED) | 409 + error code ORDER_NOT_SHIPPED | P1 |
| F-API-010 | Parameter Type Validation | Various interfaces | Input non-expected type (e.g., string instead of int) | 400 + error code INVALID_PARAM_TYPE | P0 |
| F-API-011 | XSS/SQL Injection Validation | Various interfaces | Address field injection <script> or ';DROP TABLE-- | Input escaped/rejected, no script or SQL execution | P0 |

### 4.2 Error Code Specification Validation

| ID | Test Case Name | Validation Content | Expected Result | Priority |
|---|---|---|---|---|
| F-API-012 | Error Code Return Format Unified | All interface error responses | Unified format: { "code": "ERROR_CODE", "message": "Chinese prompt", "requestId": "traceId" } | P0 |
| F-API-013 | Error Code Uniqueness | Check all error code definitions | Each error scenario has unique error code, no duplicates | P1 |
| F-API-014 | Error Code Covers All Known Exceptions |对照 business exception list check | Each business exception has corresponding error code, no裸奔500 | P0 |

### 4.3 Callback/Writeback Field Validation

| ID | Test Case Name | Interface | Validation Content | Expected Result | Priority |
|---|---|---|---|---|---|
| F-API-015 | Payment Callback - Required Fields | /callback/payment | Missing transactionId or sign | 400 + callback rejected | P0 |
| F-API-016 | Payment Callback - Signature Validation | /callback/payment | Forged/wrong signature | 401 + signature verification failed, security log recorded | P0 |
| F-API-017 | Payment Callback - Amount Validation | /callback/payment | Callback amount differs from order amount | Amount mismatch alert, order enters manual processing queue | P0 |
| F-API-018 | WMS Shipping Callback - Required Fields | /callback/wms/ship | Missing orderId or trackingNo | 400 + callback rejected | P0 |
| F-API-019 | WMS Shipping Callback - Order Status Validation | /callback/wms/ship | Corresponding order not in PAID status | If SHIPPED→idempotent return; if CANCELLED→alert+manual intervention | P0 |
| F-API-020 | WMS Return Callback - Quantity Validation | /callback/wms/return | Callback return quantity differs from return order | Alert+correct with return order as standard | P1 |
| F-API-021 | Callback Replay Attack Protection | Various callback interfaces | Use same parameters to send repeatedly in short time | Idempotent return, no duplicate processing, identify replay and record | P1 |

---

## 5. Priority Summary

| Priority | Quantity | Description |
|---|---|---|
| P0 | 35 | Core Happy Path, inventory sync critical path, API required fields/signature validation |
| P1 | 25 | Boundary scenarios, out-of-order messages, partial shipping, return process, idempotency |
| P2 | 1 | Partial shipping - partial receipt |

> **Total: 61 Functional Module Test Cases**

---

## 6. Strategy Explanation (F Module Focus)

The core goal of the F module is to **ensure system functional behavior meets expectations, covering all normal and abnormal paths**. The testing strategy focuses on inventory synchronization correctness and message processing robustness:

1. **Happy Path First** — Ensure the main chain is打通, and all core interfaces can be called normally.
2. **Inventory Synchronization is the Top Priority** — The idempotency and consistency of the four operations (reservation, deduction, release, restore) directly relate to funds and user experience.
3. **Out-of-Order Messages Cannot Be Ignored** — In distributed systems, out-of-order messages are the norm; the combination of payment → shipping → receipt must be covered.
4. **High-Concurrency Flash Sales are Typical Boundaries** — Not overselling is the bottom line of e-commerce systems; concurrent testing is needed for verification.
5. **API Contracts Focus on Field Validation and Error Codes** — Illegal input interception happens at the gateway layer; unified error code standards facilitate troubleshooting.