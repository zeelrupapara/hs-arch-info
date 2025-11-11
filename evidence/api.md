# VFX Server API Documentation (v1)

**Base URL:** `/api/v1`
**API Version:** v1
**Developer:** Saif Hamdan

---

## Table of Contents

1. [Authentication & OAuth](#1-authentication--oauth)
2. [WebSocket](#2-websocket)
3. [System - Monitor](#3-system---monitor)
4. [System - Roles](#4-system---roles)
5. [System - Resources](#5-system---resources)
6. [System - Policies](#6-system---policies)
7. [System - Tokens](#7-system---tokens)
8. [System - Sessions](#8-system---sessions)
9. [System - Operations](#9-system---operations)
10. [System - Support](#10-system---support)
11. [Broker](#11-broker)
12. [Accounts](#12-accounts)
13. [Groups](#13-groups)
14. [Commissions](#14-commissions)
15. [Deals](#15-deals)
16. [Orders](#16-orders)
17. [Positions](#17-positions)
18. [Routing Rules](#18-routing-rules)
19. [Reports](#19-reports)
20. [News](#20-news)
21. [Configs](#21-configs)
22. [Symbols](#22-symbols)
23. [Watchlist](#23-watchlist)
24. [Alerts](#24-alerts)
25. [Emails](#25-emails)
26. [Workspace Profile](#26-workspace-profile)
27. [Automation](#27-automation)
28. [Import/Export](#28-importexport)
29. [Data Providers](#29-data-providers)
30. [LP Providers](#30-lp-providers)
31. [Market Feed](#31-market-feed)
32. [Public APIs](#32-public-apis)

---

## 1. Authentication & OAuth

**Module:** Authentication
**Base Path:** `/auth/v1/oauth2`
**Auth Middleware:** Varies by endpoint
**Global Middlewares:** UserAgentParser, HeaderReader, RequestsLogger

### Endpoints

| Method | Path | Auth Required | Middleware | Description |
|--------|------|---------------|------------|-------------|
| `POST` | `/login` | No | BasicAuthParser | User login with OAuth2 credentials |
| `POST` | `/token` | No | BasicAuthParser | Obtain OAuth2 access token |
| `POST` | `/refresh/token` | No | - | Refresh expired access token |
| `DELETE` | `/logout` | Yes | Protect | Revoke current session and logout |

**Idempotency:**
- POST (login, token, refresh) — Not idempotent
- DELETE (logout) — Idempotent

**Purpose:**
OAuth2-based authentication system for user login, token generation, refresh, and logout operations.

---

## 2. WebSocket

**Module:** WebSocket Communication
**Base Path:** `/ws/v1`
**Auth Middleware:** Protect
**Global Middlewares:** UserAgentParser, HeaderReader, RequestsLogger

### Endpoints

| Method | Path | Auth Required | Description |
|--------|------|---------------|-------------|
| `GET` | `/` | Yes | Establish WebSocket connection for real-time communication |

**Idempotency:** N/A (WebSocket connection)

**Purpose:**
Real-time bidirectional communication channel for live market data, notifications, and system events.

---

## 3. System - Monitor

**Module:** System Monitoring
**Base Path:** `/api/v1/system/monitor`
**Auth Middleware:** Protect
**Authorization:** None (system-level endpoint)

### Endpoints

| Method | Path | Auth Required | Authorization | Description |
|--------|------|---------------|---------------|-------------|
| `GET` | `/health` | Yes | - | Check system health status |

**Idempotency:** GET — Idempotent

**Purpose:**
Monitor system health and availability for infrastructure monitoring and health checks.

---

## 4. System - Roles

**Module:** Role Management
**Base Path:** `/api/v1/system/roles`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Roles_Read` | List all roles in the system |
| `POST` | `/` | Yes | `Resources_Roles_Manage` | Create a new role |
| `PATCH` | `/:role_id` | Yes | `Resources_Roles_Manage` | Update an existing role |
| `DELETE` | `/:role_id` | Yes | `Resources_Roles_Manage` | Delete a role |

**Idempotency:**
- GET — Idempotent
- POST — Not idempotent
- PATCH — Idempotent
- DELETE — Idempotent

**Purpose:**
Manage user roles and permissions for role-based access control (RBAC) system.

---

## 5. System - Resources

**Module:** System Resources
**Base Path:** `/api/v1/system/resources`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Roles_Read` | List all available resources |
| `GET` | `/:role_id/role` | Yes | `Resources_Roles_Read` | Get resources assigned to a specific role |

**Idempotency:** GET — Idempotent

**Purpose:**
View system resources and their associations with roles for permission management.

---

## 6. System - Policies

**Module:** Policy Management
**Base Path:** `/api/v1/system/policies`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/:role_id` | Yes | `Resources_Roles_Manage` | Create policies for a role |
| `PUT` | `/:role_id` | Yes | `Resources_Roles_Manage` | Update policies for a role |
| `POST` | `/` | Yes | `Resources_Roles_Manage` | Delete specific policies |

**Idempotency:**
- POST (create) — Not idempotent
- PUT — Idempotent
- POST (delete) — Idempotent

**Purpose:**
Manage access control policies that define what resources roles can access.

---

## 7. System - Tokens

**Module:** Token History Management
**Base Path:** `/api/v1/system/tokens`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/history` | Yes | `Resources_Tokens_Read` | View all token generation history |
| `DELETE` | `/history` | Yes | `Resources_Tokens_Delete` | Clear token history records |

**Idempotency:**
- GET — Idempotent
- DELETE — Idempotent

**Purpose:**
Audit and manage OAuth2 token generation history for security and compliance.

---

## 8. System - Sessions

**Module:** Session Management
**Base Path:** `/api/v1/system/sessions`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/active` | Yes | `Resources_HistorySessions_Read` | List all active user sessions |
| `GET` | `/history` | Yes | `Resources_HistorySessions_Read` | View session history |
| `DELETE` | `/history` | Yes | `Resources_HistorySessions_Delete` | Clear session history |
| `DELETE` | `/active` | Yes | `Resources_OnlineSessions_Disconnect` | Disconnect all active sessions |
| `DELETE` | `/active/:id` | Yes | `Resources_OnlineSessions_Disconnect` | Disconnect a specific session |

**Idempotency:** GET, DELETE — Idempotent

**Purpose:**
Monitor and manage user sessions, including viewing active sessions and forcefully disconnecting users.

---

## 9. System - Operations

**Module:** Operation Logs
**Base Path:** `/api/v1/system/operations`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_OperationLogs_Read` | List all system operation logs |
| `GET` | `/:operation_id` | Yes | `Resources_OperationLogs_Read` | Get details of a specific operation |

**Idempotency:** GET — Idempotent

**Purpose:**
Audit trail viewer for system operations and administrative actions (read-only, deletion is against regulations).

---

## 10. System - Support

**Module:** In-App Support Ticketing
**Base Path:** `/api/v1/system/support`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/tickets` | Yes | `Resources_InAppSupport_Read` | List all support tickets |
| `GET` | `/tickets/:ticket_id` | Yes | `Resources_InAppSupport_Read` | Get ticket details |
| `POST` | `/tickets` | Yes | `Resources_InAppSupport_Manage` | Create a new support ticket |
| `PUT` | `/tickets/:ticket_id` | Yes | `Resources_InAppSupport_Manage` | Update ticket status/details |
| `GET` | `/tickets/:ticket_id/article` | Yes | `Resources_InAppSupport_Read` | Get ticket conversation articles |
| `POST` | `/tickets/:ticket_id/article` | Yes | `Resources_InAppSupport_Manage` | Add a reply to ticket |

**Idempotency:**
- GET — Idempotent
- POST — Not idempotent
- PUT — Idempotent

**Purpose:**
Internal support ticket system for user assistance and issue tracking with conversation threads.

---

## 11. Broker

**Module:** Broker Information & Settings
**Base Path:** `/api/v1/broker`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Broker_Read` | Get broker information and settings |
| `GET` | `/navigation` | Yes | `Resources_Broker_Read` | Get broker navigation tree structure |
| `PUT` | `/` | Yes | `Resources_Broker_Update` | Update broker settings |

**Idempotency:**
- GET — Idempotent
- PUT — Idempotent

**Purpose:**
Manage broker-level configuration, branding, and navigation structure for the platform.

---

## 12. Accounts

**Module:** Account Management
**Base Path:** `/api/v1/accounts`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints Overview

This module is divided into sub-sections for better organization:

#### 12.1 Account List & Details

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Accounts_Read` | List all trading accounts |
| `GET` | `/:account_id` | Yes | `Resources_Accounts_Read` | Get specific account details |

#### 12.2 My Account (Self-Service)

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/me` | Yes | `Resources_MyAccount_Read` | Get current user's profile |
| `GET` | `/me/policies/ui` | Yes | `Resources_MyAccount_Read` | Get UI permissions for current user |
| `POST` | `/me/request-change-password` | Yes | `Resources_MyAccount_ChangePassword` | Request password change (sends verification code) |
| `POST` | `/me/verify-change-password-code` | Yes | `Resources_MyAccount_ChangePassword` | Verify the password change code |
| `POST` | `/me/change-password` | Yes | `Resources_MyAccount_ChangePassword` | Change current user's password |
| `POST` | `/me/investor-password` | Yes | `Resources_MyAccount_ChangeInvestorPassword` | Change investor (read-only) password |
| `POST` | `/me/client-secret` | Yes | `Resources_MyAccount_ClientSecret` | Generate API client secret |
| `DELETE` | `/me/client-secret` | Yes | `Resources_MyAccount_ClientSecret` | Revoke API client secret |
| `GET` | `/me/journal` | Yes | `Resources_MyAccount_Journal` | View personal trading journal |

#### 12.3 Account Administration

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/` | Yes | `Resources_Accounts_Create` | Create a new trading account |
| `PUT` | `/:account_id` | Yes | `Resources_Accounts_Update` | Update account details |
| `PATCH` | `/:account_id/approve` | Yes | `Resources_Accounts_Update` | Approve pending account |
| `PATCH` | `/:account_id/decline` | Yes | `Resources_Accounts_Update` | Decline pending account |
| `PATCH` | `/:account_id/change-password` | Yes | `Resources_Accounts_ChangePasswords` | Admin: Change user's password |
| `DELETE` | `/:account_id` | Yes | `Resources_Accounts_Delete` | Delete an account |
| `PATCH` | `/:account_id/groups/:group_id` | Yes | `Resources_Accounts_Update` | Move account to different group |
| `GET` | `/:account_id/journal` | Yes | `Resources_Accounts_Journal` | View account's trading journal |

#### 12.4 Favourites

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/favourites` | Yes | `Resources_Accounts_Favourites` | List favourite accounts |
| `POST` | `/:account_id/favourites` | Yes | `Resources_Accounts_Favourites` | Add account to favourites |
| `DELETE` | `/:account_id/favourites` | Yes | `Resources_Accounts_Favourites` | Remove account from favourites |

#### 12.5 Money Operations

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/:account_id/money/deposit` | Yes | `Resources_Accounts_Money` | Deposit funds to account |
| `POST` | `/:account_id/money/withdrawal` | Yes | `Resources_Accounts_Money` | Withdraw funds from account |
| `POST` | `/:account_id/money/adjust` | Yes | `Resources_Accounts_Money` | Adjust account balance (correction) |
| `POST` | `/:account_id/money/credit-in` | Yes | `Resources_Accounts_Money` | Add credit to account |
| `POST` | `/:account_id/money/credit-out` | Yes | `Resources_Accounts_Money` | Remove credit from account |

#### 12.6 Position Analysis & Balance Verification

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/:account_id/position-analysis` | Yes | `Resources_BulkOperations_Read` | Analyze account positions for corrections |
| `POST` | `/:account_id/apply-position-changes` | Yes | `Resources_BulkOperations_Update` | Apply position corrections (reopen positions) |
| `GET` | `/:account_id/verify-balance` | Yes | `Resources_Accounts_Read` | Verify account balance integrity |
| `POST` | `/:account_id/apply-balance-correction` | Yes | `Resources_Accounts_Money` | Apply balance corrections if discrepancies found |

**Idempotency:**
- GET — Idempotent
- POST (create, money operations) — Not idempotent
- PUT, PATCH — Idempotent
- DELETE — Idempotent

**Purpose:**
Comprehensive account lifecycle management including creation, approval, money operations, profile management, favourites, and balance verification.

---

## 13. Groups

**Module:** Account Group Management
**Base Path:** `/api/v1/groups`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Groups_Read` | List all account groups |
| `POST` | `/` | Yes | `Resources_Groups_Create` | Create a new account group |
| `PUT` | `/:group_id` | Yes | `Resources_Groups_Update` | Update group settings |
| `PATCH` | `/:group_id/copy/:copy_group_id` | Yes | `Resources_Groups_Update` | Copy settings from another group |
| `DELETE` | `/:group_id` | Yes | `Resources_Groups_Delete` | Delete a group |

**Idempotency:**
- GET — Idempotent
- POST — Not idempotent
- PUT, PATCH — Idempotent
- DELETE — Idempotent

**Purpose:**
Manage account groups with shared trading parameters, margin requirements, and permission sets.

---

## 14. Commissions

**Module:** Commission Management
**Base Path:** `/api/v1/commissions`
**Auth Middleware:** Protect (via parent)
**Authorization Middleware:** `Authorization(Resources_Groups_Commissions)` (applied to all routes)

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Groups_Commissions` | List all commission templates |
| `GET` | `/groups/:group_id` | Yes | `Resources_Groups_Commissions` | Get all commissions for a group |
| `GET` | `/:commission_id/groups/:group_id` | Yes | `Resources_Groups_Commissions` | Get specific commission for a group |
| `POST` | `/groups/:group_id` | Yes | `Resources_Groups_Commissions` | Create commission structure for group |
| `PUT` | `/:commission_id` | Yes | `Resources_Groups_Commissions` | Update commission structure |
| `DELETE` | `/:commission_id` | Yes | `Resources_Groups_Commissions` | Delete commission structure |
| `POST` | `/:commission_id/tiers` | Yes | `Resources_Groups_Commissions` | Add commission tier (volume-based) |
| `PUT` | `/:commission_id/tiers/:tier_id` | Yes | `Resources_Groups_Commissions` | Update commission tier |
| `DELETE` | `/:commission_id/tiers/:tier_id` | Yes | `Resources_Groups_Commissions` | Delete commission tier |

**Idempotency:**
- GET — Idempotent
- POST — Not idempotent
- PUT — Idempotent
- DELETE — Idempotent

**Purpose:**
Configure tiered commission structures for account groups with volume-based pricing models.

---

## 15. Deals

**Module:** Deal History
**Base Path:** `/api/v1/deals`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Trades_Overview` | List all completed deals |
| `GET` | `/bulk` | Yes | `Resources_BulkOperations_Read` | Get deals in bulk for mass operations |
| `GET` | `/accounts/me` | Yes | `Resources_MyDeals_Read` | Get current user's deals |
| `GET` | `/accounts/:account_id` | Yes | `Resources_Trades_Overview` | Get deals for specific account |
| `GET` | `/:deal_id` | Yes | `Resources_Trades_Overview` | Get specific deal details |
| `POST` | `/bulk` | Yes | `Resources_BulkOperations_Delete` | Delete multiple deals (bulk operation) |
| `DELETE` | `/` | Yes | `Resources_BulkOperations_Delete` | Delete deals by IDs array |

**Idempotency:**
- GET — Idempotent
- POST (bulk delete) — Idempotent
- DELETE — Idempotent

**Purpose:**
View and manage closed trade history with bulk operation support for administrative tasks.

---

## 16. Orders

**Module:** Order Management
**Base Path:** `/api/v1/orders`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

#### 16.1 Order Viewing

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Trades_Overview` | List all pending orders |
| `GET` | `/accounts/me` | Yes | `Resources_MyOrders_Read` | Get current user's orders |
| `GET` | `/accounts/:account_id` | Yes | `Resources_Trades_Overview` | Get orders for specific account |
| `GET` | `/:order_id` | Yes | `Resources_Trades_Overview` | Get specific order details |

#### 16.2 Order Creation

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/accounts/me` | Yes | `Resources_MyOrders_Create` | Create order for current user |
| `POST` | `/` | Yes | `Resources_Trades_Manage` | Admin: Create order for any account |

#### 16.3 Order Modification

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `PUT` | `/:order_id` | Yes | `Resources_Trades_Manage` | Admin: Modify pending order |
| `PUT` | `/:order_id/accounts/me` | Yes | `Resources_MyOrders_Update` | Modify own pending order |

#### 16.4 Order Cancellation

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/:order_id` | Yes | `Resources_Trades_Manage` | Admin: Cancel any order |
| `POST` | `/:order_id/accounts/me` | Yes | `Resources_MyOrders_Cancel` | Cancel own order |

**Idempotency:**
- GET — Idempotent
- POST (create) — Not idempotent
- PUT — Idempotent
- POST (cancel) — Idempotent

**Purpose:**
Full order lifecycle management including creation, modification, and cancellation for pending orders (limit, stop, etc.).

**Note:** Broker/trader approval routes are commented out in the current implementation.

---

## 17. Positions

**Module:** Position Management
**Base Path:** `/api/v1/positions`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

#### 17.1 Position Viewing

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Trades_Overview` | List all open positions |
| `GET` | `/bulk` | Yes | `Resources_BulkOperations_Read` | Get positions in bulk for mass operations |
| `GET` | `/accounts/me` | Yes | `Resources_MyPositions_Read` | Get current user's open positions |
| `GET` | `/accounts/:account_id` | Yes | `Resources_Trades_Overview` | Get positions for specific account |
| `GET` | `/:position_id` | Yes | `Resources_Trades_Overview` | Get specific position details |

#### 17.2 Position Modification

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `PUT` | `/:position_id` | Yes | `Resources_Trades_Manage` | Admin: Modify position (SL/TP) |
| `PUT` | `/:position_id/accounts/me` | Yes | `Resources_MyPositions_Update` | Modify own position (SL/TP) |

#### 17.3 Position Closing

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/bulk` | Yes | `Resources_BulkOperations_Update` | Close multiple positions (bulk) |
| `POST` | `/:position_id` | Yes | `Resources_Trades_Manage` | Admin: Close any position |
| `POST` | `/:position_id/accounts/me` | Yes | `Resources_MyPositions_Close` | Close own position |
| `POST` | `/:position_id/close-by-hedge` | Yes | `Resources_MyOrders_Create` | Close hedged positions (opposite positions) |

**Idempotency:**
- GET — Idempotent
- PUT — Idempotent
- POST (close operations) — Idempotent

**Purpose:**
Manage open market positions including modification of stop-loss/take-profit, position closing, bulk operations, and hedge closing.

---

## 18. Routing Rules

**Module:** Order Routing Configuration
**Base Path:** `/api/v1/rules`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

#### 18.1 Configuration & Viewing

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/config` | Yes | `Resources_RoutingRule_Read` | Get routing configuration metadata |
| `GET` | `/` | Yes | `Resources_RoutingRule_Read` | List all routing rules |
| `GET` | `/:rule_id` | Yes | `Resources_RoutingRule_Read` | Get specific routing rule |
| `GET` | `/:rule_id/conditions/:cond_id` | Yes | `Resources_RoutingRule_Read` | Get specific routing condition |

#### 18.2 Rule Management

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/` | Yes | `Resources_RoutingRule_Create` | Create new routing rule |
| `PUT` | `/:rule_id` | Yes | `Resources_RoutingRule_Update` | Update routing rule |
| `DELETE` | `/:rule_id` | Yes | `Resources_RoutingRule_Delete` | Delete routing rule |
| `POST` | `/swap-priority` | Yes | `Resources_RoutingRule_Update` | Change rule execution priority order |

#### 18.3 Condition Management

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/:rule_id/conditions` | Yes | `Resources_RoutingRule_Create` | Add condition to routing rule |
| `PUT` | `/:rule_id/conditions/:cond_id` | Yes | `Resources_RoutingRule_Update` | Update routing condition |
| `DELETE` | `/:rule_id/conditions/:cond_id` | Yes | `Resources_RoutingRule_Delete` | Delete routing condition |

**Idempotency:**
- GET — Idempotent
- POST (create, swap) — Not idempotent (swap), Not idempotent (create)
- PUT — Idempotent
- DELETE — Idempotent

**Purpose:**
Configure intelligent order routing rules with conditions to direct orders to different liquidity providers or execution methods.

---

## 19. Reports

**Module:** Report Generation & Management
**Base Path:** `/api/v1/reports`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Reports_Read` | List all generated reports |
| `POST` | `/` | Yes | `Resources_Reports_Create` | Generate a new report |
| `GET` | `/:id` | Yes | `Resources_Reports_Read` | Get report metadata |
| `GET` | `/:id/file` | Yes | `Resources_Reports_Read` | Download report file |
| `DELETE` | `/:id` | Yes | `Resources_Reports_Delete` | Delete a report |

**Idempotency:**
- GET — Idempotent
- POST — Not idempotent
- DELETE — Idempotent

**Purpose:**
Generate, view, and manage custom reports (trading activity, account statements, P&L, etc.).

---

## 20. News

**Module:** News Feed
**Base Path:** `/api/v1/news`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/feed` | Yes | `Resources_News_Read` | Get market news feed |

**Idempotency:** GET — Idempotent

**Purpose:**
Access real-time financial news and market updates.

---

## 21. Configs

**Module:** Configuration Management
**Base Path:** `/api/v1/configs`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Config_Read` | List all system configurations |
| `GET` | `/:config_id` | Yes | `Resources_Config_Read` | Get specific configuration |
| `PATCH` | `/:config_id` | Yes | `Resources_Config_Update` | Update configuration value |
| `GET` | `/groups/:group_id` | Yes | `Resources_Config_Read` | Get configurations for a group |

**Idempotency:**
- GET — Idempotent
- PATCH — Idempotent

**Purpose:**
Manage system-wide and group-specific configuration parameters.

---

## 22. Symbols

**Module:** Trading Instruments Management
**Base Path:** `/api/v1/symbols`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints Overview

This module is divided into sub-sections:

#### 22.1 Symbol Viewing

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Symbols_Read` | List all trading symbols |
| `GET` | `/tree` | Yes | `Resources_Symbols_Read` | Get symbols in tree structure |
| `GET` | `/me` | Yes | `Resources_MySymbols_Read` | Get symbols available to current user |
| `GET` | `/me/tree` | Yes | `Resources_MySymbols_Read` | Get user symbols in tree structure |
| `GET` | `/me/by_name` | Yes | `Resources_MySymbols_Read` | Get user's symbol by name |
| `GET` | `/:symbol_id` | Yes | `Resources_Symbols_Read` | Get specific symbol details |
| `GET` | `/accounts/:account_id` | Yes | `Resources_Symbols_Read` | Get symbols available to account |

#### 22.2 Symbol Management

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/` | Yes | `Resources_Symbols_Create` | Create new trading symbol |
| `PUT` | `/:symbol_id` | Yes | `Resources_Symbols_Update` | Update symbol parameters |
| `DELETE` | `/:symbol_id` | Yes | `Resources_Symbols_Delete` | Delete a symbol |
| `DELETE` | `/bulk` | Yes | `Resources_Symbols_Delete` | Bulk delete symbols |

#### 22.3 Symbol Classes

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/classes` | Yes | `Resources_SymbolClasses_Create` | Create symbol class (category) |
| `PATCH` | `/classes/:class_id` | Yes | `Resources_SymbolClasses_Update` | Update symbol class |
| `DELETE` | `/classes/:class_id` | Yes | `Resources_SymbolClasses_Delete` | Delete symbol class |
| `DELETE` | `/classes/bulk_delete` | Yes | `Resources_SymbolClasses_Delete` | Bulk delete symbols by class |
| `PUT` | `/classes/bulk_symbols_config_update` | Yes | `Resources_SymbolClasses_Update` | Bulk update symbols in a class |

#### 22.4 Group-Specific Symbols

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/groups/:group_id` | Yes | `Resources_Groups_Symbols` | Get symbols for a group |
| `PUT` | `/groups/:symbol_group_id` | Yes | `Resources_Groups_Symbols` | Update symbol for a group |
| `DELETE` | `/groups/bulk_delete` | Yes | `Resources_Groups_Symbols` | Bulk delete symbols by group |
| `PUT` | `/groups/bulk_symbols_config_update` | Yes | `Resources_Groups_Symbols` | Bulk update group-specific symbols |

#### 22.5 Symbol Inheritance

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/symbol-inheritance/:symbol_group_id` | Yes | `Resources_Symbols_Read` | Get symbol inheritance settings |
| `PATCH` | `/symbol-inheritance/:symbol_group_id` | Yes | `Resources_SymbolClasses_Update` | Update symbol inheritance settings |

**Idempotency:**
- GET — Idempotent
- POST — Not idempotent
- PUT, PATCH — Idempotent
- DELETE — Idempotent

**Purpose:**
Comprehensive management of trading instruments including creation, classification, group-specific configurations, and inheritance rules.

---

## 23. Watchlist

**Module:** Watchlist Management
**Base Path:** `/api/v1/watchlist`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Watchlist_Read` | List all watchlists |
| `GET` | `/tree` | Yes | `Resources_Watchlist_Read` | Get watchlists in tree structure |
| `GET` | `/:watchlist_id` | Yes | `Resources_Watchlist_Read` | Get specific watchlist |
| `POST` | `/` | Yes | `Resources_Watchlist_Create` | Create new watchlist |
| `PATCH` | `/:watchlist_id` | Yes | `Resources_Watchlist_Update` | Update watchlist details |
| `DELETE` | `/:watchlist_id` | Yes | `Resources_Watchlist_Delete` | Delete a watchlist |
| `POST` | `/:watchlist_id/symbols/:symbol_id` | Yes | `Resources_Watchlist_Update` | Add symbol to watchlist |
| `DELETE` | `/:watchlist_id/symbols/:symbol_id` | Yes | `Resources_Watchlist_Update` | Remove symbol from watchlist |

**Idempotency:**
- GET — Idempotent
- POST (create) — Not idempotent
- POST (add symbol) — Idempotent
- PATCH — Idempotent
- DELETE — Idempotent

**Purpose:**
Create and manage custom watchlists for tracking preferred trading instruments.

---

## 24. Alerts

**Module:** Price Alert Management
**Base Path:** `/api/v1/alerts`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Alerts_Read` | List all price alerts |
| `GET` | `/:alert_id` | Yes | `Resources_Alerts_Read` | Get specific alert details |
| `POST` | `/` | Yes | `Resources_Alerts_Create` | Create new price alert |
| `PATCH` | `/:alert_id` | Yes | `Resources_Alerts_Update` | Update alert parameters |
| `DELETE` | `/:alert_id` | Yes | `Resources_Alerts_Delete` | Delete an alert |

**Idempotency:**
- GET — Idempotent
- POST — Not idempotent
- PATCH — Idempotent
- DELETE — Idempotent

**Purpose:**
Set up price-based alerts for market movements and receive notifications.

---

## 25. Emails

**Module:** Internal Email System
**Base Path:** `/api/v1/emails`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

#### 25.1 Admin Email Management

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_Emails_Read` | Admin: List all emails |
| `GET` | `/:id` | Yes | `Resources_Emails_Read` | Admin: Get specific email |
| `POST` | `/` | Yes | `Resources_MyEmails_Create` | Create/send new email |
| `PUT` | `/:id` | Yes | `Resources_MyEmails_Update` | Update email (draft) |
| `DELETE` | `/:id` | Yes | `Resources_MyEmails_Delete` | Delete email |

#### 25.2 User Email Folders

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/me/inbox` | Yes | `Resources_MyEmails_Read` | Get inbox emails |
| `GET` | `/me/outbox` | Yes | `Resources_MyEmails_Read` | Get sent emails |
| `GET` | `/me/draft` | Yes | `Resources_MyEmails_Read` | Get draft emails |
| `GET` | `/me/bin` | Yes | `Resources_MyEmails_Read` | Get deleted emails (trash) |
| `GET` | `/me/:tracking_id` | Yes | `Resources_MyEmails_Read` | Get specific email by tracking ID |

**Idempotency:**
- GET — Idempotent
- POST — Not idempotent
- PUT — Idempotent
- DELETE — Idempotent

**Purpose:**
Internal messaging system with inbox, outbox, drafts, and trash folders for platform communication.

**Note:** SMTP test route is commented out.

---

## 26. Workspace Profile

**Module:** Workspace Profile Management
**Base Path:** `/api/v1/workspace_profile`
**Auth Middleware:** Protect
**Authorization:** None specified (general access)

### Endpoints

| Method | Path | Auth Required | Authorization | Description |
|--------|------|---------------|---------------|-------------|
| `GET` | `/` | Yes | - | List all workspace profiles |
| `GET` | `/:profile_id` | Yes | - | Get specific workspace profile |
| `POST` | `/` | Yes | - | Create new workspace profile |
| `PATCH` | `/:profile_id` | Yes | - | Update workspace profile |
| `DELETE` | `/:profile_id` | Yes | - | Delete workspace profile |

**Idempotency:**
- GET — Idempotent
- POST — Not idempotent
- PATCH — Idempotent
- DELETE — Idempotent

**Purpose:**
Save and manage user workspace layouts, chart configurations, and UI preferences.

---

## 27. Automation

**Module:** Trading Automation Rules
**Base Path:** `/api/v1/automation`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/config` | Yes | `Resources_Automation_Read` | Get automation configuration options |
| `GET` | `/rules` | Yes | `Resources_Automation_Read` | List all automation rules |
| `GET` | `/rules/:rule_id` | Yes | `Resources_Automation_Read` | Get specific automation rule |
| `POST` | `/rules` | Yes | `Resources_Automation_Create` | Create new automation rule |
| `PUT` | `/rules/:rule_id` | Yes | `Resources_Automation_Update` | Update automation rule |
| `DELETE` | `/rules/:rule_id` | Yes | `Resources_Automation_Delete` | Delete automation rule |
| `POST` | `/rules/:rule_id/clone` | Yes | `Resources_Automation_Create` | Clone existing automation rule |
| `PATCH` | `/rules/:rule_id/toggle` | Yes | `Resources_Automation_Update` | Enable/disable automation rule |
| `GET` | `/logs` | Yes | `Resources_Automation_Read` | View automation execution logs |
| `POST` | `/test` | Yes | `Resources_Automation_Read` | Test automation rule without executing |

**Idempotency:**
- GET — Idempotent
- POST (create, clone) — Not idempotent
- POST (test) — Idempotent
- PUT, PATCH — Idempotent
- DELETE — Idempotent

**Purpose:**
Create and manage automated trading rules with triggers, conditions, and actions for systematic trading.

---

## 28. Import/Export

**Module:** Configuration Import/Export
**Base Path:** `/api/v1/import_export`
**Auth Middleware:** None (inherits from v1)
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

#### 28.1 Group & Symbol Configuration

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/import` | Yes | `Resources_Symbols_Import` | Import symbols and groups config |
| `POST` | `/export` | Yes | `Resources_Symbols_Export` | Export symbols and groups config |
| `POST` | `/import/:group_id` | Yes | `Resources_Symbols_Import` | Import configs for specific group |

#### 28.2 Historical Data

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/import/single_file/symbol_history` | Yes | `Resources_Symbols_Import` | Import single symbol history file |
| `POST` | `/import/folder/symbol_history` | Yes | `Resources_Symbols_Import` | Import folder of symbol histories |
| `GET` | `/export_history` | Yes | `Resources_Symbols_Export` | Export historical data |
| `GET` | `/import/history` | Yes | `Resources_Symbols_Import` | View import history |
| `GET` | `/import/folder/history` | Yes | `Resources_Symbols_Import` | View folder import history |
| `GET` | `/demo/csv` | Yes | `Resources_Symbols_Import` | Download demo CSV template |

**Idempotency:**
- GET — Idempotent
- POST — Not idempotent

**Purpose:**
Bulk import/export of symbol configurations, group settings, and historical price data with CSV templates.

---

## 29. Data Providers

**Module:** Market Data Feed Providers
**Base Path:** `/api/v1/data-providers`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints Overview

#### 29.1 Provider Management

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_DataProviders_Read` | List all data providers |
| `GET` | `/:id` | Yes | `Resources_DataProviders_Read` | Get specific data provider |
| `POST` | `/` | Yes | `Resources_DataProviders_Create` | Create new data provider |
| `PUT` | `/:id` | Yes | `Resources_DataProviders_Update` | Update data provider |
| `DELETE` | `/:id` | Yes | `Resources_DataProviders_Delete` | Delete data provider |

#### 29.2 CSV Operations

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/:id/csv/upload` | Yes | `Resources_DataProviders_Update` | Upload provider symbol mapping CSV |
| `GET` | `/:id/csv/download` | Yes | `Resources_DataProviders_Read` | Download provider CSV configuration |

#### 29.3 Provider Control

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `POST` | `/:id/activate` | Yes | `Resources_DataProviders_Update` | Activate data provider connection |
| `POST` | `/:id/deactivate` | Yes | `Resources_DataProviders_Update` | Deactivate data provider |
| `GET` | `/:id/status` | Yes | `Resources_DataProviders_Read` | Get provider connection status |
| `POST` | `/:id/sync-datasource` | Yes | `Resources_DataProviders_Update` | Sync data source from provider |

#### 29.4 Configuration Management

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/:id/configs` | Yes | `Resources_DataProviders_Read` | Get provider configurations |
| `PUT` | `/:id/configs` | Yes | `Resources_DataProviders_Update` | Update provider configurations |
| `PATCH` | `/:id/config/file` | Yes | `Resources_DataProviders_Update` | Update provider config file |
| `GET` | `/config-template` | Yes | `Resources_DataProviders_Read` | Get configuration template |

#### 29.5 Symbol Management (for vfxmarket provider)

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/:id/symbols` | Yes | `Resources_DataProviders_Read` | Get provider's symbols |
| `POST` | `/:id/symbols` | Yes | `Resources_DataProviders_Update` | Add symbols to provider |
| `DELETE` | `/:id/symbols` | Yes | `Resources_DataProviders_Update` | Clear all provider symbols |
| `PATCH` | `/:id/symbols/:symbol_id` | Yes | `Resources_DataProviders_Update` | Update specific symbol mapping |
| `DELETE` | `/:id/symbols/:symbol_id` | Yes | `Resources_DataProviders_Update` | Remove specific symbol |

**Idempotency:**
- GET — Idempotent
- POST (create, upload) — Not idempotent
- POST (activate, deactivate, sync) — Idempotent
- PUT, PATCH — Idempotent
- DELETE — Idempotent

**Purpose:**
Manage market data feed providers (e.g., vfxmarket) including connection setup, symbol mapping, activation, and data synchronization.

---

## 30. LP Providers

**Module:** Liquidity Provider Management
**Base Path:** `/api/v1/lp-providers`
**Auth Middleware:** Protect
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints Overview

#### 30.1 LP Provider Management

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/` | Yes | `Resources_DataProviders_Read` | List all LP providers |
| `GET` | `/routing/selection` | Yes | `Resources_RoutingRule_Read` | Get LP providers for routing rules |
| `GET` | `/:id` | Yes | `Resources_DataProviders_Read` | Get specific LP provider |
| `POST` | `/` | Yes | `Resources_DataProviders_Create` | Create new LP provider |
| `PUT` | `/:id` | Yes | `Resources_DataProviders_Update` | Update LP provider |
| `DELETE` | `/:id` | Yes | `Resources_DataProviders_Delete` | Delete LP provider |
| `POST` | `/swap-priority` | Yes | `Resources_DataProviders_Create` | Change LP provider priority order |

#### 30.2 Configuration & Activation

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `PATCH` | `/:id/config/file` | Yes | `Resources_DataProviders_Update` | Update LP provider config file |
| `POST` | `/:id/activate` | Yes | `Resources_DataProviders_Update` | Activate LP connection |
| `POST` | `/:id/deactivate` | Yes | `Resources_DataProviders_Update` | Deactivate LP connection |
| `GET` | `/:id/status` | Yes | `Resources_DataProviders_Read` | Get LP connection status |

#### 30.3 Config Management

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/:id/configs` | Yes | `Resources_DataProviders_Read` | Get LP provider configurations |
| `PUT` | `/:id/configs` | Yes | `Resources_DataProviders_Update` | Update LP configurations |
| `GET` | `/config-template` | Yes | `Resources_DataProviders_Read` | Get LP config template |

**Idempotency:**
- GET — Idempotent
- POST (create, swap) — Not idempotent (swap), Not idempotent (create)
- POST (activate, deactivate) — Idempotent
- PUT, PATCH — Idempotent
- DELETE — Idempotent

**Purpose:**
Manage liquidity provider connections for order execution and routing with priority-based selection.

---

## 31. Market Feed

**Module:** Market Data & History
**Base Path:** `/api/v1/market`
**Auth Middleware:** None (inherits from v1)
**Authorization Middleware:** `Authorization(rbac)`

### Endpoints

| Method | Path | Auth Required | Authorization Resource | Description |
|--------|------|---------------|------------------------|-------------|
| `GET` | `/history` | Yes | `Resources_MySymbols_MarketHistory` | Get historical market data (OHLC) |
| `GET` | `/datasource` | Yes | `Resources_Symbols_Read` | Get available market data sources |

**Idempotency:** GET — Idempotent

**Purpose:**
Access historical market data (candlestick/OHLC data) and view available data sources.

**Note:** Market control routes (close, open, restart) are commented out.

---

## 32. Public APIs

**Module:** Public Access Endpoints
**Base Path:** `/api/v1/public`
**Auth Middleware:** None
**Authorization:** None (publicly accessible)

### Endpoints

| Method | Path | Auth Required | Description |
|--------|------|---------------|-------------|
| `GET` | `/reports/:id` | No | Download public report by ID |
| `GET` | `/broker` | No | Get public broker information |
| `POST` | `/verify-unique-username` | No | Check if username is available |
| `POST` | `/account` | No | Open new account (registration) |
| `POST` | `/forgot-password` | No | Request password reset (sends code) |
| `POST` | `/verify-forgot-password-code` | No | Verify password reset code |
| `POST` | `/reset-password` | No | Reset password with verified code |

**Idempotency:**
- GET — Idempotent
- POST — Not idempotent

**Purpose:**
Public-facing APIs for account registration, password recovery, and broker information without authentication.

---

## Additional Routes

### Static & Documentation Routes

**Base Path:** `/` (root)

| Path | Auth Required | Description |
|------|---------------|-------------|
| `/swagger/*` | Yes (ProtectStatic) | Swagger API documentation |
| `/trader/docs` | Yes (ProtectStatic) | Trader documentation |
| `/admin/docs` | Yes (ProtectStatic + Admin) | Admin documentation |
| `/static/*` | No | Static assets |
| `/others/*` | No | Other public files |
| `/icons/*` | No | Icon files |
| `/` | No | Web application (SPA) |
| `*` (fallback) | No | Serves index.html for SPA routing |

---

## Global Middlewares Summary

### Applied to All API Routes (`/api/*`)
- **UserAgentParser** — Parses user agent information
- **HeaderReader** — Reads and processes request headers
- **RequestsLogger** — Logs all incoming requests

### Applied to WebSocket Routes (`/ws/*`)
- Same as API routes plus:
- **Protect** — Requires authentication

### Applied to System Routes (`/system/*`)
- **Protect** — Requires authentication
- **Authorization(rbac)** — Role-based access control per endpoint

### Applied to Business Routes (`/v1/*`)
- Varies by module, typically **Protect** + **Authorization(rbac)**

---

## HTTP Method Idempotency Summary

| Method | Idempotent | Description |
|--------|------------|-------------|
| `GET` | Yes | Safe to retry, no side effects |
| `PUT` | Yes | Updates are idempotent |
| `PATCH` | Yes | Partial updates are idempotent |
| `DELETE` | Yes | Safe to retry, same result |
| `POST` | No* | Creates new resources, not safe to retry |

**Note:** Some POST operations like cancellation, closing, and disconnection are functionally idempotent despite using POST.

---

## Common Response Codes

| Code | Description |
|------|-------------|
| `200` | Success |
| `201` | Created |
| `400` | Bad Request |
| `401` | Unauthorized |
| `403` | Forbidden |
| `404` | Not Found |
| `500` | Internal Server Error |

---

## Notes for Developers

1. **Authentication:** Most endpoints require Bearer token in `Authorization` header
2. **RBAC:** Role-based permissions are strictly enforced via authorization middleware
3. **Rate Limiting:** Consider implementing rate limiting for production
4. **Pagination:** Check specific endpoints for pagination parameters
5. **Filtering:** Many list endpoints support query parameter filtering
6. **WebSocket:** Maintain persistent connection for real-time updates
7. **Bulk Operations:** Use bulk endpoints for efficiency when performing mass operations
8. **Commented Routes:** Some routes are disabled in the current version (see code comments)

---

**Document Version:** 1.0
**Last Updated:** 2025
**Maintainer:** Development Team
