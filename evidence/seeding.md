# Database Seeding Documentation

## Overview

This document describes the database seeding process for the VFX Trading Server. The seeding script initializes the database with essential data required for the system to function properly.

**File Location:** [data/seed_data.go](data/seed_data.go)

## Table of Contents

1. [Roles](#1-roles)
2. [Casbin Authorization Rules](#2-casbin-authorization-rules)
3. [System Configuration](#3-system-configuration)
4. [Broker Information](#4-broker-information)
5. [Trading Symbols](#5-trading-symbols)
6. [Groups](#6-groups)
7. [User Accounts](#7-user-accounts)
8. [Automation Rules](#8-automation-rules)
9. [Data Providers](#9-data-providers)

---

## 1. Roles

Creates six predefined user roles with different permission levels.

### Seeded Roles

| Role Name | Type | Description |
|-----------|------|-------------|
| **Admin** | Admin | Full system administration access |
| **Dealer** | Dealer | Back-office operations and client management |
| **Trader** | Trader | Standard trading account with full trading capabilities |
| **Investor** | Trader | Read-only trading account (view-only access) |
| **API_Trader** | Trader | Programmatic trading via API |
| **API_Admin** | Admin | Programmatic administration via API |

### Example

```go
Role{
    Desc:     "Admin",
    Original: true,
    RoleType: model.RoleType_Admin,
}
```

---

## 2. Casbin Authorization Rules

Implements role-based access control (RBAC) using Casbin policies.

### Permission Structure

Permissions follow the format: `Resource_Action`

**Example:** `Accounts_Create`, `Trades_Manage`, `MyOrders_Read`

### Role Permissions Summary

#### Admin Permissions (68 rules)
- Full system access
- All configuration management
- All account operations
- All trading operations
- System monitoring and logs
- Reports and analytics

**Key Permissions:**
```
- Config_Read, Config_Update
- Symbols_Create, Symbols_Update, Symbols_Delete
- Accounts_Create, Accounts_Delete, Accounts_Money
- Trades_Manage
- Groups_Create, Groups_Delete
- OnlineSessions_Disconnect
```

#### Dealer Permissions (42 rules)
- Client management
- Group management
- Account operations within their group
- Trading oversight
- Reports generation

**Key Permissions:**
```
- Accounts_Create, Accounts_Update, Accounts_Money
- Groups_Create, Groups_Update
- Trades_Manage
- DealingRoom_Manage
- Dealing_ViewGroup (group-restricted)
- OnlineSessions_GroupOnly
```

#### Trader Permissions (24 rules)
- Personal trading operations
- Portfolio management
- Alerts and watchlists
- Account settings

**Key Permissions:**
```
- MyOrders_Create, MyOrders_Update, MyOrders_Cancel
- MyPositions_Update, MyPositions_Close
- MyAccount_ChangePassword
- Alerts_Create, Alerts_Delete
- Watchlist_Create, Watchlist_Delete
```

#### Investor Permissions (11 rules)
- Read-only access
- View positions and orders
- View account information
- No trading capabilities

**Key Permissions:**
```
- MyAccount_Read
- MyOrders_Read
- MyPositions_Read
- MyDeals_Read
- Reports_Read
```

#### API_Trader Permissions (18 rules)
Similar to Trader but via API access only.

#### API_Admin Permissions (40 rules)
Administrative operations via API (subset of Admin permissions).

---

## 3. System Configuration

Seeds essential system-wide configuration parameters.

### Configuration Keys

| Key | Type | Default Value | Description |
|-----|------|---------------|-------------|
| **SystemBrokerSetup** | int32 | 1 | Broker setup completion flag |
| **SystemEndOfDay** | string | "23:59" | End of trading day time |
| **SystemMaxOrdersAndPositions** | int32 | 250 | Maximum orders/positions per account |
| **SystemDefaultGroupForDemoAccounts** | int32 | 24100001 | Default group ID for demo accounts |
| **SystemDefaultGroupForLiveAccounts** | int32 | 24100001 | Default group ID for live accounts |
| **SystemAutoBrokerMarketOrders** | int32 | 0 | Auto-broker market orders (0=disabled) |
| **SystemAutoBrokerPendingOrders** | int32 | 0 | Auto-broker pending orders (0=disabled) |
| **SystemAutoBrokerLiquidationOrders** | int32 | 0 | Auto-broker liquidation orders (0=disabled) |
| **SystemChattingPassOrderAfter** | int32 | 30 | Seconds before chatting passes order |
| **SystemChattingTimeoutAfter** | int32 | 30 | Seconds before chatting timeout |
| **SystemSystemChattingTimeoutAction** | int32 | 0 | Action on chatting timeout |
| **SystemLpMultiplyAmountWithContractSize** | bool | false | Multiply LP amount with contract size |
| **SystemEnableProp** | bool | from config | Enable proprietary trading features |

### Example

```go
Config{
    Desc:        "SystemEndOfDay",
    ValueType:   model.ValueType_string,
    ConfigClass: model.ConfigClass_System,
    Value:       "23:59",
}
```

---

## 4. Broker Information

Creates the default broker profile with company information.

### Broker Details

```go
Broker{
    BusinessName: "HS Trader (Demo System)",
    Address:      "Amman, Jordan",
    Country:      "Jordan",
    Tel:          "+962798726136",
    Website:      "https://www.hybridsolutions.com",
    Facebook:     "https://www.facebook.com/VertexFXTrader",
    Twitter:      "https://twitter.com/VertexFXTrader",
    Instagram:    "https://www.instagram.com/vertexfxtrader",
    LinkedIn:     "https://www.linkedin.com/company/hybrid-solutions-hs-",
    Youtube:      "https://www.youtube.com/c/hybridsolutions",
}
```

---

## 5. Trading Symbols

Seeds trading instruments organized by asset classes.

### Asset Classes and Symbols

#### 5.1 Forex

##### Majors (5 pairs)
- **EURUSD** - Euro vs US Dollar
- **GBPUSD** - British Pound vs US Dollar
- **USDCAD** - US Dollar vs Canadian Dollar
- **USDJPY** - US Dollar vs Japanese Yen
- **USDCHF** - US Dollar vs Swiss Franc

##### Minors (4 pairs)
- **EURAUD** - Euro vs Australian Dollar
- **AUDJPY** - Australian Dollar vs Japanese Yen
- **AUDNZD** - Australian Dollar vs New Zealand Dollar
- **AUDUSD** - Australian Dollar vs US Dollar

##### Exotics (1 pair)
- **USDTUR** - US Dollar vs Turkish Lira

#### 5.2 Metals (5 symbols)
- **GOLD** - Gold vs US Dollar
- **SILVER** - Silver vs US Dollar
- **PALLADIUM** - Palladium vs US Dollar
- **PLATINUM** - Platinum vs US Dollar
- **COPPER** - Copper vs US Dollar

#### 5.3 Crypto (4 symbols)
- **Bitcoin** - Bitcoin vs US Dollar
- **Ethereum** - US Dollar vs Ethereum
- **ETHXBT** - ETHXBT vs US Dollar
- **Ripple** - Ripple vs US Dollar

#### 5.4 FIX Integration (1 symbol)
- **FIX_EURUSD** - EUR vs US Dollar (via FIX protocol)

#### 5.5 Demo/Simulator (3 symbols)
- **DEMO_EURUSD** - Demo Euro vs US Dollar
- **DEMO_USDJPY** - Demo US Dollar vs Japanese Yen
- **DEMO_GBPUSD** - Demo British Pound vs US Dollar

### Symbol Configuration Example

```go
Symbol{
    Source:          "dde",
    DataSource:      "EUR A0-FX",
    Symbol:          "EURUSD",
    SymbolMap:       "EURUSD",
    ISIN:            "1",
    Desc:            "Euro vs US Dollar",
    BaseCurrency:    "EUR",
    QuoteCurrency:   "USD",
    Enabled:         true,
    Digits:          5,
    ContractSize:    1000,
    QuoteSessions:   "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00",
    TradeSessions:   "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00",
    Orders:          "0,1,2,3,4,5",
    Expiration:      "0,1,2,3",
    Filling:         model.FillPolicy_fill_or_kill,
    MinValue:        0.1,
    MaxValue:        100,
    Timeout:         30,
    MarginBuy:       1,
    MarginSell:      1,
    MaintenanceBuy:  1,
    MaintenanceSell: 1,
}
```

**Total Symbols:** 18 trading instruments across 5 asset classes

---

## 6. Groups

Creates two default user groups for organizing accounts.

### Default Groups

#### Traders Group
```go
Group{
    Root:      true,
    Desc:      "Traders",
    ManagerId: 24100001,
    GroupType: model.GroupType_client_group,
}
```
- Client-facing group
- Contains trader accounts
- Has associated symbols and configurations

#### Dealers Group
```go
Group{
    Root:      true,
    Desc:      "Dealers",
    GroupType: model.GroupType_admin_group,
}
```
- Administrative group
- Contains dealer and admin accounts
- Has associated symbols and configurations

### Group Initialization

Each group is initialized with:
- Symbol permissions (via `SeedSymbolsForGroup`)
- Configuration settings (via `SeedConfigsForGroup`)

---

## 7. User Accounts

Seeds three default user accounts for immediate system access.

### Default Accounts

#### Admin Account
```go
Account{
    UserId:       "admin",
    UserPassword: "123" (encrypted),
    RoleId:       roleAdmin.Id,
    GroupId:      dealersGroup.Id,
    AccountType:  model.AccountType_admin_account,
    Status:       model.AccountStatus_active,
    Identity: {
        FirstName:   "Admin",
        FamilyName:  "HS",
        Email:       "saif.h@hybridsolutions.com",
        Telephone:   "+962798726136",
        AddressLine: "Amman, Jordan",
    }
}
```
- **Username:** `admin`
- **Password:** `123`
- **Role:** Admin
- **Permissions:** Full system access

#### Dealer Account
```go
Account{
    UserId:       "dealer",
    UserPassword: "123" (encrypted),
    RoleId:       roleDealer.Id,
    GroupId:      dealersGroup.Id,
    AccountType:  model.AccountType_admin_account,
    Status:       model.AccountStatus_active,
    Identity: {
        FirstName:   "Dealer",
        FamilyName:  "HS",
        Email:       "saif.h@hybridsolutions.com",
        Telephone:   "+962798726136",
        AddressLine: "Amman, Jordan",
    }
}
```
- **Username:** `dealer`
- **Password:** `123`
- **Role:** Dealer
- **Permissions:** Client management and dealing room

#### Trader Account
```go
Account{
    UserId:       "trader",
    UserPassword: "123" (encrypted),
    RoleId:       roleTrader.Id,
    GroupId:      tradersGroup.Id,
    AccountType:  model.AccountType_client_account,
    Status:       model.AccountStatus_active,
    Balance:      1000000,
    Equity:       1000000,
    FreeMargin:   1000000,
    UsedMargin:   0,
    Credit:       10000,
    Leverage:     100000,
    Identity: {
        FirstName:   "Trader",
        FamilyName:  "HS",
        Email:       "saif.h@hybridsolutions.com",
        Telephone:   "+962798726136",
        AddressLine: "Amman, Jordan",
    }
}
```
- **Username:** `trader`
- **Password:** `123`
- **Role:** Trader
- **Starting Balance:** $1,000,000
- **Credit:** $10,000
- **Leverage:** 1:100,000

### Security Note

All default passwords are `123` (encrypted using oauth2.EncryptPassword). **Change these passwords immediately in production environments.**

---

## 8. Automation Rules

Seeds 7 automation rule templates for common trading scenarios.

### Automation Templates

#### 8.1 Low Balance Protection
```go
AutoRule{
    Name:        "Low Balance Protection",
    Enabled:     false,
    TriggerType: model.TriggerType_account_balance,
    Period:      model.PeriodType_minute,
    Conditions: [{
        Field:    "accounts.balance",
        Operator: "<",
        Value:    "100",
    }],
    Actions: [{
        ActionType: model.ActionType_disable_trading,
        Parameters: {
            "reason": "Account balance below minimum threshold",
        }
    }]
}
```
**Purpose:** Automatically disable trading when account balance falls below $100

#### 8.2 Margin Call Alert
```go
AutoRule{
    Name:        "Margin Call Alert",
    Enabled:     false,
    TriggerType: model.TriggerType_account_margin,
    Period:      model.PeriodType_minute,
    Conditions: [{
        Field:    "accounts.margin_level",
        Operator: "<",
        Value:    "100",
    }],
    Actions: [{
        ActionType: model.ActionType_send_email,
        Parameters: {
            "subject":  "Margin Call Warning",
            "template": "margin_call",
            "body":     "Your margin level is {{margin_level}}%. Please add funds to avoid liquidation.",
        }
    }]
}
```
**Purpose:** Send email alert when margin level falls below 100%

#### 8.3 Welcome Email
```go
AutoRule{
    Name:        "Welcome Email",
    Enabled:     false,
    TriggerType: model.TriggerType_new_account,
    Period:      model.PeriodType_once,
    Actions: [{
        ActionType: model.ActionType_send_email,
        Parameters: {
            "subject":  "Welcome to Our Trading Platform",
            "template": "welcome",
            "body":     "Welcome {{user_id}}! Your account has been created successfully.",
        }
    }]
}
```
**Purpose:** Send welcome email to new account registrations

#### 8.4 First Deposit Bonus
```go
AutoRule{
    Name:        "First Deposit Bonus",
    Enabled:     false,
    TriggerType: model.TriggerType_new_account,
    Period:      model.PeriodType_once,
    Conditions: [{
        Field:    "accounts.balance",
        Operator: ">",
        Value:    "0",
    }],
    Actions: [{
        ActionType: model.ActionType_add_credit,
        Parameters: {
            "amount": 50,
            "reason": "First deposit bonus",
        }
    }]
}
```
**Purpose:** Add $50 credit bonus for first-time deposits

#### 8.5 Stop Out Protection
```go
AutoRule{
    Name:        "Stop Out Protection",
    Enabled:     false,
    TriggerType: model.TriggerType_account_margin,
    Period:      model.PeriodType_minute,
    Conditions: [{
        Field:    "accounts.margin_level",
        Operator: "<=",
        Value:    "50",
    }],
    Actions: [
        {
            ActionType: model.ActionType_close_positions,
            Parameters: {
                "reason": "Stop out protection - margin level critical",
            }
        },
        {
            ActionType: model.ActionType_send_push,
            Parameters: {
                "title":   "Stop Out Executed",
                "message": "Your positions have been closed due to insufficient margin",
            }
        }
    ]
}
```
**Purpose:** Automatically close all positions when margin level ≤ 50% and notify trader

#### 8.6 High Profit Alert
```go
AutoRule{
    Name:        "High Profit Alert",
    Enabled:     false,
    TriggerType: model.TriggerType_position_closed,
    Period:      model.PeriodType_once,
    Conditions: [{
        Field:    "positions.profit",
        Operator: ">",
        Value:    "1000",
    }],
    Actions: [{
        ActionType: model.ActionType_send_push,
        Parameters: {
            "title":   "Great Trade!",
            "message": "You just made {{profit}} profit on {{symbol}}",
        }
    }]
}
```
**Purpose:** Send notification for profitable trades exceeding $1000

#### 8.7 Daily Trading Summary
```go
AutoRule{
    Name:        "Daily Trading Summary",
    Enabled:     false,
    TriggerType: model.TriggerType_time,
    StartedAt:   1704067200000, // Jan 1, 2024
    Period:      model.PeriodType_day,
    Hours:       23,
    Minutes:     59,
    Actions: [{
        ActionType: model.ActionType_send_email,
        SendBy:     model.SendByType_group,
        Parameters: {
            "subject":  "Daily Trading Summary",
            "template": "daily_summary",
            "body":     "Your daily trading summary is ready.",
            "group_id": clientGroup.Id,
        }
    }]
}
```
**Purpose:** Send daily trading summary email at 23:59 to all traders

### Important Notes

- All automation rules are created with `Enabled: false` by default
- Rules must be manually enabled through the admin interface
- Rules are marked as `RecordType_seed` to identify them as templates
- Supports variable interpolation (e.g., `{{margin_level}}`, `{{profit}}`)

---

## 9. Data Providers

Seeds three data provider configurations for market data feeds.

### Provider Types

#### 9.1 DDE Provider
```go
DataProvider{
    Name:         "dde",
    DisplayName:  "DDE Provider",
    ProviderType: model.DataProviderType_dde,
    Status:       model.DataProviderStatus_inactive,
    Priority:     1,
    IsActive:     false,
    Description:  "DDE market data provider for real-time price feeds",
}
```

**Credentials:**
- `host` - DDE server host address
- `port` - DDE server port
- `token` - DDE authentication token

**Purpose:** Real-time market data via DDE (Dynamic Data Exchange) protocol

#### 9.2 F4B FIX Provider
```go
DataProvider{
    Name:         "fix_f4b44",
    DisplayName:  "F4B FIX Provider",
    ProviderType: model.DataProviderType_fix,
    Status:       model.DataProviderStatus_inactive,
    Priority:     2,
    IsActive:     false,
    Description:  "FIX protocol provider for F4B liquidity",
}
```

**Credentials:**
- `username` - FIX username
- `password` - FIX password

**FIX Configuration (replaces .cfg file):**

**DEFAULT Section:**
- `ConnectionType` = "initiator"
- `HeartBtInt` = 30 seconds
- `EncryptMethod` = 0 (None)
- `SocketConnectHost` = (empty, configure via admin)
- `SocketConnectPort` = (empty, configure via admin)
- `ReconnectInterval` = 5 seconds
- `CheckLatency` = N
- `ResetOnLogon` = Y
- `TimeStampPrecision` = MILLIS

**SESSION Section:**
- `BeginString` = "FIX.4.4"
- `SenderCompID` = (empty, configure via admin)
- `TargetCompID` = (empty, configure via admin)

**Purpose:** Institutional liquidity via FIX 4.4 protocol

#### 9.3 FIX Simulator
```go
DataProvider{
    Name:         "fix_simulator44",
    DisplayName:  "FIX Simulator",
    ProviderType: model.DataProviderType_fix,
    Status:       model.DataProviderStatus_inactive,
    Priority:     3,
    IsActive:     false,
    Description:  "Simulated FIX provider for testing",
}
```

**Purpose:** Testing and development with simulated FIX data

### Generic Provider Configurations

All providers include these generic configs:
- `heartbeat_interval` = 30 seconds
- `reconnect_delay` = 5 seconds
- `max_reconnect_attempts` = 10

### Configuration Notes

- All providers start as **inactive** and must be configured and activated manually
- Credentials are **not encrypted** in the database (consider encryption for production)
- FIX configuration is stored in the database (no .cfg file needed)
- Priority determines data feed precedence when multiple providers are active

---

## Running the Seeder

### Prerequisites

1. MySQL database running
2. Configuration file set up
3. Database migrations completed

### Execution

```bash
# From the project root
cd data
go run seed_data.go
```

### What Happens

The seeder runs in a **transaction** with the following sequence:

1. ✅ Seed Roles
2. ✅ Seed Casbin Rules (Authorization)
3. ✅ Seed System Configuration
4. ✅ Seed Broker Information
5. ✅ Seed Trading Symbols
6. ✅ Seed Groups (with configs and symbols)
7. ✅ Seed User Accounts
8. ✅ Seed Automation Rules
9. ✅ Seed Data Providers

If any step fails, the entire transaction is **rolled back**.

### Success Output

```
data seeded successfully
```

---

## Post-Seeding Tasks

After running the seeder:

### 1. Change Default Passwords
```sql
UPDATE accounts SET user_password = '<new_encrypted_password>' WHERE user_id IN ('admin', 'dealer', 'trader');
```

### 2. Configure Data Providers
- Navigate to Admin > Data Providers
- Add credentials for DDE or FIX providers
- Configure connection settings
- Activate desired providers

### 3. Enable Automation Rules
- Navigate to Admin > Automation
- Review seeded automation templates
- Customize parameters as needed
- Enable desired rules

### 4. Update Broker Information
- Navigate to Admin > Broker Setup
- Update company details
- Add logo and branding

### 5. Customize System Configuration
- Navigate to Admin > System Config
- Adjust end-of-day time
- Configure auto-brokering settings
- Set limits and timeouts

---

## Database Schema References

### Main Tables Seeded

- `roles` - User role definitions
- `casbin_rule` - Authorization policies
- `configs` - System configuration
- `brokers` - Broker information
- `symbol_classes` - Symbol hierarchy
- `symbols` - Trading instruments
- `groups` - Account groups
- `accounts` - User accounts
- `auto_rules` - Automation templates
- `data_providers` - Market data sources
- `data_provider_credentials` - Provider authentication
- `data_provider_configs` - Provider settings

---

## Troubleshooting

### Common Issues

#### 1. Migration Not Run
**Error:** `Table doesn't exist`

**Solution:** Run migrations first
```bash
# Migrations are run automatically in the seeder via DBSess.Migrate()
```

#### 2. Duplicate Entry Errors
**Error:** `Duplicate entry for key 'PRIMARY'`

**Solution:** Drop and recreate the database, or use a fresh database

#### 3. Foreign Key Constraint Fails
**Error:** `Cannot add or update a child row: a foreign key constraint fails`

**Solution:** Ensure the seeding order is maintained (roles before accounts, groups before accounts, etc.)

#### 4. Config Not Found
**Error:** `Config file not found`

**Solution:** Ensure your config file is in the correct location and properly formatted

---

## Security Considerations

1. **Default Passwords:** All accounts use password `123` - change immediately
2. **Credentials:** Data provider credentials are not encrypted by default
3. **Email Addresses:** Update placeholder email addresses
4. **API Access:** Secure API accounts with strong credentials
5. **Automation Rules:** Review and test automation rules before enabling in production

---

## Summary

The seeding process initializes:
- ✅ **6 User Roles** (Admin, Dealer, Trader, Investor, API Trader, API Admin)
- ✅ **244 Authorization Rules** across all roles
- ✅ **13 System Configuration** parameters
- ✅ **1 Broker** profile
- ✅ **18 Trading Symbols** across 5 asset classes
- ✅ **2 Account Groups** (Traders, Dealers)
- ✅ **3 User Accounts** (admin, dealer, trader)
- ✅ **7 Automation Templates** for common scenarios
- ✅ **3 Data Providers** (DDE, FIX F4B, FIX Simulator)

This provides a complete foundation for a functional trading system ready for configuration and deployment.


