# CoeurStrike.AI - Web Classification API

*WHOISXMLAPI — Rapid-Development Group*

CoeurStrike.AI classifies domain names against the IAB Tech Lab Content Taxonomy 3.0. One HTTP call returns the primary, secondary, and (when applicable) tertiary IAB categories for a given fully-qualified domain name. Classifications are cached on our side for 30 days, so repeat lookups for the same domain are near-instant.

Typical use cases: contextual advertising, content filtering, brand-safety signalling, audience segmentation, and analytics enrichment.

---

## Getting Started

### Base URL

```
https://webcat.coeurstrike.com
```

### Authentication

All programmatic requests require an API key in the `X-API-Key` header:

```
X-API-Key: wl8_your_api_key_here
```

### Requesting an API Key

Visit the registration page:

```
https://webcat.coeurstrike.com/register
```

Provide your company name and email. A key is issued immediately, but the account is in a pending state until an administrator approves it. Once approved you can start making API calls with the key; queries against a pending account return `403`.

---

## API Reference

### Classify a Domain

```
GET /api/v1/classify/{domain}
```

**Query parameters**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `force` | bool | `false` | If `true`, bypasses the 30-day cache and re-classifies the domain via OpenAI. Useful when a prior classification returned `N/A` for all three slots and you want to retry with a fresh attempt. |

**Example: cached lookup**

```bash
curl -X GET "https://webcat.coeurstrike.com/api/v1/classify/cnn.com" \
  -H "X-API-Key: wl8_your_api_key_here"
```

**Success response (200 OK)**

```json
{
  "fqdn": "cnn.com",
  "auditdate": "2026-04-21 14:30:00",
  "iab_category1": "IAB12 - News",
  "iab_category2": "IAB12-1 - International News",
  "iab_category3": "IAB11 - Law, Government & Politics",
  "cached": true
}
```

**Example: force re-classification**

```bash
curl -X GET "https://webcat.coeurstrike.com/api/v1/classify/obscure-site.example?force=1" \
  -H "X-API-Key: wl8_your_api_key_here"
```

The `force=1` parameter re-invokes the AI classifier and returns a fresh result. Counts toward your daily quota the same as a normal request.

**Response timing**

- **Cache hit** (domain classified within the last 30 days) — typically <100ms
- **Cache miss or `force=1`** — typically 1–3 seconds (includes the AI classification round-trip)

The `cached` boolean in the response tells you which path served your request.

---

### Check API Usage

```
GET /api/v1/usage
```

Returns your current daily quota usage and account details.

**Example**

```bash
curl -X GET "https://webcat.coeurstrike.com/api/v1/usage" \
  -H "X-API-Key: wl8_your_api_key_here"
```

**Response**

```json
{
  "customer_id": "cust_abc123def456",
  "company_name": "Acme Corp",
  "queries_per_day": 100,
  "queries_used_today": 15,
  "queries_remaining": 85,
  "expiration_date": "2026-05-21"
}
```

The `queries_used_today` counter resets at 00:00 UTC every day.

---

### Health Check

```
GET /api/v1/health
```

Returns service status and the running API version. No authentication required.

```json
{
  "status": "ok",
  "version": "2.4",
  "openai_model": "gpt-4.1-mini",
  "max_concurrent": 10
}
```

---

## Response Fields (classify endpoint)

| Field | Type | Description |
|---|---|---|
| `fqdn` | string | The normalized domain (lowercased, scheme/path/`www.` stripped) |
| `auditdate` | string | UTC timestamp of when this classification was produced (`YYYY-MM-DD HH:MM:SS`) |
| `iab_category1` | string | Primary IAB 3.0 category, or `"N/A"` if undeterminable |
| `iab_category2` | string | Secondary IAB 3.0 category, or `"N/A"` if undeterminable |
| `iab_category3` | string | Tertiary IAB 3.0 category, populated when a third distinct category genuinely applies, otherwise `"N/A"` |
| `cached` | bool | `true` if served from our cache, `false` if freshly classified on this request |

### About `"N/A"` responses

Every category slot is either a valid IAB 3.0 category string (e.g. `"IAB12 - News"`) or the literal string `"N/A"`. `N/A` indicates the classifier couldn't determine a category with reasonable confidence — commonly seen for brand-new domains, domains the underlying model isn't familiar with, or very niche/regional properties.

**Possible response shapes:**

```json
// Three categories apply
{"iab_category1": "IAB12 - News",
 "iab_category2": "IAB12-1 - International News",
 "iab_category3": "IAB11 - Law, Government & Politics"}

// Only two categories apply
{"iab_category1": "IAB19 - Technology & Computing",
 "iab_category2": "IAB19-11 - Internet Technology",
 "iab_category3": "N/A"}

// Only primary determinable
{"iab_category1": "IAB9 - Hobbies & Interests",
 "iab_category2": "N/A",
 "iab_category3": "N/A"}

// Domain could not be classified at all
{"iab_category1": "N/A", "iab_category2": "N/A", "iab_category3": "N/A"}
```

If you get all-`N/A` for a domain you believe should be classifiable, try `?force=1` to re-run the classifier — occasionally a transient issue produces an empty result that a retry resolves.

---

## Error Codes

All errors return a JSON body of the form:

```json
{"detail": {"message": "<short human-readable message>", "error_code": "<code>"}}
```

| HTTP | `error_code` | When it happens | Retry? |
|---|---|---|---|
| 200 | — | Classification returned successfully | — |
| 401 | — | Invalid or missing API key | No — check your key |
| 403 | — | Account pending approval or API key expired | No — contact your account rep |
| 408 | `timeout` | Classifier exceeded its internal deadline | Yes — retry after a few seconds |
| 429 | — (no body code) | Your daily quota is exhausted | No until tomorrow (00:00 UTC) |
| 429 | `rate_limit` | Upstream AI provider is throttling | Yes — retry with 15–60s backoff |
| 502 | `upstream_auth` | Our classifier's upstream credentials are misconfigured | No — contact support |
| 502 | `upstream_bad_request` | Upstream rejected the prompt for this input | No for this domain |
| 503 | `classifier_unavailable` | Temporary classifier failure | Yes — retry with exponential backoff |

**Distinguishing the two 429 cases:** a 429 without an `error_code` means you've hit your customer daily quota (wait until tomorrow or request a limit increase). A 429 with `error_code: "rate_limit"` means the upstream AI provider is temporarily throttling — just retry in a few seconds.

---

## Code Examples

### Python (single domain)

```python
import requests

API_KEY = "wl8_your_api_key_here"
BASE_URL = "https://webcat.coeurstrike.com"

def classify(domain, force=False):
    params = {"force": "1"} if force else None
    r = requests.get(
        f"{BASE_URL}/api/v1/classify/{domain}",
        headers={"X-API-Key": API_KEY},
        params=params,
        timeout=60,
    )
    if r.status_code == 200:
        return r.json()
    print(f"Error {r.status_code}: {r.text}")
    return None

result = classify("cnn.com")
if result:
    print(f"Primary:   {result['iab_category1']}")
    print(f"Secondary: {result['iab_category2']}")
    print(f"Tertiary:  {result['iab_category3']}")
    print(f"Cached:    {result['cached']}")
```

### Python (bulk, concurrent)

The API is built for concurrency. Fire multiple requests in parallel with `asyncio` + `httpx`:

```python
import asyncio
import httpx

API_KEY = "wl8_your_api_key_here"
BASE_URL = "https://webcat.coeurstrike.com"

async def classify(client, domain):
    r = await client.get(
        f"{BASE_URL}/api/v1/classify/{domain}",
        headers={"X-API-Key": API_KEY},
        timeout=60,
    )
    return domain, r.json() if r.status_code == 200 else None

async def main():
    domains = ["cnn.com", "espn.com", "github.com", "amazon.com"]
    async with httpx.AsyncClient() as client:
        # 20 is a good starting concurrency; adjust up or down based on experience.
        sem = asyncio.Semaphore(20)
        async def bounded(d):
            async with sem:
                return await classify(client, d)
        results = await asyncio.gather(*(bounded(d) for d in domains))
    for domain, result in results:
        if result:
            print(f"{domain}: {result['iab_category1']}")

asyncio.run(main())
```

For production bulk jobs, implement retry-with-backoff: 15–30s on `429 rate_limit`, 5–10s on `408 timeout`, 1–2s on `503`. Terminal errors (`401`, `403`, `429 quota`, `502 upstream_auth`) should not be retried.

### JavaScript (Node.js)

```javascript
const API_KEY = "wl8_your_api_key_here";
const BASE_URL = "https://webcat.coeurstrike.com";

async function classify(domain, { force = false } = {}) {
  const url = new URL(`${BASE_URL}/api/v1/classify/${domain}`);
  if (force) url.searchParams.set("force", "1");
  const r = await fetch(url, { headers: { "X-API-Key": API_KEY } });
  if (r.status === 200) return await r.json();
  console.error(`Error ${r.status}`);
  return null;
}

classify("cnn.com").then(result => {
  if (result) {
    console.log(`Primary:   ${result.iab_category1}`);
    console.log(`Secondary: ${result.iab_category2}`);
    console.log(`Tertiary:  ${result.iab_category3}`);
  }
});
```

### PHP

```php
<?php
$api_key  = "wl8_your_api_key_here";
$base_url = "https://webcat.coeurstrike.com";

function classify($domain, $force = false) {
    global $api_key, $base_url;
    $url = "$base_url/api/v1/classify/$domain" . ($force ? "?force=1" : "");
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL            => $url,
        CURLOPT_HTTPHEADER     => ["X-API-Key: $api_key"],
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT        => 60,
    ]);
    $response  = curl_exec($ch);
    $http_code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    return $http_code === 200 ? json_decode($response, true) : null;
}

$result = classify("cnn.com");
if ($result) {
    echo "Category: " . $result["iab_category1"] . "\n";
}
```

### cURL (shell batch)

```bash
#!/bin/bash
API_KEY="wl8_your_api_key_here"

for domain in cnn.com espn.com github.com amazon.com; do
    echo -n "$domain: "
    curl -s "https://webcat.coeurstrike.com/api/v1/classify/$domain" \
      -H "X-API-Key: $API_KEY" | jq -r '.iab_category1'
done
```

---

## IAB 3.0 Categories

Classifications use the IAB Tech Lab Content Taxonomy 3.0. Tier-1 categories returned by the API:

| Code | Category |
|---|---|
| IAB1  | Arts & Entertainment |
| IAB2  | Automotive |
| IAB3  | Business |
| IAB4  | Careers |
| IAB5  | Education |
| IAB6  | Family & Parenting |
| IAB7  | Health & Fitness |
| IAB8  | Food & Drink |
| IAB9  | Hobbies & Interests |
| IAB10 | Home & Garden |
| IAB11 | Law, Government & Politics |
| IAB12 | News |
| IAB13 | Personal Finance |
| IAB14 | Society |
| IAB15 | Science |
| IAB16 | Pets |
| IAB17 | Sports |
| IAB18 | Style & Fashion |
| IAB19 | Technology & Computing |
| IAB20 | Travel |
| IAB21 | Real Estate |
| IAB22 | Shopping |
| IAB23 | Religion & Spirituality |

Tier-2 subcategories (e.g. `IAB12-1 - International News`, `IAB19-11 - Internet Technology`) are returned where applicable. Full taxonomy reference: https://iabtechlab.com/standards/content-taxonomy/

---

## Rate Limits & Caching

- **Daily query limit** is per-account, set by your subscription plan. Check your remaining quota at `/api/v1/usage`.
- **No per-second rate limit** is enforced on our side beyond what your daily quota allows. We recommend keeping concurrency reasonable (20–50 in-flight requests is a good starting point).
- **Results are cached for 30 days.** Cache hits do not invoke the AI classifier but still count against your daily quota. Use `?force=1` to bypass the cache when needed.

---

## Best Practices

1. **Set a 60-second HTTP timeout.** Cache hits are near-instant, but cache-miss classifications may take 1–3 seconds, and upstream retries can occasionally push this higher.

2. **Cache results on your side.** Classifications for established domains are stable for months. A local cache reduces API calls, saves your quota, and insulates you from transient upstream issues.

3. **Clean domain input before submitting.** Strip `http://`, `https://`, `www.`, and paths:
   - &#9989; `example.com`
   - &#10060; `https://www.example.com/page`

   The server cleans these as well, but URL-style input with embedded slashes will `404` against the route.

4. **Fire requests concurrently for bulk work.** The backend classifies multiple domains in parallel. Batch jobs of 10K+ domains are routine.

5. **Handle retryable errors with backoff.** `408`, `429 rate_limit`, and `503` are transient — exponential backoff (15s, 30s, 60s) works well. Terminal errors (`401`, `403`, `429` without code, `502 upstream_auth`) should not be retried.

6. **Monitor quota consumption.** Poll `/api/v1/usage` periodically or after each batch so you know where you stand.

7. **Use `?force=1` sparingly.** Reserved for cases where a prior `N/A` result was suspicious and you want a fresh classification attempt. Forced classifications always hit the AI and always count against quota.

---

## Web Interface

For manual domain lookups (useful for spot-checks and demos):

```
https://webcat.coeurstrike.com
```

Sign in with the email you registered with and your API key, then enter a domain to see the full classification with copy-to-clipboard JSON. The signed-in state persists for 24 hours, and the running API version is displayed in the footer.

---

## Support

For API key requests, technical support, quota increases, or billing inquiries, contact your account representative.

---

&copy; 2026 CoeurStrike.AI — a WHOISXMLAPI Rapid-Development Group service
