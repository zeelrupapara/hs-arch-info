# Validation Documentation - Quick Reference

## Overview
This document covers validation rules for all major system components. Each section follows the pattern: **What** is validated → **How** it's done → **What** happens on error.

---

## 1. Account Creation

**Key Rules:**
- Account type: `client_account` or `admin_account`
- Clients need Trader role, admins need non-Trader role
- Email must be unique and valid
- Balance ≥ 0 for clients
- Leverage: 1-5000 (integer)
- Demo accounts get 100,000 balance if deposit < 3,000

**Validation:** Struct validation + DB checks for role/group existence

**Errors:** 400 (invalid data), 404 (role/group not found), 500 (DB error)

**Integration:** Publishes event → Sends email → Logs operation → Deposits balance

```go
if data.AccountType == model.AccountType_client_account {
    if role.RoleType != model.RoleType_Trader {
        return error("role not allowed for client account")
    }
}
```

---

## 2. Password Management

| Rule | Details |
|------|---------|
| Length | 3-32 characters |
| Uniqueness | User ≠ Investor password |
| Change Process | 5-digit code, 15-min expiry |
| Encryption | Always encrypted before storage |

**Errors:** 400 (invalid), 403 (wrong code), 404 (not found), 500 (DB/encryption)

**Integration:** Email verification → Publish event → Log

---

## 3. Order Validation

**Required Fields:** account_id, symbol_id, volume (≥0), order_price, type, side, expiration_policy

**Special Rules:**
- Market orders: no limit price
- Pending orders: can't update market orders
- Volume: >0 for create, ≤ position volume for close
- Expiration: if `specified_time/day`, `expiry_at` required

**Validation:** Struct + enum checks + status validation

**Errors:** 400 (invalid), 404 (not found), 500 (publish error)

```go
if payload.Type == model.OrderType_market_order {
    order.OrderPrice = payload.OrderPrice
} else {
    order.OrderLimitPrice = payload.OrderPrice
}
```

---

## 4. Position Validation

**Update Rules:**
- Position must be `open_position` status
- Volume ≤ position volume
- Comment: 0-255 chars
- Hedge close: same symbol, opposite sides, both open

**Errors:** 400 (not open/invalid volume), 404 (not found), 500 (DB/publish)

---

## 5. Margin & Leverage

**Calculations:**
```
Margin Level = (Equity / Used Margin) × 100%
Free Margin = Equity - Used Margin
```

**Thresholds:**
- Margin Call: < 50%
- Stop Out: < 20%

**Validation:** Real-time in vfxcore, config-based thresholds

**Integration:** Auto-liquidation if enabled

---

## 6. Liquidation Rules

**Types:** `equity` or `marginlevel`

**Status:** `no_liquidation` → `under_liquidation` → `liquidated`

**Process:** Triggered by vfxcore → Closes positions → Publishes events

---

## 7. Automation Rules

**Structure:** Trigger → Conditions → Actions

**Key Components:**
- **Triggers:** time, account_balance, etc.
- **Conditions:** field/operator/value (e.g., `accounts.equity > 1000`)
- **Actions:** send_email, etc. (with parameters)
- **Scheduling:** period, weekdays, months arrays

**Validation:** Config-based (automation.json) + struct validation

**Test Mode:** Simulate without saving

```go
if !config.ValidateConditionField(cond.Field, cond.Operator) {
    return error("Invalid condition field or operator")
}
```

---

## 8. Routing Rules

**Fields:** name, enable, action_label/type/value, request_type, order_type, priority, conditions

**Validation:**
- Request/order types: comma-separated, must be valid
- Conditions: category (accounts/symbols), field, operator, value
- Priority: unique, swappable

**Config:** routing.json for validation rules

---

## 9. Symbol Validation

| Field | Format | Example |
|-------|--------|---------|
| Sessions | 7 comma-separated time ranges | `08:00-16:00` per weekday |
| Swap Days | 7 comma-separated floats | `1.5,2.0,...` |
| Time Format | HH:MM | `15:04` |
| Dates | YYYY-MM-DD HH:MM:SS | `2024-01-15 09:30:00` |

**Validation:** Custom parsers + enum checks

```go
sessions := strings.Split(session, ",")
if len(sessions) != 7 {
    return error("invalid session format should be 7 days")
}
```

---

## 10. Data Provider

**Provider Types:** FIX, DDE, Simulator, WebSocket

**Key Fields:**
- Name: unique, required
- Status: inactive/active/error
- Credentials: key/value (encrypted)
- CSV Upload: symbol mappings

**Integration:** Publishes to vfxcore → Syncs with vfxmarket

---

## 11. Report Validation

**Model:** `vfxdata.QueryRequest` (request_id, report_type, report_data_type, account_ids, filter, history)

**Validation:** Manual type-checking + ownership enforcement

**Key Points:**
- Path params must be valid integers
- Ownership: users access only their reports
- Files checked with `os.Stat` before operations

**Events:**
- Logs via `queueSystemOperationLog`
- DeleteReport → publishes `EventReportDelete`
- CreateReport → async via operation logs

---

## 12. LP (Liquidity Provider)

**DTO Requirements:**
- name: 3-50 chars
- display_name: 3-100 chars
- FIX config: `.cfg` extension, must parse via QuickFIX

**Lifecycle:**
1. **Create/Update:** Persist provider + credentials + configs
2. **Config File:** Upload `.cfg` → validate → save to `FIX_FILE_PATH`
3. **Activate:** Set status=active → publish event
4. **Priority Swap:** Lock rows → swap priorities

**Validation:** Struct tags + manual checks + QuickFIX parsing

**Events:** Published to NATS "system.lp_providers" → Routes to websocket clients

---

## Common Error Patterns

| Code | Meaning | Common Causes |
|------|---------|---------------|
| 400 | Bad Request | Invalid fields, validation failures, format errors |
| 403 | Forbidden | Invalid verification code |
| 404 | Not Found | Resource doesn't exist, ownership check failed |
| 500 | Internal Server Error | DB errors, publish failures, filesystem issues |

---

## Integration Flow

```
Request → Struct Validation → Custom Checks → DB Operations
    ↓
Events Published (NATS) → Logs Created → WebSocket Clients Notified
```

**Key Components:**
- **Validation:** go-playground/validator + custom checks
- **Events:** NATS messaging system
- **Logs:** OperationsLog + Journal events
- **WebSocket:** Real-time client updates via NatsClientRouter

---

## Quick Tips

✅ **Always validate:** Use struct tags + custom logic  
✅ **Check ownership:** Enforce account-level access  
✅ **Handle errors:** Return appropriate HTTP codes  
✅ **Publish events:** Keep system components in sync  
✅ **Log operations:** Maintain audit trail  
✅ **Test thoroughly:** Use test mode where available (automation rules)