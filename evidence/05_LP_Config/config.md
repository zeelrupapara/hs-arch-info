# 🧩 FIX Configuration Overview

---

## 🕹️ Previous Version

- LP connections were **manually configured and static**.  
- Each provider (e.g., **Centroid**, **Simulator**, **F4B**) required **separate FIX setups** and manual upload of configuration files.  
- SSL encryption via **stunnel** was **non-dynamic** — every new provider required a manually created stunnel entry.  
- This process caused **developer dependency**, longer onboarding, and higher support overhead.

---

## ⚙️ Current Version

- LP configurations are now **centralized and database-driven**.  
- Providers can be **created or modified directly through the application UI**.  
- The **stunnel-manager** service automatically handles:
  - SSL tunnel setup  
  - Tunnel modification  
  - Live updates when LP parameters change  

✅ **Benefits:**
- Dynamic FIX session management  
- Faster LP onboarding  
- Reduced operational overhead  
- No manual SSL intervention required  

---

## 🔄 FIX Connection Flow

| **Stage** | **Description** |
|------------|----------------|
| **Heartbeat (HeartBtInt)** | Defines interval for periodic keepalive messages. |
| **Timeout & Reconnect** | Automatic reconnection triggered if no heartbeat/test request is received. |
| **Logon** | Authenticates and synchronizes FIX session sequence numbers. |
| **Logout** | Gracefully ends the session and updates sequence tracking. |

---

## 📂 Configuration Examples

### 🔹 Previous Version of LP Configuration
```ini
# Manual FIX Configuration Example (Legacy)
[DEFAULT]
ResetOnLogon=Y
ResetOnLogout=Y
ResetOnDisconnect=Y
ResetSeqNumFlag=Y
FileLogPath=logs/fix/centroid44
FileStorePath=logs/fix/centroid44
HeartBtInt=30
ReconnectInterval=5
ValidateFieldsHaveValues=N
CheckLatency=N
EncryptMethod=0
SocketConnectHost=stunnel
SocketConnectPort=50002
ConnectionType=initiator
EndTime=23:59:59
StartTime=00:00:00
TimeStampPrecision=MILLIS


[SESSION]
BeginString=FIX.4.4
SenderCompID=HSA
TargetCompID=HSA2

ENV Session
  - FIX_IMPL=centroid44
  - FIX_NAME=centroid44  