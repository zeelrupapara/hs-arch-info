# 🧩 Stunnel Manager / gRPC Server Configuration Report

---

## ⚙️ Server Configuration

| **Parameter** | **Value** |
|----------------|-----------|
| **gRPC Host** | `0.0.0.0` |
| **gRPC Port** | `50055` |
| **Config Path** | `/etc/stunnel/stunnel.conf` |
| **PID File** | `/etc/stunnel/stunnel.pid` |
| **Log Level** | `info` |

---

## 🔄 Stunnel4 Flow

The following flow defines how **VFXLP** interacts with the **stunnel-manager (gRPC service)** for managing FIX provider sessions.

### 🧠 Process Flow Description

1. **Session Validation**  
   - VFXLP checks whether a FIX session for the Liquidity Provider (LP) already exists.  
   - If an active session is found, the currently running provider is **disabled or stopped** before continuing.  

2. **Request Forwarding**  
   - After validation, VFXLP passes the provider configuration request to the **stunnel-manager** service through gRPC.  
   - This service manages SSL tunnel creation, modification, and termination dynamically.

3. **Tunnel Management**  
   - The stunnel-manager updates or reloads the corresponding **`stunnel.cfg`** file.  
   - It communicates with the **stunnel process** via a shared **PID file** to:
     - **Activate** a new LP  
     - **Reload** tunnel parameters for an updated LP  
     - **Kill/Stop** an inactive or disabled provider  

4. **Confirmation & Logging**  
   - Once the tunnel is created or updated, stunnel-manager confirms the operation status back to VFXLP.  
   - Logs are recorded under `/var/log/stunnel/` and the provider’s individual directory in `/var/log/vfxlp/fix/{provider}/`.

---


