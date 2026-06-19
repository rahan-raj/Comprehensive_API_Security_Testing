# Input Validation & Injection Testing Report

**Tester:** Rahan Raj K R  
**Date:** June 2026  
**Target Application:** crAPI — http://localhost:8888  
**Tools Used:** Postman, Burp Suite, curl  

---

## Summary

This report documents testing performed against crAPI's input handling, including SQL/NoSQL injection, command injection, mass assignment, and general input sanitization. **2 vulnerabilities were confirmed** (one Critical — Command Injection, one High — Negative Quantity / Balance Manipulation), one Low severity information disclosure issue was found incidentally, and SQL Injection, NoSQL Injection, and Mass Assignment were tested thoroughly and found **not vulnerable** on the endpoints/fields tested.

---

## 1. Test 1 — SQL Injection in Login Field

**Objective:** Determine whether the `email` field in the login endpoint is vulnerable to SQL injection.

**Payloads Tested:**
```json
{"email": "test@test.com' OR '1'='1", "password": "anything"}
{"email": "admin'--", "password": "wrongpassword123"}
{"email": "admin'--", "password": "x"}
```

**Request:**
```http
POST /identity/api/auth/login HTTP/1.1
Host: localhost:8888
Content-Type: application/json

{"email": "test@test.com' OR '1'='1", "password": "anything"}
```

**Result:** Both payloads were treated as literal email address strings, not as SQL commands. The server responded with `"Given Email is not registered!"` for the `' OR '1'='1` payload — a normal, safe rejection. For the `admin'--` payload, a first attempt using a 1-character password (`"x"`) triggered a separate validation error (see Finding 3.1b below) before the email field was ever evaluated; retrying with a valid-length password (`"wrongpassword123"`) again returned `"Given Email is not registered!"`. No authentication bypass occurred and no SQL error or database information was leaked.

**Finding 3.1 — SQL Injection on Login Endpoint**

| Field | Details |
|---|---|
| **Severity** | Informational (Not Vulnerable) |
| **OWASP Category** | API8:2023 — Security Misconfiguration / Injection |
| **Affected Endpoint** | `POST /identity/api/auth/login` |
| **Status** | Tested, Not Vulnerable |

**Description:** Both classic SQL injection payloads (`' OR '1'='1`, `admin'--`) were treated as literal, non-existent email strings rather than executed as SQL. This indicates the backend uses parameterized queries or an ORM that safely separates user input from query structure.

**Evidence:** ![evidence](./evidence/phase%203/sql%20injection%201.PNG)

**Impact:** None. Endpoint appears to correctly use parameterized queries, preventing SQL injection via the email field.

**Remediation:** No action required for this finding. Continue using parameterized queries / ORM-based data access for all future endpoints.

---

**Finding 3.1b — Verbose Internal Error Message / Information Disclosure**

| Field | Details |
|---|---|
| **Severity** | Low |
| **OWASP Category** | API8:2023 — Security Misconfiguration |
| **Affected Endpoint** | `POST /identity/api/auth/login` |
| **Status** | Confirmed |

**Description:** Discovered incidentally while testing SQL injection: when a request fails server-side input validation (e.g. a password shorter than the minimum allowed length), the API returns a raw internal stack trace instead of a generic error message.

**Steps to Reproduce:**
1. Send `POST /identity/api/auth/login` with body: `{"email": "admin'--", "password": "x"}`
2. Observe the response.

**Response Received:**
```json
{
  "message": "Validation failed",
  "details": "org.springframework.validation.BeanPropertyBindingResult: 1 errors\nField error in object 'loginForm' on field 'password'; rejected value [x]; codes [Size.loginForm.password,...]; default message [size must be between 4 and 100]"
}
```

**Evidence:** ![evidence](./evidence/phase%203/error%20in%20sql%20injection.PNG)

**Impact:** Reveals internal implementation details — the backend framework (Spring), internal class/field names, and validation logic — helping an attacker fingerprint the technology stack and craft more targeted attacks.

**Remediation:** Catch validation exceptions server-side and return a generic, user-friendly error message (e.g. `"Invalid input"`) without exposing internal class names, stack traces, or framework-specific details.

---

## 2. Test 2 — Mass Assignment Vulnerability

**Objective:** Determine whether the signup endpoint accepts and processes unexpected fields such as `isAdmin` or `role`.

**Request:**
```http
POST /identity/api/auth/signup HTTP/1.1
Host: localhost:8888
Content-Type: application/json

{
  "name": "Test Hacker",
  "email": "hacker2@test.com",
  "number": "1234567890",
  "password": "Test1234!",
  "isAdmin": true,
  "role": "admin",
  "credit": 9999
}
```

**Follow-up check:**
```http
GET /identity/api/v2/user/dashboard HTTP/1.1
Authorization: Bearer <new_account_token>
```

**Result:** Two separate signup attempts were made with privileged fields injected: (1) `isAdmin: true, role: "admin"`, and (2) `available_credit: 99999, credit: 99999`. In both cases the account was created successfully, but upon login and checking `GET /identity/api/v2/user/dashboard`, the account showed `"role": "ROLE_USER"` (the standard default) and `"available_credit": 100.0` (the standard default) — both injected values were silently ignored.

**Finding 3.2 — Mass Assignment on Signup Endpoint**

| Field | Details |
|---|---|
| **Severity** | Informational (Not Vulnerable) |
| **OWASP Category** | API3:2023 — Broken Object Property Level Authorization |
| **Affected Endpoint** | `POST /identity/api/auth/signup` |
| **Status** | Tested, Not Vulnerable |

**Description:** The signup endpoint does not process additional unexpected fields beyond the documented schema. Both privilege-related (`isAdmin`, `role`) and financial (`credit`, `available_credit`) injected fields were ignored; the server assigned its own default values in both cases.

**Evidence:** ![evidence](./evidence/phase%203/mass_assignment_test.PNG)

**Impact:** None on this endpoint. The signup flow correctly enforces server-side control over privileged and financial fields.

**Remediation:** No action required for this endpoint. Recommend applying the same allowlist validation pattern to other write endpoints (profile update, order placement) as a precaution, since mass assignment risk can vary endpoint-by-endpoint.

---

## 3. Test 3 — NoSQL Injection

**Objective:** Determine whether query parameters or JSON bodies are vulnerable to NoSQL operator injection.

**Payloads Tested:**
```
GET /community/api/v2/community/posts/recent?limit[$gt]=0
```
```json
POST /community/api/v2/community/posts
{"title": {"$gt": ""}, "content": "test"}
```

**Baseline:** `GET /community/api/v2/community/posts/recent` (no injection) returned 4 posts (`"total": 4`) under normal conditions, establishing a comparison point.

**Result:** The `limit[$gt]=0` query parameter was ignored entirely — the response was identical to the baseline (4 posts, `total: 4`), indicating it was not interpreted as a MongoDB operator. The JSON body injection was rejected outright with a strict type error: `"cannot unmarshal object into Go struct field Post.title of type string"` — confirming the backend enforces strict typing on request fields (the `title` field must be a string), which structurally prevents operator objects from ever reaching the database query layer.

**Finding 3.3 — NoSQL Injection**

| Field | Details |
|---|---|
| **Severity** | Informational (Not Vulnerable) |
| **OWASP Category** | API8:2023 — Security Misconfiguration / Injection |
| **Status** | Tested, Not Vulnerable |

**Description:** Tested whether MongoDB query operators (`$gt`, object-based payloads) could be injected via URL query parameters and JSON request bodies on the community service (which uses MongoDB). The query-parameter injection was silently ignored, and the JSON-body injection was rejected by strict server-side type validation (strongly-typed Go structs) before it could reach the database layer.

**Evidence:** ![evidence](./evidence/phase%203/noSQL%20injection.PNG)

**Impact:** None. Strict type enforcement at the application layer (Go struct typing) is an effective mitigation against this class of injection on the fields tested.

**Remediation:** No action required for the fields tested. Continue enforcing strict typing on all request body fields across the application as a baseline defense against NoSQL injection.

---

## 4. Test 4 — Command Injection

**Objective:** Determine whether any field that may be processed server-side allows OS command injection.

**Background:** Video objects in crAPI include a `conversion_params` field (e.g. `"-v codec h264"`), suggesting this value is passed to a server-side video-conversion process (likely invoking a tool such as `ffmpeg` via a shell command).

**Payloads Tested (via `PUT /identity/api/v2/user/videos/{id}`):**
```json
{"video_name": "test", "conversion_params": "-v codec h264"}            // control, no injection
{"video_name": "test", "conversion_params": "-v codec h264; sleep 10"}  // injection attempt #1
{"video_name": "test", "conversion_params": "-v codec h264; sleep 5"}   // injection attempt #2
```

**Result:** The control request (no injected command) returned instantly. The `; sleep 10` request delayed the response by approximately 10 seconds. The `; sleep 5` request delayed the response by approximately 5 seconds. The response delay tracked precisely with the injected sleep duration across all three tests, which is very difficult to explain by network latency or coincidence — this is a time-based confirmation that the injected shell command is being executed server-side.

**Finding 3.4 — Command Injection via conversion_params Field**

| Field | Details |
|---|---|
| **Severity** | Critical |
| **OWASP Category** | API8:2023 — Security Misconfiguration / Injection |
| **Affected Endpoint** | `PUT /identity/api/v2/user/videos/{id}` |
| **Affected Field** | `conversion_params` |
| **Status** | Confirmed |

**Description:** The `conversion_params` field is passed to a server-side process without sanitization. Appending a shell command separator (`;`) followed by an arbitrary command (`sleep <n>`) causes the server's response time to delay by exactly `<n>` seconds, confirming the injected command is executed by the underlying operating system shell.

**Steps to Reproduce:**
1. Authenticate and obtain a valid JWT token.
2. Send `PUT /identity/api/v2/user/videos/{own_video_id}` with body `{"video_name": "test", "conversion_params": "-v codec h264; sleep 10"}`.
3. Observe ~10 second response delay.
4. Repeat with `"; sleep 5"` — delay scales to ~5 seconds, confirming direct control over execution time.
5. Control test with no injected command returns instantly, ruling out network/server lag as the cause.

**Evidence:** ![evidence](./evidence/phase%203/command%20injection.PNG)

**Impact:** Critical. An attacker with any valid, low-privilege account can execute arbitrary operating system commands on the server. Depending on the permissions of the process running the video conversion, this could lead to full server compromise, theft of other users' data or server files, lateral movement into internal systems, or complete denial of service. This is the most severe finding in this assessment.

**Remediation:**
1. Never pass user-controlled input directly into shell commands.
2. Use a strict allowlist of permitted `conversion_params` flag values rather than accepting free-form text.
3. If shelling out is unavoidable, use parameterized/argument-array subprocess APIs (passing arguments as a list rather than a concatenated string) so shell metacharacters (`;`, `|`, `&&`) cannot be interpreted.
4. Run the video-processing subprocess in a sandboxed, minimal-privilege environment to limit impact even if injection occurs.

---

## 5. Test 5 — Improper Input Sanitization (General)

**Objective:** Test boundary and malformed inputs across multiple endpoints (oversized strings, wrong data types, special characters).

**Payload tested** via `/workshop/api/shop/orders`
```
{
  "product_id": <id>, 
  "quantity": -5
}

```


**Finding 3.5 — Negative Quantity Accepted in Order Endpoint (Balance Manipulation)**

| Field | Details |
|---|---|
| **Severity** | High |
| **OWASP Category** | API6:2023 — Unrestricted Access to Sensitive Business Flows (also an input validation failure) |
| **Affected Endpoint** | `POST /workshop/api/shop/orders` |
| **Status** | Confirmed |

**Description:** The order placement endpoint does not validate that the `quantity` field is a positive integer. Submitting a negative quantity causes the server to calculate a negative total cost, which is then **added** to the user's balance instead of being rejected — effectively allowing users to generate free credit on demand.

**Steps to Reproduce:**
1. Create a fresh account; confirm starting `available_credit` = `100.0` via `GET /identity/api/v2/user/dashboard`.
2. Retrieve a valid `product_id` via `GET /workshop/api/shop/products`.
3. Send `POST /workshop/api/shop/orders` with body: `{"product_id": <id>, "quantity": -5}`.
4. Response: `{"id": 7, "message": "Order sent successfully.", "credit": 150.0}`.
5. Balance increased from `100.0` to `150.0` after a single negative-quantity order, confirming the exploit is reliably reproducible.

**Evidence:** ![evidence](./evidence/phase%203/improper%20input%20sanitization.PNG)

**Impact:** A malicious user can repeat this request indefinitely to generate unlimited account balance with no real payment, resulting in direct, measurable financial loss to the business. This is a high-severity, trivially exploitable flaw requiring only a valid account.

**Remediation:**
1. Enforce server-side validation that `quantity` is a positive integer within a range (e.g. 1–100); reject zero, negative, or non-integer values with `HTTP 400`.
2. Never trust client-supplied values to directly produce a credit/debit without independent server-side recalculation and sanity checks (e.g. assert `total_cost >= 0` before applying it to the balance).
3. Add automated regression tests covering boundary and negative-value cases for all financial calculation endpoints.

---

## 7. Findings Summary

| # | Finding | Severity | Status |
|---|---|---|---|
| 3.1 | SQL Injection — login | Informational | Tested, Not Vulnerable |
| 3.1b | Verbose error message / info disclosure | Low | Confirmed |
| 3.2 | Mass Assignment — signup | Informational | Tested, Not Vulnerable |
| 3.3 | NoSQL Injection — community endpoints | Informational | Tested, Not Vulnerable |
| 3.4 | Command Injection — conversion_params | **Critical** | **Confirmed** |
| 3.5 | Negative quantity / balance manipulation | **High** | **Confirmed** |

---

## 8. Evidence Index

All screenshots referenced above are stored in `evidence/phase3/`.
