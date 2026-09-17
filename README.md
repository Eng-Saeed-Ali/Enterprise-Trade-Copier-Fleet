# ⚡ Enterprise Dual-Platform Trade Copier Fleet (MT5 + MT4)
### Sub-Millisecond Multi-Instance Trade Replication Engine with Native Shared-Memory IPC

![MQL5](https://img.shields.io/badge/Language-MQL5%20%2F%20MQL4%20%2F%20C%2B%2B-blue?style=for-the-badge)
![PowerShell](https://img.shields.io/badge/Automation-PowerShell%20%26%20Win32-purple?style=for-the-badge)
![Latency](https://img.shields.io/badge/Latency-%3C%200.1ms%20(Local%20IPC)-brightgreen?style=for-the-badge)
![Architecture](https://img.shields.io/badge/Architecture-Serverless%20File--IPC-orange?style=for-the-badge)

An institutional-grade, zero-hub trade replication ecosystem engineered to mirror high-frequency orders from a single Master account to a fleet of **100+ concurrent slave terminals** across MetaTrader 5 and MetaTrader 4 in under 0.1 milliseconds.

---

## 🏗️ System Architecture

Traditional copiers rely on local TCP sockets or external brokers that suffer from port exhaustion, WAF blocks, and single-point-of-failure crashes. This engine implements a **Serverless Shared-Memory Ring Buffer (`FILE_COMMON`)** mapped across the Windows OS page cache.
┌────────────────────────────────────────────────────────────────────────┐
│ MASTER MT5 TERMINAL │
│ (High-Frequency Broadcaster - Master_Copier_Engine.ex5) │
└───────────────────────────────────┬────────────────────────────────────┘
│ Direct Signal Stream (< 0.1ms)
▼
┌────────────────────────────────────────────────────────────────────────┐
│ MQL SHARED MEMORY IPC RING BUFFER │
│ Path: %APPDATA%\MetaQuotes\Terminal\Common\Files\master_signals.txt │
│ Concurrency: Atomic Lock-free FILE_SHARE_READ | FILE_SHARE_WRITE │
└───────────────────┬────────────────────────────────┬───────────────────┘
│ │
┌───────────┴───────────┐ ┌───────────┴───────────┐
│ Continuous Read (250ms)│ │ Continuous Read (100ms)│
▼ ▼ ▼ ▼
┌───────────────────────────────┐ ┌───────────────────────────────┐
│ 100+ SLAVE MT5 FLEET │ │ 5 SLAVE MT4 FLEET │
│ C:\TradingHub\Terminals\ │ │ C:\TradingHub\Terminals_MT4\ │
│ Terminal_001 ... Terminal_115 │ │ Terminal_001 ... Terminal_005 │
│ Slave_Copier_Engine.ex5 │ │ Slave_Copier_Engine_MT4.ex4 │
└───────────────────────────────┘ └───────────────────────────────┘
code
Code
---

## ⚡ Key Engineering Highlights

### 1. Universal Auto-Symbol Matcher (v7.5)
Eliminates cross-broker naming discrepancies through an autonomous multi-tier resolution algorithm:
- **Gold & Precious Metals:** Dynamically matches `XAUUSD` $\leftrightarrow$ `GOLD`, `XAUUSD_`, `XAUUSD.pro`, `XAUUSDm`, `XAUUSDc`, and broker-specific variations.
- **Forex Suffix Stripping:** Automatically removes `.pro`, `.ecn`, `.std`, `m`, `c`, `_`, `!` and queries the target broker's active MarketWatch catalog in real time.

### 2. Overcoming Windows GDI / User32 Session Limits
A single Windows interactive desktop session physically caps at **65,536 USER/GDI handles**, which restricts unoptimized systems to ~30 GUI instances. This architecture breaks this barrier by:
- Expanding the Windows Desktop Heap (`SharedSection=1024,65536,2048`) via Registry manipulation.
- Stripping chart bloat (single-chart headless profile with `flags=339` and `MaxBars=1`).
- Running slave terminals in portable, minimized background sessions.

### 3. Smart Auto-Discovery Watchdog (v6.1)
- **Deep Process Auditing:** Uses `Get-CimInstance Win32_Process` to query full executable command lines.
- **Auto-Discovery:** Scans terminal directories on startup, dynamically launching *only* active, configured accounts while ignoring empty terminal slots.
- **Staggered Launch Queue:** Boots terminals with a safe 3-to-10 second stagger delay to completely eliminate CPU spikes and disk I/O thrashing.

### 4. Deterministic Idempotency & Direct Position Matching
- **Zero-Duplicate Guarantee:** Enforces order uniqueness via sequential lineage tokens (`master_event_id`).
- **Direct Comment Tracking:** `CLOSE` and `MODIFY` operations parse position comments directly (`CP|M:<master_ticket>|E:<event_id>`), eliminating state loss during transient memory crashes or platform reboots.

---

## 📂 Repository Structure

```text
├── src/
│   ├── mql5/
│   │   ├── Master_Copier_Engine.mq5       # Master Broadcaster EA
│   │   └── Slave_Copier_Engine.mq5        # Universal Slave EA (v7.5)
│   └── mql4/
│       └── Slave_Copier_Engine_MT4.mq4    # MT4 Cross-Platform Slave EA
├── automation/
│   ├── 01_Setup_Terminals.ps1             # Multi-Terminal Clean Provisioning
│   ├── 03_Deep_Watchdog.ps1               # 24/7 Staggered Fleet Supervisor
│   └── 04_Setup_Slaves_Engine.ps1         # Auto-Skip Account Setup Wizard
├── tools/
│   ├── 1_Launch_Master_Terminal.bat       # Master Launcher
│   ├── Setup_Slaves_Wizard.bat            # Interactive Sequential Setup Wizard
│   ├── Start_System_Watchdog.bat          # Background Fleet Watchdog Runner
│   └── 2_WIPE_AND_RESET_SYSTEM.bat        # Safe Signal Buffer Cleaner
└── docs/
    └── PRODUCTION_MANUAL_AR.md            # Arabic Operations Manual
