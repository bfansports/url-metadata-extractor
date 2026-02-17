# url-metadata-extractor

## What This Is

Microservice API that extracts metadata from article URLs. Given a URL, it fetches the HTML and parses OpenGraph tags, Twitter Cards, meta tags, and item properties to return structured metadata (title, description, image, author, date, etc.). Used by the bFAN platform to generate rich previews of shared links in posts and chat.

## Tech Stack

- **Node.js 0.12** (Express.js framework) -- END OF LIFE, must upgrade
- **Express 4.12** (HTTP server)
- **Cheerio 0.19** (HTML parsing, jQuery-like API)
- **Superagent 1.1** (HTTP client for fetching URLs)
- **Bluebird 2.9** (Promises library)
- **Moment.js** (date parsing — in maintenance mode, consider replacing)
- **Mocha** (test framework)
- **Chai** (assertions)
- **Docker** (containerized deployment)

## Quick Start

**Run with Docker:**
```bash
docker run -d -p 3000:80 blikk/url-metadata-extractor
```

**Build and run locally:**
```bash
npm install
npm start
```

**Development mode (with nodemon):**
```bash
npm run dev
```

**Test:**
```bash
npm test
```

**Extract metadata from a URL:**
```bash
curl -XPOST http://localhost:3000/extract \
  --header "Content-Type:application/json" \
  --data '{"url": "https://example.com/article"}'
```

## Project Structure

```
app.js                       # Express app setup (body-parser, logging, error handlers)
bin/www                      # HTTP server entry point (port from PORT env var, default 3000)
routes/
└── index.js                 # API routes (POST /extract) — validates URL presence, calls extractor
lib/
└── extractor.js             # Metadata extraction logic — fetches URL, parses HTML with Cheerio
test/
├── apiSpec.js               # HTTP-level API tests (supertest)
├── extractorSpec.js          # Unit tests for extractor (nock + HTML fixtures)
└── htmls/                   # HTML fixture files (techcrunch, youtube, mashable, etc.)
Dockerfile                   # Container definition (FROM node:0.12 — OUTDATED)
circle.yml                   # CI/CD config (CircleCI — builds Docker, pushes to Docker Hub)
package.json                 # Node.js dependencies
.jshintrc                    # Linting config
```

## Dependencies

**Other bFAN repos:**
<!-- Ask: Which bFAN services call this API? sa_site_v2? Lambda APIs? -->
<!-- Ask: Is this used in mobile apps for link previews? -->
<!-- Ask: Integration with chat or social post features? -->

**External services:**
- Fetches arbitrary URLs from the public internet
- No caching layer — every request triggers a fresh HTTP fetch
- No proxy service — direct outbound connections
- No rate limiting on inbound or outbound requests

## API / Interface

**Endpoint:**
```
POST /extract
Content-Type: application/json
```

**Request Body:**
```json
{
  "url": "https://example.com/article"
}
```

**Response (200):**
```json
{
  "url": "https://example.com/article",
  "title": "Article Title",
  "description": "Article description",
  "image": "https://example.com/image.jpg",
  "contentType": "article",
  "site": "Example Site",
  "date": "2015-01-06T06:00:51+00:00",
  "metatags": {
    "og:title": "Article Title",
    "og:description": "Article description",
    "og:image": "https://example.com/image.jpg",
    "twitter:card": "summary_large_image"
  }
}
```

**Error Responses:**
- `400` — Missing `url` in request body (returns text: `"You must specify a URL."`)
- `404` — URL fetch failed (unreachable, DNS failure, malformed URL — returns empty body via `sendStatus`)
- `500` — Internal error (returns error object in dev, message string in prod)

**Extracted Metadata (priority order):**
- `title` — og:title > twitter:title > meta title
- `description` — og:description > twitter:description > meta description
- `image` — og:image > twitter:image
- `contentType` — og:type
- `author` — meta author > article:author > og:article:author
- `site` — og:site_name > og:site > twitter:site
- `date` — article:published_time > og:article:published_time > meta date > datepublished > pub_date > sailthru:date > displaydate > HTML time[itemprop=datePublished] > .date[data-time] > .article-timestamp-published
- `metatags` — raw dictionary of all meta tags

## Key Patterns

- **Metadata Prioritization**: Prefers OpenGraph tags, falls back to Twitter Cards, then generic meta tags, then HTML element scraping
- **HTML Parsing**: Uses Cheerio (server-side jQuery) to parse HTML — parsed 3 times per request (perf issue)
- **Asynchronous Fetching**: Superagent for HTTP requests, Bluebird promises (promisifyAll pattern)
- **Stateless API**: No database, no session, pure request-response
- **Docker Deployment**: Containerized for easy deployment
- **Superagent follows redirects** (up to 5 by default)

## Security Warnings

**SSRF RISK: This service fetches arbitrary user-supplied URLs.** This is the #1 security concern. Any code change must consider:

1. **No URL validation exists.** The service will fetch ANY URL — including `http://169.254.169.254/` (AWS metadata), `http://localhost:*`, private IPs (`10.*`, `172.16.*`, `192.168.*`), and `file://` URIs.
2. **No timeout configured.** `superagent.get(url)` has no timeout — a malicious slow server can hold connections open indefinitely.
3. **No response size limit.** A URL pointing to a multi-GB file will be loaded entirely into memory.
4. **No Content-Type check.** The service parses any response as HTML, including binary files.
5. **No rate limiting.** The service can be used as an SSRF amplification proxy.
6. **No authentication.** The endpoint is completely open.

**Before adding features or modifying the fetch logic, these issues MUST be addressed first.** See FINDINGS.md for full details and remediation steps.

**If you are working on this service:**
- Do NOT expose it to the public internet without URL validation and private IP blocking
- Do NOT trust any URL from the request body
- Do NOT increase fetch capabilities (e.g., adding JavaScript rendering) without fixing SSRF first
- Any URL validation must happen AFTER DNS resolution to prevent DNS rebinding attacks

## Environment

**Environment variables:**
- `PORT` — HTTP listen port (default: `3000`, set to `80` in Docker)
- `NODE_ENV` — `production` in Docker (affects error response verbosity)
- `LOG_NAME` — Logger name (set to `url-metadata-extractor` in Docker/CI)
- `.env` file loaded via `dotenv` at startup

## Deployment

**Docker:**
```bash
# Build
docker build -t url-metadata-extractor .

# Run
docker run -d -p 3000:80 url-metadata-extractor
```

**CI/CD (CircleCI — `circle.yml`):**
- Runs `mocha` tests
- Builds Docker image tagged with: `latest`, git SHA, version from package.json
- On merge to `master`: pushes to Docker Hub as `blikk/url-metadata-extractor`
- Docker Hub credentials: `DOCKER_USER=blikkdeploy`, password from `$DOCKER_PASSWORD` env var

**GitHub Workflow:**
- `.github/workflows/github-backup.yml` — S3 backup (triggers on `develop` branch — likely misconfigured, default branch is `master`)

## Testing

**Test Framework:**
- Mocha (test runner)
- Chai + chai-as-promised (assertions)
- Supertest + supertest-as-promised (HTTP assertions)
- Nock (HTTP mocking)

**Run Tests:**
```bash
npm test
```

**Test coverage:**
- 8 HTML fixture files (techcrunch, youtube, mashable, kickstarter, arstechnica, wired, cnet, dailymail)
- Tests cover: valid URL extraction, missing URL (400), inaccessible URL (404), malformed URL (404), date extraction from various site formats
- Tests do NOT cover: SSRF scenarios, timeout behavior, large responses, non-HTML content, redirect chains, authentication
- Test and dev packages (`chai`, `nock`, `supertest`) are incorrectly in `dependencies` instead of `devDependencies`

## Gotchas

- **SSRF is the primary risk** — see Security Warnings above
- **Node.js 0.12 is EOL** (since Dec 2016). Dockerfile uses `FROM node:0.12`. Must upgrade before any other work.
- **All dependencies are ~11 years old** with known CVEs. Full `npm audit` and upgrade required.
- **No NLP** — extracts only structured metadata from HTML tags, no natural language processing
- **Depends on site markup** — if a site lacks OpenGraph/Twitter Card tags, metadata will be incomplete
- **Timeout not configured** — slow sites hang requests indefinitely
- **HTML parsed 3x per request** — `cheerio.load()` called separately in `getMetatagsFromHtml`, `findCanonicalUrl`, and `findDate`
- **Dot replacement bug** — `name.replace('.', ':')` only replaces the first dot (not global regex)
- **Jade view engine configured but unused** — dead code in `app.js`, no views directory exists
- **`package.json` description references Python** — stale from earlier implementation
- **GitHub backup workflow triggers on `develop`** but default branch is `master` — backup never runs
- **Redirects are followed** but the destination URL is not re-validated (SSRF bypass vector)
- **Error format is inconsistent** — text for 400, empty for 404, object/string for 500
