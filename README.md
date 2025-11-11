# HS Trader — Step 0 Baseline Freeze: Developer Evidence Pack

## Project Information

| Field | Details |
|-------|---------|
| **Project** | HS Trader System |
| **Phase** | Step 0 — Baseline Freeze |
| **Prepared by** | Golang Back end Team |
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
| Gateway Config | Screenshots (Word file) | Authentication + Rate limit setup |
| Sample Logs | Docs file | Request/response proof |
| Error Logs | Error log documentation | Validation failures evidence |

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
| DDL Dump | `schema_dump` | Structure dump from script |
| ERD Diagram | ERD Diagram files | Table relationships |
| Migration Script | `seeder code snippet` | Seeder for default records |

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
| NATS Subject List | Document.docx |

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
| Validation Code Snippet | `validation` | Source snippet from account validation logic |

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
| Sample Report (CSV) | CSV file | Example of CAT regulatory report with header fields. |
| PDF Export | PDF file | Generated daily performance report. |
| HTML | HTML file | Visual dashboard showing report status and generation history. |
| Dashboard | Dashboard file | Cron job for automated report generation. |

---

## Developer Output Format

### API: Order Placement

**Current State:**
REST endpoint `/api/v1/orders` secured with OAuth authentication. Order processing performed in VFXCore. No idempotency token implemented yet. Logs stored under `/logs/api/orders.log`.

**Evidence:**
- Screenshot of API gateway config (showing rate limits and auth middleware)
- Sample request/response JSON log captured via Postman

### DB Schema: Orders

**Current State:**
Table `orders(id, symbol, type, price, qty, timestamp, status)` with FLOAT types for monetary fields. No explicit audit trail for modifications; linked with accounts and positions.

**Evidence:**
- DDL dump (`schema_dump.sql`)
- ERD diagram (`vfxcore.png`) highlighting foreign keys and indexes

### Event Topic: system.order

**Current State:**
NATS subject `system.order` used for publishing order events to VFXCore. No persistence or retention policy enabled (default fire-and-forget).

**Evidence:**
- Output of `grep nats.Publish` from source code
- Broker configuration screenshot showing subject subscriptions

### Validation Rule: Max Order Size

**Current State:**
Hardcoded rule in `vfxcore/order_validation.go` enforces a maximum of 250 open orders per account (configured via `seed_data.go`). No runtime override or audit tracking for violations.

**Evidence:**
- Code snippet from `order_validation.go`
- Test execution log (`validation_test.log`) showing rejection when limit exceeded

### LP Config: LPABC

**Current State:**
Configured through `vfx_LPProviderConfig` table. Static JSON used for FIX session details (Host, Port, HeartBtInt). Manual failover required. No latency KPIs or monitoring alerts configured.

**Evidence:**
- Config file (`lp_config.docx`) showing FIX parameters
- Monitoring dashboard screenshot (`monitoring_dashboard.png`) displaying connection status

### Report: CAT

**Current State:**
CAT (Consolidated Audit Trail) report generated manually via `/api/v1/reports` endpoint. Output in CSV format and emailed to compliance team. No archival or automated scheduler configured yet.

**Evidence:**
- Sample CAT report (`cat_report.csv`)
- Email delivery screenshot showing transmission to compliance distribution list

### Report: Account Statement

**Current State:**
Displays account balance, equity, and margin summary generated through the report API. Available on demand for traders and admins.

**Evidence:**
- Generated CSV (`account_statement.csv`)
- Report dashboard screenshot

### Report: Online Session

**Current State:**
Lists user login sessions with timestamp, IP, and session status. Used for security auditing and access monitoring.

**Evidence:**
- Session log (`session_log.csv`)
- Screenshot of report execution from UI dashboard
