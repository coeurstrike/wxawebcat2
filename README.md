# WebL8.AI - Intelligent Web Classification System

A production-ready web classification system using IAB 3.0 Taxonomy, powered 100% by OpenAI GPT models.

![Python](https://img.shields.io/badge/python-3.10+-blue.svg)
![FastAPI](https://img.shields.io/badge/FastAPI-0.109+-green.svg)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15+-blue.svg)
![License](https://img.shields.io/badge/license-MIT-green.svg)

## 🚀 Features

- **FastAPI Web Application** - Modern REST API with interactive documentation
- **Beautiful Web Interface** - Cybersecurity-themed UI for single and batch classification
- **PostgreSQL Database** - Persistent storage with caching and staleness detection
- **Two-Stage Pipeline** - Separate crawler and classifier services for scalability
- **Customer Management** - API keys, rate limiting, and usage tracking
- **IAB 3.0 Taxonomy** - Industry-standard classification with 700+ categories

## 📋 Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Web/API       │────▶│    Crawler      │────▶│   Classifier    │
│   (FastAPI)     │     │   (Script 1)    │     │   (Script 2)    │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         │                       ▼                       │
         │              ┌─────────────────┐              │
         │              │  Queue (Files)  │              │
         │              │  ./data/queue/  │              │
         │              └─────────────────┘              │
         │                                               │
         ▼                                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     PostgreSQL Database                          │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ classifications  │  │  customers   │  │   query_log      │   │
│  └──────────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## 📁 Project Structure

```
wxacat/
├── wxacat.conf         # Configuration file
├── config.py           # Config loader
├── database.py         # SQLAlchemy models
├── crawler.py          # Script 1: Web crawler
├── classifier.py       # Script 2: OpenAI classifier
├── api.py              # FastAPI application
├── requirements.txt    # Python dependencies
├── templates/          # HTML templates
│   ├── index.html      # Home page
│   ├── result.html     # Classification result
│   ├── pending.html    # Pending status
│   └── batch_result.html
├── static/             # Static files (CSS, JS)
└── data/               # Working directories
    ├── queue/          # Pending crawl jobs
    ├── processing/     # Currently processing
    ├── processed/      # Completed jobs
    └── failed/         # Failed jobs
```

## 🛠️ Installation

### Prerequisites

- Python 3.10+
- PostgreSQL 15+
- OpenAI API key

### Setup

```bash
# Clone the repository
git clone https://github.com/yourusername/wxacat.git
cd wxacat

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate     # Windows

# Install dependencies
pip install -r requirements.txt

# Install Playwright browsers
python -m playwright install

# Create PostgreSQL database
createdb wxacat_db

# Configure settings
cp wxacat.conf.example wxacat.conf
# Edit wxacat.conf with your settings
```

### Configuration

Edit `wxacat.conf`:

```ini
[database]
host = localhost
port = 5432
name = wxacat_db
user = your_user
password = your_password

[openai]
api_key = sk-your-openai-api-key

[cache]
stale_days = 30
```

Or use environment variables:
```bash
export WXACAT_DATABASE_PASSWORD=your_password
export WXACAT_OPENAI_API_KEY=sk-your-key
```

## 🚀 Running the System

### 1. Start the API Server

```bash
python api.py
# Or with uvicorn
uvicorn api:app --host 0.0.0.0 --port 8000 --reload
```

Access at: http://localhost:8000

### 2. Start the Crawler Service

```bash
# Process single domain
python crawler.py example.com

# Batch from file
python crawler.py -f urls.txt

# With custom concurrency
python crawler.py -f urls.txt -c 10
```

### 3. Start the Classifier Service

```bash
# Run as daemon (continuous)
python classifier.py

# Process once and exit
python classifier.py --once

# Custom poll interval
python classifier.py --poll-interval 5
```

## 📡 API Endpoints

### Classification

```bash
# Single domain (requires API key)
curl -X GET "http://localhost:8000/api/v1/classify/example.com" \
     -H "X-API-Key: wl8_your_api_key"

# Batch classification
curl -X POST "http://localhost:8000/api/v1/classify/batch" \
     -H "X-API-Key: wl8_your_api_key" \
     -H "Content-Type: application/json" \
     -d '{"domains": ["cnn.com", "espn.com", "github.com"]}'
```

### Usage Stats

```bash
curl -X GET "http://localhost:8000/api/v1/usage" \
     -H "X-API-Key: wl8_your_api_key"
```

### Admin (Create Customer)

```bash
curl -X POST "http://localhost:8000/api/admin/customers" \
     -H "Content-Type: application/json" \
     -d '{
       "company_name": "Acme Corp",
       "email": "admin@acme.com",
       "queries_per_day": 1000,
       "expiration_days": 365
     }'
```

## 🗄️ Database Schema

### domain_classifications

| Column | Type | Description |
|--------|------|-------------|
| id | SERIAL | Primary key |
| fqdn | VARCHAR(255) | Domain name (unique, indexed) |
| url | VARCHAR(2048) | Full URL |
| title | VARCHAR(512) | Page title |
| description | TEXT | Meta description |
| iab_category1 | VARCHAR(50) | Primary IAB category |
| iab_category2 | VARCHAR(50) | Secondary category |
| iab_category3 | VARCHAR(50) | Tertiary category |
| source | VARCHAR(50) | crawl, http_fallback, known |
| audit_date | TIMESTAMP | Last classification date |
| is_classified | BOOLEAN | Classification status |

### customers

| Column | Type | Description |
|--------|------|-------------|
| id | SERIAL | Primary key |
| customer_id | VARCHAR(50) | Unique customer ID |
| api_key | VARCHAR(100) | API key (indexed) |
| company_name | VARCHAR(255) | Company name |
| email | VARCHAR(255) | Contact email |
| queries_per_day | INTEGER | Daily rate limit |
| queries_used_today | INTEGER | Usage counter |
| expiration_date | DATE | Account expiration |
| is_active | BOOLEAN | Account status |

### query_log

| Column | Type | Description |
|--------|------|-------------|
| id | SERIAL | Primary key |
| customer_id | VARCHAR(50) | Customer reference |
| fqdn | VARCHAR(255) | Queried domain |
| query_time | TIMESTAMP | Query timestamp |
| response_time_ms | INTEGER | Response time |
| cache_hit | BOOLEAN | Cache hit flag |

## ⚙️ Configuration Reference

### wxacat.conf sections

| Section | Key | Default | Description |
|---------|-----|---------|-------------|
| database | host | localhost | PostgreSQL host |
| database | port | 5432 | PostgreSQL port |
| database | name | wxacat_db | Database name |
| cache | stale_days | 30 | Days before refresh |
| crawler | concurrency | 5 | Parallel crawlers |
| classifier | poll_interval | 10 | Queue poll seconds |
| openai | model | gpt-4o-mini | OpenAI model |
| api | port | 8000 | API server port |

## 🔄 Workflow

1. **Request** arrives via API or Web UI
2. **Check Database** for cached classification
   - If fresh: return immediately
   - If stale or missing: queue for crawling
3. **Crawler** picks up queued domains
   - Fetches page with stealth browser
   - Extracts metadata, headings, content
   - Saves to queue directory
4. **Classifier** monitors queue
   - Reads crawled data
   - Classifies with OpenAI
   - Saves to PostgreSQL
   - Moves files to processed/

## 🛡️ Security

- API key authentication required
- Rate limiting per customer
- Query logging for audit
- Environment variable support for secrets

## 📈 Scaling

For high-volume deployments:

1. **Multiple Crawler Instances** - Run on different machines
2. **Multiple Classifier Instances** - Process queue in parallel
3. **Redis Queue** - Replace file-based queue
4. **Read Replicas** - For PostgreSQL reads
5. **CDN/Cache** - For static assets

## 🤝 Contributing

Contributions welcome! Please read our contributing guidelines.

## 📄 License

MIT License - see [LICENSE](LICENSE) for details.

---

Built with ❤️ by WebL8.AI
