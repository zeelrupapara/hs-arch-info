# HybridiSolutions Hstrader - Third Party Services Documentation

## Overview
This document provides comprehensive details about all notification, payment gateway, email, SMS, and data provider services integrated into the Hstrader application. Each service is documented with integration details, configuration, usage, and trigger mechanisms.

---

## Table of Contents
1. [Email Services](#email-services)
2. [Notification Services](#notification-services)
3. [Payment Gateway Services](#payment-gateway-services)
4. [SMS Services](#sms-services)
5. [Data Provider Services](#data-provider-services)

---

## Email Services

### 1. **SMTP Email Service (Simple Mail Transfer Protocol)**

#### Service Overview
- **Service Name:** SMTP Email Service
- **Integration Type:** Direct SMTP Server Connection
- **Status:** ✅ **ACTIVELY USED**
- **Location:** `/vfxserver/pkg/smtp/`
- **Configuration:** Environment Variables

#### Configuration Details

**Environment Variables Required:**
```
SMTP_HOST        - SMTP Server hostname
SMTP_PORT        - SMTP Server port (465 for SSL, 587 for TLS)
SMTP_FROM        - Sender email address
SMTP_LOGIN       - SMTP login username (used for port 587)
SMTP_PASSWORD    - SMTP password for authentication
```

**Config Structure:**
```go
type SMTP struct {
    Host     string    // SMTP Host
    Port     int32     // SMTP Port
    From     string    // Sender email address
    Login    string    // SMTP login username
    Password string    // SMTP password
    DB       *db.MysqlDB
    Log      *logger.Logger
    Corn     *gocron.Scheduler
    Pool     *smtppool.Pool  // Connection pool
}
```

#### Authentication Mechanism
- **Port 465 (SSL):** Uses `Plain Auth` with From and Password
- **Port 587 (TLS):** Uses `Plain Auth` with Login and Password
- **TLS Configuration:** InsecureSkipVerify enabled, custom ServerName

#### Connection Management
- **Connection Pool:** Uses `smtppool` library
- **Max Connections:** 10
- **Idle Timeout:** 10 seconds
- **Pool Wait Timeout:** 5 seconds
- **Thread Safety:** RWMutex protected

#### Features Implemented

##### Email Sending Functionality
1. **Email Queueing System**
   - Emails are queued in database with status `queue`
   - Processed asynchronously every 1 second
   - Batch sending with error handling

2. **Email Types and Templates**
   - **Report Email:** Account statement reports
   - **Demo Account Email:** Demo account approval notifications
   - **Password Change Email:** Password change verification codes
   
3. **Template System**
   - HTML template-based emails
   - Dynamic placeholder replacement:
     - `{{business_name}}` - Broker business name
     - `{{account_name}}` - Account holder name
     - `{{website}}` - Broker website URL
     - `{{play_store}}` - Google Play Store link
     - `{{app_store}}` - Apple App Store link
     - `{{facebook}}`, `{{instagram}}`, `{{twitter}}`, `{{linkedin}}` - Social media links

#### Database Integration
**Table:** `vfx_Mail`

**Email Status Tracking:**
```go
type MailStatus int32

const (
    MailStatus_queue   = 0  // Waiting to be sent
    MailStatus_sent    = 1  // Successfully sent
    MailStatus_failed  = 2  // Failed to send
    MailStatus_bounce  = 3  // Email bounced
)
```

**Email Types:**
```go
type MailType int32

const (
    MailType_smtp   = 0  // SMTP/External email
    MailType_inemail = 1 // In-app email system
    MailType_chat   = 2  // Chat/messaging system
)
```

#### Use Cases & Triggers

| Use Case | Trigger Event | Recipient | Template |
|----------|---------------|-----------|----------|
| **Account Report Delivery** | Report generation completed | Account owner email | `report_template.html` |
| **Demo Account Approval** | Admin approves demo account | New trader email | `demo_account_template.html` |
| **Password Reset** | User requests password change | Account email | `change_password_template.html` |
| **Automation Email Action** | Automation rule triggers `send_email` action | Specified recipient or account email | Dynamic HTML content |

#### Health Monitoring
- **Monitor Job:** Runs every 10 seconds
- **Checks:**
  - SMTP connection availability
  - Connection pool health
  - Queue processing status

#### Error Handling
- Failed emails are logged with error details
- Retry mechanism through cron scheduler
- Failed emails marked with status `failed`

#### Code Example: Sending Email
```go
// Queue an email
mail := &model.Mail{
    To:      "user@example.com",
    From:    "", // Uses configured SMTP_FROM
    Subject: "Your Report is Ready",
    Body:    "<html>...</html>",
    Type:    model.MailType_smtp,
    Status:  model.MailStatus_queue,
    OwnerId: accountId,
    AccountId: accountId,
}
err := db.Create(mail).Error
```

---

### 2. **In-App Email System (InEmail)**

#### Service Overview
- **Service Name:** In-App Email System
- **Status:** ✅ **ACTIVELY USED**
- **Location:** `/vfxserver/internal/server/v1/emails.go`
- **Database Table:** `vfx_Mail` (with `type = MailType_inemail`)

#### Functionality

**Email Inbox Management:**
- Accounts can send/receive emails within the application
- Separate inbox and outbox views
- Email tracking with unique tracking IDs
- Soft delete support (deleted flag)

**Endpoints:**
1. **GET /api/v1/emails** - Get all system emails (admin only)
2. **GET /api/v1/emails/me/inbox** - Get user's inbox emails
3. **GET /api/v1/emails/me/outbox** - Get user's sent emails

**Email Deduplication:**
- Uses `EmailTrackingId` to prevent duplicate inbox entries
- Shows only latest email version in inbox

#### Use Cases
- **Internal Communication:** System notifications to users
- **Support Tickets:** Email-based support communication
- **Account Notifications:** Important account updates

---

### 3. **Chat/Messaging System**

#### Service Overview
- **Status:** 🔄 **INFRASTRUCTURE SUPPORT ONLY**
- **Type:** MailType_chat
- **Current Implementation:** Not actively sending chat messages
- **Potential Use:** Real-time chat notifications

---

## Notification Services

### 1. **OneSignal Push Notifications**

#### Service Overview
- **Service Name:** OneSignal Push Notification Service
- **Integration Type:** Third-party Push Service (RESTful API)
- **Status:** ✅ **ACTIVELY USED**
- **Location:** `/vfxcore/pkg/onesignal/`
- **Platforms Supported:** iOS, Android, Windows, Huawei

#### Configuration Details

**Environment Variables Required:**
```
ONE_SIGNAL_APP_ID         - OneSignal Application ID
ONE_SIGNAL_USER_AUTH_KEY  - OneSignal User Authentication Key
ONE_SIGNAL_REST_API_KEY   - OneSignal REST API Key for server requests
```

**Config Structure:**
```go
type OneSignal struct {
    AppID      string  // OneSignal App ID
    AuthKey    string  // User Auth Key
    RestAPIKey string  // REST API Key
}
```

#### API Integration

**Client Wrapper:**
```go
type OneSignal struct {
    config          *config.Config
    oneSignalConfig *onesignal.Configuration
    apiClient       *onesignal.APIClient
}
```

**Authentication:**
- Uses OAuth context with `AppAuth` token
- REST API Key passed in request context
- Configuration managed by OneSignal Go SDK

#### Notification Features

**Notification Structure:**
```go
type Notification struct {
    PlayerIDs []string           // Target device IDs
    Heading   *StringMap         // Multi-language heading
    Content   *StringMap         // Multi-language content
    ImageURL  string            // Notification image
    IsIos     bool              // iOS support
    IsAndroid bool              // Android support
    IsWindows bool              // Windows support
    IsHuawei  bool              // Huawei device support
}
```

**Multi-Language Support:**
- English (En)
- Arabic (Ar)
- Hindi (Hi)
- Same content sent to all language variants

#### Use Cases & Triggers

| Use Case | Trigger Event | Integration Point |
|----------|---------------|-------------------|
| **Automation Push Action** | Automation rule triggers `send_push` action | `vfxcore/handler/automation_actions.go` |
| **Price Alerts** | Market alert conditions met | Alert notification system |
| **Trading Notifications** | Trade executions, order fills | Trading execution system |
| **Account Updates** | Important account changes | Account management system |

#### Code Example: Sending Push Notification
```go
// Create notification
n := onesignal.NewNotification(
    []string{"account_id_1", "account_id_2"}, // Player IDs
    "Trade Executed",                          // Title
    "Your EUR/USD order has been filled",     // Message
    "https://example.com/image.png",          // Image URL
)

// Send through OneSignal
if v.OneSignal != nil {
    err := v.OneSignal.Push(n)
}
```

#### Error Handling
- Validates notification ID in response
- Returns error if notification ID is empty
- Logs failures with account ID context
- Continues with other actions even if push fails

#### Validation Requirements
- **AppID:** Cannot be empty (required)
- **AuthKey:** Cannot be empty (required)
- **RestAPIKey:** Cannot be empty (required)

---

### 2. **In-App Notification System**

#### Service Overview
- **Status:** 🔄 **PLANNED/INFRASTRUCTURE READY**
- **Type:** Real-time in-app notifications
- **Potential Integration:** WebSocket-based notifications

---

## Payment Gateway Services

### **Status: ❌ NOT IMPLEMENTED**

#### Overview
**No payment gateway service is currently integrated into the application.**

#### Architecture Support
- **Payment Fields Present:** Yes
  - `PaymentsType` - Type of payment method (field in Broker model)
  - `NextPayment` - Next payment due date
  - `CurrentPayment` - Current payment amount
  - `DuePayment` - Outstanding payment amount

- **Purpose:** These fields support broker license payment tracking
- **Current Implementation:** Manual payment processing/tracking only

#### Potential Services (Not Integrated)
- Stripe
- Razorpay
- PayPal
- Square

#### API Endpoints (Payment Info Only)
- **PUT /api/v1/brokers** - Update broker payment information
  - Updates: `PaymentsType`, `NextPayment`, `CurrentPayment`, `DuePayment`
  - No automated payment processing

#### Future Considerations
- Payment gateway integration would require:
  1. API credentials configuration
  2. Payment processing handlers
  3. Webhook listeners for payment events
  4. Transaction logging and reconciliation
  5. PCI DSS compliance implementation

---

## SMS Services

### **Status: ❌ NOT IMPLEMENTED**

#### Overview
**No SMS service is currently integrated into the application.**

#### Architecture Support
- **Email Field Present:** Yes
  - User email stored in `Identity` model
  - No phone number field for SMS delivery

#### Database Consideration
- Email field: `Identity.Email`
- No phone/mobile field in current schema

#### Potential Services (Not Integrated)
- Twilio
- AWS SNS
- Firebase Cloud Messaging
- Nexmo/Vonage

#### Implementation Requirements (If Needed)
1. Add phone number field to `Identity` model
2. Phone number validation during registration
3. SMS gateway configuration
4. OTP/SMS template system
5. Delivery tracking

---

## Data Provider Services

### 1. **Market Data Provider System**

#### Service Overview
- **Service Name:** Market Data Feed Provider
- **Status:** ✅ **ACTIVELY USED**
- **Location:** `/vfxmarket/model/common/v1/data_provider.go`
- **Purpose:** Provides real-time and historical market data for trading

#### Supported Data Provider Types

```go
type DataProviderType int32

const (
    DataProviderType_fix       = 0  // FIX Protocol
    DataProviderType_dde       = 1  // DDE (Dynamic Data Exchange)
    DataProviderType_simulator = 2  // Simulator for testing
    DataProviderType_ws        = 3  // WebSocket feed
)
```

#### Data Provider Status

```go
type DataProviderStatus int32

const (
    DataProviderStatus_inactive = 0  // Inactive
    DataProviderStatus_active   = 1  // Active and working
    DataProviderStatus_error    = 2  // Error state
)
```

#### Database Schema

**Table:** `vfx_DataProvider`

**Fields:**
```go
type DataProvider struct {
    Id              int32                    // Unique ID
    Name            string                   // Unique name (e.g., "fix_f4b44")
    DisplayName     string                   // User-friendly name
    ProviderType    DataProviderType         // Type of provider
    Type            string                   // String version for compatibility
    Status          DataProviderStatus       // Current status
    Priority        int32                    // Priority for fallback
    IsActive        bool                     // Active flag
    Description     string                   // Description
    
    // Relations
    Credentials     []DataProviderCredential // Provider credentials
    SymbolCSV       *DataProviderSymbolCSV   // Symbol mappings
    ConnectionConfigs []DataProviderConfig   // Connection settings
}
```

#### Credential Management

**Table:** `vfx_DataProviderCredential`

**Purpose:** Secure storage of provider-specific credentials

**Fields:**
```go
type DataProviderCredential struct {
    Id          int32   // Unique ID
    ProviderId  int32   // Reference to DataProvider
    Key         string  // Credential key (e.g., "sendercompid", "username", "host")
    Value       string  // Credential value
    IsEncrypted bool    // Encryption flag
    Description string  // Credential description
}
```

**Common Credential Keys:**
- FIX Protocol: `sendercompid`, `targetcompid`, `host`, `port`, `username`, `password`
- DDE: `host`, `topic`, `item`, `username`, `password`
- WebSocket: `url`, `apikey`, `secret`

#### Symbol Mapping System

**Table:** `vfx_DataProviderSymbolCSV`

**Purpose:** Manage symbol CSV uploads and mappings

**Fields:**
```go
type DataProviderSymbolCSV struct {
    Id                  int32   // Unique ID
    ProviderId          int32   // Reference to DataProvider
    SymbolCount         int32   // Number of symbols in CSV
    ValidationStatus    string  // pending, validated, failed
    ValidationErrors    string  // Error details if validation fails
    FileName            string  // Original CSV filename
    FileSize            int64   // Size of CSV file
}
```

**Symbol Mapping Table:** `vfx_DataProviderSymbol`

```go
type DataProviderSymbol struct {
    Id              int32   // Unique ID
    ProviderId      int32   // Reference to DataProvider
    ProviderSymbol  string  // Symbol code from provider
    PlatformSymbol  string  // Internal platform symbol
    Description     string  // Symbol description
}
```

#### Connection Configuration

**Table:** `vfx_DataProviderConfig`

**Purpose:** Provider-specific configuration settings

**Fields:**
```go
type DataProviderConfig struct {
    Id              int32   // Unique ID
    ProviderId      int32   // Reference to DataProvider
    ConfigKey       string  // Configuration key
    ConfigValue     string  // Configuration value
    ConfigType      string  // Type (string, int, float, bool)
    Description     string  // Description
}
```

#### Data Provider Events

**Event Types:**
```go
type DataProviderEventType int32

const (
    DataProviderEvent_status          = 0  // Status change
    DataProviderEvent_create          = 1  // Provider created
    DataProviderEvent_update          = 2  // Provider updated
    DataProviderEvent_delete          = 3  // Provider deleted
    DataProviderEvent_activate        = 4  // Provider activated
    DataProviderEvent_deactivate      = 5  // Provider deactivated
    DataProviderEvent_syncsource      = 6  // Sync symbol source
    DataProviderEvent_config_updated  = 7  // Config updated
)
```

**NATS Integration:**
- **Subject:** `system.data_providers`
- **Message Type:** `DataProviderEvent`
- **Listener:** `NatsDataProviderEventListener` in vfxmarket

#### Use Cases & Triggers

| Use Case | Trigger Event | Handler |
|----------|---------------|---------|
| **Market Data Subscription** | Data provider created/activated | vfxmarket handler subscribes |
| **Symbol Mapping** | CSV upload and validation | DataProvider updates symbols |
| **Connection Status** | Provider status changes | Event published to NATS |
| **Fallback Routing** | Primary provider fails | Routes to backup by priority |
| **Configuration Update** | Admin updates credentials | Event triggers reconnection |

#### Real-Time Market Data Flow

**Path:** Data Provider → vfxmarket → vfxcore → Trading Engine

1. **Data Provider connects** (FIX, DDE, WS, Simulator)
2. **Market ticks received** by vfxmarket
3. **Tick data published** via NATS to vfxcore
4. **Positions/Alerts calculated** in real-time
5. **Account balances updated** based on market movements

#### Example: FIX Protocol Provider
- **Type:** FIX (Financial Information Exchange)
- **Protocol:** Binary message-based
- **Credentials Needed:**
  - SenderCompID
  - TargetCompID
  - Host/Port
  - Username/Password
- **Common Providers:** Interactive Brokers, Lightspeed, etc.

#### Example: WebSocket Provider
- **Type:** ws (WebSocket)
- **Protocol:** JSON over WebSocket
- **Credentials Needed:**
  - WebSocket URL
  - API Key
  - API Secret
- **Advantages:** Lower latency, real-time updates
- **Common Providers:** Crypto feeds, Modern forex APIs

#### Market Data in Trading Engine

**Market Data Usage in vfxcore:**
```go
// Market data cache maintained per symbol
MarketData map[int32]*model.Tick

// Used for:
- Position P&L calculation
- Alert trigger evaluation
- Order execution pricing
- Account equity calculation
- Margin level computation
```

#### Monitoring and Health

**Status Checks:**
- Connection status: `active`, `inactive`, `error`
- Symbol count validation
- Credential validation
- Feed latency monitoring
- Data quality checks

---

### 2. **Data Provider CLI Tools (vfxmarket)**

#### Service Overview
- **Location:** `/vfxmarket/` microservice
- **Status:** ✅ **ACTIVELY USED**
- **Features:**
  - Data provider management
  - Connection testing
  - Symbol synchronization
  - Health monitoring

---

## Integration Architecture

### Service Communication Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    HSTRADER APPLICATION                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   vfxserver  │    │   vfxcore    │    │  vfxmarket   │   │
│  │              │    │              │    │              │   │
│  │ • SMTP Mail  │    │ • OneSignal  │    │ • Data Feed  │   │
│  │ • In-Email   │    │ • Alerts     │    │ • Symbols    │   │
│  │ • REST API   │    │ • Automation │    │ • Market Data│   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│        ↓                   ↓                   ↓              │
│     NATS Message Bus (Event-Driven)                          │
│        ↓                   ↓                   ↓              │
│   [Mail Queue]      [Alert Events]     [Data Events]        │
│        │                   │                   │              │
└────────┼───────────────────┼───────────────────┼──────────────┘
         │                   │                   │
         ↓                   ↓                   ↓
    [SMTP Server]     [OneSignal API]    [Data Providers]
```

---

## Configuration Management

### Environment Variables Summary

#### SMTP Configuration
```bash
SMTP_HOST=mail.example.com
SMTP_PORT=587
SMTP_FROM=noreply@example.com
SMTP_LOGIN=username@example.com
SMTP_PASSWORD=password
```

#### OneSignal Configuration
```bash
ONE_SIGNAL_APP_ID=app_id_here
ONE_SIGNAL_USER_AUTH_KEY=auth_key_here
ONE_SIGNAL_REST_API_KEY=rest_api_key_here
```

#### Data Provider Support
```bash
# Configured in application through Admin UI
# Credentials stored encrypted in database
# No environment variables needed (managed via API)
```

---

## Automation System Integration

### Automation Actions Using Services

#### Email Action
```go
ActionType_send_email = "send_email"

// Parameters:
{
    "subject": "Alert Subject",
    "body": "Alert message with {{placeholders}}",
    "to": "user@example.com"  // Optional, uses account email if empty
}

// Flow:
1. Automation rule triggers
2. Email queued to vfx_Mail table
3. SMTP service processes queue every second
4. Email sent via configured SMTP server
```

#### Push Notification Action
```go
ActionType_send_push = "send_push"

// Parameters:
{
    "subject": "Push Title",
    "message": "Push message content",
    "account_id": ["player_id_1", "player_id_2"]
}

// Flow:
1. Automation rule triggers
2. OneSignal client initialized
3. Push notification sent to specified devices
4. Multi-language content (En, Ar, Hi)
5. All platforms targeted (iOS, Android, Windows, Huawei)
```

---

## Alert Notification System

### Alert Types Supported
```go
AlertType_balance      // Account balance alert
AlertType_equity       // Equity alert
AlertType_margin_level // Margin level alert
AlertType_sp           // Stop-out price alert
AlertType_market_ask   // Market ask price alert
AlertType_market_bid   // Market bid price alert
```

### Alert Delivery Methods
1. **In-App Alert:** Stored in vfx_Alert table
2. **Email Notification:** Via SMTP service
3. **Push Notification:** Via OneSignal (through automation)
4. **Real-time Update:** WebSocket to trading platform

---

## Security Considerations

### Credential Management
- **SMTP Password:** Stored in environment variables
- **OneSignal Keys:** Stored in environment variables
- **Data Provider Credentials:** Encrypted in database
- **Encryption Flag:** `IsEncrypted = true` for sensitive data

### Data Privacy
- **Email Tracking:** Uses EmailTrackingId for deduplication
- **Soft Deletes:** Emails can be marked deleted but retained in database
- **Access Control:** Account-level filtering on inbox/outbox
- **User Identity:** Email linked to Identity model (with privacy controls)

### API Security
- **SMTP TLS:** Supported on ports 465 (SSL) and 587 (TLS)
- **OneSignal OAuth:** Context-based authentication with API keys
- **NATS Auth:** Message broker security configured separately

---

## Performance Metrics

### Email Service Performance
- **Queue Processing:** Every 1 second
- **Health Monitoring:** Every 10 seconds
- **Connection Pool:** 10 max connections
- **Batch Size:** Processed with efficient SMTP send

### Push Notification Performance
- **Sending:** Real-time, non-blocking
- **Targets:** Multi-platform (iOS, Android, Windows, Huawei)
- **Languages:** 3 language variants per notification

### Data Provider Performance
- **Market Data:** Real-time tick updates
- **Symbol Resolution:** CSV-based mapping
- **Failover:** Priority-based provider selection

---

## Troubleshooting Guide

### SMTP Issues
**Problem:** Emails not sending
**Solutions:**
1. Verify SMTP credentials in environment
2. Check firewall allows SMTP port (465/587)
3. Verify sender email is authorized
4. Check database Mail table for queued emails
5. Review SMTP health monitor logs

### OneSignal Issues
**Problem:** Push notifications not received
**Solutions:**
1. Verify OneSignal App ID configured
2. Verify Auth Keys in environment variables
3. Confirm device player IDs valid
4. Check OneSignal dashboard for failures
5. Verify account has OneSignal SDK installed

### Data Provider Issues
**Problem:** No market data received
**Solutions:**
1. Verify data provider credentials
2. Check provider status in database
3. Validate symbol mappings in CSV
4. Confirm provider connection settings
5. Monitor NATS topic: `system.data_providers`

---

## Summary Table

| Service | Type | Status | Implementation | Key Location |
|---------|------|--------|-----------------|---|
| SMTP Email | External | ✅ Active | Direct connection | `/vfxserver/pkg/smtp/` |
| In-Email | Internal | ✅ Active | Database table | `/vfxserver/internal/server/v1/emails.go` |
| OneSignal Push | External | ✅ Active | REST API | `/vfxcore/pkg/onesignal/` |
| Payment Gateway | External | ❌ Not Implemented | Tracking only | `/vfxserver/model/common/v1/broker.go` |
| SMS Service | External | ❌ Not Implemented | Not integrated | - |
| Data Providers | External | ✅ Active | Multiple types | `/vfxmarket/model/common/v1/data_provider.go` |

---

## Conclusion

The Hstrader application has a robust notification and communication infrastructure focused on:
- **Email Services:** SMTP-based sending and in-app messaging
- **Push Notifications:** OneSignal integration for mobile platforms
- **Market Data:** Multiple provider types (FIX, DDE, WebSocket, Simulator)
- **Automation:** Email and push actions triggered by trading events

**Not Implemented:** Payment gateways and SMS services, though the architecture supports future integration of these services through the automation system and mail queue.

All services are production-ready with monitoring, error handling, and security measures in place.
