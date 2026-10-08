# CYS-319-MJEP Web API Security – Practical Tests

This README summarizes all practical tests contained in **Practical Practice Questions with Solutions** for the Web API Security practicals.

## Part A

1. **Postman CRUD on JSONPlaceholder** — GET, POST, PUT and DELETE requests; expected 200/201 status codes and REST concepts.
2. **Flask Student REST API – Five Operations** — GET all, GET by ID, POST, PUT and DELETE on the Student API.
3. **Nginx Rate Limiting with Client Script** — Nginx rate `1r/s`, burst `3`; rapid requests produce 200 for the initial allowed requests and 503 for blocked requests.
4. **Quota Gateway Monitoring** — Per-client quota of 5 requests/15 seconds with a 20-second cooldown; compares `client_A` and `client_B`.
5. **OWASP Juice Shop – Password Hash Leak** — Use Burp HTTP history to identify an API response exposing a password hash and explain why this is unsafe.
6. **SQL Injection Login Bypass** — Compare vulnerable and secure login APIs using `admin' --`; secure version uses parameterized queries/input validation.
7. **Basic Auth, API Key and JWT** — Test missing/incorrect and correct credentials for Basic Auth, API Key and JWT endpoints.

## Part B

8. **Duplicate-ID Bug in Student API** — Demonstrate duplicate IDs caused by `len(students)+1`, then fix using a counter or `max(id)+1`.
9. **Nginx Gateway – Allowed vs Blocked** — Show 200 and 503 entries in `access.log` and explain why blocked requests never reach Flask.
10. **Per-Client Quota and Cooldown** — Block `client_A` after exceeding five requests, verify `client_B` remains allowed, then verify automatic recovery after 20 seconds.
11. **OWASP Juice Shop – Broken Object Level Authorization** — Test `View Basket` or `Forged Feedback` by changing an object identifier/UserId in Burp Repeater.
12. **HTTP Parameter Pollution on `/transfer`** — Test `amount=100&amount=99999`; vulnerable validation and processing use different occurrences, while secure API rejects duplicates.
13. **JWT Login and Expiry** — Login to obtain a JWT valid for 30 seconds, inspect it, use it successfully, then verify `401 token expired` after expiry.
14. **HTTPS and Burp Response Tampering** — Use Burp as a trusted proxy to modify a response and explain what TLS protects and what it does not protect at the endpoint/application layer.

## Detailed Test Reference

### 1. Postman CRUD on JSONPlaceholder

**Endpoint:** `https://jsonplaceholder.typicode.com/posts`

| Test | Method | Endpoint | Expected Status |
|---|---|---|---|
| Get post | GET | `/posts/1` | 200 OK |
| Create post | POST | `/posts` | 201 Created |
| Update post | PUT | `/posts/1` | 200 OK |
| Delete post | DELETE | `/posts/1` | 200 OK |

POST/PUT body:

```json
{"title":"hi","body":"test","userId":1}
```

Take a screenshot of every request/response pair and record method, endpoint, response body and status code. Key concepts: `201 Created`, PUT replacement, PATCH partial update, DELETE idempotency.

### 2. Flask Student REST API

Start with:

```bash
python p2_server.py
```

Server: `http://127.0.0.1:5000`

Test `GET /students`, `GET /students/1`, invalid `GET /students/999`, `POST /students`, `PUT /students/1`, and `DELETE /students/1`. Expected results are 200 for successful reads/updates, 201 for creation, and 404 for nonexistent resources. Example POST body:

```json
{"name":"Amit","marks":80}
```

### 3. Nginx Rate Limiting

Start `p3_backend.py` on port 5000, Nginx on port 8080, then run `p3_client.py` (20 rapid requests). The supplied configuration uses:

```nginx
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=1r/s;
limit_req zone=mylimit burst=3;
limit_req_status 503;
proxy_pass http://127.0.0.1:5000;
```

The first approximately four requests are allowed and excess requests return `503 Service Temporarily Unavailable`. Verify `access.log`; Flask should only see requests forwarded by Nginx.

### 4. Quota Gateway Monitoring

Run `p4_gateway.py`, then `p4_simulate_clients.py` and `p4_monitor.py`. Configuration: 5 requests per 15 seconds per client and 20-second cooldown. `client_A` sends 8 requests: first 5 are 200 and later requests are 403. `client_B` sends 3 and remains at 200. The monitor reports counts, blocked client and remaining cooldown. After about 20 seconds, `client_A` becomes allowed again.

### 5. Juice Shop – Password Hash Leak

Run OWASP Juice Shop, route the browser through Burp, log in, and inspect Burp HTTP history for the user/API response containing a password hash. The source gives an example MD5 hash in a `password` field. Record the exact response and screenshot it. Verify the challenge on the Juice Shop Score Board. Reason: a hash can be cracked offline and exposing it unnecessarily increases its exposure.

### 6. SQL Injection Login Bypass

Run `p6_vuln.py` on port 5000 and `p6_secure.py` on port 5001. Submit username `admin' --` and password `x`. The vulnerable API returns `{"login":"SUCCESS"}` because the comment sequence removes the password check; the secure API returns `{"login":"REJECTED"}`. The supplied fix is parameterized queries, e.g. `cursor.execute("SELECT * FROM users WHERE username=? AND password=?", (u,p))`, plus input validation.

### 7. Basic Auth, API Key and JWT

Run `p7_auth_all.py` or `p7_all_client.py`. Test:

- Basic Auth: `GET /basic/students` — missing/wrong credentials 401, correct credentials 200.
- API Key: `GET /apikey/students` — missing/wrong `X-API-Key` 401/403, correct key 200.
- JWT: `POST /login`, then `GET /jwt/students` — missing/bad token 401, `Authorization: Bearer <token>` 200.

Record all six cases and screenshots.

### 8. Duplicate-ID Bug

Run `p2_server.py`. POST student A, POST student B, delete A, then POST student C. The buggy `new_id = len(students) + 1` can assign C the same ID as B. Fix with a persistent counter or `max([s['id'] for s in students], default=0) + 1`. Repeat and show unique IDs.

### 9. Nginx Allowed vs Blocked

Run `p3_backend.py` and Nginx with 1 request/sec plus burst 3. Slow requests to `http://127.0.0.1:8080/students` should return 200; rapid requests should produce 503. In `access.log`, identify one 200 and one 503. Nginx rejects excess traffic before `proxy_pass`, so Flask never receives blocked requests.

### 10. Per-Client Quota and Cooldown

Run `p4_gateway.py`. Send requests with `X-Client-Id: client_A` until the sixth request returns 403. Send fewer requests as `client_B`; it should continue receiving 200. Wait 20 seconds and send again as `client_A`; it should return 200 without restarting the server.

### 11. Broken Object Level Authorization

Using Burp with Juice Shop, complete either **View Basket** or **Forged Feedback**. For View Basket, change the basket ID in a captured request. For Forged Feedback, change the `UserId` in the request body. A vulnerable API returns another user's object or accepts the forged user ID. Show original request, modified request, response and Score Board result. Cause: authentication is checked but object ownership/authorization is not.

### 12. HTTP Parameter Pollution

Run `p6_transfer_vuln.py` on 5003 and `p6_transfer_secure.py` on 5004. Send:

```text
GET /transfer?amount=100&amount=99999
```

The vulnerable API validates 100 but transfers 99999 because validation and processing use different occurrences. The secure API rejects duplicate `amount` values, for example when `len(request.args.getlist('amount')) != 1`, returning 400. Show both responses.

### 13. JWT Login and Expiry

Run `p7_jwt.py` or `p7_jwt_client.py`. Login with the supplied test credentials to obtain a JWT with a 30-second expiry. Inspect header/payload/signature on jwt.io. Immediately call `GET /students` with `Authorization: Bearer <token>` and expect 200. Wait 31 seconds and repeat; expect `401 token expired`.

### 14. HTTPS and Burp Response Tampering

Run `p8_gen_cert.py` once and then `p8_https_app.py` on `https://127.0.0.1:5443`. Trust Burp's CA in the browser, proxy the HTTPS traffic, capture `/students/1`, send it to Repeater, and modify the response field from `"marks": 80` to `"marks": 100`. The browser displays the tampered value. Conclusion: TLS protects data in transit from outsiders, but a trusted interception proxy can decrypt and modify traffic; TLS does not replace authentication, authorization, input validation or server-side checks.

## Practical Evidence Checklist

- [ ] Server/terminal screenshot
- [ ] Request screenshot
- [ ] Response screenshot
- [ ] HTTP status code
- [ ] Request body and headers where relevant
- [ ] Relevant log output
- [ ] Before/after screenshots for vulnerable vs fixed tests
- [ ] Short explanation of the vulnerability
- [ ] Short explanation of the fix/mitigation

## Test Summary

| # | Practical | Main Concept |
|---:|---|---|
| 1 | Postman CRUD | REST CRUD, methods, status codes |
| 2 | Flask Student API | REST API operations |
| 3 | Nginx Rate Limiting | Rate limiting, reverse proxy |
| 4 | Quota Gateway | Per-client quotas, cooldown |
| 5 | Password Hash Leak | Sensitive data exposure |
| 6 | SQL Injection | Parameterized queries |
| 7 | Basic Auth/API Key/JWT | Authentication |
| 8 | Duplicate-ID Bug | Data integrity |
| 9 | Nginx Allowed/Blocked | Gateway protection |
| 10 | Client Quota/Cooldown | Throttling |
| 11 | BOLA | Authorization |
| 12 | Parameter Pollution | Input validation |
| 13 | JWT Expiry | Token lifecycle |
| 14 | HTTPS/Burp Tampering | TLS and application security |

> **Note:** Script names, ports, URLs, keys and field names follow the supplied practical document. Use the actual files/configuration supplied for your lab and capture your own screenshots and outputs.
