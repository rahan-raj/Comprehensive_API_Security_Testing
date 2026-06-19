# Comprehensive_API_Security_Testing
API Security Assessment — OWASP crAPI | ArcByte Cybersecurity Internship 2026

**Intern** :- Rahan Raj K R  
**Task** :-  Perform Comprehensive API Security Testing and Assessment   
**Target Application** :- crAPI (OWASP Completely Ridiculous API) — local lab  

---

## Overview

This repository contains the deliverables for the Cybersecurity Internship Assessment on API Security Testing. The assessment follows a 6-phase methodology to identify, exploit (safely), and document API security vulnerabilities, then provide professional remediation recommendations.

**Important:** All testing was performed exclusively on an intentionally vulnerable, locally-hosted lab environment (crAPI). No production systems were tested.

---

## ⚡ Step 1 — Install These (One Time Only)

| Tool | Link | What It Does |
|---|---|---|
| Docker Desktop | [docker.com](https://docker.com/products/docker-desktop) | Runs the practice app |
| Postman | [postman.com](https://postman.com/downloads) | Sends API requests |
| Burp Suite Community | [portswigger.net](https://portswigger.net/burp/communitydownload) | Records all traffic |

---

## ⚡ Step 2 — Start the Lab (Every Time)

Open **Command Prompt** and run these 3 lines:

```cmd
cd crAPI\deploy\docker
docker compose up -d
```

Wait 2 minutes, then open **Chrome** and go to:

```
http://localhost:8888   ← the practice app
http://localhost:8025   ← fake email inbox (MailHog)
```

---

## ⚡ Step 3 — Create 2 Test Accounts

Go to `http://localhost:8888` → Register → make two accounts:

```
User A → usera@test.com / Test1234!
User B → userb@test.com / Test1234!
```

Verify both at `http://localhost:8025` (click the verification link in each email).

---

## ⚡ Step 4 — Get Your Token in Postman (Do This Every Session)

Open Postman → New Request → `POST` →
```
http://localhost:8888/identity/api/auth/login
```
Body → raw → JSON:
```json
{"email": "usera@test.com", "password": "Test1234!"}
```
Click **Send** → copy the `token` value → paste into Notepad labeled `USER A TOKEN`.

Repeat for User B → label it `USER B TOKEN`.

---

## ⚡ Step 5 — How to Use a Token in Every Request

Authorization tab → Bearer Token → paste the token → Send.

> ⚠️ Always verify which token is pasted by checking the first 3 letters match your Notepad.

---

| Phase | What You're Doing | Main Tool | Key Test |
|---|---|---|---|
| **1 — Discovery** | Find all API endpoints | Burp Suite + Postman | Import `http://localhost:8888/openapi.json` into Postman |
| **2 — Auth** | Test login & access controls | Postman + jwt.io | IDOR, rate limiting, JWT decode |
| **3 — Injection** | Try breaking inputs | Postman + curl | Command injection, SQL/NoSQL, mass assignment |
| **4 — Business Logic** | Game the app's rules | Burp Repeater + Postman | Race condition, order abuse, data exposure |
| **5 — Report** | Write it all up | Any text editor | Use the finding template below |

---

## 📋 Finding Template (Copy for Each Bug)

```
Finding #: [Name of vulnerability]
Severity: Critical / High / Medium / Low
OWASP: API1:2023 — [Category name]
Endpoint: GET/POST http://localhost:8888/...

Description:
[1–2 sentences — what is broken]

Steps to Reproduce:
1.
2.
3.

Evidence: [ the file path of evidence .png ]

Impact:
[What can an attacker do with this?]

Remediation:
[What should the developer fix?]
```

---

## 📋 Key Endpoints Quick Reference

```
LOGIN         POST   /identity/api/auth/login
SIGNUP        POST   /identity/api/auth/signup
DASHBOARD     GET    /identity/api/v2/user/dashboard
VIDEOS        GET    /identity/api/v2/user/videos
VEHICLES      GET    /identity/api/v2/vehicle/vehicles
COMMUNITY     GET    /community/api/v2/community/posts/recent
PRODUCTS      GET    /workshop/api/shop/products
ORDERS        POST   /workshop/api/shop/orders
COUPON        POST   /workshop/api/shop/apply_coupon
```

---

## 📋 Most Important Tests (In Order)

```
1. Decode your JWT          → paste token at jwt.io
2. IDOR on videos           → GET /user/videos/6 using User A's token
3. IDOR on vehicle          → GET /vehicle/{User_B_id}/location using User A's token
4. Rate limiting            → run curl loop 10 times with wrong password
5. Command injection        → PUT /user/videos/{id} with "conversion_params": "-v codec h264; sleep 10"
6. Negative quantity        → POST /shop/orders with "quantity": -5
7. Oversized quantity       → POST /shop/orders with "quantity": 999999
8. IDOR on orders           → GET /shop/orders/8 using User B's token
9. Pagination               → GET /posts/recent?limit=999999
10. Sensitive data          → look at every response for emails/phone numbers
```

---

## 📋 Evidence Screenshots — What to Capture

For every finding, take **3 screenshots**:

1. **Before** — dashboard showing starting balance / current state
2. **During** — Postman showing the request URL + body + response in one frame
3. **After** — dashboard showing the changed state (new balance, modified data, etc.)

---

## 📋 Severity Quick Guide

| Label | Meaning | Example |
|---|---|---|
| **Critical** | Any user can cause major damage | Command injection, unlimited order quantity |
| **High** | Logged-in user accesses others' private data | IDOR on videos, orders |
| **Medium** | Real issue but needs specific conditions | Missing rate limiting, data exposure |
| **Low** | Minor, best-practice issue | Verbose error messages, pagination |
| **Informational** | Tested, no vulnerability found | SQL injection, mass assignment (crAPI) |

---

## 📋 If Something Breaks

| Problem | Fix |
|---|---|
| `localhost:8888` won't load | Run `docker compose ps` — check all say "Up (healthy)" |
| Postman gives 401 | Your token expired — log in again and copy a fresh one |
| Got 502 / 504 error | Wait 30 seconds and retry — container had a hiccup |
| "Already claimed" on coupon | Token belongs to wrong account — decode at jwt.io to verify |
| Burp not capturing traffic | Check proxy is ON in browser (FoxyProxy) and Burp is open |
| Command not found in cmd | Run `cd crAPI\deploy\docker` first, then retry |

---
