# API Penetration Testing: Combined Checklist + Scenario List

*A unified, actionable framework for **design-time security**, **pen-testing**, and **real-world attack scenarios**  all in one place.*

---

## **How to Use This Guide**

| Phase | Purpose |
| --- | --- |
| **Design & Dev** | Use **Checklist** items as secure coding standards |
| **Pentesting** | Use **Scenarios** as test cases (with tools & PoCs) |
| **Reporting** | Map findings to **Checklist** for remediation |

---

# **1. API SECURITY CHECKLIST**

*(Secure-by-Design Countermeasures)*

### **Authentication**

- Avoid Basic Auth → Use **JWT**, **OAuth 2.0**, or **API Keys**
- Use established libraries (e.g., jsonwebtoken, oauthlib)
- Enforce **max retries + account lockout** on login
- Encrypt all sensitive data at rest & in transit

### **JWT Security**

- Use **strong, random secret** (≥256-bit for HS256)
- **Force algorithm** (don’t trust header) → HS256 or RS256
- Short **TTL** (≤15 min) + **Refresh Token Rotation**
- **No sensitive data** in payload
- Limit JWT size (<4KB recommended)

### **Access Control**

- **Rate limiting / throttling** (per IP, user, API key)
- Enforce **HTTPS** (TLS 1.2+, strong ciphers, HSTS)
- Disable directory listing
- Whitelist IPs for private APIs

### **Authorization**

- Validate redirect_uri server-side (OAuth)
- Use **authorization code flow**, not implicit
- Include state param to prevent CSRF
- Define & validate scope per client

### **Input Validation**

- Enforce correct **HTTP methods** → return 405 if invalid
- Validate Content-Type & Accept headers → return 406 if invalid
- Validate **all user inputs** (XSS, SQLi, RCE, etc.)
- **Never** put secrets in URLs → use Authorization header
- Use **API Gateway** for caching, rate limits, dynamic routing

### **Processing**

- Protect **all endpoints** with auth
- Use /me/ instead of numeric IDs
- Use **UUIDv4**, not auto-increment IDs
- Disable **XXE** & **entity expansion** in XML/YAML parsers
- Offload heavy tasks to **workers/queues**
- **Disable DEBUG mode in production**

### **Output**

- Send security headers:
    
    
    ```bash
    X-Content-Type-Options: nosniff
    X-Frame-Options: deny
    Content-Security-Policy: default-src 'none'
    Strict-Transport-Security: max-age=31536000; includeSubDomains
    Referrer-Policy: no-referrer
    Permissions-Policy: geolocation=(), microphone=()
    ```
    
- Remove fingerprint headers (Server, X-Powered-By)
- Return correct **HTTP status codes**
- **Never** return credentials or tokens

### **CI/CD & Monitoring**

- 90%+ unit/integration test coverage
- Mandatory code review (no self-approve)
- SAST/DAST + dependency scanning
- Rollback plan
- Centralized logging (no sensitive data)
- IDS/IPS + real-time alerts (Slack, Email, etc.)

---

# **2. API PENTESTING SCENARIOS & TEST CASES**

*(Attack Scenarios with Tools & PoCs)*

---

## **Recon & Fuzzing**

| Test | Tool | Notes |
| --- | --- | --- |
| Fuzz paths, params, headers | **ffuf**, **Burp Intruder** | Try /api/v1, /v2, /beta, /internal |
| Check HTTP vs HTTPS | **curl**, **nmap** | Is port 80 open? |
| Test allowed methods | OPTIONS /endpoint | Look for unexpected PUT, DELETE |
| Discover undocumented APIs | **Postman**, **Burp**, **OpenAPI leaks** | Check JS files, Swagger, Postman collections |

---

## **Authentication Attacks**

| Scenario | Test Case | Tool |
| --- | --- | --- |
| **User Enumeration** | Login with user vs nonexistent → timing/response diff | Burp Repeater |
| **Credential Stuffing** | Spray known password lists | Burp Intruder |
| **No Rate Limit on Login** | 1000+ attempts on one user | Burp Intruder + IP Rotator |
| **Weak Password Policy** | Try 123456, password | Manual |
| **Token Reuse / No Expiry** | Use old/expired JWT | JWT.io, Burp |
| **None Algorithm Attack** | {"alg":"none"} | **jwt_tool**, Burp |
| **Weak JWT Secret** | Brute-force HS256 key | **hashcat**, **john** |
| **JWT in URL (GET)** | Check logs or referrer leaks | Burp |
| **No Re-Auth for Sensitive Actions** | Change email without password | Manual |

---

## **Authorization & Access Control**

| Vulnerability | Test | Example |
| --- | --- | --- |
| **BOLA / IDOR** | Change user_id=123 → 124 | /users/124/profile |
| **BFLA** | User calls admin endpoint | /admin/users |
| **Mass Assignment** | Add "admin":true in registration | Param Miner, Burp |
| **Excessive Data Exposure** | GET /users/me returns SSN | Manual review |

---

## **Injection Attacks**

| Type | Test | Tool |
| --- | --- | --- |
| **SQLi / NoSQLi** | ' OR 1=1--, {"$ne": null} | **sqlmap**, Burp |
| **Command Injection** | ; ls, && whoami | Burp Repeater |
| **XXE** | Convert JSON → XML + <!ENTITY> | **Burp Content Type Converter** |
| **SSTI** | {{7*7}}, ${7*7} | Burp |
| **XSS via API** | <script>alert(1)</script> in response | Burp Collaborator |

---

## **Business Logic & Resource Abuse**

| Scenario | Test |
| --- | --- |
| **Rate Limit Bypass** | X-Forwarded-For: 127.0.0.1, X-Real-IP |
| **Pricing Manipulation** | price=-100, discount=999 |
| **Inventory Spoofing** | product_id=999999 |
| **Race Condition** | Parallel requests to /transfer |
| **File Upload RCE** | .php, .jsp in upload endpoint |
| **Unrestricted Resource Use** | Upload 10GB file → DoS |

---

## **Password Reset Flaws (HIGH-IMPACT)**

| Attack | PoC | Mitigation |
| --- | --- | --- |
| **Email Parameter Pollution** | email=victim@site.com&email=attacker@evil.com | Parse only first email |
| **Host Header Poisoning** | Host: attacker.com → reset link sent to attacker | Whitelist Host |
| **Reset Token in Referrer** | Click external link after reset | Use POST, not GET |
| **Token Prediction** | Analyze pattern (timestamp, user ID) | Use cryptographically random tokens |
| **Brute Force Reset Token** | Numeric token + IP rotation | Rate limit + strong tokens |
| **Use My Token on Your Email** | email=victim@site.com&token=MYTOKEN | Bind token to email/session |
| **Registration = Password Reset (Upsert)** | Register with existing email + new password | Reject duplicate emails |
| **Skip Old Password Check** | skipOldPwdCheck=true | Remove pre-auth paths |

---

## **JWT-Specific Test Cases**

| Test | Tool |
| --- | --- |
| alg: none | **jwt_tool** |
| Key brute-force (HS256) | **hashcat -m 16500** |
| No exp validation | Modify exp to future |
| Kid path traversal | kid: ../../dev/null |
| JKU/X5U SSRF | Host malicious JWKS |

---

## **GraphQL-Specific**

- Is **introspection enabled**? (query { __schema { types { name } }})
- Batch query DoS
- Apply all REST tests (IDOR, injection, etc.)

---

## **Cloud & Infrastructure**

- Enumerate S3/GCS buckets (cloud_enum)
- SSRF via API → internal metadata (169.254.169.254)
- Open redirects (?url=http://evil.com)

---

## **HTTP & Web Misconfigs**

| Test | Expected |
| --- | --- |
| HTTP Request Smuggling | **HTTP Request Smuggler** |
| CORS * with credentials | Origin: evil.com |
| Host Header Attack | Host: attacker.com |
| Missing Security Headers | **nuclei**, **curl** |

---

# **TOOLBOX**

| Tool | Use Case |
| --- | --- |
| **Burp Suite** | Proxy, Intruder, Repeater, Extensions |
| **Postman** | API exploration, collections |
| **OWASP ZAP** | Automated scanning |
| **ffuf / dirsearch** | Fuzzing |
| **sqlmap** | SQLi |
| **jwt_tool** | JWT attacks |
| **Param Miner** | Mass assignment |
| **Hashcat / John** | Brute-force secrets |
| **nuclei** | Header & config scanning |

---

# **REAL-WORLD BOUNTY EXAMPLES**

| Bug | Payout | Lesson |
| --- | --- | --- |
| IDOR in Healthcare API | $10,000 | Always validate object ownership |
| Rate Limit Bypass via X-Forwarded-For | $5,000 | Don’t trust client headers |
| Password Reset via Host Header | $7,500 | Whitelist allowed hosts |
| JWT none algorithm | $3,000 | Force algorithm server-side |

---

# **FINAL CHECKLIST SUMMARY (Printable)**

```
[ ] HTTPS + HSTS
[ ] No Basic Auth
[ ] JWT: strong secret, short TTL, no sensitive data
[ ] Rate limiting on all endpoints
[ ] Input validation + correct HTTP methods
[ ] All endpoints behind auth
[ ] UUIDs, not sequential IDs
[ ] Security headers (CSP, XFO, etc.)
[ ] No debug mode
[ ] SAST/DAST + dependency scanning
[ ] Centralized logging (no PII)
```

---

**Pro Tip**:

> "The best API pentest = 20% tools, 80% logic."
Focus on business logic, state transitions, and trust boundaries.
> 

---
