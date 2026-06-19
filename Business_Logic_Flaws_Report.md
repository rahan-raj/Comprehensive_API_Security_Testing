# Business Logic Flaws Report

**Tester:** Rahan Raj K R  
**Date:** June 2026  
**Target Application:** crAPI — http://localhost:8888  
**Tools Used:** Burp Suite (Repeater), Postman, curl  

---

##  Summary

This report documents testing performed against crAPI's business logic, including race conditions, sensitive data exposure, resource/pagination limits, and abuse scenarios such as price/quantity manipulation and IDOR in complex order workflows. **3 vulnerabilities were confirmed** (one Critical — unrestricted order quantity/balance bypass, one High — IDOR exposing PII via order lookup, one Medium — sensitive data exposure across multiple endpoints), plus one Low severity pagination validation issue, while the coupon race condition and direct price-field manipulation were tested thoroughly and found **not vulnerable**.

---

## 1. Test 1 — Race Condition on Coupon Application

**Objective:** Determine whether the coupon application endpoint can be abused by sending multiple simultaneous requests to apply the same coupon more than once.

**Setup:** Obtained a valid coupon code (shared via the community forum, e.g. `TRAC075`).

**Steps Performed:**
1. Captured a valid `POST /workshop/api/shop/apply_coupon` request in Burp Suite.
2. Sent the request to Repeater and duplicated it across 5 tabs.
3. Triggered all 5 requests as close to simultaneously as possible.
4. Checked account balance before and after.

**Request:**
```http
POST /workshop/api/shop/apply_coupon HTTP/1.1
Host: localhost:8888
Authorization: Bearer <token>
Content-Type: application/json

{"coupon_code": "TRAC075", "amount": 75}
```

**Alternative — curl parallel test:**
```bash
for i in {1..5}; do
  curl -X POST http://localhost:8888/workshop/api/shop/apply_coupon \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{"coupon_code":"TRAC075","amount":75}' &
done
wait
```

**Result:**

Testing was performed across two separate fresh accounts to ensure clean, reproducible conditions.

| Metric | hacker2@test.com | 
|---|---|
| Balance before any coupon | 100.0 |
| Balance after ONE legitimate coupon use | 175.0 (+75) |
| Balance after 5 simultaneous duplicate requests | 175.0 (unchanged) | 
| Individual response for all 5 simultaneous requests | `400 Bad Request` — "already claimed by you" (all 5) |

Each of the 5 simultaneous Repeater requests was inspected individually (not just the final balance) to confirm none slipped through with a false "success" response — all 5 were consistently and correctly rejected on both test accounts.

![evidence](./evidence/phase%204/race_condition_1.PNG)

**Finding 4.1 — Race Condition Testing on Coupon Application Endpoint**

| Field | Details |
|---|---|
| **Severity** | Informational (Not Vulnerable) |
| **OWASP Category** | API6:2023 — Unrestricted Access to Sensitive Business Flows |
| **Affected Endpoint** | `POST /workshop/api/shop/apply_coupon` |
| **Status** | Tested, Not Vulnerable |

**Description:** The coupon redemption endpoint correctly rejected all 5 simultaneous duplicate requests with `"already claimed by you"`, even when fired as close to concurrently as possible via Burp Suite Repeater. This result was reproduced identically on two separate test accounts. The endpoint appears to enforce atomicity correctly (e.g. via a database-level unique constraint or locking mechanism on coupon redemption per user).

**Evidence:** ![evidence](./evidence/phase%204/burp_suite_race_condition.PNG)

**Impact:** None. The coupon redemption logic correctly prevents duplicate application even under concurrent request conditions.

**Remediation:** No action required. Continue using the current atomic redemption-checking approach as a model for other sensitive business flows.

---

## 2. Test 2 — Sensitive Data Exposure in API Responses

**Objective:** Identify any fields returned in API responses that expose more data than necessary.

**Endpoints Reviewed:**

| Endpoint | Sensitive Field Found | Description |
|---|---|---|
| `GET /workshop/api/shop/orders/{id}` | `user.email`, `user.number` | Returns the order owner's full email and phone number to ANY authenticated user who requests that order ID, not just the owner |
| `GET /community/api/v2/community/posts/recent` | `author.email` | Returns every forum post author's full email address to any logged-in user browsing the forum |
| `GET /identity/api/v2/user/videos/{id}` | `profileVideo` (base64) | Embeds raw base64 image/video data directly in the JSON response rather than via a separately-authorized media URL |

**Sample Response — Order Lookup (Finding A):**
```json
{
  "order": {
    "id": 8,
    "user": {
      "email": "racetest@test.com",
      "number": "9876500004"
    },
    "product": {"id": 2, "name": "Wheel", "price": "10.00"},
    "quantity": 1,
    "status": "delivered"
  }
}
```
This was returned when the request was made by a completely unrelated account (`racetest2@test.com`), not the order's owner — see the corresponding IDOR finding (4.5) for full reproduction steps.

**Sample Response — Community Posts (Finding B):**
```json
"author": {
  "nickname": "Robot",
  "email": "robot001@example.com",
  ...
}
```

**Finding 4.2 — Sensitive Data Exposure in API Responses**

| Field | Details |
|---|---|
| **Severity** | Medium |
| **OWASP Category** | API3:2023 — Broken Object Property Level Authorization |
| **Status** | Confirmed |

**Description:** Multiple endpoints return personally identifiable information (PII) about OTHER users beyond what the requesting user needs or is entitled to see. The order lookup endpoint exposes another user's full email and phone number (directly tied to the IDOR in Finding 4.5). The community posts endpoint exposes every author's email address to any logged-in user as standard behavior. Additionally, video/profile endpoints embed raw base64 media data directly inline in JSON responses rather than serving it through a separately-authorized media endpoint, which is a design pattern worth reviewing even though no additional vulnerability beyond the existing IDOR (Phase 2, Finding 2.2) was confirmed here.

**Evidence:** 
**( Order PII Exposure )**
![evidence](./evidence/phase%204/order_pii_exposure.PNG) 

**( Community Posts Email Exposure )**
![evidence](./evidence/phase%204/community_posts_email_exposure.PNG)

**Impact:** Finding A allows trivial harvesting of real user contact details (email + phone) by enumerating order IDs — a direct, scalable privacy breach. Finding B exposes every forum participant's email address to the entire user base by design, which is poor practice for a public-facing community feature.

**Remediation:**
1. Never include full email/phone in order payloads returned to non-owners; gate contact info behind a separate, properly authorized endpoint if needed.
2. Replace full email addresses in community posts with nickname/display name only; implement in-app messaging if direct contact is a desired feature.
3. Consider serving profile images/videos via dedicated, separately-authorized media endpoints rather than embedding raw base64 data in general resource JSON responses.

---

## 3. Test 3 — Lack of Resource Limiting / Pagination

**Objective:** Determine whether list endpoints enforce a maximum page size.

**Requests:**
```http
GET /community/api/v2/community/posts/recent?limit=999999 HTTP/1.1
GET /community/api/v2/community/posts/recent?limit=0 HTTP/1.1
GET /community/api/v2/community/posts/recent?limit=-1 HTTP/1.1
Authorization: Bearer <token>
```

**Baseline (no parameters):** Returns all 4 existing posts, `total: 4`, `next_offset: null`.

**Result:**

| Test | Result |
|---|---|
| `limit=999999` | Returned all 4 posts with no error or cap applied; no upper bound enforced |
| `limit=0` | Returned all 4 posts (not an empty list as logically expected); malformed metadata: `next_offset: 0`, `previous_offset: 0` |
| `limit=-1` | Returned 1 post with nonsensical metadata: `next_offset: -1`, `previous_offset: 1` |

While the small size of this lab's dataset (4 seed posts) limits the immediate visible impact, the underlying lack of validation on the `limit` parameter would pose a real risk on a production system with a large dataset.

**Finding 4.3 — Missing Input Validation on Pagination Parameters**

| Field | Details |
|---|---|
| **Severity** | Low |
| **OWASP Category** | API4:2023 — Unrestricted Resource Consumption |
| **Affected Endpoint** | `GET /community/api/v2/community/posts/recent` |
| **Status** | Confirmed |

**Description:** The `limit` parameter accepts any integer value — including excessively large numbers, zero, and negative numbers — with no server-side validation or sanitization. Zero and negative values also produce malformed pagination metadata (`next_offset`/`previous_offset`) rather than being rejected or normalized.

**Evidence:**
 ![evidence](./evidence/phase%204/pagination-validation-test.png.jpeg)

**Impact:** On a dataset of realistic production size, an unbounded `limit` value could allow a single request to retrieve the entire dataset in one call, leading to excessive server load, potential denial-of-service, or bulk data scraping. The malformed offset values for invalid inputs also indicate the pagination logic is not robust against unexpected input.

**Remediation:** Enforce a hard maximum on `limit` server-side (e.g. cap at 50–100); reject or default invalid values (zero, negative, non-numeric) to a sane default rather than producing malformed metadata.

---

## 4. Test 4 — API Abuse / Price Manipulation

**Objective:** Determine whether order quantity or pricing fields can be manipulated client-side.

**Test A — Client-Supplied Price Field**

**Request:**
```http
POST /workshop/api/shop/orders HTTP/1.1
Authorization: Bearer <token>
Content-Type: application/json

{"product_id": 1, "quantity": 1, "price": 0.01}
```

**Result:** Balance before: 175.0. Balance after: 165.0 — a drop of exactly 10.00 (the real product price), not 0.01 as injected. The server ignored the client-supplied `price` field entirely and charged the actual product price looked up server-side.

**Finding 4.4a — Price Field Injection (Not Vulnerable)**

| Field | Details |
|---|---|
| **Severity** | Informational (Not Vulnerable) |
| **OWASP Category** | API6:2023 — Unrestricted Access to Sensitive Business Flows |
| **Status** | Tested, Not Vulnerable |

**Description:** The order endpoint correctly ignores a client-supplied `price` field and calculates the real price server-side from the product record.

**Evidence:**
![evidence](./evidence/phase%204/API_abuse_scenario_2.PNG) 
![evidence](./evidence/phase%204/API_abuse_scenario_1.PNG)


**Impact:** None. Confirms order pricing is calculated authoritatively on the server.

**Remediation:** No action required. Continue calculating all prices server-side.

---

**Test B — Unrestricted Order Quantity / No Balance Check**

**Request:**
```http
POST /workshop/api/shop/orders HTTP/1.1
Authorization: Bearer <token>
Content-Type: application/json

{"product_id": 1, "quantity": 999999}
```

**Result:** Balance before: 165.0. Order accepted with no rejection. Balance after: **-9999825.0** (165.0 − (999999 × 10.00) = −9999825.0, confirming the full order value was deducted with no validation).

**Finding 4.4b — No Quantity Limit or Balance Check on Order Endpoint**

| Field | Details |
|---|---|
| **Severity** | **Critical** |
| **OWASP Category** | API4:2023 — Unrestricted Resource Consumption / API6:2023 — Unrestricted Access to Sensitive Business Flows |
| **Status** | Confirmed |

**Description:** The order endpoint does not enforce any upper limit on the `quantity` field, nor does it verify that the user has sufficient balance before processing the order. An order requesting an arbitrarily large quantity is processed successfully regardless of cost or available credit, driving the account balance deeply negative.

**Evidence:** 
![evidence](./evidence/phase%204/API_abuse_scenario_3.PNG)
![evidence](./evidence/phase%204/API_abuse_scenario_4.PNG)

**Impact:** Critical business logic flaw. Users can rack up effectively unlimited debt with no enforcement — potentially exploitable for fraud (ordering massive quantities of real goods with no ability or intention to pay). Combined with the Phase 3 finding (negative quantity inflating balance), this confirms the order/balance system has no meaningful server-side financial integrity controls at all. At scale, this could be abused to place fraudulent bulk orders, causing direct financial and operational damage.

**Remediation:**
1. Enforce a maximum quantity per order (e.g. 1–50) with server-side validation, rejecting anything above the cap with `HTTP 400`.
2. Verify `available_credit >= total_order_cost` BEFORE processing any order; reject with a clear "Insufficient balance" error if not met (this exact error message already exists elsewhere in the application — see Phase 3 testing — but is not applied here).
3. Add server-side maximum order value checks independent of quantity, as defense-in-depth.
4. Add automated regression tests covering this exact scenario (large quantity vs. low balance).

---

## 5. Test 5 — IDOR in Order Workflow

**Objective:** Determine whether a user can view or modify another user's orders within the order workflow.

**Setup:** User A (`racetest@test.com`) placed an order via `POST /workshop/api/shop/orders`, receiving `order id: 8`.

**Request (made AS User B, targeting User A's order):**
```http
GET /workshop/api/shop/orders/8 HTTP/1.1
Authorization: Bearer <racetest2_token>
```

**Result:** The server returned User A's **full order details**, including their email address and phone number, despite the request being authenticated as a completely unrelated account (`racetest2@test.com`) that had never interacted with order 8:

```json
{
  "order": {
    "id": 8,
    "user": {
      "email": "racetest@test.com",
      "number": "9876500004"
    },
    "product": {"id": 2, "name": "Wheel", "price": "10.00"},
    "quantity": 1,
    "status": "delivered",
    "transaction_id": "91e213db-854c-4b31-b22c-3606b902cced"
  }
}
```

**Control test:** `GET /workshop/api/shop/orders/all` using User B's token correctly returned an empty list (`count: 0`), confirming User B has no orders of their own. This proves the *listing* endpoint is correctly scoped to the authenticated user, while the *single-order lookup* endpoint is **not** — an inconsistency in authorization enforcement across two closely related endpoints in the same feature.

**Finding 4.5 — Insecure Direct Object Reference (IDOR) on Single Order Lookup Endpoint**

| Field | Details |
|---|---|
| **Severity** | High |
| **OWASP Category** | API1:2023 — Broken Object Level Authorization |
| **Affected Endpoint** | `GET /workshop/api/shop/orders/{id}` |
| **Status** | Confirmed |

**Description:** The single-order lookup endpoint does not verify that the authenticated user is the owner of the requested order. Any authenticated user can retrieve another user's complete order details — including PII (email, phone number) and purchase history — by simply guessing or incrementing the order ID in the URL.

**Evidence:** 
![evidence](./evidence/phase%204/IDOR_in_complex_workflows_1.PNG)

**Impact:** Any authenticated user — even a brand-new account with no purchase history — can enumerate sequential order IDs to harvest other users' email addresses, phone numbers, and purchase history at scale. This is a direct, high-impact privacy breach.

**Remediation:**
1. Before returning order data, verify server-side that `order.user_id == authenticated_user.id`; return `403 Forbidden` or `404 Not Found` if they don't match.
2. Apply this same ownership check consistently across all `{id}`-based endpoints, not just listing endpoints — audit every ID-based route in the orders, vehicles, and videos services for the same pattern (see also Phase 2, Findings 2.2/2.3).
3. Avoid returning full user contact details (email, phone) in order responses at all where not strictly necessary.

---

##  Findings Summary

| # | Finding | Severity | Status |
|---|---|---|---|
| 4.1 | Race condition — coupon redemption | Informational | Tested, Not Vulnerable |
| 4.2 | Sensitive data exposure (orders, community posts) | Medium | Confirmed |
| 4.3 | Missing input validation on pagination parameters | Low | Confirmed |
| 4.4a | Price field injection | Informational | Tested, Not Vulnerable |
| 4.4b | No quantity limit / balance check on orders | **Critical** | **Confirmed** |
| 4.5 | IDOR on single order lookup endpoint | **High** | **Confirmed** |

---

## 8. Evidence Index

All screenshots referenced above are stored in `evidence/phase4/`.
