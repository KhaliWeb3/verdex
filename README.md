[README.md](https://github.com/user-attachments/files/26329658/README.md)
# Verdex

**Cross-market intelligence agent for smarter buying decisions.**

Verdex aggregates products across Amazon, eBay, Alibaba, and AliExpress — then validates, scores, and explains every recommendation before a user acts. It behaves less like a price comparison tool and more like a procurement analyst: one that challenges pricing inefficiencies, surfaces hidden alternatives, and quantifies trust instead of assuming it.

![Status](https://img.shields.io/badge/status-in%20development-yellow?style=flat-square)
![Architecture](https://img.shields.io/badge/architecture-microservices-0ea5e9?style=flat-square)
![Database](https://img.shields.io/badge/database-PostgreSQL%20%2B%20pgvector-336791?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-22c55e?style=flat-square)

---

## Table of Contents

- [The Problem](#the-problem)
- [How It Works](#how-it-works)
- [System Architecture](#system-architecture)
- [Microservices](#microservices)
- [Database Schema](#database-schema)
- [Event Bus](#event-bus)
- [End-to-End Flow](#end-to-end-flow)
- [Tech Stack](#tech-stack)
- [Project Status](#project-status)
- [Contributing](#contributing)
- [License](#license)

---

## The Problem

Shopping across marketplaces is broken in a specific, underappreciated way:

- The same product appears under dozens of different titles and listings
- Cheaper does not mean equivalent — but no platform tells you the difference
- No existing tool explains *why* it's recommending something
- Users are left guessing on counterfeit risk, spec mismatches, and seller reliability

The result is friction, mistrust, and bad purchases. Verdex is designed to eliminate all three.

---

## How It Works

Every search triggers three sequential operations:

### 1 — Intelligent Sourcing

Verdex identifies a product's canonical signature — brand, specs, materials, manufacturer patterns — and searches all integrated marketplaces for exact matches and near-equivalents simultaneously.

### 2 — Equivalence Validation

This is where most platforms fail. Rather than assuming cheaper equals better, Verdex builds a **confidence score** from five independent signals:

| Signal | What It Measures |
|---|---|
| Spec matching | Hard attribute comparison against canonical product |
| Visual similarity | Image-based comparison via embeddings |
| Supplier consistency | Manufacturer pattern and origin matching |
| Review analysis | Pattern integrity and sentiment scoring |
| Price history | Historical pricing behaviour and anomaly detection |

Every result is classified as one of four types:

| Classification | Meaning |
|---|---|
| `exact_match` | Identical product, different listing |
| `equivalent` | Functionally the same, different source |
| `inferior` | Missing key attributes relative to original |
| `risky` | Counterfeit likelihood or seller red flags present |

No blind recommendations. Every suggestion is explainable.

### 3 — Transparent Decision Support

Before any purchase, the user sees a full decision panel:

- Side-by-side comparison: original vs. alternative
- Price delta vs. value delta
- Confidence score with line-by-line reasoning
- Risk flags: quality risk, counterfeit likelihood, shipping reliability

The user decides. Verdex removes the guesswork.

---

## System Architecture

Verdex is built as a three-layer system:

```
┌─────────────────────────────────┐
│      User Experience Layer      │  Search UI · Decision Panel · Confidence Display
└────────────────┬────────────────┘
                 ↓
┌─────────────────────────────────┐
│       Intelligence Layer        │  Matching · Validation · Risk Scoring
└────────────────┬────────────────┘
                 ↓
┌─────────────────────────────────┐
│     Execution & Data Layer      │  Orders · Analytics · B2B API · Event Bus
└─────────────────────────────────┘
```

Internally, this resolves into **six distinct systems**:

| System | Responsibility |
|---|---|
| User Experience Engine | Search interface, confidence UI, decision support panels |
| Intelligence Engine | Product matching, equivalence validation, risk scoring |
| Data Aggregation Layer | Marketplace scraping, price history, supplier reliability tracking |
| Execution Layer | Order creation and routing |
| Analytics Layer | User behaviour, pricing trend analysis, demand signals |
| B2B API Layer | External developer access, usage metering, API key management |

---

## Microservices

Each service owns its schema exclusively. No shared databases. Cross-service communication happens through events and internal APIs — never direct database joins.

### Service Topology

```
                    [ API Gateway ]
                          ↓
┌─────────────────────────────────────────────────┐
│  Auth        │  Search     │  Listings           │
│  Matching    │  Validation │  Risk               │
│  Orders      │  Analytics  │  B2B API            │
└─────────────────────────────────────────────────┘
```

### Language Assignment

| Service | Language | Rationale |
|---|---|---|
| API Gateway | Node.js | Routing, middleware, rate limiting |
| Auth | Node.js | JWT/session ecosystem |
| Search | Node.js | Query orchestration and NLP parsing |
| Listings | Go | High-volume concurrent marketplace fetching |
| Matching | Go | Vector math, CPU-intensive scoring |
| Validation | Go | Multi-signal scoring pipelines |
| Risk | Go | Fast rule evaluation at scale |
| Orders | Node.js | Business logic and state management |
| Analytics | Go | High-throughput event ingestion |
| B2B API | Go | Performance-critical, external-facing |

---

### API Gateway

Entry point for all client requests. Handles routing, token validation, and rate limiting.

```
POST   /api/search
GET    /api/product/:id
POST   /api/checkout
GET    /api/recommendations/:query
```

---

### Auth Service

Handles user identity, session management, and internal token validation.

```
POST   /auth/register
POST   /auth/login
GET    /auth/me
POST   /auth/logout

GET    /internal/validate-token     ← internal only
```

---

### Search Service

Parses raw user input into structured product attributes for downstream services.

```
POST   /search
```

Request:
```json
{
  "query": "iPhone 13 Pro 256GB"
}
```

Response:
```json
{
  "query_id": "uuid",
  "parsed": {
    "brand": "Apple",
    "model": "iPhone 13 Pro",
    "storage": "256GB",
    "category": "smartphones"
  }
}
```

---

### Listings Service

Fetches and stores raw product data from all integrated marketplaces.

```
GET    /listings?query=iphone13&marketplace=all
GET    /listings/:id

POST   /internal/fetch-marketplace  ← internal worker trigger
```

---

### Matching Service

The core intelligence layer. Maps marketplace listings to canonical products and computes confidence scores.

```
POST   /matching/run
```

Request:
```json
{
  "listings": ["listing_id_1", "listing_id_2"],
  "product_id": "canonical_product_uuid"
}
```

Response:
```json
{
  "matches": [
    {
      "listing_id": "listing_id_1",
      "product_variant_id": "variant_uuid",
      "confidence": 0.91,
      "match_type": "equivalent"
    },
    {
      "listing_id": "listing_id_2",
      "product_variant_id": "variant_uuid",
      "confidence": 0.43,
      "match_type": "inferior"
    }
  ]
}
```

---

### Validation Service

Explains each match by breaking down the confidence score into individual signal scores.

```
POST   /validation/score
```

Request:
```json
{
  "listing_match_id": "match_uuid"
}
```

Response:
```json
{
  "listing_match_id": "match_uuid",
  "spec_score": 0.95,
  "image_score": 0.88,
  "review_score": 0.90,
  "seller_score": 0.85,
  "composite_score": 0.91
}
```

---

### Risk Service

Flags listing-level anomalies: price manipulation, counterfeit signals, unreliable sellers.

```
POST   /risk/evaluate
```

Request:
```json
{
  "listing_id": "listing_uuid"
}
```

Response:
```json
{
  "listing_id": "listing_uuid",
  "risk_level": "low",
  "flags": ["price_anomaly"],
  "severity": 1
}
```

---

### Orders Service

Manages order creation, status tracking, and confirmation flow.

```
POST   /orders/create
GET    /orders/:id
POST   /orders/:id/confirm
```

---

### Analytics Service

Ingests all system events for behaviour analysis and product intelligence.

```
POST   /analytics/event
GET    /analytics/user/:id
GET    /analytics/product/:id
```

---

### B2B API Service

External developer-facing layer. Exposes Verdex intelligence via metered API access.

```
GET    /b2b/best-price?product=iphone13&region=global
GET    /b2b/supplier-score?supplier_id=uuid
GET    /b2b/risk-report?listing_id=uuid
```

Example response (`/b2b/best-price`):
```json
{
  "product": "iPhone 13 Pro 256GB",
  "best_supplier": "Alibaba",
  "price": 78.00,
  "currency": "USD",
  "confidence": 0.91,
  "risk_level": "low",
  "alternatives_found": 6
}
```

---

## Database Schema

Each schema listed below belongs exclusively to the named service.

---

### Auth Service

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT UNIQUE NOT NULL,
    password_hash   TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'user',   -- user | admin | b2b
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE sessions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token       TEXT NOT NULL UNIQUE,
    expires_at  TIMESTAMP NOT NULL
);
```

---

### Search Service

```sql
CREATE TABLE search_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID,                  -- nullable for anonymous searches
    raw_query       TEXT NOT NULL,
    parsed_query    JSONB,                 -- structured output from NLP parser
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_search_queries_user ON search_queries(user_id);
CREATE INDEX idx_search_queries_created ON search_queries(created_at DESC);
```

---

### Listings Service

```sql
CREATE TABLE marketplaces (
    id      SERIAL PRIMARY KEY,
    name    TEXT UNIQUE NOT NULL,          -- Amazon | eBay | Alibaba | AliExpress
    base_url TEXT
);

CREATE TABLE listings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marketplace_id  INT NOT NULL REFERENCES marketplaces(id),
    external_id     TEXT NOT NULL,         -- ID from source marketplace
    title           TEXT NOT NULL,
    description     TEXT,
    price           NUMERIC(12, 2) NOT NULL,
    currency        TEXT NOT NULL DEFAULT 'USD',
    url             TEXT NOT NULL,
    image_url       TEXT,
    rating          FLOAT,
    review_count    INT DEFAULT 0,
    seller_name     TEXT,
    raw_data        JSONB,                 -- full scraped payload preserved
    fetched_at      TIMESTAMP DEFAULT NOW(),
    UNIQUE(marketplace_id, external_id)
);

CREATE INDEX idx_listings_marketplace  ON listings(marketplace_id);
CREATE INDEX idx_listings_price        ON listings(price);
CREATE INDEX idx_listings_fetched      ON listings(fetched_at DESC);
```

---

### Matching Service

The canonical product layer — the truth behind every listing.

```sql
-- Normalised product identity
CREATE TABLE products (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title       TEXT NOT NULL,
    brand       TEXT,
    category    TEXT,
    model       TEXT,
    attributes  JSONB,                     -- flexible spec store
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Variant handling: size, storage, colour, etc.
CREATE TABLE product_variants (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id          UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    variant_attributes  JSONB NOT NULL,    -- { "storage": "256GB", "colour": "black" }
    sku                 TEXT,
    created_at          TIMESTAMP DEFAULT NOW()
);

-- Bridge between raw listings and canonical products
CREATE TABLE listing_matches (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id          UUID NOT NULL,
    product_variant_id  UUID NOT NULL REFERENCES product_variants(id),
    match_type          TEXT NOT NULL,     -- exact | equivalent | inferior | risky
    confidence_score    FLOAT NOT NULL CHECK (confidence_score BETWEEN 0 AND 1),
    created_at          TIMESTAMP DEFAULT NOW(),
    UNIQUE(listing_id, product_variant_id)
);

CREATE INDEX idx_matches_confidence ON listing_matches(confidence_score DESC);
CREATE INDEX idx_matches_listing    ON listing_matches(listing_id);
CREATE INDEX idx_products_category  ON products(category);
```

---

### Validation Service

```sql
CREATE TABLE match_explanations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_match_id    UUID NOT NULL UNIQUE,
    spec_score          FLOAT NOT NULL CHECK (spec_score BETWEEN 0 AND 1),
    image_score         FLOAT NOT NULL CHECK (image_score BETWEEN 0 AND 1),
    review_score        FLOAT NOT NULL CHECK (review_score BETWEEN 0 AND 1),
    seller_score        FLOAT NOT NULL CHECK (seller_score BETWEEN 0 AND 1),
    composite_score     FLOAT GENERATED ALWAYS AS (
                            (spec_score + image_score + review_score + seller_score) / 4
                        ) STORED,
    created_at          TIMESTAMP DEFAULT NOW()
);
```

---

### Risk Service

```sql
CREATE TABLE risk_flags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID NOT NULL,
    flag_type       TEXT NOT NULL,         -- price_anomaly | counterfeit | unreliable_seller | suspicious_reviews
    severity        INT NOT NULL CHECK (severity BETWEEN 1 AND 5),
    description     TEXT,
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_risk_listing  ON risk_flags(listing_id);
CREATE INDEX idx_risk_type     ON risk_flags(flag_type);
CREATE INDEX idx_risk_severity ON risk_flags(severity DESC);
```

---

### Orders Service

```sql
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,
    listing_id      UUID NOT NULL,
    total_amount    NUMERIC(12, 2) NOT NULL,
    currency        TEXT NOT NULL DEFAULT 'USD',
    status          TEXT NOT NULL DEFAULT 'pending',  -- pending | confirmed | fulfilled | cancelled
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE order_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id            UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_variant_id  UUID NOT NULL,
    quantity            INT NOT NULL CHECK (quantity > 0),
    unit_price          NUMERIC(12, 2) NOT NULL
);

CREATE INDEX idx_orders_user    ON orders(user_id);
CREATE INDEX idx_orders_status  ON orders(status);
```

---

### Analytics Service

```sql
-- Append-only event store — never update, never delete
CREATE TABLE events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type  TEXT NOT NULL,    -- UserSearched | MatchCompleted | RiskEvaluated | OrderCreated
    user_id     UUID,
    session_id  UUID,
    payload     JSONB NOT NULL,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_events_type    ON events(event_type);
CREATE INDEX idx_events_user    ON events(user_id);
CREATE INDEX idx_events_created ON events(created_at DESC);
```

---

### B2B API Service

```sql
CREATE TABLE api_clients (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    api_key     TEXT NOT NULL UNIQUE,
    plan        TEXT NOT NULL DEFAULT 'starter',  -- starter | growth | enterprise
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE api_usage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES api_clients(id),
    endpoint        TEXT NOT NULL,
    requests_count  INT NOT NULL DEFAULT 0,
    period_start    TIMESTAMP NOT NULL,
    period_end      TIMESTAMP NOT NULL
);

CREATE INDEX idx_api_usage_client ON api_usage(client_id);
CREATE INDEX idx_api_usage_period ON api_usage(period_start, period_end);
```

---

### Vector Search (Similarity Engine)

Powers product and listing similarity — used by the Matching Service.

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     TEXT NOT NULL,         -- product | listing
    entity_id       UUID NOT NULL UNIQUE,
    embedding       VECTOR(1536) NOT NULL, -- OpenAI text-embedding-3-small
    created_at      TIMESTAMP DEFAULT NOW()
);

-- IVFFlat index for approximate nearest neighbour search
CREATE INDEX idx_embeddings_vector
ON embeddings USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

---

## Event Bus

Every critical operation emits an event. Services subscribe and react asynchronously — no service waits on another.

**Broker:** Kafka (recommended) or RabbitMQ

| Event | Emitted By | Consumed By |
|---|---|---|
| `UserSearched` | Search | Analytics, Listings |
| `ListingsFetched` | Listings | Matching |
| `MatchCompleted` | Matching | Validation, Risk, Analytics |
| `RiskEvaluated` | Risk | API Gateway (response assembly) |
| `OrderCreated` | Orders | Analytics |

---

## End-to-End Flow

```
Step 1   User submits query
         API Gateway → Search Service (parses query into structured attributes)

Step 2   Listings fetched
         Search → Listings Service (pulls from eBay, Amazon, Alibaba, AliExpress)
         Event: ListingsFetched

Step 3   Intelligence runs
         Listings → Matching Service (confidence scored per listing)
         Event: MatchCompleted
         → Validation Service (per-signal score breakdown)
         → Risk Service (anomaly and counterfeit flags)
         Event: RiskEvaluated

Step 4   Response assembled
         API Gateway returns to client:
         {
           "original_price": 120,
           "best_alternative_price": 80,
           "savings": 40,
           "confidence": 0.91,
           "match_type": "equivalent",
           "risk_level": "low",
           "explanation": { ... }
         }

Step 5   User action
         → Redirect to marketplace (affiliate model)
         → OR place order via Orders Service
         Event: OrderCreated
```

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| API Gateway | Node.js + Fastify | Routing, auth middleware, rate limiting |
| Service runtime (compute) | Go | Listings, Matching, Validation, Risk, Analytics, B2B |
| Service runtime (logic) | Node.js | Auth, Search, Orders |
| Primary database | PostgreSQL 15 | Core transactional storage |
| Vector search | pgvector | Similarity matching via embeddings |
| Caching | Redis | Listing cache, session store |
| Message broker | Kafka | Async event bus between services |
| Embeddings | OpenAI `text-embedding-3-small` | Product and listing vectorisation |
| Data warehouse | BigQuery | Long-term analytics and pricing trends |
| Containerisation | Docker + Docker Compose | Local development and deployment |

---

## Project Status

| Milestone | Status |
|---|---|
| System architecture | ✅ Complete |
| Per-service database schemas | ✅ Complete |
| Microservice endpoints defined | ✅ Complete |
| Event bus topology | ✅ Complete |
| Service scaffolding | 🔲 Pending |
| Matching engine v1 | 🔲 Pending |
| Validation + confidence scoring | 🔲 Pending |
| Risk evaluation engine | 🔲 Pending |
| Listings integrations (all 4 marketplaces) | 🔲 Pending |
| Vector similarity search | 🔲 Pending |
| B2B API (public) | 🔲 Pending |

---

## Contributing

Verdex is in active early development. Contributions are welcome across any of the service layers.

### Getting Started

```bash
git clone https://github.com/KhaliWeb3/verdex.git
cd verdex
```

Each service lives in its own directory under `/services` and contains a `README.md` with local setup instructions.

### Guidelines

**One service per pull request.** Changes that span multiple services belong in separate PRs — it keeps reviews clean and rollbacks surgical.

**Own your schema.** If you're contributing to a service, changes to that service's database schema must be included in the same PR as the code that depends on them. No schema changes without a migration file.

**Events over direct calls.** If your contribution requires one service to react to another, use the event bus. Avoid introducing synchronous inter-service dependencies unless absolutely necessary.

**Test your confidence logic.** The Matching and Validation services are the core of Verdex's value. Any changes to scoring algorithms must include test cases with documented expected outputs.

**Write explainable code.** This project is about explainability — that philosophy extends to the codebase. Non-obvious logic should be commented with *why*, not just *what*.

### Opening an Issue

If you spot an architectural inconsistency, a schema gap, or want to propose a new signal for the confidence scoring system — open an issue. Label it clearly: `architecture`, `schema`, `intelligence`, `api`, or `docs`.

---

## License

MIT © [KhaliWeb3](https://github.com/KhaliWeb3)
