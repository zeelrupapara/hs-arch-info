# 🌱 VFXServer Database Seeder Documentation (Developer Reference)

This developer reference documents the complete VFXServer database seeder source code (Go), with full, unabridged code snippets and explanations for each seeding step.  
The file is organized to match the runtime order used in `main()` and provides context for why each seed exists and what it inserts into the database.

---

## Table of Contents

1. Main Seeder Entry Point
2. seedRoles
3. seedBroker
4. seedGroups
5. seedAccount
6. seedSymbols (full list)
7. seedSystemConfig
8. seedCasbinRules
9. Helper functions for policies/resources mapping
10. seedAutomation
11. seedDataProviders
12. ptrInt64 helper
13. Summary

---

## 🏁 Main Seeder Entry Point

**Purpose:** Initialize configuration, logger, database connection and run all seeding steps in a transaction. Any failure rolls back the transaction.

```go
package main 



import ( 

"fmt" 

"math/rand" 

"strings" 

"vfxserver/config" 

model "vfxserver/model/common/v1" 

"vfxserver/pkg/authz" 

"vfxserver/pkg/db" 

"vfxserver/pkg/logger" 

"vfxserver/pkg/oauth2" 



"gorm.io/gorm" 

) 



func main() { 

// config instant 

cfg := config.NewConfig() 



// logger 

log, err := logger.NewLogger(cfg) 

if err != nil { 

log.Logger.Panicf("Error reading config: %v", err) 

} 



// MYSQL 

DBSess, err := db.NewMysqDB(cfg) 

if err != nil { 

log.Logger.Panicf("Error connecting to MYSQL DB: %v", err) 

} 



err = DBSess.Migrate() 

if err != nil { 

log.Logger.Panicf("Error in mirgrating: %v", err) 

} 



err = DBSess.AlterTableIds() 

if err != nil { 

log.Logger.Panicf("Error in altering table ids: %v", err) 

} 



authz, err := authz.NewAuthz(DBSess.DB) 

if err != nil { 

log.Logger.Panicf("Error in creating authz: %v", err) 

} 



tx := DBSess.DB.Begin() 



err = seedRoles(tx) 

if err != nil { 

tx.Rollback() 

log.Logger.Fatal(err) 

} 



err = seedCasbinRules(tx, authz) 

if err != nil { 

tx.Rollback() 

log.Logger.Fatal(err) 

} 



err = seedSystemConfig(tx) 

if err != nil { 

tx.Rollback() 

log.Logger.Fatal(err) 

} 



err = seedBroker(tx) 

if err != nil { 

tx.Rollback() 

log.Logger.Fatal(err) 

} 



err = seedSymbols(tx) 

if err != nil { 

tx.Rollback() 

log.Logger.Fatal(err) 

} 



err = seedGroups(tx) 

if err != nil { 

tx.Rollback() 

log.Logger.Fatal(err) 

} 



err = seedAccount(tx) 

if err != nil { 

tx.Rollback() 

log.Logger.Fatal(err) 

} 



err = seedAutomation(tx) 

if err != nil { 

tx.Rollback() 

log.Logger.Fatal(err) 

} 



err = seedDataProviders(tx) 

if err != nil { 

tx.Rollback() 

log.Logger.Fatal(err) 

} 



tx.Commit() 

log.Logger.Info("data seeded successfully") 

} 
```

---

## 1️⃣ seedRoles

**Purpose:** Insert default system roles into the `roles` table. Roles are marked `Original: true` to indicate they are built-in.

```go
func seedRoles(tx *gorm.DB) error { 

roles := []*model.Role{ 

{ 

Desc: authz.Roles_Admin, 

Original: true, 

RoleType: model.RoleType_Admin, 

}, 

{ 

Desc: authz.Roles_Dealer, 

Original: true, 

RoleType: model.RoleType_Dealer, 

}, 

{ 

Desc: authz.Roles_Trader, 

Original: true, 

RoleType: model.RoleType_Trader, 

}, 

{ 

Desc: authz.Roles_Investor, 

Original: true, 

RoleType: model.RoleType_Trader, 

}, 

{ 

Desc: authz.Roles_API_Trader, 

Original: true, 

RoleType: model.RoleType_Trader, 

}, 

{ 

Desc: authz.Roles_API_Admin, 

Original: true, 

RoleType: model.RoleType_Admin, 

}, 

} 

return tx.Save(roles).Error 

} 
```

---

## 2️⃣ seedBroker

**Purpose:** Insert a demo broker entry that acts as the parent organization for accounts, groups, symbols, etc.

```go
func seedBroker(tx *gorm.DB) error { 

broker := &model.Broker{ 

BusinessName: "HS Trader (Demo System)", 

Address: "Amman, Jordan", 

Country: "Jordan", 

Tel: "+962798726136", 

Website: "https://www.hybridsolutions.com", 

Facebook: "https://www.facebook.com/VertexFXTrader", 

Twitter: "https://twitter.com/VertexFXTrader", 

Instagram: "https://www.instagram.com/vertexfxtrader", 

LinkedIn: "https://www.linkedin.com/company/hybrid-solutions-hs-", 

Youtube: "https://www.youtube.com/c/hybridsolutions", 

} 

return tx.Save(broker).Error 

} 
```

---

## 3️⃣ seedGroups

**Purpose:** Create root groups — `Traders` (client group) and `Dealers` (admin group) — then seed group-specific configs and symbols for each created group.

```go
func seedGroups(tx *gorm.DB) error { 

//jana: config instant 

cfg := config.NewConfig() 



g1 := &model.Group{ 

Root: true, 

Desc: "Traders", 

ManagerId: 24100001, 

GroupType: model.GroupType_client_group, 

} 

g2 := &model.Group{ 

Root: true, 

Desc: "Dealers", 

GroupType: model.GroupType_admin_group, 

} 



err := tx.Save([]*model.Group{g1, g2}).Error 

if err != nil { 

return err 

} 



err = db.SeedConfigsForGroup(tx, g1.Id, cfg) 

if err != nil { 

return err 

} 

err = db.SeedSymbolsForGroup(tx, g1.Id) 

if err != nil { 

return err 

} 



err = db.SeedConfigsForGroup(tx, g2.Id, cfg) 

if err != nil { 

return err 

} 

err = db.SeedSymbolsForGroup(tx, g2.Id) 

if err != nil { 

return err 

} 



return nil 

} 
```

---

## 4️⃣ seedAccount

**Purpose:** Create default accounts (Admin, Dealer, Trader) linked to the broker and groups. The default password `"123"` is encrypted via `oauth2.EncryptPassword` and assigned to each account. For the `Trader` account, initial balances and leverage are set.

```go
func seedAccount(tx *gorm.DB) error { 



// Get broker 

broker := &model.Broker{} 

err := tx.First(broker).Error 

if err != nil { 

return err 

} 



// Get Group "Dealers" 

dealersGroup := &model.Group{} 

err = tx.Where("group_type = ?", model.GroupType_admin_group).First(dealersGroup).Error 

if err != nil { 

return err 

} 



// Get Group "Traders" 

tradersGroup := &model.Group{} 

err = tx.Where("group_type = ?", model.GroupType_client_group).First(tradersGroup).Error 

if err != nil { 

return err 

} 



// Get Role "Admin" 

roleAdmin := &model.Role{} 

err = tx.Where("`desc` = ?", authz.Roles_Admin).First(roleAdmin).Error 

if err != nil { 

return err 

} 



// Get Role "Dealer" 

roleDealer := &model.Role{} 

err = tx.Where("`desc` = ?", authz.Roles_Dealer).First(roleDealer).Error 

if err != nil { 

return err 

} 



// Get Role "Trader" 

roleTrader := &model.Role{} 

err = tx.Where("`desc` = ?", authz.Roles_Trader).First(roleTrader).Error 

if err != nil { 

return err 

} 



password, err := oauth2.EncryptPassword("123") 

if err != nil { 

return err 

} 



// create accounts 

accounts := []*model.Account{ 

{ 

BrokerId: broker.Id, 

AccountType: model.AccountType_admin_account, 

Status: model.AccountStatus_active, 

// UserId: "admin", 

UserPassword: password, 

RoleId: roleAdmin.Id, 

GroupId: dealersGroup.Id, 

Identity: model.Identity{ 

FirstName: "Admin", 

FamilyName: "HS", 

Email: "saif.h@hybridsolutions.com", 

Telephone: "+962798726136", 

AddressLine: "Amman, Jordan", 

}, 

}, 

{ 

BrokerId: broker.Id, 

AccountType: model.AccountType_admin_account, 

Status: model.AccountStatus_active, 

// UserId: "dealer", 

UserPassword: password, 

RoleId: roleDealer.Id, 

GroupId: dealersGroup.Id, 

Identity: model.Identity{ 

FirstName: "Dealer", 

FamilyName: "HS", 

Email: "saif.h@hybridsolutions.com", 

Telephone: "+962798726136", 

AddressLine: "Amman, Jordan", 

}, 

}, 

{ 

BrokerId: broker.Id, 

AccountType: model.AccountType_client_account, 

Status: model.AccountStatus_active, 

// UserId: "trader", 

UserPassword: password, 

RoleId: roleTrader.Id, 

GroupId: tradersGroup.Id, 

Balance: 1000000, 

Equity: 1000000, 

FreeMargin: 1000000, 

UsedMargin: 0, 

Credit: 10000, 

Leverage: 100000, 

Identity: model.Identity{ 

FirstName: "Trader", 

FamilyName: "HS", 

Email: "saif.h@hybridsolutions.com", 

Telephone: "+962798726136", 

AddressLine: "Amman, Jordan", 

}, 

}, 

} 

err = tx.Save(accounts).Error 

if err != nil { 

return err 

} 



for i := range accounts { 

accounts[i].ClientId = fmt.Sprintf("%d_%d", accounts[i].Id, rand.Int63n(1e10)) 

} 



return tx.Save(accounts).Error 

} 
```

---

## 5️⃣ seedSymbols (FULL CONTENT)

**Purpose:** Seed the full instrument catalog used by the trading platform. This includes categories (SymbolClass) like Forex (Majors/Minors/Exotics), Metals, Crypto, Fix, Demo, etc. Each `Symbol` entry contains trading configuration fields (spread, digits, contract size, sessions, margins, etc).

> NOTE: This is the full unabridged `seedSymbols` function with all symbol definitions exactly as provided.

```go
func seedSymbols(tx *gorm.DB) error { 



symbolClass := []*model.SymbolClass{ 

{ 

Desc: "Forex", 

SymbolClasses: []*model.SymbolClass{ 

{ 

Desc: "Majors", 

Symbols: []*model.Symbol{ 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "EUR A0-FX", 

Symbol: "EURUSD", 

SymbolMap: "EURUSD", 

ISIN: "1", 

Desc: "Euro vs US Dollar", 

BaseCurrency: "EUR", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "GBP A0-FX", 

Symbol: "GBPUSD", 

SymbolMap: "GBPUSD", 

ISIN: "1", 

Desc: "British Pound vs US Dollar", 

BaseCurrency: "GBP", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "NZDJPY@FXCM A0-FX", 

Symbol: "USDCAD", 

SymbolMap: "USDCAD", 

ISIN: "1", 

Desc: "US Dollar vs Canadian Dollar", 

BaseCurrency: "USD", 

QuoteCurrency: "CAD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "JPY@FXCM A0-FX", 

Symbol: "USDJPY", 

SymbolMap: "USDJPY", 

ISIN: "1", 

Desc: "US Dollar vs Japanese Yen", 

BaseCurrency: "USD", 

QuoteCurrency: "JPY", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "CHF", 

Symbol: "USDCHF", 

SymbolMap: "USDCHF", 

ISIN: "1", 

Desc: "US Dollar vs Swiss Franc", 

BaseCurrency: "USD", 

QuoteCurrency: "CHF", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

}, 

}, 

{ 

Desc: "Minors", 

Symbols: []*model.Symbol{ 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "EURAUD", 

Symbol: "EURAUD", 

SymbolMap: "EURAUD", 

ISIN: "1", 

Desc: "Euro vs Australian Dollar", 

BaseCurrency: "EUR", 

QuoteCurrency: "AUD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "AUDJPY", 

Symbol: "AUDJPY", 

SymbolMap: "AUDJPY", 

ISIN: "1", 

Desc: "Australian Dollar vs Japanese Yen", 

BaseCurrency: "AUD", 

QuoteCurrency: "JPY", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

Execution: model.ExecutionMode_Instant, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "AUDNZD", 

Symbol: "AUDNZD", 

SymbolMap: "AUDNZD", 

ISIN: "1", 

Desc: "Australian Dollar vs New Zealand Dollar", 

BaseCurrency: "AUD", 

QuoteCurrency: "NZD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "AUD", 

Symbol: "AUDUSD", 

SymbolMap: "AUDUSD", 

ISIN: "1", 

Desc: "Australian Dollar vs US Dollar", 

BaseCurrency: "AUD", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

}, 

}, 

{ 

Desc: "Exotics", 

Symbols: []*model.Symbol{ 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "AUD", 

Symbol: "USDTUR", 

SymbolMap: "USDTUR", 

ISIN: "1", 

Desc: "US Dollar vs Turkish Lira", 

BaseCurrency: "USD", 

QuoteCurrency: "TUR", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

}, 

}, 

}, 

}, 

{ 

Desc: "Metals", 

Symbols: []*model.Symbol{ 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "gold", 

Symbol: "GOLD", 

SymbolMap: "GOLD", 

ISIN: "1", 

Desc: "GOLD vs US Dollar", 

BaseCurrency: "GOLD", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "MXN", 

Symbol: "PALLADIUM", 

SymbolMap: "PALLADIUM", 

ISIN: "1", 

Desc: "PALLADIUM vs US Dollar", 

BaseCurrency: "PALLADIUM", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "", 

Symbol: "COPPER", 

SymbolMap: "COPPER", 

ISIN: "1", 

Desc: "COPPER vs US Dollar", 

BaseCurrency: "COPPER", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "", 

Symbol: "PLATINUM", 

SymbolMap: "PLATINUM", 

ISIN: "1", 

Desc: "PLATINUM vs US Dollar", 

BaseCurrency: "PLATINUM", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "XAG", 

Symbol: "SILVER", 

SymbolMap: "SILVER", 

ISIN: "1", 

Desc: "SILVER vs US Dollar", 

BaseCurrency: "SILVER", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

}, 

}, 

{ 

Desc: "Crypto", 

Symbols: []*model.Symbol{ 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "BITCOIN", 

Symbol: "Bitcoin", 

SymbolMap: "Bitcoin", 

ISIN: "1", 

Desc: "Bitcoin vs US Dollar", 

BaseCurrency: "Bitcoin", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "USDETH", 

Symbol: "Ethereum", 

SymbolMap: "Ethereum", 

ISIN: "1", 

Desc: "US Dollar vs Ethereum", 

BaseCurrency: "USD", 

QuoteCurrency: "Ethereum", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "XBTCAD@KKN", 

Symbol: "ETHXBT", 

SymbolMap: "ETHXBT", 

ISIN: "1", 

Desc: "ETHXBT vs US Dollar", 

BaseCurrency: "ETHXBT", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

ExternalSymbol: "0", 

Source: "dde", 

DataSource: "LTCUSD", 

Symbol: "Ripple", 

SymbolMap: "Ripple", 

ISIN: "1", 

Desc: "Ripple vs US Dollar", 

BaseCurrency: "Ripple", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

}, 

}, 

{ 

Desc: "Fix", 

Symbols: []*model.Symbol{ 

{ 

ExternalSymbol: "0", 

Source: "fix", 

DataSource: "FIX_EURUSD", 

Symbol: "FIX_EURUSD", 

SymbolMap: "FIX_EURUSD", 

ISIN: "1", 

Desc: "EUR vs US Dollar", 

BaseCurrency: "EUR", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

}, 

}, 

{ 

Desc: "Demo", 

Symbols: []*model.Symbol{ 

{ 

Source: "fix_simulator", 

DataSource: "DEMO_EURUSD", 

Symbol: "DEMO_EURUSD", 

SymbolMap: "DEMO_EURUSD", 

ExternalSymbol: "DEMO_EURUSD", 

ISIN: "1", 

Desc: "Demo Euro vs US Dollar", 

BaseCurrency: "EUR", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

Source: "fix_simulator", 

DataSource: "DEMO_USDJPY", 

Symbol: "DEMO_USDJPY", 

SymbolMap: "DEMO_USDJPY", 

ExternalSymbol: "DEMO_USDJPY", 

ISIN: "1", 

Desc: "Demo US Dollar vs Japanese Yen", 

BaseCurrency: "USD", 

QuoteCurrency: "JPY", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

{ 

Source: "fix_simulator", 

DataSource: "DEMO_GBPUSD", 

Symbol: "DEMO_GBPUSD", 

SymbolMap: "DEMO_GBPUSD", 

ExternalSymbol: "DEMO_GBPUSD", 

ISIN: "1", 

Desc: "Demo British Pound vs US Dollar", 

BaseCurrency: "GBP", 

QuoteCurrency: "USD", 

Enabled: true, 

Digits: 5, 

Spread: nil, 

SpreadBalance: 0, 

ContractSize: 1000, 

QuoteSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TradeSessions: "00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00,00:00-00:00", 

TimeLimit: "", 

Orders: "0,1,2,3,4,5", 

Expiration: "0,1,2,3", 

Filling: model.FillPolicy_fill_or_kill, 

MinValue: 0.1, 

MaxValue: 100, 

Step: 1, 

Timeout: 30, 

MarginBuy: 1, 

MarginSell: 1, 

MaintenanceBuy: 1, 

MaintenanceSell: 1, 

}, 

}, 

}, 

} 

 

return tx.Save(&symbolClass).Error 

} 
```

---

## 6️⃣ seedSystemConfig

**Purpose:** Insert core system-wide configuration values used during runtime.

```go
func seedSystemConfig(tx *gorm.DB) error { 

//jana: config instant 

cfg := config.NewConfig() 



configs := []*model.Config{ 

{ 

Desc: model.KeyDesc_SystemBrokerSetup, 

ValueType: model.ValueType_int32, 

ConfigClass: model.ConfigClass_System, 

Value: "1", 

}, 

{ 

Desc: model.KeyDesc_SystemEndOfDay, 

ValueType: model.ValueType_string, 

ConfigClass: model.ConfigClass_System, 

Value: "23:59", 

}, 

{ 

Desc: model.KeyDesc_SystemMaxOrdersAndPositions, 

ValueType: model.ValueType_int32, 

ConfigClass: model.ConfigClass_System, 

Value: "250", 

}, 

{ 

Desc: model.KeyDesc_SystemDefaultGroupForDemoAccounts, 

ValueType: model.ValueType_int32, 

ConfigClass: model.ConfigClass_System, 

Value: "24100001", 

}, 

{ 

Desc: model.KeyDesc_SystemDefaultGroupForLiveAccounts, 

ValueType: model.ValueType_int32, 

ConfigClass: model.ConfigClass_System, 

Value: "24100001", 

}, 

{ 

Desc: model.KeyDesc_SystemAutoBrokerMarketOrders, 

ValueType: model.ValueType_int32, 

ConfigClass: model.ConfigClass_System, 

Value: "0", 

}, 

{ 

Desc: model.KeyDesc_SystemAutoBrokerPendingOrders, 

ValueType: model.ValueType_int32, 

ConfigClass: model.ConfigClass_System, 

Value: "0", 

}, 

{ 

Desc: model.KeyDesc_SystemAutoBrokerLiquidationOrders, 

ValueType: model.ValueType_int32, 

ConfigClass: model.ConfigClass_System, 

Value: "0", 

}, 

{ 

Desc: model.KeyDesc_SystemChattingPassOrderAfter, 

ValueType: model.ValueType_int32, 

ConfigClass: model.ConfigClass_System, 

Value: "30", 

}, 

{ 

Desc: model.KeyDesc_SystemChattingTimeoutAfter, 

ValueType: model.ValueType_int32, 

ConfigClass: model.ConfigClass_System, 

Value: "30", 

}, 

{ 

Desc: model.KeyDesc_SystemSystemChattingTimeoutAction, 

ValueType: model.ValueType_int32, 

ConfigClass: model.ConfigClass_System, 

Value: "0", 

}, 

{ 

Desc: model.KeyDesc_SystemLpMultiplyAmountWithContractSize, 

ValueType: model.ValueType_bool, 

ConfigClass: model.ConfigClass_System, 

Value: "0", 

}, 

{ 

Desc: model.KeyDesc_SystemEnableProp, 

ValueType: model.ValueType_bool, 

ConfigClass: model.ConfigClass_System, 

Value: cfg.Setting.EnableProp, 

}, 

} 

return tx.Save(configs).Error 

} 
```

---

## 7️⃣ seedCasbinRules

**Purpose:** Map all permission rule strings into `model.Resource` entries and then create role policies for Casbin enforcement.

> This includes `mapRulesToResource` and `mapRulesToPolicies` helpers used below.

```go
func seedCasbinRules(tx *gorm.DB, az *authz.Authz) error { 

allRules := []string{ 

authz.Resources_Type_Admin, 

authz.Resources_Type_Dealer, 

authz.Resources_Type_Trader, 

authz.Resources_Config_Read, 

authz.Resources_Config_Update, 

authz.Resources_SystemConfig_General, 

authz.Resources_SystemConfig_Dealing, 

authz.Resources_Symbols_Read, 

authz.Resources_Symbols_Create, 

authz.Resources_Symbols_Update, 

authz.Resources_Symbols_Delete, 

authz.Resources_SymbolClasses_Create, 

authz.Resources_SymbolClasses_Update, 

authz.Resources_SymbolClasses_Delete, 

authz.Resources_Broker_Read, 

authz.Resources_Broker_Update, 

authz.Resources_OperationLogs_Read, 

authz.Resources_OperationLogs_Delete, 

authz.Resources_Roles_Read, 

authz.Resources_Roles_Manage, 

authz.Resources_HistorySessions_Read, 

authz.Resources_HistorySessions_Delete, 

authz.Resources_Tokens_Read, 

authz.Resources_Tokens_Delete, 

authz.Resources_Emails_Read, 

authz.Resources_Groups_Read, 

authz.Resources_Groups_Create, 

authz.Resources_Groups_Update, 

authz.Resources_Groups_Delete, 

authz.Resources_Groups_Symbols, 

authz.Resources_Groups_General, 

authz.Resources_Groups_Commissions, 

authz.Resources_Groups_Trades, 

authz.Resources_Accounts_Read, 

authz.Resources_Accounts_Create, 

authz.Resources_Accounts_Update, 

authz.Resources_Accounts_Delete, 

authz.Resources_Accounts_Money, 

authz.Resources_Accounts_Favourites, 

authz.Resources_Accounts_ChangePasswords, 

authz.Resources_Accounts_Journal, 

authz.Resources_Trades_Overview, 

authz.Resources_Trades_Manage, 

authz.Resources_DealingRoom_Manage, 

authz.Resources_Dealing_ViewAll, 

authz.Resources_Dealing_ViewGroup, 

authz.Resources_OnlineSessions_Read, 

authz.Resources_OnlineSessions_Disconnect, 

authz.Resources_OnlineSessions_GroupOnly, 

authz.Resources_InAppSupport_Read, 

authz.Resources_InAppSupport_Manage, 

authz.Resources_MyOrders_Read, 

authz.Resources_MyOrders_Create, 

authz.Resources_MyOrders_Update, 

authz.Resources_MyOrders_Cancel, 

authz.Resources_MyPositions_Read, 

authz.Resources_MyPositions_Update, 

authz.Resources_MyPositions_Close, 

authz.Resources_MyDeals_Read, 

authz.Resources_Alerts_Read, 

authz.Resources_Alerts_Create, 

authz.Resources_Alerts_Update, 

authz.Resources_Alerts_Delete, 

authz.Resources_MyAccount_Read, 

authz.Resources_MyAccount_ChangePassword, 

authz.Resources_MyAccount_ChangeInvestorPassword, 

authz.Resources_MyAccount_ClientSecret, 

authz.Resources_MyAccount_Journal, 

authz.Resources_MySymbols_Read, 

authz.Resources_MySymbols_MarketHistory, 

authz.Resources_Watchlist_Read, 

authz.Resources_Watchlist_Create, 

authz.Resources_Watchlist_Update, 

authz.Resources_Watchlist_Delete, 

authz.Resources_MyEmails_Read, 

authz.Resources_MyEmails_Create, 

authz.Resources_MyEmails_Update, 

authz.Resources_MyEmails_Delete, 

authz.Resources_Reports_Read, 

authz.Resources_Reports_Create, 

authz.Resources_Reports_Delete, 

authz.Resources_News_Read, 

authz.Resources_BulkOperations_Read, 

authz.Resources_BulkOperations_Update, 

authz.Resources_RoutingRule_Create, 

authz.Resources_RoutingRule_Update, 

authz.Resources_RoutingRule_Delete, 

authz.Resources_RoutingRule_Read, 

authz.Resources_Automation_Read, 

authz.Resources_Automation_Create, 

authz.Resources_Automation_Update, 

authz.Resources_Automation_Delete, 

authz.Resources_DataProviders_Create, 

authz.Resources_DataProviders_Update, 

authz.Resources_DataProviders_Delete, 

authz.Resources_DataProviders_Read, 

authz.Resources_BulkOperations_Delete, 

} 



resources, err := mapRulesToResource(allRules) 

if err != nil { 

return err 

} 



err = tx.Create(&resources).Error 

if err != nil { 

return err 

} 



adminRules := []string{ 

authz.Resources_MyAccount_Read, 

authz.Resources_MyAccount_ChangePassword, 

authz.Resources_MyAccount_Journal, 

authz.Resources_MySymbols_Read, 

authz.Resources_MySymbols_MarketHistory, 

authz.Resources_MyEmails_Read, 

authz.Resources_MyEmails_Create, 

authz.Resources_MyEmails_Update, 

authz.Resources_MyEmails_Delete, 

authz.Resources_Type_Admin, 

authz.Resources_Type_Dealer, 

authz.Resources_Type_Trader, 

authz.Resources_Config_Read, 

authz.Resources_Config_Update, 

authz.Resources_SystemConfig_General, 

authz.Resources_SystemConfig_Dealing, 

authz.Resources_Symbols_Read, 

authz.Resources_Symbols_Create, 

authz.Resources_Symbols_Update, 

authz.Resources_Symbols_Delete, 

authz.Resources_SymbolClasses_Create, 

authz.Resources_SymbolClasses_Update, 

authz.Resources_SymbolClasses_Delete, 

authz.Resources_Broker_Read, 

authz.Resources_Broker_Update, 

authz.Resources_OperationLogs_Read, 

authz.Resources_OperationLogs_Delete, 

authz.Resources_Roles_Read, 

authz.Resources_Roles_Manage, 

authz.Resources_HistorySessions_Read, 

authz.Resources_HistorySessions_Delete, 

authz.Resources_Tokens_Read, 

authz.Resources_Tokens_Delete, 

authz.Resources_Emails_Read, 

authz.Resources_Groups_Read, 

authz.Resources_Groups_Create, 

authz.Resources_Groups_Update, 

authz.Resources_Groups_Delete, 

authz.Resources_Groups_Symbols, 

authz.Resources_Groups_General, 

authz.Resources_Groups_Commissions, 

authz.Resources_Groups_Trades, 

authz.Resources_Accounts_Read, 

authz.Resources_Accounts_Create, 

authz.Resources_Accounts_Update, 

authz.Resources_Accounts_Delete, 

authz.Resources_Accounts_Money, 

authz.Resources_Accounts_Favourites, 

authz.Resources_Accounts_ChangePasswords, 

authz.Resources_MyAccount_ClientSecret, 

authz.Resources_Accounts_Journal, 

authz.Resources_Trades_Overview, 

authz.Resources_Trades_Manage, 

authz.Resources_Dealing_ViewAll, 

authz.Resources_DealingRoom_Manage, 

authz.Resources_OnlineSessions_Disconnect, 

authz.Resources_OnlineSessions_Read, 

authz.Resources_InAppSupport_Read, 

authz.Resources_InAppSupport_Manage, 

authz.Resources_Watchlist_Read, 

authz.Resources_Watchlist_Create, 

authz.Resources_Watchlist_Update, 

authz.Resources_Watchlist_Delete, 

authz.Resources_Reports_Read, 

authz.Resources_Reports_Create, 

authz.Resources_Reports_Delete, 

authz.Resources_News_Read, 

authz.Resources_BulkOperations_Read, 

authz.Resources_BulkOperations_Update, 

authz.Resources_RoutingRule_Create, 

authz.Resources_RoutingRule_Update, 

authz.Resources_RoutingRule_Delete, 

authz.Resources_RoutingRule_Read, 

authz.Resources_Automation_Read, 

authz.Resources_Automation_Create, 

authz.Resources_Automation_Update, 

authz.Resources_Automation_Delete, 

authz.Resources_DataProviders_Create, 

authz.Resources_DataProviders_Update, 

authz.Resources_DataProviders_Delete, 

authz.Resources_DataProviders_Read, 

authz.Resources_Symbols_Export, 

authz.Resources_Symbols_Import, 

authz.Resources_BulkOperations_Delete, 

} 

dealerRules := []string{
authz.Resources_MyAccount_Read,
authz.Resources_MyAccount_ChangePassword,
authz.Resources_MySymbols_Read,
authz.Resources_MySymbols_MarketHistory,
authz.Resources_MyEmails_Create,
authz.Resources_MyEmails_Update,
authz.Resources_MyEmails_Delete,
authz.Resources_Config_Read,
authz.Resources_Config_Update,
authz.Resources_Symbols_Read,
authz.Resources_Broker_Read,
authz.Resources_Emails_Read,
authz.Resources_Groups_Read,
authz.Resources_Groups_Create,
authz.Resources_Groups_Update,
authz.Resources_Groups_Delete,
authz.Resources_Groups_General,
authz.Resources_Groups_Trades,
authz.Resources_Groups_Symbols,
authz.Resources_Groups_Commissions,
authz.Resources_Accounts_Read,
authz.Resources_Accounts_Create,
authz.Resources_Accounts_Update,
authz.Resources_Accounts_Delete,
authz.Resources_Accounts_Money,
authz.Resources_Accounts_Favourites,
authz.Resources_Accounts_ChangePasswords,
authz.Resources_Accounts_Journal,
authz.Resources_Trades_Overview,
authz.Resources_Trades_Manage,
authz.Resources_DealingRoom_Manage,
authz.Resources_Dealing_ViewGroup,
authz.Resources_OnlineSessions_GroupOnly,
authz.Resources_OnlineSessions_Disconnect,
authz.Resources_InAppSupport_Read,
authz.Resources_InAppSupport_Manage,
authz.Resources_Roles_Read,
authz.Resources_MyEmails_Read,
authz.Resources_Reports_Read,
authz.Resources_Reports_Create,
authz.Resources_Reports_Delete,
authz.Resources_BulkOperations_Read,
authz.Resources_BulkOperations_Update,
authz.Resources_RoutingRule_Read,
authz.Resources_DataProviders_Read,
authz.Resources_BulkOperations_Delete,
}

traderRules := []string{
authz.Resources_MyAccount_Read,
authz.Resources_MyAccount_Journal,
authz.Resources_MyAccount_ChangePassword,
authz.Resources_MyAccount_ChangeInvestorPassword,
authz.Resources_MyAccount_ClientSecret,
authz.Resources_MySymbols_Read,
authz.Resources_MySymbols_MarketHistory,
authz.Resources_MyEmails_Read,
authz.Resources_MyEmails_Create,
authz.Resources_MyEmails_Update,
authz.Resources_MyEmails_Delete,
authz.Resources_MyOrders_Read,
authz.Resources_MyOrders_Create,
authz.Resources_MyOrders_Update,
authz.Resources_MyOrders_Cancel,
authz.Resources_MyPositions_Read,
authz.Resources_MyPositions_Update,
authz.Resources_MyPositions_Close,
authz.Resources_MyDeals_Read,
authz.Resources_Alerts_Read,
authz.Resources_Alerts_Create,
authz.Resources_Alerts_Update,
authz.Resources_Alerts_Delete,
authz.Resources_Watchlist_Read,
authz.Resources_Watchlist_Create,
authz.Resources_Watchlist_Update,
authz.Resources_Watchlist_Delete,
authz.Resources_Reports_Read,
authz.Resources_Reports_Create,
authz.Resources_Reports_Delete,
authz.Resources_News_Read,
}

investorRules := []string{
authz.Resources_MyEmails_Read,
authz.Resources_MyAccount_Read,
authz.Resources_MyAccount_Journal,
authz.Resources_MySymbols_Read,
authz.Resources_MySymbols_MarketHistory,
authz.Resources_MyOrders_Read,
authz.Resources_MyPositions_Read,
authz.Resources_MyDeals_Read,
authz.Resources_Reports_Read,
authz.Resources_Reports_Create,
authz.Resources_Alerts_Read,
authz.Resources_Watchlist_Read,
authz.Resources_News_Read,
}

apiTraderRules := []string{
authz.Resources_MyAccount_Read,
authz.Resources_MyAccount_Journal,
authz.Resources_MySymbols_Read,
authz.Resources_MySymbols_MarketHistory,
authz.Resources_MyOrders_Read,
authz.Resources_MyOrders_Create,
authz.Resources_MyOrders_Update,
authz.Resources_MyOrders_Cancel,
authz.Resources_MyPositions_Read,
authz.Resources_MyPositions_Update,
authz.Resources_MyPositions_Close,
authz.Resources_MyDeals_Read,
authz.Resources_Alerts_Read,
authz.Resources_Alerts_Create,
authz.Resources_Alerts_Update,
authz.Resources_Alerts_Delete,
authz.Resources_Watchlist_Read,
authz.Resources_Watchlist_Create,
authz.Resources_Watchlist_Update,
authz.Resources_Watchlist_Delete,
authz.Resources_Reports_Read,
authz.Resources_Reports_Create,
authz.Resources_Reports_Delete,
authz.Resources_News_Read,
}

apiAdminRules := []string{
authz.Resources_MyAccount_Read,
authz.Resources_MyAccount_Journal,
authz.Resources_MySymbols_Read,
authz.Resources_MySymbols_MarketHistory,
authz.Resources_Type_Admin,
authz.Resources_Type_Dealer,
authz.Resources_Type_Trader,
authz.Resources_Config_Read,
authz.Resources_Config_Update,
authz.Resources_SystemConfig_General,
authz.Resources_SystemConfig_Dealing,
authz.Resources_Symbols_Read,
authz.Resources_Broker_Read,
authz.Resources_OperationLogs_Read,
authz.Resources_Roles_Read,
authz.Resources_HistorySessions_Read,
authz.Resources_Groups_Read,
authz.Resources_Groups_Create,
authz.Resources_Groups_Update,
authz.Resources_Groups_Delete,
authz.Resources_Groups_Symbols,
authz.Resources_Groups_General,
authz.Resources_Groups_Commissions,
authz.Resources_Groups_Trades,
authz.Resources_Accounts_Read,
authz.Resources_Accounts_Create,
authz.Resources_Accounts_Update,
authz.Resources_Accounts_Delete,
authz.Resources_Accounts_Money,
authz.Resources_Accounts_Favourites,
authz.Resources_Accounts_ChangePasswords,
authz.Resources_Accounts_Journal,
authz.Resources_Trades_Overview,
authz.Resources_OnlineSessions_Read,
authz.Resources_OnlineSessions_Disconnect,
authz.Resources_Reports_Read,
authz.Resources_Reports_Create,
authz.Resources_Reports_Delete,
authz.Resources_News_Read,
}

policies, err := mapRulesToPolicies("Admin", adminRules)
if err != nil {
return err
}
d, err := mapRulesToPolicies("Dealer", dealerRules)
if err != nil {
return err
}
t, err := mapRulesToPolicies("Trader", traderRules)
if err != nil {
return err
}
i, err := mapRulesToPolicies("Investor", investorRules)
if err != nil {
return err
}
ap, err := mapRulesToPolicies("API_Trader", apiTraderRules)
if err != nil {
return err
}
ad, err := mapRulesToPolicies("API_Admin", apiAdminRules)
if err != nil {
return err
}

policies = append(policies, d...)
policies = append(policies, t...)
policies = append(policies, i...)
policies = append(policies, ap...)
policies = append(policies, ad...)

_, err = az.Enforcer.AddNamedPolicies("p", policies)

return err

}

func mapRulesToPolicies(role string, rules []string) ([][]string, error) {
policies := make([][]string, 0)
for _, rule := range rules {
r := strings.Split(rule, "_")
if len(r) != 2 {
return nil, fmt.Errorf("invalid rule %s", rule)
}
policies = append(policies, []string{role, r[0], r[1]})
}
return policies, nil
}

func mapRulesToResource(rules []string) ([]model.Resource, error) {

resourcesMap := make(map[string]*model.Resource)
for _, rule := range rules {
r := strings.Split(rule, "_")
if len(r) != 2 {
return nil, fmt.Errorf("invalid rule format")
}
if _, ok := resourcesMap[r[0]]; !ok {
resourcesMap[r[0]] = &model.Resource{Desc: r[0], Actions: []model.Action{{Desc: r[1]}}, Type: model.ResourceType_api}
} else {
resourcesMap[r[0]].Actions = append(resourcesMap[r[0]].Actions, model.Action{Desc: r[1]})
}
}

resources := make([]model.Resource, 0)
for _, res := range resourcesMap {
resources = append(resources, *res)
}

return resources, nil
}

func seedAutomation(tx *gorm.DB) error {
 // Get a client group for the templates
 var clientGroup model.Group
 err := tx.Where("group_type = ?", model.GroupType_client_group).First(&clientGroup).Error
 if err != nil {
 return fmt.Errorf("failed to find client group: %w", err)
 }

 // Create automation rule templates
 templates := []*model.AutoRule{
 {
 Name: "Low Balance Protection",
 Enabled: false,
 TriggerType: model.TriggerType_account_balance,
 Period: model.PeriodType_minute,
 Repetitions: 0,
 RecordType: model.RecordType_seed,
 Conditions: []model.AutoCondition{
 {
 Field: "accounts.balance",
 Operator: "<",
 Value: "100",
 Position: 0,
 },
 },
 Actions: []model.AutoAction{
 {
 Name: "Disable Trading on Low Balance",
 ActionType: model.ActionType_disable_trading,
 SendBy: model.SendByType_trigger_login,
 Parameters: model.Parameters{
 "reason": "Account balance below minimum threshold",
 },
 Position: 0,
 },
 },
 },
 {
 Name: "Margin Call Alert",
 Enabled: false,
 TriggerType: model.TriggerType_account_margin,
 Period: model.PeriodType_minute,
 Repetitions: 0,
 RecordType: model.RecordType_seed,
 Conditions: []model.AutoCondition{
 {
 Field: "accounts.margin_level",
 Operator: "<",
 Value: "100",
 Position: 0,
 },
 },
 Actions: []model.AutoAction{
 {
 Name: "Send Margin Call Email",
 ActionType: model.ActionType_send_email,
 SendBy: model.SendByType_trigger_login,
 Parameters: model.Parameters{
 "subject": "Margin Call Warning",
 "template": "margin_call",
 "body": "Your margin level is {{margin_level}}%. Please add funds to avoid liquidation.",
 },
 Position: 0,
 },
 },
 },
 {
 Name: "Welcome Email",
 Enabled: false,
 TriggerType: model.TriggerType_new_account,
 Period: model.PeriodType_once,
 Repetitions: 1,
 RecordType: model.RecordType_seed,
 Actions: []model.AutoAction{
 {
 Name: "Send Welcome Email",
 ActionType: model.ActionType_send_email,
 SendBy: model.SendByType_trigger_login,
 Parameters: model.Parameters{
 "subject": "Welcome to Our Trading Platform",
 "template": "welcome",
 "body": "Welcome {{user_id}}! Your account has been created successfully.",
 },
 Position: 0,
 },
 },
 },
 {
 Name: "First Deposit Bonus",
 Enabled: false,
 TriggerType: model.TriggerType_new_account,
 Period: model.PeriodType_once,
 Repetitions: 1,
 RecordType: model.RecordType_seed,
 Conditions: []model.AutoCondition{
 {
 Field: "accounts.balance",
 Operator: ">",
 Value: "0",
 Position: 0,
 },
 },
 Actions: []model.AutoAction{
 {
 Name: "Add First Deposit Bonus",
 ActionType: model.ActionType_add_credit,
 SendBy: model.SendByType_trigger_login,
 Parameters: model.Parameters{
 "amount": 50,
 "reason": "First deposit bonus",
 },
 Position: 0,
 },
 },
 },
 {
 Name: "Stop Out Protection",
 Enabled: false,
 TriggerType: model.TriggerType_account_margin,
 Period: model.PeriodType_minute,
 Repetitions: 0,
 RecordType: model.RecordType_seed,
 Conditions: []model.AutoCondition{
 {
 Field: "accounts.margin_level",
 Operator: "<=",
 Value: "50",
 Position: 0,
 },
 },
 Actions: []model.AutoAction{
 {
 Name: "Close All Positions",
 ActionType: model.ActionType_close_positions,
 SendBy: model.SendByType_trigger_login,
 Parameters: model.Parameters{
 "reason": "Stop out protection - margin level critical",
 },
 Position: 0,
 },
 {
 Name: "Send Stop Out Notification",
 ActionType: model.ActionType_send_push,
 SendBy: model.SendByType_trigger_login,
 Parameters: model.Parameters{
 "title": "Stop Out Executed",
 "message": "Your positions have been closed due to insufficient margin",
 },
 Position: 1,
 },
 },
 },
 {
 Name: "High Profit Alert",
 Enabled: false,
 TriggerType: model.TriggerType_position_closed,
 Period: model.PeriodType_once,
 Repetitions: 0,
 RecordType: model.RecordType_seed,
 Conditions: []model.AutoCondition{
 {
 Field: "positions.profit",
 Operator: ">",
 Value: "1000",
 Position: 0,
 },
 },
 Actions: []model.AutoAction{
 {
 Name: "Send Profit Notification",
 ActionType: model.ActionType_send_push,
 SendBy: model.SendByType_trigger_login,
 Parameters: model.Parameters{
 "title": "Great Trade!",
 "message": "You just made {{profit}} profit on {{symbol}}",
 },
 Position: 0,
 },
 },
 },
 {
 Name: "Daily Trading Summary",
 Enabled: false,
 TriggerType: model.TriggerType_time,
 StartedAt: ptrInt64(1704067200000), // Jan 1, 2024
 Period: model.PeriodType_day,
 Hours: 23,
 Minutes: 59,
 Repetitions: 0,
 RecordType: model.RecordType_seed,
 Actions: []model.AutoAction{
 {
 Name: "Send Daily Summary",
 ActionType: model.ActionType_send_email,
 SendBy: model.SendByType_group,
 Parameters: model.Parameters{
 "subject": "Daily Trading Summary",
 "template": "daily_summary",
 "body": "Your daily trading summary is ready.",
 "group_id": clientGroup.Id,
 },
 Position: 0,
 },
 },
 },
 }

 // Save all templates
 for _, template := range templates {
 if err := tx.Create(template).Error; err != nil {
 return fmt.Errorf("failed to create automation template '%s': %w", template.Name, err)
 }
 }

 return nil
}

func seedDataProviders(tx *gorm.DB) error {
 // Create default data providers
 providers := []*model.DataProvider{
 {
 Name: "dde",
 DisplayName: "DDE Provider",
 ProviderType: model.DataProviderType_dde,
 Type: "dde", // String version for vfxmarket compatibility
 Status: model.DataProviderStatus_inactive,
 Priority: 1,
 IsActive: false,
 Description: "DDE market data provider for real-time price feeds",
 },
 {
 Name: "fix_f4b44",
 DisplayName: "F4B FIX Provider",
 ProviderType: model.DataProviderType_fix,
 Type: "fix", // String version for vfxmarket compatibility
 Status: model.DataProviderStatus_inactive,
 Priority: 2,
 IsActive: false,
 Description: "FIX protocol provider for F4B liquidity",
 },
 {
 Name: "fix_simulator44",
 DisplayName: "FIX Simulator",
 ProviderType: model.DataProviderType_fix,
 Type: "fix", // String version for vfxmarket compatibility
 Status: model.DataProviderStatus_inactive,
 Priority: 3,
 IsActive: false,
 Description: "Simulated FIX provider for testing",
 },
 }

 // Save providers
 for _, provider := range providers {
 if err := tx.Create(provider).Error; err != nil {
 return fmt.Errorf("failed to create data provider '%s': %w", provider.Name, err)
 }

 // Add default credentials and configurations based on provider type
 var credentials []*model.DataProviderCredential
 var configs []*model.DataProviderConfig

 switch provider.ProviderType {
 case model.DataProviderType_dde:
 credentials = []*model.DataProviderCredential{
 {
 ProviderId: provider.Id,
 Key: "host",
 Value: "", // Empty, to be configured by admin
 IsEncrypted: false,
 Description: "DDE server host address",
 },
 {
 ProviderId: provider.Id,
 Key: "port",
 Value: "",
 IsEncrypted: false,
 Description: "DDE server port",
 },
 {
 ProviderId: provider.Id,
 Key: "token",
 Value: "",
 IsEncrypted: false, // No encryption per recent changes
 Description: "DDE authentication token",
 },
 }
 case model.DataProviderType_fix:
 // FIX credentials - these will be in the credentials table
 credentials = []*model.DataProviderCredential{
 {
 ProviderId: provider.Id,
 Key: "username",
 Value: "",
 IsEncrypted: false,
 Description: "FIX username for authentication",
 },
 {
 ProviderId: provider.Id,
 Key: "password",
 Value: "",
 IsEncrypted: false, // No encryption per recent changes
 Description: "FIX password for authentication",
 },
 }

 // FIX configuration parameters - these will be in the config table
 // These replace the need for a .cfg file
 configs = []*model.DataProviderConfig{
 // DEFAULT section
 {
 ProviderId: provider.Id,
 Key: "ConnectionType",
 Value: "initiator",
 Category: "DEFAULT",
 DataType: "string",
 Description: "Connection type (initiator/acceptor)",
 },
 {
 ProviderId: provider.Id,
 Key: "HeartBtInt",
 Value: "30",
 Category: "DEFAULT",
 DataType: "int",
 Description: "Heartbeat interval in seconds",
 },
 {
 ProviderId: provider.Id,
 Key: "EncryptMethod",
 Value: "0",
 Category: "DEFAULT",
 DataType: "int",
 Description: "Encryption method (0=None)",
 },
 {
 ProviderId: provider.Id,
 Key: "SocketConnectHost",
 Value: "",
 Category: "DEFAULT",
 DataType: "string",
 Description: "FIX server host address",
 },
 {
 ProviderId: provider.Id,
 Key: "SocketConnectPort",
 Value: "",
 Category: "DEFAULT",
 DataType: "int",
 Description: "FIX server port number",
 },
 {
 ProviderId: provider.Id,
 Key: "ReconnectInterval",
 Value: "5",
 Category: "DEFAULT",
 DataType: "int",
 Description: "Reconnection interval in seconds",
 },
 {
 ProviderId: provider.Id,
 Key: "CheckLatency",
 Value: "N",
 Category: "DEFAULT",
 DataType: "string",
 Description: "Check message latency (Y/N)",
 },
 {
 ProviderId: provider.Id,
 Key: "ResetOnLogon",
 Value: "Y",
 Category: "DEFAULT",
 DataType: "string",
 Description: "Reset sequence on logon (Y/N)",
 },
 {
 ProviderId: provider.Id,
 Key: "TimeStampPrecision",
 Value: "MILLIS",
 Category: "DEFAULT",
 DataType: "string",
 Description: "Timestamp precision (MILLIS/MICROS/NANOS)",
 },
 // SESSION section
 {
 ProviderId: provider.Id,
 Key: "BeginString",
 Value: "FIX.4.4",
 Category: "SESSION",
 DataType: "string",
 Description: "FIX protocol version",
 },
 {
 ProviderId: provider.Id,
 Key: "SenderCompID",
 Value: "",
 Category: "SESSION",
 DataType: "string",
 Description: "Sender company ID",
 },
 {
 ProviderId: provider.Id,
 Key: "TargetCompID",
 Value: "",
 Category: "SESSION",
 DataType: "string",
 Description: "Target company ID",
 },
 }
 case model.DataProviderType_simulator:
 // Simulator doesn't need credentials
 credentials = []*model.DataProviderCredential{}
 }

 // Save credentials
 for _, cred := range credentials {
 if err := tx.Create(cred).Error; err != nil {
 return fmt.Errorf("failed to create credential for provider '%s': %w", provider.Name, err)
 }
 }

 // Add generic configs if not already defined
 if configs == nil {
 configs = []*model.DataProviderConfig{
 {
 ProviderId: provider.Id,
 Key: "heartbeat_interval",
 Value: "30",
 Category: "GENERIC",
 DataType: "int",
 Description: "Heartbeat interval for connection monitoring",
 },
 {
 ProviderId: provider.Id,
 Key: "reconnect_delay",
 Value: "5",
 Category: "GENERIC",
 DataType: "int",
 Description: "Delay before reconnection attempt",
 },
 {
 ProviderId: provider.Id,
 Key: "max_reconnect_attempts",
 Value: "10",
 Category: "GENERIC",
 DataType: "int",
 Description: "Maximum reconnection attempts",
 },
 }
 }

 // Save all configs
 for _, config := range configs {
 if err := tx.Create(config).Error; err != nil {
 return fmt.Errorf("failed to create config for provider '%s': %w", provider.Name, err)
 }
 }
 }

 return nil
}

func ptrInt64(v int64) *int64 {
 return &v
}
