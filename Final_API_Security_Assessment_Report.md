# API Security Assessment Report
## crAPI — Completely Ridiculous API

---


 **Prepared by** :- Rahan Raj K R     
 **Target Application** :- crAPI (Completely Ridiculous API)   
 **Environment** :- Local lab — http://localhost:8888   

---

## 1. Executive Summary

A comprehensive API security assessment was conducted against the crAPI application between June 11 and June 20, 2026, as part of the ArcByte Cybersecurity Internship Program. The assessment followed the OWASP API Security Top 10 (2023) methodology across six structured phases: API discovery, authentication and authorization testing, input validation and injection testing, business logic testing, professional reporting, and remediation guidance.

The assessment identified **11 total findings** across the application:

| Severity | Count |
|---|---|
| Critical | 2 |
| High | 3 |
| Medium | 2 |
| Low | 2 |
| Informational (Not Vulnerable / Tested Clean) | 7 |

The two most critical findings are a **Command Injection vulnerability** in the video conversion feature, which allows any authenticated user to execute arbitrary operating system commands on the server, and an **unrestricted order quantity flaw** that allows users to place orders of any size regardless of their available balance — effectively generating unlimited debt. Both require immediate remediation.

Three High severity findings were also identified: two Insecure Direct Object Reference (IDOR) vulnerabilities allowing any authenticated user to access other users' private videos, vehicle GPS locations, and order details including personal contact information (email address and phone number), and a missing rate limit on the login endpoint that exposes the application to automated brute-force attacks.

Positively, several areas demonstrated sound security practices: the coupon redemption system correctly resisted concurrent race-condition attacks, the signup endpoint properly ignored attempted privilege-escalation fields (isAdmin, role, credit), and the login endpoint resisted SQL and NoSQL injection on tested fields.

Immediate remediation is recommended for all Critical and High severity findings. Detailed reproduction steps, evidence references, and specific remediation guidance for every finding are provided in Section 4.

---

## 2. Scope & Methodology

### 2.1 Scope

| Item | Detail |
|---|---|
| **Target Application** | crAPI (Completely Ridiculous API) |
| **Environment** | Locally hosted via Docker — http://localhost:8888 |
| **In Scope** | All API endpoints exposed by crAPI's identity, community, and workshop services |
| **Out of Scope** | Any production system, third-party integrations, physical security, host OS hardening |


### 2.2 Lab Environment

crAPI is an intentionally vulnerable API application created by OWASP for security training purposes. It simulates a car-management platform with user authentication, a community forum, a vehicle tracking system, and an e-commerce shop. All testing was performed exclusively on this locally-hosted lab — no production systems, external networks, or real user data were accessed or affected at any point during this assessment.

### 2.3 Methodology

Testing followed the **OWASP API Security Top 10 (2023)** framework, structured across six phases:

| Phase | Focus Area | Duration |
|---|---|---|
| 1 | Reconnaissance & API Discovery | Day 1 |
| 2 | Authentication & Authorization Testing | Days 1–3 |
| 3 | Input Validation & Injection Testing | Days 3–5 |
| 4 | Business Logic & Advanced Testing | Days 5–7 |
| 5 | Professional Reporting | Days 7–8 |
| 6 | Remediation Guidance  | Final Day |

### 2.4 Tools Used

| Tool | Purpose |
|---|---|
| **Burp Suite Community Edition** | Traffic interception, Repeater for race-condition and concurrent-request testing |
| **Postman** | Manual API request crafting, testing, and collection management |
| **curl** | Command-line request automation for rate-limit testing |
| **jwt.io** | JWT token decoding and algorithm analysis |
| **Docker Desktop** | Running the crAPI lab environment |

### 2.5 API Services Tested

crAPI exposes three distinct backend services, each with its own base path:

| Service | Base Path | Technology |
|---|---|---|
| Identity Service | `/identity/api/` | Java / Spring Framework |
| Community Service | `/community/api/` | Go / MongoDB |
| Workshop Service | `/workshop/api/` | Python |

---

## 3. Risk Rating Summary

All findings are listed below, ordered by severity. Full details for each finding appear in Section 4.

| ID | Finding | Severity  | Phase | Status |
|---|---|---|---|---|
| 01 | Command Injection via conversion_params | **Critical**  | 3 | Confirmed |
| 02 | Unrestricted Order Quantity / No Balance Check | **Critical**  | 4 | Confirmed |
| 03 | IDOR — Video Endpoint | **High**  | 2 | Confirmed |
| 04 | IDOR — Vehicle Location Endpoint | **High**  | 2 | Confirmed |
| 05 | IDOR — Single Order Lookup (PII Exposure) | **High**  | 4 | Confirmed |
| 06 | Missing Rate Limiting on Login Endpoint | Medium  | 2 | Confirmed |
| 07 | Sensitive Data Exposure (Orders + Community) | Medium  | 4 | Confirmed |
| 08 | Negative Quantity / Balance Manipulation | High  | 3 | Confirmed |
| 09 | Verbose Internal Error Message — Info Disclosure | Low | 3 | Confirmed |
| 10 | Missing Pagination Input Validation | Low  | 4 | Confirmed |
| 11 | JWT Configuration (Algorithm / Expiry) | [TBD after jwt.io analysis]  | 2 | [TBD] |

**Not Vulnerable (Tested Clean):**
- SQL Injection — Login endpoint
- NoSQL Injection — Community endpoints
- Mass Assignment — Signup endpoint (role, credit fields)
- Price Field Injection — Order endpoint
- Race Condition — Coupon redemption
- Horizontal Privilege Escalation — Video update endpoint
- Password Brute-force (basic rate limiting appears present on some paths)

---

## 4. Detailed Findings

---

### F-01 — Command Injection via conversion_params Field

| Field | Details |
|---|---|
| **Severity** | Critical |
| **OWASP Category** | API8:2023 — Security Misconfiguration / Injection |
| **Affected Endpoint** | `PUT /identity/api/v2/user/videos/{id}` |
| **Affected Field** | `conversion_params` |
| **Authentication Required** | Yes (any valid JWT) |

**Description:**
The `conversion_params` field, intended to hold video-conversion flags (e.g. `-v codec h264`), is passed to a server-side process without sanitization — most likely a shell command invoking a video-conversion tool such as ffmpeg. Appending a shell command separator (`;`) followed by any arbitrary command causes that command to be executed by the underlying operating system.

**Steps to Reproduce:**
1. Authenticate as any valid user and obtain a JWT token.
2. Send the following request:
```http
PUT /identity/api/v2/user/videos/{own_video_id} HTTP/1.1
Host: localhost:8888
Authorization: Bearer <token>
Content-Type: application/json

{"video_name": "test", "conversion_params": "-v codec h264; sleep 10"}
```
3. Observe that the response is delayed by approximately 10 seconds.
4. Repeat with `; sleep 5` — response delay scales to ~5 seconds, confirming the injected value directly controls server-side execution time.
5. Control test using `-v codec h264` (no injection) returns instantly, ruling out network/server lag.

**Evidence:** 
![evidence](./evidence/phase%203/command%20injection.PNG)

**Impact:**
Any authenticated user — even a low-privilege account — can execute arbitrary operating system commands on the server. Depending on the permissions of the process running the video conversion, this could lead to full server compromise, theft of all users' data and server files, lateral movement into internal networks, or a complete denial of service. This is the most severe finding in this assessment.

**Remediation:**
1. Never pass user-controlled input directly into shell commands.
2. Use a strict server-side allowlist of permitted `conversion_params` values rather than accepting free-form text.
3. If shelling out is unavoidable, use parameterized subprocess APIs (passing arguments as an array, not a concatenated string) so shell metacharacters like `;`, `|`, and `&&` cannot be interpreted.
4. Run the video-processing subprocess in a sandboxed, minimum-privilege environment.

---

### F-02 — Unrestricted Order Quantity / No Balance Check

| Field | Details |
|---|---|
| **Severity** | Critical |
| **OWASP Category** | API4:2023 — Unrestricted Resource Consumption / API6:2023 — Unrestricted Access to Sensitive Business Flows |
| **Affected Endpoint** | `POST /workshop/api/shop/orders` |
| **Authentication Required** | Yes (any valid JWT) |

**Description:**
The order endpoint does not enforce any upper limit on the `quantity` field, nor does it verify that the user has sufficient balance before processing the order. An order requesting an arbitrarily large quantity is processed successfully regardless of cost or available credit.

**Steps to Reproduce:**
1. Confirm starting `available_credit` = `165.0` via `GET /identity/api/v2/user/dashboard`.
2. Send:
```http
POST /workshop/api/shop/orders HTTP/1.1
Authorization: Bearer <token>
Content-Type: application/json

{"product_id": 1, "quantity": 999999}
```
3. Order is accepted with no rejection.
4. Check dashboard — `available_credit` is now `-9999825.0`.

**Evidence:**
![evidence](./evidence/phase%204/API_abuse_scenario_3.PNG) 
![evidence](./evidence/phase%204/API_abuse_scenario_4.PNG)

**Impact:**
Users can place orders of unlimited value with no balance enforcement — exploitable for fraud against real inventory/fulfillment systems. Combined with F-08 (negative quantity), the entire financial integrity of the order system is compromised.

**Remediation:**
1. Enforce server-side maximum quantity per order (e.g. 1–50); reject above-cap values with `HTTP 400`.
2. Verify `available_credit >= total_order_cost` before processing; reject with a clear error if insufficient.
3. Add automated regression tests covering boundary values (large quantity, zero balance) for all order endpoints.

---

### F-03 — IDOR on Video Endpoint

| Field | Details |
|---|---|
| **Severity** | High |
| **OWASP Category** | API1:2023 — Broken Object Level Authorization |
| **Affected Endpoint** | `GET /identity/api/v2/user/videos/{id}` |
| **Authentication Required** | Yes (any valid JWT) |

**Description:**
The video endpoint does not verify that the authenticated user owns the requested video. Any authenticated user can retrieve another user's private video by incrementing the `id` parameter.

**Steps to Reproduce:**
1. Log in as User A, call `GET /identity/api/v2/user/videos` — note your video ID (e.g. `5`).
2. Log in as User B, call `GET /identity/api/v2/user/videos` — note your video ID (e.g. `6`).
3. Using **User A's token**, call `GET /identity/api/v2/user/videos/6`.
4. Response returns User B's video data.


**Impact:** Any authenticated user can enumerate and access private videos belonging to other users.

**Remediation:** Verify `resource.owner_id == authenticated_user.id` server-side before returning any resource, on every request.

---

### F-04 — IDOR on Vehicle Location Endpoint

| Field | Details |
|---|---|
| **Severity** | High |
| **OWASP Category** | API1:2023 — Broken Object Level Authorization |
| **Affected Endpoint** | `GET /identity/api/v2/vehicle/{vehicleId}/location` |
| **Authentication Required** | Yes (any valid JWT) |

**Description:**
Vehicle GPS location data belonging to other users is accessible by referencing their vehicle ID with one's own valid token. This follows the same pattern as F-03 but exposes real-time/historical GPS coordinates — a significant physical safety risk.

**Steps to Reproduce:**
1. Log in as User A, call `GET /identity/api/v2/vehicle/vehicles` — note User A's vehicle UUID.
2. Log in as User B to obtain their vehicle UUID via the same endpoint.
3. Using **User A's token**, call `GET /identity/api/v2/vehicle/{User_B_vehicle_UUID}/location`.
4. Response returns User B's GPS location data.

**Evidence:** 
![evidence](./evidence/phase%201/vehicle%20endpoints/location.PNG)

**Impact:** Exposure of real-time GPS coordinates could enable physical tracking of victims — a direct personal safety risk beyond a typical data breach.

**Remediation:** Apply the same ownership verification as F-03 to all vehicle-related endpoints. Conduct a full audit of all `{id}`-based endpoints across all three services for the same pattern.

---

### F-05 — IDOR on Single Order Lookup (PII Exposure)

| Field | Details |
|---|---|
| **Severity** | High |
| **OWASP Category** | API1:2023 — Broken Object Level Authorization |
| **Affected Endpoint** | `GET /workshop/api/shop/orders/{id}` |
| **Authentication Required** | Yes (any valid JWT) |

**Description:**
The single-order lookup endpoint does not verify that the authenticated user owns the requested order. Any authenticated user can retrieve another user's complete order — including their email address, phone number, and purchase history — by guessing or incrementing the order ID.

**Steps to Reproduce:**
1. Log in as User A (`racetest@test.com`), place an order — order `id: 8` created.
2. Log in as User B (`racetest2@test.com`) — a separate account that has never interacted with order 8.
3. Using **User B's token**, call `GET /workshop/api/shop/orders/8`.
4. Response returns User A's full order including:
   - Email: `racetest@test.com`
   - Phone: `9876500004`
   - Product, quantity, status, transaction ID

**Control Test:** `GET /workshop/api/shop/orders/all` using User B's token returned an empty list (`count: 0`), confirming the listing endpoint is correctly scoped while the single-lookup endpoint is not — an inconsistency in authorization enforcement across two closely related endpoints.

**Evidence:** 
![evidence](./evidence/phase%204/IDOR_in_complex_workflows_1.PNG)

**Impact:** Any authenticated user can enumerate sequential order IDs to harvest other users' email addresses and phone numbers at scale — a direct, scalable privacy breach.

**Remediation:**
1. Before returning order data, verify `order.user_id == authenticated_user.id`; return `403 Forbidden` or `404 Not Found` if they don't match.
2. Audit all `{id}`-based routes across the workshop service for the same pattern.
3. Consider not including full contact details (email, phone) in order response payloads at all.

---

### F-06 — Missing Rate Limiting on Login Endpoint

| Field | Details |
|---|---|
| **Severity** | Medium |
| **OWASP Category** | API4:2023 — Unrestricted Resource Consumption |
| **Affected Endpoint** | `POST /identity/api/auth/login` |
| **Authentication Required** | No |

**Description:**
The login endpoint accepted 10 consecutive failed authentication attempts without any throttling, lockout, or CAPTCHA challenge, leaving it open to automated brute-force and credential-stuffing attacks.

**Steps to Reproduce:**
Run the following in Command Prompt on Windows:
```cmd
for /L %i in (1,1,10) do curl -s -o NUL -w "Request %i: HTTP %%{http_code}\n" -X POST http://localhost:8888/identity/api/auth/login -H "Content-Type: application/json" -d "{\"email\":\"test@test.com\",\"password\":\"wrongpassword\"}"
```
All 10 requests return `HTTP 401` (wrong password) with no blocking or throttling occurring.

**Evidence:** 
![evidence](./evidence/phase%202/missing%20rate%20limiting%20on%20login.PNG)

**Impact:** Attackers can automate unlimited password-guessing attempts with no consequence, making the application vulnerable to credential-stuffing attacks using leaked password lists.

**Remediation:**
1. Implement rate limiting (e.g. max 5 failed attempts per 10 minutes per IP/account); return `HTTP 429 Too Many Requests` once exceeded.
2. Consider progressive delays and account lockout after repeated failures.
3. Add CAPTCHA after a defined number of failed attempts.

---

### F-07 — Sensitive Data Exposure in API Responses

| Field | Details |
|---|---|
| **Severity** | Medium |
| **OWASP Category** | API3:2023 — Broken Object Property Level Authorization |
| **Affected Endpoints** | `GET /workshop/api/shop/orders/{id}`, `GET /community/api/v2/community/posts/recent` |
| **Authentication Required** | Yes |

**Description:**
Multiple endpoints return personally identifiable information about OTHER users beyond what the requesting user needs or is entitled to see. The order endpoint returns another user's full email and phone number (directly linked to F-05). The community posts endpoint returns every post author's full email address to any logged-in user browsing the forum.

**Evidence:** 
**( order  pii exposure )**
![evidence](./evidence/phase%204/order_pii_exposure.PNG)
**( community posts email exposure )**
![evidence](./evidence/phase%204/community_posts_email_exposure.PNG)

**Impact:** Enables bulk harvesting of user contact details (email + phone) from orders, and exposes every forum participant's email address to the entire user base.

**Remediation:**
1. Remove full email/phone from order response payloads for non-owners.
2. Replace email addresses in community post author objects with display name/nickname only.
3. Implement in-app messaging if user-to-user contact is a desired product feature.

---

### F-08 — Negative Quantity Accepted / Balance Manipulation

| Field | Details |
|---|---|
| **Severity** | High |
| **OWASP Category** | API6:2023 — Unrestricted Access to Sensitive Business Flows |
| **Affected Endpoint** | `POST /workshop/api/shop/orders` |
| **Authentication Required** | Yes |

**Description:**
The order endpoint accepts negative values for the `quantity` field. A negative quantity causes the server to calculate a negative total cost, which is then added to the user's balance — effectively allowing users to generate free credit on demand.

**Steps to Reproduce:**
1. Confirm starting `available_credit` = `100.0`.
2. Send `POST /workshop/api/shop/orders` with body: `{"product_id": 1, "quantity": -5}`.
3. Response: `{"message": "Order sent successfully.", "credit": 150.0}`.
4. Balance increased from `100.0` to `150.0` — confirming the exploit.

**Evidence:** 
![evidence](./evidence/phase%203/improper%20input%20sanitization.PNG)

**Impact:** Users can repeat this indefinitely to generate unlimited account credit, resulting in direct financial loss. Shares the same root cause as F-02 — both confirm the absence of basic server-side financial integrity controls in the order system.

**Remediation:** Validate that `quantity` is a positive integer (≥ 1) before processing any order; reject zero, negative, or non-integer values with `HTTP 400`.

---

### F-09 — Verbose Internal Error Message / Information Disclosure

| Field | Details |
|---|---|
| **Severity** | Low |
| **OWASP Category** | API8:2023 — Security Misconfiguration |
| **Affected Endpoint** | `POST /identity/api/auth/login` |
| **Authentication Required** | No |

**Description:**
When a request fails server-side input validation (e.g. a password shorter than the minimum length), the API returns a raw internal stack trace rather than a generic error message, revealing the backend framework (Spring), internal class and field names, and validation logic.

**Steps to Reproduce:**
Send `POST /identity/api/auth/login` with body: `{"email": "test@test.com", "password": "x"}`

**Response:**
```json
{
  "message": "Validation failed",
  "details": "org.springframework.validation.BeanPropertyBindingResult: 1 errors\nField error in object 'loginForm' on field 'password'..."
}
```

**Evidence:** 
![evidence](./evidence/phase%203/error%20in%20sql%20injection.PNG)

**Impact:** Helps attackers fingerprint the technology stack (Spring Framework, Java) to craft more targeted exploits.

**Remediation:** Catch all validation exceptions server-side and return a generic, user-friendly error (e.g. `"Invalid input"`) without exposing internal class names, stack traces, or framework-specific details.

---

### F-10 — Missing Input Validation on Pagination Parameters

| Field | Details |
|---|---|
| **Severity** | Low |
| **OWASP Category** | API4:2023 — Unrestricted Resource Consumption |
| **Affected Endpoint** | `GET /community/api/v2/community/posts/recent` |
| **Authentication Required** | Yes |

**Description:**
The `limit` query parameter accepts any integer value — including excessively large numbers, zero, and negative values — with no server-side validation. Invalid values also produce malformed pagination metadata.

| Test | Result |
|---|---|
| `limit=999999` | Returned all records with no cap applied |
| `limit=0` | Returned all records instead of an empty list; malformed metadata |
| `limit=-1` | Returned 1 record with nonsensical `next_offset: -1`, `previous_offset: 1` |

**Evidence:** 
![evidence](./evidence/phase%204/pagination-validation-test.png.jpeg)

**Impact:** On a production dataset, an unbounded `limit` value could allow bulk data scraping or resource-exhaustion attacks in a single request.

**Remediation:**
1. Enforce a hard server-side maximum page size (e.g. 50–100), regardless of client input.
2. Reject or normalize invalid values (zero, negative, non-integer) to a safe default with an appropriate error.

---

### F-11 — JWT Configuration Analysis

| Field | Details |
|---|---|
| **Severity** | [Update after completing jwt.io analysis] |
| **OWASP Category** | API2:2023 — Broken Authentication |
| **Affected Endpoint** | `POST /identity/api/auth/login` (token issuance) |

**Observations from jwt.io:**
- Algorithm (`alg`): `RS256` (asymmetric — generally good)
- Token expiry (`exp`): Present — `iat + 604800` seconds (7 days — long-lived, see below)
- Payload claims: `sub` (email), `iat`, `exp`, `role`

**Description:**
The JWT algorithm (`RS256`) is appropriate and not susceptible to the `none` algorithm confusion attack (confirmed: sending a token with `alg: none` returned `401 Unauthorized`). However, the 7-day token lifetime is unusually long — a stolen token remains valid for up to a week with no mechanism to revoke it early.

**Evidence:** 
![evidence](./evidence/phase%202/JWT.PNG)

**Impact:** Medium — a stolen token (via XSS, interception, or device theft) remains exploitable for 7 days with no server-side way to invalidate it.

**Remediation:**
1. Reduce access token lifetime to 15–60 minutes and implement refresh-token rotation for longer sessions.
2. Consider implementing a token revocation list (blocklist) for logout and account-compromise scenarios.

---

## 5. Recommendations

The following recommendations are listed in priority order, from most to least urgent:

### Priority 1 — Fix Immediately (Critical)

**1. Eliminate Command Injection (F-01)**
Never pass user-supplied text directly into shell commands. Use a strict allowlist of permitted conversion flag values, and invoke video-conversion tools via parameterized subprocess APIs (not concatenated shell strings). This is the single highest-priority fix — it allows full server compromise from any valid account.

**2. Add Financial Validation to the Order System (F-02, F-08)**
The order system lacks basic financial integrity checks. Enforce: (a) `quantity` must be a positive integer within a defined maximum; (b) `available_credit >= total_order_cost` must be verified server-side before any order is processed. These two checks together close both the negative-quantity and oversized-quantity exploits, which share the same root cause.

### Priority 2 — Fix Urgently (High)

**3. Implement Object-Level Authorization on All ID-Based Endpoints (F-03, F-04, F-05)**
Every endpoint that returns or modifies a specific resource using an ID in the URL must verify the authenticated user owns or is permitted to access that resource — server-side, on every request. Currently this check is missing from at least three separate endpoints across two services (videos, vehicles, orders), suggesting it was never applied as a standard pattern. Conduct a full audit of every `{id}`-based route across all three services and apply ownership verification consistently.

**4. Rate-Limit the Login Endpoint (F-06)**
Add throttling (max 5 attempts / 10 minutes / IP+account) and return `HTTP 429` to close the brute-force exposure on the authentication endpoint.

### Priority 3 — Fix Soon (Medium)

**5. Remove Unnecessary PII from API Responses (F-07)**
Audit every response payload and apply the principle of least exposure: return only the fields the client actually requires. Specifically: remove email/phone from order responses visible to non-owners, and replace email addresses in community post author objects with display names only.

### Priority 4 — Fix in Next Development Cycle (Low)

**6. Suppress Internal Error Details (F-09)**
Return generic, user-friendly error messages for all validation failures; never expose framework class names, stack traces, or field names in API error responses.

**7. Validate and Cap Pagination Parameters (F-10)**
Add a hard server-side cap on the `limit` parameter (e.g. 50 or 100), and normalize or reject invalid values (zero, negative, non-integer).

**8. Reduce JWT Token Lifetime (F-11)**
Shorten access token expiry from 7 days to 15–60 minutes and implement refresh-token rotation to reduce the window of exposure for any stolen token.

### General Long-Term Recommendations

**9. Establish Consistent Security Patterns Across All Services**
The assessment found that authorization enforcement is inconsistent across closely related endpoints within the same feature (e.g. `/orders/all` correctly scopes to the authenticated user while `/orders/{id}` does not). This suggests security controls are being applied endpoint-by-endpoint rather than at the framework/middleware level. Implement centralized authorization middleware that applies object-level checks uniformly, rather than relying on individual developers to remember to add them per endpoint.

**10. Integrate Security Testing into the Development Pipeline**
Tools such as OWASP ZAP can be configured to run automated API security scans against a staging environment on every code push, catching regressions of the issues found in this assessment before they reach production.

**11. Adopt a Secure-by-Default API Framework Pattern**
Consider adopting API framework conventions that make secure behavior the default — for example, response serializers that require explicit field allowlisting (preventing mass assignment and data over-exposure by default), and middleware that enforces authentication and ownership checks before any controller logic runs.

---

## 6. Conclusion

This assessment identified a total of 11 security findings in crAPI, including 2 Critical vulnerabilities that could each independently result in full application compromise or significant financial fraud with minimal attacker skill required. The most concerning pattern across the findings is not any single vulnerability, but the systemic absence of consistent authorization and input validation controls — the same class of bug (broken object-level authorization) appears independently in at least three separate endpoints across two different backend services, and the same class of financial logic bug (no server-side validation on order values) appears in two different entry points.

Addressing the root-cause patterns (centralized authorization enforcement, consistent input validation, server-side financial integrity checks) rather than fixing each finding in isolation will result in a significantly more robust API surface and reduce the likelihood of similar findings recurring in future assessments.

All testing was performed ethically on an intentionally vulnerable local lab environment. No production systems, real user data, or external networks were accessed or affected at any point during this assessment.

---


##  OWASP API Security Top 10 (2023) Mapping

| OWASP ID | Name | Findings Mapped |
|---|---|---|
| API1:2023 | Broken Object Level Authorization | F-03, F-04, F-05 |
| API2:2023 | Broken Authentication | F-11 |
| API3:2023 | Broken Object Property Level Authorization | F-07 |
| API4:2023 | Unrestricted Resource Consumption | F-02, F-06, F-10 |
| API5:2023 | Broken Function Level Authorization | Tested — not confirmed |
| API6:2023 | Unrestricted Access to Sensitive Business Flows | F-02, F-08 |
| API7:2023 | Server Side Request Forgery | Not tested |
| API8:2023 | Security Misconfiguration | F-01, F-09 |
| API9:2023 | Improper Inventory Management | Addressed via Phase 1 API Discovery |
| API10:2023 | Unsafe Consumption of APIs | Not tested |

---

*ArcByte Studios Pvt Ltd · Cybersecurity Internship Program · June 2026*
*This report was produced in a controlled lab environment for educational purposes only.*
