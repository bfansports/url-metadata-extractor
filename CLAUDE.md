# url-metadata-extractor

## What This Is

Microservice API that extracts metadata from article URLs. Given a URL, it fetches the HTML and parses OpenGraph tags, Twitter Cards, meta tags, and item properties to return structured metadata (title, description, image, author, date, etc.). Used by the bFAN platform to generate rich previews of shared links in posts and chat.

## Tech Stack

- **Node.js** (Express.js framework)
- **Express 4.12** (HTTP server)
- **Cheerio 0.19** (HTML parsing, jQuery-like API)
- **Superagent 1.1** (HTTP client for fetching URLs)
- **Bluebird 2.9** (Promises library)
- **Moment.js** (date parsing)
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
app.js                       # Express app setup
bin/www                      # HTTP server entry point
routes/
└── index.js                 # API routes (POST /extract)
lib/
└── extractor.js             # Metadata extraction logic
test/                        # Mocha tests
Dockerfile                   # Container definition
circle.yml                   # CI/CD config (CircleCI)
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
<!-- Ask: Any rate limiting or caching? -->
<!-- Ask: Does it use a proxy service? -->

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

**Response:**
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
    "twitter:card": "summary_large_image",
    ...
  }
}
```

**Extracted Metadata:**
- `title` — article/page title (from og:title, twitter:title, or <title>)
- `description` — summary (from og:description, twitter:description, or meta description)
- `image` — featured image URL (from og:image, twitter:image, etc.)
- `contentType` — content type (article, website, video, etc.)
- `site` — site name (from og:site_name)
- `date` — publication date (from article:published_time, sailthru:date, etc.)
- `metatags` — raw metatag dictionary

**Error Handling:**
<!-- Ask: What HTTP status codes are returned on error? -->
<!-- Ask: How are malformed URLs handled? -->
<!-- Ask: Timeout behavior for slow sites? -->

## Key Patterns

- **Metadata Prioritization**: Prefers OpenGraph tags, falls back to Twitter Cards, then generic meta tags
- **HTML Parsing**: Uses Cheerio (server-side jQuery) to parse HTML
- **Asynchronous Fetching**: Superagent for HTTP requests, Bluebird promises
- **Stateless API**: No database, no session, pure request-response
- **Docker Deployment**: Containerized for easy deployment

<!-- Ask: Caching strategy? Redis? In-memory? -->
<!-- Ask: How are 404s and timeouts handled? -->
<!-- Ask: Does it follow redirects? -->

## Environment

**Required environment variables:**
<!-- Ask: Any env vars needed? API keys? Timeout settings? -->

**Configuration:**
- `.env` file (via dotenv package)
- Environment-specific settings
<!-- Ask: What's in the .env file? -->

**Deployment:**
- Docker container on port 80 (inside container)
- Exposed as port 3000 (example, configurable)

## Deployment

**Docker:**
```bash
# Build
docker build -t url-metadata-extractor .

# Run
docker run -d -p 3000:80 url-metadata-extractor
```

**CircleCI:**
- `circle.yml` defines CI/CD pipeline
<!-- Ask: What does CircleCI do? Build Docker image? Deploy to ECS? -->
<!-- Ask: What environments exist? dev/qa/prod? -->
<!-- Ask: Deployment trigger? Merge to master? -->

**GitHub Workflow:**
- `.github/workflows/github-backup.yml` — repository backup automation

## Testing

**Test Framework:**
- Mocha (test runner)
- Chai (assertions)
- Supertest (HTTP assertions)
- Nock (HTTP mocking)

**Run Tests:**
```bash
npm test
```

**Test Coverage:**
<!-- Ask: Coverage percentage? -->
<!-- Ask: Integration tests vs unit tests? -->
<!-- Ask: How are external URLs mocked in tests? -->

## Gotchas

- **No NLP**: Extracts only structured metadata from HTML tags — no natural language processing
- **Depends on Site Markup**: If a site doesn't have OpenGraph/Twitter Card tags, metadata will be incomplete
- **External URL Fetching**: Service must be able to reach arbitrary URLs — firewall rules matter
- **Timeout Risks**: Slow sites can hang requests — ensure timeout is configured
- **HTML Parsing**: Malformed HTML may cause parsing errors
- **Rate Limiting**: Fetching many URLs quickly may trigger rate limits on target sites
- **Redirects**: Must handle HTTP redirects correctly
- **HTTPS**: Must support HTTPS URLs

<!-- Ask: What's the default timeout for fetching URLs? -->
<!-- Ask: How are blocked or private URLs handled? -->
<!-- Ask: Error rate monitoring? -->
<!-- Ask: Known sites that don't work well? -->