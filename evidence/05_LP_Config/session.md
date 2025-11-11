## 🧩 Session Manager Flow

The **Session Manager** within **VFXLP** is responsible for establishing, maintaining, and gracefully terminating FIX sessions with Liquidity Providers (LPs).  
It works in close coordination with the **stunnel-manager** and database-driven configuration engine.

---

### 🧠 Process Flow Description

1. **Session Discovery & Initialization**
   - On startup or LP activation, the Session Manager checks if a session entry already exists in the database (`vfx_LPProviderConfig`).
   - If the LP is already active, the system ensures the existing connection is gracefully stopped before initiating a new session.
   - Configuration parameters (Host, Port, SenderCompID, TargetCompID, HeartBtInt) are loaded dynamically from the DB.

2. **Connection Establishment**
   - Once configuration validation succeeds, Session Manager initiates a FIX 4.4 connection via `vfxlp`.
   - If SSL is enabled for the LP, the request is routed through the `stunnel-manager` before the FIX handshake begins.
   - Successful **Logon** triggers sequence synchronization and status update (`status = 1` in DB).

3. **Heartbeat & Session Maintenance**
   - A periodic **Heartbeat** is sent at the configured `HeartBtInt` interval.
   - The Session Manager monitors Test Requests, sequence resets, and missed heartbeats.
   - On inactivity or missed heartbeats, automatic **reconnect logic** is triggered to restore session stability.

4. **Failure Handling & Recovery**
   - If the FIX session enters an error state, it updates DB status to `2 (error)` and publishes a **NATS event** (`vfxlp.events.lpstatus`) for observability.
   - Recovery attempts are automatically retried based on system retry policy (e.g., exponential backoff).

5. **Deactivation / Kill Switch**
   - When an LP is deactivated via admin API, the Session Manager sends a **Logout** message, terminates the session, and stops the corresponding process.
   - The status in DB is updated to `0 (inactive)` to prevent routing from using that LP.

6. **Session Tracking & Logging**
   - Each FIX session writes logs under `/var/log/vfxlp/fix/{provider}/`.
   - Key events such as Logon, Logout, Heartbeat, TestRequest, and Disconnect are timestamped and sequence-numbered for traceability.

---


