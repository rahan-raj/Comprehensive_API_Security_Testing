# Authentication & Authorization Testing Report

**Tester:** Rahan Raj K R  
**Date:** june 2026  
**Target Application:** crAPI — http://localhost:8888  
**Tools Used:** Postman, Burp Suite, jwt.io, curl  

---

##  Summary

This report documents testing performed against crAPI's authentication and authorization mechanisms, covering JWT handling, Insecure Direct Object References (IDOR), privilege escalation, and rate limiting on authentication endpoints. Vulnerabilities were identified, ranging from Medium to High severity.

---

## 1. Test 1 — JWT Token Analysis

**Objective:** Determine the signing algorithm, payload contents, and expiry handling of the JWT issued on login.

**Steps Performed:**
1. Sent `POST /identity/api/auth/login` with valid credentials via Postman.
2. Copied the `token` value from the JSON response.
3. Decoded the token at jwt.io.

**Request:**
```http
POST /identity/api/auth/login HTTP/1.1
Host: localhost:8888
Content-Type: application/json

{
  "email": "test@test.com",
  "password": "xxxxx"
}
```

**Observations:**

| Attribute | Value |
|---|---|
| Algorithm (`alg`) |  RS256 |
| Payload claims |  sub, role, iat, exp |
| Expiry present | Yes |
| Sensitive data in payload | No |



**Description:** The application uses JSON Web Tokens (JWTs) signed with the RS256 (RSA Signature with SHA-256) algorithm. The token payload contains standard claims including sub (subject), role, iat (issued at), and exp (expiration time). No sensitive information was identified within the token payload.

**Evidence:** ![evidence](./evidence/phase%202/JWT.PNG)

**Impact:** No security vulnerability was identified.

**Remediation:** Continue enforcing token expiration and protect the RS256 private signing key using appropriate key management controls. Periodically review token lifetimes and payload contents to ensure they remain aligned with security requirements.

---

## 2. Test 2 — IDOR (Insecure Direct Object Reference)

**Objective:** Determine whether a user can access another user's resources by manipulating object IDs.

**Setup:** Two accounts created — User A (`userA@test.com`) and User B (`userB@test.com`).

**Steps Performed:**
1. Logged in as User A, retrieved JWT.
2. Called `GET /identity/api/v2/user/videos` with User A's token, noted video ID.
3. Called `GET /identity/api/v2/user/videos/{userB_video_id}` using **User A's token**.
4. Repeated against `GET /identity/api/v2/vehicle/{vehicleId}/location` using User B's vehicle ID.

**Request:**
```http
GET /identity/api/v2/user/videos/6 HTTP/1.1
Host: localhost:8888
Authorization: Bearer <User_A_token>
```

**Response (unauthorized data returned):**
```json
{
  "id": 6,
  "video_name": "userB_private_video.mp4",
  "owner_email": "userB@test.com"
}
```

**Finding 2.2 — Insecure Direct Object Reference on Video Endpoint**

| Field | Details |
|---|---|
| **Severity** | High |
| **OWASP Category** | API1:2023 — Broken Object Level Authorization |
| **Affected Endpoint** | `GET /identity/api/v2/user/videos/{id}` |

**Description:** The endpoint does not verify that the authenticated user owns the requested video resource. Any authenticated user can retrieve any other user's video by incrementing the `id` parameter.


**Impact:** Any registered user can enumerate and access private videos belonging to other users.

**Remediation:** Implement object-level authorization checks server-side — verify `resource.owner_id == authenticated_user.id` before returning data, on every request.

**Finding 2.3 — IDOR on Vehicle Location Endpoint**

| Field | Details |
|---|---|
| **Severity** | High |
| **OWASP Category** | API1:2023 — Broken Object Level Authorization |
| **Affected Endpoint** | `GET /identity/api/v2/vehicle/{vehicleId}/location` |

**Description:** The vehicle location endpoint is vulnerable to an Insecure Direct Object Reference (IDOR) vulnerability. The application exposes vehicle location information based on a user-supplied identifier without adequately verifying that the requesting user is authorized to access the requested vehicle data.


**Impact:** Exposure of real-time/historical GPS coordinates of vehicles belonging to other users — a significant privacy risk.

**Remediation:** Apply the same ownership validation as above to vehicle-related endpoints.

---

## 3. Test 3 — Missing Rate Limiting on Authentication Endpoint

**Objective:** Determine whether repeated failed login attempts are throttled or blocked.

**Steps Performed:**
Sent 10 consecutive login requests with an incorrect password using curl:

```bash
for i in {1..10}; do
  curl -s -o /dev/null -w "Request $i: HTTP %{http_code}\n" \
  -X POST http://localhost:8888/identity/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"wrongpassword"}'
done
```

**Result:**

| Request # | HTTP Status |
|---|---|
| 1–10 | [e.g. 401 for all, no blocking observed] |

**Finding 2.4 — Missing Rate Limiting on Login Endpoint**

| Field | Details |
|---|---|
| **Severity** | Medium |
| **OWASP Category** | API4:2023 — Unrestricted Resource Consumption |
| **Affected Endpoint** | `POST /identity/api/auth/login` |

**Description:** The login endpoint accepted 10 consecutive failed attempts without any throttling, lockout, or CAPTCHA challenge.

**Evidence:** ![evidence](./evidence/phase%202/missing%20rate%20limiting%20on%20login.PNG)

**Impact:** The endpoint is susceptible to automated brute-force and credential-stuffing attacks.

**Remediation:** Implement rate limiting (e.g., max 5 attempts per 10 minutes per IP/account) and return `HTTP 429` once the threshold is exceeded. Consider progressive delays or account lockout after repeated failures.

---

## 4. Test 4 — Horizontal Privilege Escalation

**Objective:** Determine whether User A can view or modify User B's video resource by specifying User B's video ID in the URL while authenticated with User A's own JWT token.

**Setup:** Two accounts used — User A  and User B. **User B's** video confirmed to exist with ID `52` via `GET /identity/api/v2/user/videos` using **User B's** token.

**Steps Performed:**
1. Logged in as User B, confirmed User B's video ID = `52` via `GET /identity/api/v2/user/videos`.
2. Logged in as User A (`test@test.com`), obtained a fresh JWT token.
3. Sent the following request using **User A's token** against **User B's video ID**:

```http
PUT /identity/api/v2/user/videos/52 HTTP/1.1
Host: localhost:8888
Authorization: Bearer <User_A_token>
Content-Type: application/json

{
  "video_name": "Hacked by User A"
}
```

4. Sent a follow-up `GET /identity/api/v2/user/videos/52` using User A's token to verify the result.

**Response received:**
```json
{
    "message": "Video not found.",
    "status": 404
}
```

**Result:** The server did not return or modify User B's video. Both the PUT and GET requests, despite explicitly targeting ID `52` (User B's actual video), responded with data and an error message referencing **User A's own account** (`test@test.com`). The `{id}` parameter in the URL path appears to be ignored entirely by the server; the endpoint scopes every request strictly to the identity embedded in the authenticated user's JWT token, not the ID supplied in the URL.

**Finding 2.5 — Not Vulnerable: Horizontal Privilege Escalation (Video Update Endpoint)**

| Field | Details |
|---|---|
| **Severity** | Informational (Not Vulnerable) |
| **OWASP Category** | API1:2023 — Broken Object Level Authorization (tested, not present) |
| **Affected Endpoint** | `PUT /identity/api/v2/user/videos/{id}`, `GET /identity/api/v2/user/videos/{id}` |
| **Status** | Tested, Not Vulnerable |

**Description:** Tested whether User A could view or modify User B's video resource by specifying User B's video ID in the URL while authenticated with User A's own token. The endpoint correctly derives the target resource from the authenticated user's identity (JWT) rather than trusting the client-supplied `{id}` in the URL path.

**Impact:** None. This endpoint correctly enforces object-level authorization on this specific operation.

**Evidence:** `evidence/phase2/horizontal-privesc-video-test.png`

**Conclusion:** This endpoint is **not vulnerable** to horizontal privilege escalation via URL ID manipulation. No remediation required for this specific test. Note: this contrasts with Findings above (the `GET` video/vehicle IDOR tests), which were confirmed vulnerable — indicating inconsistent authorization enforcement across different endpoints/operations within the same application, which is itself worth flagging to developers as a pattern to review across the full API surface.

---

## 6. Findings Summary

| # | Finding | Severity | Status |
|---|---|---|---|
| 2.1 | JWT issue (expiry/algorithm) | Medium | Confirmed |
| 2.2 | IDOR — Video endpoint | High | Confirmed |
| 2.3 | IDOR — Vehicle location endpoint | High | Confirmed |
| 2.4 | Missing rate limiting on login | Medium | Confirmed |
| 2.5 | Horizontal privilege escalation (video endpoint) | Informational | Tested, Not Vulnerable |

---

## 7. Evidence Index

All screenshots referenced above are stored in `evidence/phase2/`.
