# Security Audit Findings — url-metadata-extractor

**Date:** 2026-02-17  
**Auditor:** AI (Frontend Developer agent, Claude Opus 4.6)  
**Scope:** Full codebase review — SSRF, input validation, timeout handling, resource exhaustion, caching  
**Repo:** bfansports/url-metadata-extractor  
**Branch:** master  
**Commit:** 3080be3  

---

## Critical

### C-1: Server-Side Request Forgery (SSRF) — No URL validation

**File:** `lib/extractor.js:15`, `routes/index.js:11-13`

**Description:** The `/extract` endpoint accepts any URL from the request body and passes it directly to `superagent.get(url)` with zero validation. An attacker can request internal network resources, cloud metadata endpoints, or localhost services.

**Attack vectors:**
- `{"url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"}` — AWS instance metadata (IAM credentials)
- `{"url": "http://localhost:6379/"}` — Redis, databases, or other internal services
- `{"url": "http://10.0.0.1/admin"}` — internal network scanning
- `{"url": "file:///etc/passwd"}` — local file read (superagent may support file:// protocol)
- `{"url": "http://[::1]:80/"}` — IPv6 localhost bypass

**Impact:** Full SSRF. If this service runs on AWS (which it does — Docker/ECS), an attacker can steal IAM credentials from the instance metadata service, access internal VPC resources, and pivot to other services.

**Remediation:**
1. **Whitelist protocols** — only allow `http://` and `https://`
2. **Block private/reserved IPs** — reject `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`, `::1`, `fd00::/8`
3. **DNS resolution check** — resolve the hostname first, verify the resolved IP is not private, THEN fetch. This prevents DNS rebinding attacks where `evil.com` resolves to `127.0.0.1`.
4. **Block cloud metadata** — explicitly block `169.254.169.254` and the newer IMDSv2 token endpoint
5. **Use IMDSv2** — require tokens for instance metadata on the EC2/ECS host as defense-in-depth

**Severity:** CRITICAL — this is the single most important finding. The service is an open SSRF proxy.

---

### C-2: No request timeout on outbound HTTP fetches

**File:** `lib/extractor.js:15`

**Description:** `request.get(url).end(callback)` has no timeout configured. Superagent's default is no timeout — it will wait indefinitely for a response.

**Impact:**
- An attacker sends a URL pointing to a server that holds the connection open (slowloris-style). The Node.js process accumulates hanging connections until it runs out of file descriptors or memory.
- A few hundred concurrent slow requests can DoS the entire service.
- Even without malicious intent, a slow third-party site can cascade failures.

**Remediation:**
```javascript
request.get(url)
  .timeout({ response: 5000, deadline: 10000 })
  .end(callback);
```
Set both a response timeout (time to first byte) and a deadline timeout (total time).

**Severity:** CRITICAL — trivially exploitable for denial of service.

---

## High

### H-1: No response body size limit

**File:** `lib/extractor.js:15-18`

**Description:** The service fetches the entire response body into memory (`res.text`) with no size cap. An attacker can point the service at a URL serving a multi-gigabyte file.

**Impact:** Memory exhaustion, process crash, denial of service. Since `res.text` is stored as a string, a 1GB response will consume ~2GB of V8 heap (UTF-16 encoding).

**Remediation:**
- Use superagent's `.maxResponseSize()` or manually stream and truncate
- Recommended limit: 5MB (generous for HTML with metadata)
- Reject non-HTML content types before reading the body

---

### H-2: No Content-Type validation on fetched responses

**File:** `lib/extractor.js:16-18`

**Description:** The service passes `res.text` to Cheerio regardless of the response's Content-Type. It will attempt to parse binary files (images, PDFs, executables) as HTML.

**Impact:**
- Memory waste parsing non-HTML content
- Potential crashes from Cheerio parsing binary data
- Information leakage if binary content matches meta tag patterns

**Remediation:**
Check `res.headers['content-type']` — only proceed if it starts with `text/html` or `application/xhtml+xml`.

---

### H-3: Critically outdated Node.js runtime

**File:** `Dockerfile:1`

**Description:** `FROM node:0.12` — Node.js 0.12 reached end-of-life in **December 2016**. It has 10+ years of unpatched security vulnerabilities including OpenSSL flaws, HTTP parsing bugs, and V8 engine exploits.

**Impact:** The entire runtime is a known-vulnerable attack surface. No amount of application-level fixes can compensate for a compromised runtime.

**Remediation:** Upgrade to Node.js 20 LTS (or 22 LTS when available). This will require updating the codebase from CommonJS/ES5 patterns but is non-negotiable for production use.

---

### H-4: All dependencies severely outdated with known CVEs

**File:** `package.json`

**Dependencies and approximate age (as of 2026):**
- `express@^4.12.3` — ~11 years old (current is 4.21+). Multiple prototype pollution and DoS CVEs.
- `superagent@^1.1.0` — ~11 years old (current is 9.x). Known redirect and header injection issues.
- `cheerio@^0.19.0` — ~11 years old (current is 1.0+). Known prototype pollution via parse5.
- `body-parser@^1.12.2` — ~11 years old. Known DoS via large payloads.
- `bluebird@^2.9.14` — ~11 years old. Bluebird 3.x had security fixes.
- `moment@^2.9.0` — ~11 years old. Moment.js itself is in maintenance mode. Known ReDoS in date parsing.
- `dotenv@^1.0.0` — ~11 years old (current is 16.x).
- `underscore@^1.8.2` — ~11 years old. Known arbitrary code execution CVE-2021-23358.

**Impact:** Dozens of known CVEs across the dependency tree. Any automated scanner will flag this service.

**Remediation:** Full dependency audit with `npm audit`. Upgrade all dependencies. Consider replacing moment.js with `date-fns` or native `Intl.DateTimeFormat`, and underscore with native ES6+ methods.

---

### H-5: No rate limiting

**File:** `app.js`, `routes/index.js`

**Description:** No rate limiting middleware. Any client can send unlimited requests.

**Impact:**
- DoS the service directly
- Use the service as an SSRF amplification proxy to scan internal networks at scale
- Trigger rate limiting on third-party sites (bFAN's IP gets blocked)

**Remediation:** Add `express-rate-limit` middleware. Suggested: 60 requests/minute per IP for the `/extract` endpoint.

---

## Medium

### M-1: No authentication or authorization

**File:** `app.js`, `routes/index.js`

**Description:** The API is completely open. No API key, JWT, or any form of authentication.

**Impact:** Anyone who discovers the endpoint can use it — for legitimate metadata extraction or as an SSRF proxy. Combined with C-1 (SSRF), this is especially dangerous.

**Remediation:** At minimum, require an API key via header (`X-API-Key`) or use network-level restrictions (security groups, ALB rules). For internal-only use, ensure the service is not exposed to the public internet.

---

### M-2: Error object leaks stack traces in development mode

**File:** `app.js:37-43`

**Description:** In development mode, the entire error object (including stack trace) is sent to the client via `res.send(err)`. The `NODE_ENV` check determines this behavior.

**Impact:** Information disclosure — stack traces reveal file paths, dependency versions, and internal logic. If `NODE_ENV` is not explicitly set to `production`, traces leak.

**Remediation:** Never send raw error objects. In development, log the full error server-side but return a sanitized message to the client.

---

### M-3: No request body size limit

**File:** `app.js:17-18`

**Description:** `bodyParser.json()` and `bodyParser.urlencoded()` are used without a `limit` option. The default is 100KB, which is reasonable for this use case, but it should be explicitly set.

**Impact:** While the default is acceptable, relying on defaults is fragile — a dependency update could change the default.

**Remediation:**
```javascript
app.use(bodyParser.json({ limit: '10kb' }));
app.use(bodyParser.urlencoded({ extended: false, limit: '10kb' }));
```
The request body only needs a URL string — 10KB is generous.

---

### M-4: No caching layer

**File:** entire codebase

**Description:** Every request for the same URL triggers a fresh HTTP fetch and parse. No in-memory cache, no Redis, no CDN caching headers.

**Impact:**
- Unnecessary load on target websites (could get bFAN's IP blocked)
- Unnecessary latency for repeated URLs
- Amplifies DoS potential — each request does real I/O work

**Remediation:** Add a cache with TTL. Options:
- In-memory LRU cache (`lru-cache` package) — simplest, works for single-instance
- Redis — works for multi-instance deployments
- HTTP cache headers in the response (e.g., `Cache-Control: max-age=3600`)

---

### M-5: HTML parsed multiple times per request

**File:** `lib/extractor.js:135-145`

**Description:** `getMetadataFromHtml` calls `getMetatagsFromHtml`, `findCanonicalUrl`, and `findDate` — each calls `cheerio.load(html)` independently, parsing the full HTML 3 times per request.

**Impact:** 3x CPU and memory usage per request. For large pages (the dailymail fixture is 240KB), this is wasteful.

**Remediation:** Call `cheerio.load(html)` once and pass the `$` object to all extraction methods.

---

### M-6: Superagent follows redirects without limit

**File:** `lib/extractor.js:15`

**Description:** Superagent's default behavior follows redirects (up to 5 by default). Combined with no SSRF protection, an attacker can set up a redirect chain: `https://evil.com` -> `http://169.254.169.254/...`

**Impact:** SSRF bypass via open redirects. Even if URL validation is added, the final resolved URL after redirects is not checked.

**Remediation:** Either disable redirects (`.redirects(0)`) or re-validate the URL after each redirect. Limit redirects to 3.

---

### M-7: dot replacement in metatag names is not global

**File:** `lib/extractor.js:36`

**Description:** `name.replace('.', ':')` only replaces the **first** dot. `String.replace` with a string argument is not global in JavaScript. For a metatag name like `og.image.url`, only the first dot becomes a colon: `og:image.url`.

**Impact:** Subtle metadata extraction bug. Some metatag names with multiple dots will be stored inconsistently.

**Remediation:** Use a regex: `name.replace(/\./g, ':').toLowerCase()`

---

## Low

### L-1: Test and dev dependencies in production `dependencies`

**File:** `package.json`

**Description:** `chai`, `chai-as-promised`, `nock`, `supertest`, `supertest-as-promised` are listed under `dependencies` instead of `devDependencies`. These are test-only packages.

**Impact:** Larger Docker image, increased attack surface (test packages installed in production). With `npm install --production`, they would not be excluded.

**Remediation:** Move test packages to `devDependencies`. Update Dockerfile to use `npm install --production`.

---

### L-2: Jade view engine configured but never used

**File:** `app.js:12-13`

**Description:** `app.set('view engine', 'jade')` is configured but there is no `views/` directory and no routes render templates. Jade is also deprecated (renamed to Pug) and has known vulnerabilities.

**Impact:** Confusing dead code. No runtime impact since no views are rendered, but Jade/Pug would be installed as a transitive dependency if present in package.json.

**Remediation:** Remove the view engine configuration lines from `app.js`.

---

### L-3: Error responses are inconsistent

**File:** `routes/index.js:22`, `app.js:39-51`

**Description:**
- Missing URL returns 400 with text: `"You must specify a URL."`
- Failed extraction returns 404 with no body (just `sendStatus(404)`)
- App-level 404 returns 404 with error message
- App-level 500 returns error object (dev) or error message string (prod)

**Impact:** Inconsistent error format makes client-side error handling fragile. Callers cannot reliably parse errors.

**Remediation:** Standardize on JSON error responses:
```json
{"error": "message", "status": 400}
```

---

### L-4: `var` used throughout instead of `const`/`let`

**File:** All source files

**Description:** The codebase uses `var` exclusively. Variable re-declaration on line 59 of `extractor.js` (`var dateClass` declared twice) demonstrates the hoisting risks.

**Impact:** Function-scoped variables increase risk of subtle bugs. The double `var dateClass` declaration is a concrete example.

**Remediation:** Part of the Node.js upgrade — switch to `const`/`let` with ES6+ syntax.

---

### L-5: CircleCI config exposes Docker credentials pattern

**File:** `circle.yml:5-6`

**Description:** `DOCKER_EMAIL` and `DOCKER_USER` are hardcoded in the CI config. The password comes from an environment variable, which is correct, but the email and username are visible.

**Impact:** Low — these are deployment credentials, not secrets. But the `docker login` command pattern is deprecated (password via CLI argument appears in process listing).

**Remediation:** Use `docker login --password-stdin` or migrate to GitHub Actions with OIDC-based authentication.

---

### L-6: GitHub backup workflow triggers on wrong branch

**File:** `.github/workflows/github-backup.yml:4`

**Description:** The backup workflow triggers on push to `develop`, but the repo's default branch is `master`. This workflow never runs.

**Impact:** Repository is not being backed up to S3 as intended.

**Remediation:** Change trigger to `master` or add both branches.

---

### L-7: `package.json` description references Python

**File:** `package.json:4`

**Description:** `"description": "API that extracts metadata from a given article URL. Uses the Python newspaper library."` — this is a Node.js project. The Python reference is stale from an earlier implementation.

**Impact:** Confusing for developers and automated tools that read package metadata.

**Remediation:** Update to: `"API that extracts metadata from a given article URL using OpenGraph, Twitter Cards, and meta tags."`

---

## Agent Skill Improvements

### S-1: CLAUDE.md needs security warnings for SSRF-sensitive codebase

The existing CLAUDE.md is a good bootstrap but lacks security-specific guidance. Any AI agent working on this codebase needs to know about the SSRF risk before making changes. Updated CLAUDE.md is included in this PR.

### S-2: Add security test cases to testing section

The test suite only covers happy-path metadata extraction. CLAUDE.md should document the gap and suggest security test scenarios for any future development.

### S-3: Document the actual error handling behavior

Many `<!-- Ask: ... -->` placeholders in CLAUDE.md can now be answered from code analysis:
- HTTP 400 for missing URL
- HTTP 404 for failed fetches
- No timeout configured (critical gap)
- Superagent follows up to 5 redirects by default

---

## Positive Observations

1. **Input validation exists for missing URL** — `routes/index.js:11` correctly returns 400 when `req.body.url` is absent. This is the right pattern; it just needs to be extended to validate the URL format and destination.

2. **Error logging is present** — Both `routes/index.js` and `app.js` log errors with structured context (`{err, url}`). This is good operational practice.

3. **Tests use HTTP mocking (Nock)** — The test suite properly mocks external HTTP calls with `nock`, avoiding flaky tests that depend on live websites. The HTML fixtures are realistic.

4. **Stateless design** — No database, no sessions, no persistent state. This makes the service easy to scale horizontally and reason about.

5. **Separation of concerns** — Clean split between HTTP routing (`routes/index.js`) and extraction logic (`lib/extractor.js`). The extractor is independently testable.

6. **Strict mode enabled** — `'use strict'` is used in all source files, preventing common JavaScript pitfalls.

7. **Docker containerization** — Proper Dockerfile with production `NODE_ENV`, even if the base image is outdated.

---

## Summary

| Severity | Count | Key Theme |
|----------|-------|-----------|
| Critical | 2 | SSRF (open proxy), No timeout (DoS) |
| High | 5 | No response size limit, no content-type check, ancient Node.js/deps, no rate limit |
| Medium | 7 | No auth, info disclosure, no caching, redundant parsing, redirect bypass, regex bug |
| Low | 7 | Dev deps in prod, dead code, inconsistent errors, stale metadata |

**Recommendation:** This service should NOT be exposed to the public internet in its current state. The SSRF vulnerability (C-1) is trivially exploitable and can compromise AWS credentials. If the service must remain running, prioritize:
1. URL validation with private IP blocking (C-1)
2. Request timeout (C-2)
3. Response size limit (H-1)
4. Rate limiting (H-5)
5. Node.js and dependency upgrade (H-3, H-4)
