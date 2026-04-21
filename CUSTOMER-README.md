# WebL8.AI - Web Classification API

WebL8.AI provides intelligent website classification using the IAB 3.0 taxonomy. Our AI-powered API analyzes domains and returns standardized content categories for advertising, content filtering, and analytics use cases.

## Getting Started

### Base URL

```
https://webcat.coeurstrike.ai
```

### Authentication

All API requests require an API key passed in the `X-API-Key` header:

```
X-API-Key: wl8_your_api_key_here
```

### Request an API Key

Visit the registration page to request access:

```
https://webcat.coeurstrike.ai/register
```

Fill out the form with your company name and email. Your API key will be provided after admin approval.

---

## API Reference

### Classify a Domain

Classify a domain and receive IAB 3.0 category classifications.

**Endpoint:**
```
GET /api/v1/classify/{domain}
```

**Example Request:**
```bash
curl -X GET "https://webcat.coeurstrike.ai/api/v1/classify/cnn.com" \
  -H "X-API-Key: wl8_your_api_key_here"
```

**Success Response (200 OK):**
```json
{
  "fqdn": "cnn.com",
  "auditdate": "2026-04-21 14:30:00",
  "iab_category1": "IAB12 - News",
  "iab_category2": "IAB12-1 - International News",
  "iab_category3": "",
  "cached": true
}
```

**Queued Response (202 Accepted):**

If the domain hasn't been classified yet, it will be queued for processing:

```json
{
  "detail": {
    "message": "Domain queued",
    "fqdn": "example.com"
  }
}
```

Wait 30 seconds and retry the request.

---

### Check API Usage

View your current usage statistics and remaining quota.

**Endpoint:**
```
GET /api/v1/usage
```

**Example Request:**
```bash
curl -X GET "https://webcat.coeurstrike.ai/api/v1/usage" \
  -H "X-API-Key: wl8_your_api_key_here"
```

**Response:**
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

---

## Response Fields

| Field | Description |
|-------|-------------|
| `fqdn` | The fully qualified domain name that was classified |
| `auditdate` | Timestamp when the classification was performed |
| `iab_category1` | Primary IAB 3.0 category |
| `iab_category2` | Secondary IAB 3.0 category (if applicable) |
| `iab_category3` | Tertiary IAB 3.0 category (if applicable) |
| `cached` | Whether the result was served from cache |

---

## Error Codes

| HTTP Code | Meaning | Description |
|-----------|---------|-------------|
| 200 | Success | Classification returned successfully |
| 202 | Accepted | Domain queued for classification - retry in 30 seconds |
| 401 | Unauthorized | Invalid or missing API key |
| 403 | Forbidden | Account pending approval or API key expired |
| 429 | Too Many Requests | Daily query limit exceeded |

---

## Code Examples

### Python

```python
import requests

API_KEY = "wl8_your_api_key_here"
BASE_URL = "https://webcat.coeurstrike.ai"

def classify_domain(domain):
    headers = {"X-API-Key": API_KEY}
    response = requests.get(f"{BASE_URL}/api/v1/classify/{domain}", headers=headers)
    
    if response.status_code == 200:
        return response.json()
    elif response.status_code == 202:
        print("Domain queued, retry in 30 seconds")
        return None
    else:
        print(f"Error: {response.status_code}")
        return None

# Example usage
result = classify_domain("cnn.com")
if result:
    print(f"Category: {result['iab_category1']}")
```

### JavaScript (Node.js)

```javascript
const API_KEY = "wl8_your_api_key_here";
const BASE_URL = "https://webcat.coeurstrike.ai";

async function classifyDomain(domain) {
  const response = await fetch(`${BASE_URL}/api/v1/classify/${domain}`, {
    headers: { "X-API-Key": API_KEY }
  });

  if (response.status === 200) {
    return await response.json();
  } else if (response.status === 202) {
    console.log("Domain queued, retry in 30 seconds");
    return null;
  } else {
    console.error(`Error: ${response.status}`);
    return null;
  }
}

// Example usage
classifyDomain("cnn.com").then(result => {
  if (result) {
    console.log(`Category: ${result.iab_category1}`);
  }
});
```

### PHP

```php
<?php
$api_key = "wl8_your_api_key_here";
$base_url = "https://webcat.coeurstrike.ai";

function classifyDomain($domain) {
    global $api_key, $base_url;
    
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, "$base_url/api/v1/classify/$domain");
    curl_setopt($ch, CURLOPT_HTTPHEADER, ["X-API-Key: $api_key"]);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    
    $response = curl_exec($ch);
    $http_code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    if ($http_code === 200) {
        return json_decode($response, true);
    }
    return null;
}

// Example usage
$result = classifyDomain("cnn.com");
if ($result) {
    echo "Category: " . $result["iab_category1"];
}
?>
```

### cURL (Batch Processing)

```bash
#!/bin/bash
API_KEY="wl8_your_api_key_here"

domains=("cnn.com" "espn.com" "github.com" "amazon.com")

for domain in "${domains[@]}"; do
    echo "Classifying: $domain"
    curl -s "https://webcat.coeurstrike.ai/api/v1/classify/$domain" \
      -H "X-API-Key: $API_KEY" | jq '.iab_category1'
    sleep 1
done
```

---

## IAB 3.0 Categories

WebL8.AI classifies domains using the IAB Tech Lab Content Taxonomy 3.0. Common categories include:

| Code | Category |
|------|----------|
| IAB1 | Arts & Entertainment |
| IAB2 | Automotive |
| IAB3 | Business |
| IAB4 | Careers |
| IAB5 | Education |
| IAB6 | Family & Parenting |
| IAB7 | Health & Fitness |
| IAB8 | Food & Drink |
| IAB9 | Hobbies & Interests |
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

For the complete taxonomy, visit: https://iabtechlab.com/standards/content-taxonomy/

---

## Rate Limits

- **Daily Limit**: Based on your subscription plan
- **Rate**: No per-second limit, but please be respectful
- **Cache**: Results are cached for 30 days to improve response times

Check your current usage with the `/api/v1/usage` endpoint.

---

## Best Practices

1. **Handle 202 responses**: New domains need processing time. Implement retry logic with 30-second delays.

2. **Cache results locally**: Classifications are stable. Cache results on your end to reduce API calls.

3. **Clean domain input**: Remove `http://`, `https://`, `www.`, and paths before submitting:
   - ✅ `example.com`
   - ❌ `https://www.example.com/page`

4. **Monitor usage**: Check `/api/v1/usage` regularly to avoid hitting limits.

---

## Web Interface

For manual lookups, visit:

```
https://webcat.coeurstrike.ai
```

Enter your API key and domain to get instant classification results with a copy-to-clipboard JSON output.

---

## Support

For API key requests, technical support, or billing inquiries, contact your account representative.

---

© 2026 WebL8.AI - Intelligent Web Classification
