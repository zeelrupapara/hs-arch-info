# HS Trader — Step 0 Baseline Freeze: Developer Evidence Pack

## Project Information

| Field | Details |
|-------|---------|
| **Project** | HS Trader System |
| **Phase** | Step 0 — Baseline Freeze |
| **Prepared by** | Golang Back End Team |
| **Date** | 08/11/2025 |
| **Version** | 1.0 |

## Purpose

To capture the current state of the HS Trader system (APIs, Databases, Messaging, LP Configs, Validation Rules, and Reports) before migration. This document includes short notes and evidence of links for each major component.

---

## Table of Contents

1. [APIs](#apis)
2. [Database Schemas](#database-schemas)
3. [Event Topics / Messaging](#event-topics--messaging)
4. [Validation Rules](#validation-rules)
5. [Liquidity Provider (LP) Configs](#liquidity-provider-lp-configs)
6. [Reports](#reports)
7. [Developer Output Format](#developer-output-format)

---

## APIs

### Overview

HS Trader exposes REST and WebSocket APIs under `/api/v1`, `/auth/v1`, and `/ws/v1`, built with Fiber (Golang). Authentication uses OAuth2 + RBAC, with structured logging via Zap.

### Base Paths

| Type | Base Path | Purpose |
|------|-----------|---------|
| REST API | `/api/v1` | Core business APIs |
| WebSocket | `/ws/v1` | Real-time data updates |
| OAuth2 | `/auth/v1/oauth2` | Authentication & token issuance |
| Public | `/api/v1/public` | Public (non-auth) endpoints |
| System | `/api/v1/system` | System admin APIs |

### Middleware & Auth Flow

| Middleware | Function |
|-----------|----------|
| `cors.New()` | Enables CORS |
| `RequestsLogger` | Logs HTTP requests |
| `UserAgentParser`, `HeaderReader` | Reads client headers |
| `Protect` | Validates Bearer token |
| `Authorization()` | RBAC policy enforcement |

**OAuth Flow:**
```
POST /login → POST /token → POST /refresh/token → DELETE /logout
```

### Module Summary

| Module | Base Path | Purpose |
|--------|-----------|---------|
| Auth | `/auth/v1/oauth2` | Token lifecycle management |
| Accounts | `/api/v1/accounts` | Account management and balances |
| Orders | `/api/v1/orders` | Order placement & updates |
| Positions | `/api/v1/positions` | Manage open positions |
| Reports | `/api/v1/reports` | Generate and fetch reports |

### Evidence

| Evidence Type | File / Link | Description |
|---------------|------------|-------------|
| Swagger Spec | https://dev.hstrader.com/swagger/index.html#/ | Full API specification (login with admin role) |
| Gateway Config | [api-flow](evidence/api.md) | Authentication + Rate limit setup |


---

## Database Schemas

### Overview

HS Trader uses a dual-database model:

- **MySQL** for transactional data (accounts, orders, positions, symbols, configs)
- **InfluxDB** for time-series data (quotes, ticks, metrics)

### Key Tables

| Table | Description |
|-------|-------------|
| Accounts | Balance, equity, margin fields (int64, float64) |
| Symbols | Instrument metadata (Forex, Metals, Crypto) |
| Groups | Hierarchical user grouping |
| Roles | RBAC roles (Admin, Dealer, Trader, Investor) |
| Config | Global settings (End of Day, Max Orders) |
| AutoRules | Automation rules (JSON conditions & actions) |
| DataProviders | Market feed sources (FIX, DDE, Simulators) |
| RoutingRules | Routing rules (JSON conditions & actions) |

### Evidence

| Evidence Type | File / Link | Description |
|---------------|------------|-------------|
| DDL Dump | [schema_dump](evidence/schema_dump.sql) | Structure dump from script |
| ERD Diagram | [ERD](evidence/vfxcore.png) | Table relationships |
| Migration Script | [seeder_script](evidence/VFXServer_Full_Seeder_Developer_Reference(1).md) | Seeder for default records |

---

## Event Topics / Messaging

### Overview

NATS messaging is used for inter-service communication between VFXServer, VFXCore, and VFXMarket.

### Subjects

| Subject | Description |
|---------|-------------|
| `orders.stream` | Real-time order updates |
| `positions.update` | Account position changes |
| `prices.ticks` | Market tick data stream |
| `lp.status` | Liquidity provider status notifications |

### Evidence

| Evidence Type | File / Link |
|---------------|------------|
| NATS Subject List | [NATS Subjects List](evidence/nats.md) |

---

## Validation Rules

### Overview

Validation rules ensure system reliability, account integrity, and risk control across the HS Trader platform. They protect against invalid operations, enforce margin discipline, and maintain trading accuracy. Rules are enforced through API handlers, Casbin RBAC policies, and VFXCore validation engines.

### Validation Categories

| Rule Type | Description | Enforcement |
|-----------|-------------|-------------|
| Account Creation Validation | Ensure valid account type, role, group, and email before account creation. | API Handler + Casbin RBAC |
| Password Management Validation | Checks password length (3–32 chars), prevents duplicate investor/user passwords, and applies rate limits. | Authentication Service |
| Order Validation | Confirms valid symbol, positive volume, margin sufficiency, and ensures account margin level compliance. | VFXCore Engine |
| Position Validation | Prevents modification of closed/locked positions; verifies correct side (Buy/Sell). | Position Handler |
| Margin & Leverage Validation | Calculates margin level (equity ÷ used_margin × 100) and triggers stop-out alerts if below threshold. | VFXCore Risk Monitor |
| Liquidation Rules | Auto-closes positions when equity/margin fall below configured thresholds. | Automation Engine |
| Automation Rule Validation | Validates trigger conditions (balance < threshold, profit > limit) before execution. | Automation Service |
| Routing Rule Validation | Checks that LP routing priorities and provider availability are valid before trade routing. | Routing Manager |
| Symbol Validation | Ensures symbol is active and enabled before accepting orders. | Symbol Handler |
| Data Provider Validation | Confirms that external data providers are connected and streaming quotes. | Data Provider Monitor |

### Evidence

| Evidence Type | File / Link | Description |
|---------------|------------|-------------|
| Validation Code Snippet | [validation](evidence/validation_docs.md) | Source snippet from account validation logic |

---

## Liquidity Provider (LP) Configs

### Overview

HS Trader connects to external Liquidity Providers (LPs) via FIX 4.4, using the internal bridge vfxlp and stunnel (SSL) for secure communication. This setup supports dynamic routing, automatic failover, and centralized monitoring for all LP connections.

### Configuration Details

| Config Item | Description |
|-------------|-------------|
| LP Connections | Each LP maintains a FIX 4.4 session managed by vfxlp. Handles Logon, Heartbeat, and Logout. |
| Config Format | Stored in DB (vfx_LPProviderConfig + optional .cfg file) with Host, Port, SenderCompID, TargetCompID, HeartBtInt. |
| Routing Logic | Priority-based routing (1 = highest); active providers only are used. |
| Failover / Kill Switch | Admin can deactivate LP via API → publishes NATS event and gracefully stops session. |
| Monitoring | Status tracked in DB (status: 0 = inactive, 1 = active, 2 = error). Logs in vfxlp/logs/fix/{provider}/. |
| Security / SSL | Managed by stunnel-manager for automatic SSL tunnel setup and updates. |

### Version Comparison

| Aspect | Previous Version | Current Version |
|--------|-----------------|-----------------|
| LP Configuration | Manual and static configuration of each LP (Centroid, Simulator, F4B). Separate stunnel entries required; developer support needed for every change. | Fully dynamic and database-driven. stunnel-manager service automatically creates and updates SSL tunnels for new or modified LPs. |

### FIX Connection Flow

| Stage | Description |
|-------|-------------|
| Heartbeat Interval (HeartBtInt) | Defines keepalive interval; sends heartbeat when idle. |
| Timeout & Disconnect Handling | Triggers automatic reconnect if heartbeat/test request missed. |
| Logon | Authenticates FIX session and syncs sequence numbers. |
| Logout | Gracefully terminates FIX session and updates sequence tracking. |

### Evidence

| Evidence Type | File / Link | Description |
|---------------|------------|-------------|
| Config File | Document.docx | FIX session setup (JSON/YAML) |
| Session Logs | Document.docx | Logon/Heartbeat traces |
| SSL Manager Log | Document.docx | Dynamic SSL tunnel creation proof |

---

## Reports

### Overview

The HS Trader reporting system is designed to generate regulatory, operational, and compliance reports. Reports are categorized based on their purpose and frequency, with both manual and automated generation options. Reports are accessible through the `/api/v1/reports` endpoint and stored locally or in cloud archival.

### Reporting Mechanism

Reports are generated by backend handlers in vfxcore and vfxserver.

**Data sources:**
- MySQL (Orders, Positions, Accounts)
- InfluxDB (Time Series Market Data)

**Generated reports can be downloaded via:**
- REST endpoint: `GET /api/v1/reports/:id`
- Internal job schedulers (CRON or Airflow based)

**Formats supported:** CSV, PDF, HTML

### Evidence

| Evidence Type | File / Link | Description |
|---------------|------------|-------------|
| Sample Report (CSV) | [CSV Reports](evidence/report/CSV/) | Example of CAT regulatory report with header fields. Available files: Account Group.csv, Account Statement.csv, History - Deals.csv, History - Order.csv, History - Position.csv, Journal.csv, Money Transaction.csv, Online Sessions.csv, Trade Account.csv |
| PDF Export | [PDF Reports](evidence/report/pdf/) | Generated daily performance report. Available files: Account Group.pdf, Account Statement.pdf, History - Deals.pdf, History - order.pdf, History - Position.pdf, Journal.pdf, Money Transaction.pdf, Online Sessions.pdf, Trade Account.pdf |
| HTML | [HTML Reports](evidence/report/HTML/) | Visual dashboard showing report status and generation history. Available files: Account Group.png, Account Statement.png, History - Deal.png, History - Order.png, History - Position.png, Journal.png, Money Transaction.png, Online Sessions.png, Trade Account.png |
| Dashboard | [Dashboard](evidence/report/Report%20dashboard%20view.png) | Cron job for automated report generation. |
