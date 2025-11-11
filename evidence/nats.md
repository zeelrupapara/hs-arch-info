# NATS Subjects List

## Overview
Complete enumeration of subjects used in Publish and Subscribe operations across **VFXServer**, **VFXCore**, and **VFXMarket** services [file:1].

NATS messaging enables inter-service communication using a publish-subscribe pattern where services publish messages to specific subjects and other services subscribe to those subjects to receive updates [web:33][web:36].

---

## Published Subjects

The following subjects are **published** by services to broadcast events and data:

| Subject | Description |
|---------|-------------|
| `system.lp_providers` | Liquidity provider configuration updates |
| `websocket.broker.routing_workflow` | WebSocket routing workflow events |
| `SubjectSystemLPIn` | Incoming liquidity provider messages |
| `SubjectSystemLPOut` | Outgoing liquidity provider messages |
| `SubjectSystemSymbols` | Symbol/instrument updates |
| `SubjectSystemMarketFeed("*")` | Market feed data (wildcard for all instruments) |
| `SubjectSystemMonitor` | System monitoring and health check events |
| `SubjectSystemDataProviders` | Data provider status updates |
| `SubjectSystemSymbolStatus` | Symbol activation/deactivation status |
| `SubjectAccountRoutingRules(accountId)` | Account-specific routing rule changes |
| `SubjectAccountOperations(accountId)` | Account operation events (per account) |
| `SubjectSystemAccounts` | System-wide account updates |
| `SubjectAccountDeals(accountId)` | Deal/trade execution events (per account) |
| `SubjectAccountOrders(accountId)` | Order updates (per account) |
| `SubjectSystemPositions` | System-wide position updates |
| `SubjectSystemOrders` | System-wide order updates |

---

## Subscribed Subjects

The following subjects are **subscribed to** by services to receive events and data:

| Subject | Description |
|---------|-------------|
| `SubjectSystemLPOut` | Listens for outgoing LP messages |
| `SubjectSystemSymbols` | Receives symbol/instrument updates |
| `SubjectSystemDataProviders` | Monitors data provider status changes |
| `SubjectSystemMarketFeed("*")` | Consumes real-time market feed data |
| `SubjectSystemMoneyTransaction` | Listens for money transaction events |
| `SubjectSystemBalanceVerification` | Receives balance verification requests |
| `SubjectSystemBalanceCorrection` | Processes balance correction events |
| `SubjectSystemAccounts` | Monitors account creation/modification |
| `SubjectSystemConfig` | Receives system configuration updates |
| `SubjectSystemSymbolStatus` | Tracks symbol status changes |
| `SubjectSystemSymbolsGroup` | Monitors symbol group updates |
| `SubjectSystemGroups` | Receives group hierarchy changes |
| `SubjectSystemCommissions` | Listens for commission configuration updates |
| `SubjectSystemCommissionsTiers` | Monitors commission tier changes |
| `SubjectSystemLPProviders` | Tracks LP provider configuration |
| `SubjectSystemLPIn` | Receives incoming LP messages |
| `SubjectSystemDealing` | Listens for dealing desk events |
| `SubjectSystemRule` | Monitors routing rule updates |
| `SubjectSystemRuleConditions` | Receives rule condition changes |
| `SubjectSystemAutoRule` | Tracks automation rule updates |
| `SubjectBackofficeStatus` | Monitors back-office system status |
| `SubjectBrokerListener` | Listens for broker event notifications |

---

## Subject Naming Conventions

### System-Level Subjects
- **Pattern:** `SubjectSystem*`
- **Purpose:** Broadcast system-wide events affecting multiple accounts or services
- **Examples:** `SubjectSystemOrders`, `SubjectSystemPositions`, `SubjectSystemSymbols`

### Account-Level Subjects
- **Pattern:** `SubjectAccount*(accountId)`
- **Purpose:** Send targeted events to specific account contexts
- **Examples:** `SubjectAccountOrders(accountId)`, `SubjectAccountDeals(accountId)`
- **Note:** These subjects use dynamic parameter substitution with account identifiers

### Wildcard Subjects
- **Pattern:** `Subject*("*")`
- **Purpose:** Subscribe to all events in a category using wildcard matching
- **Example:** `SubjectSystemMarketFeed("*")` receives data for all market instruments

---

## Messaging Architecture

### Service Communication Flow

