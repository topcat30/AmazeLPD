<div align="center">

<img src="AmazeLPD_logo.png" alt="AmazeLPD" width="120"/>

# AmazeLPD
### LPD/LPR & IPP Print Gateway for Windows

**Receive print jobs from AS/400, Linux, Unix, SAP, mainframe, and CUPS — and forward them to any Windows document-processing platform or print queue.**

[![License: Proprietary](https://img.shields.io/badge/License-Proprietary-red.svg)](LICENSE)
[![Platform: Windows](https://img.shields.io/badge/Platform-Windows-blue.svg)](https://amazelpd.com)
[![.NET Framework 4.8](https://img.shields.io/badge/.NET-Framework%204.8-purple.svg)](https://amazelpd.com)
[![Website](https://img.shields.io/badge/web-amazelpd.com-da781f.svg)](https://amazelpd.com)

[**Free Trial**](https://amazelpd.com) · [**Buy a Licence**](https://amazelpd.com#pricing) · [**Activate**](https://amazelpd.com/activate.html) · [**Getting Started**](#quick-start)

---
</div>

## The Problem

Microsoft has **deprecated the Windows LPD Print Service** (deprecated since Windows Server 2012 R2, with removal announced). Thousands of enterprises still rely on LPD/LPR to route print jobs from **AS/400 output queues, Linux, Unix, SAP, Oracle EBS, and mainframe systems** to Windows. When LPD is removed, those print pipelines break — clients that depend on it simply stop printing.

**AmazeLPD is the supported replacement.** It runs as a Windows service, receives print jobs over **LPD/LPR (RFC 1179, TCP 515)** or **IPP/1.1 (TCP 631)**, and forwards each job to your destination — an **HTTP/REST** document platform or a **local Windows print queue** — with no changes to the sending side.

---

## Proven under load

AmazeLPD was stress-tested head-to-head against the Microsoft Windows LPD service on the **same hardware** (8 vCPU, ~4 GB Windows VM), driving **500 concurrent clients × 5,000 jobs**, deliberately pushed to saturation.

| Metric | Microsoft LPD | **AmazeLPD** |
|---|---|---|
| Jobs succeeded | 205 / 5,000 (**4.1%**) | **4,989 / 5,000 (99.8%)** |
| Sustainable throughput (<2% fail) | 0 jobs/s (overloaded throughout) | **378 jobs/s** |
| Median acknowledgement latency | 6,934 ms | **152 ms** |
| Latency growth under overload | 13.3× | **1.4×** |
| Connections starved (never acknowledged) | 1,916 (**38%**) | **0** |
| Acknowledged-then-lost jobs | — | **0 (none)** |

Microsoft LPD showed the classic worker-starvation collapse; AmazeLPD **degraded gracefully instead of collapsing** — absorbing a 4,733-item internal backlog and draining it cleanly to zero without losing a single acknowledged job. Its 11 failures (0.2%) were all connection-level on the load host, never data loss: **AmazeLPD acknowledges a job only after it is durably written to disk**, so the client's success count always matches what's actually on the server — there is no acknowledged-but-lost failure mode.

> *Figures are AmazeLPD's own controlled stress testing, measured on identical hardware. See the Technical Architecture & Evaluation Report for full methodology.*

---

## Who is this for?

| You have... | AmazeLPD does... |
|---|---|
| IBM i / AS/400 output queues printing via LPR | Receives the job, forwards to your document platform or printer |
| Linux / Unix systems printing to Windows | Drops in as a replacement LPD endpoint |
| SAP or Oracle EBS legacy spool output | Accepts the print stream, routes it onward |
| Mainframe (z/OS JES) LPD output | Receives and forwards without changes to the mainframe |
| CUPS / macOS / modern Linux clients using IPP | Add AmazeLPD as a remote IPP printer — no LPR setup needed |
| Microsoft LPD being removed from Windows Server | Replaces it transparently — no changes to print clients |

---

## Key Features

- **Windows service** — installs and runs as a standard Windows service under the least-privilege **NetworkService** account; starts automatically and restarts on failure.
- **RFC 1179 compliant** — works with any LPR client: AS/400, Linux, CUPS, mainframe, and Windows `lpr.exe`.
- **IPP/1.1 support** — every queue is also exposed as a CUPS-compatible IPP printer (`ipp://server:631/printers/<QueueName>`); LPD and IPP share one spooling pipeline, so each queue is reachable both ways with no extra configuration.
- **Two destination types** — forward to an **HTTP/REST** endpoint (with Basic, Bearer, or API-key auth and custom headers/fields) **or** to a **local Windows print queue** (native winspool), per queue.
- **Acknowledge-after-spool integrity** — a job is acknowledged only once its bytes are durably on disk; no acknowledged-then-lost failure mode.
- **Concurrent queue processing** — a fast, highly-parallel receive path is decoupled from throttled dispatch; queues process simultaneously and none blocks another.
- **Per-queue configuration** — each queue has its own destination, parameters, and data conditioning.
- **Data conditioning** — strip IBM i (i-series) control characters, normalise LF/CR to CRLF, wrap the payload in an XML/JSON envelope (Base64 + custom parameters), or Base64-encode — all optional, per queue.
- **Structured diagnostic logging** — self-rotating, time-bounded log with a watchdog heartbeat, pipeline counters, slow-operation warnings and full exception traces; answers *"what is the service doing right now, and why did that job fail?"* without a debugger.
- **Job history & audit** — SQLite ledger of every job received, processed, and delivered, with per-job status, retry and retention.
- **Crash recovery** — on start-up, jobs stuck in Processing reset to Pending, orphaned spool files are recovered, and genuinely-missing jobs are failed cleanly.
- **Tunable** — concurrency, receive-thread floor (RAM auto-sized), timeouts, retention and conditioning are all operator settings.
- **CSV bulk import** — import hundreds of queues at once from a spreadsheet.
- **No heavy dependencies** — embedded SQLite; no SQL Server, no extra runtime.
- **7-day free trial** — full functionality, no credit card.

---

## Quick Start

**1. Install**
```
Download from https://amazelpd.com → run AmazeLPD_Setup.exe → the service installs and starts automatically
```

**2. Create a queue** — open **AmazeLPD Manager** → **New Queue**:
- Queue name: `MYQUEUE` (matches the `-P` in your lpr command)
- Destination: a REST endpoint URL, or a local Windows printer
- Parameters: package/branch and any custom fields your platform needs

**3. Send a test job**

Via LPR:
```cmd
lpr -S <your-server-ip> -P MYQUEUE "C:\test.pdf"
```
Via IPP (CUPS / macOS / Linux):
```bash
lpadmin -p MYQUEUE -E -v ipp://<your-server-ip>:631/printers/MYQUEUE -m everywhere
lp -d MYQUEUE test.pdf
```

**4. Done.** The job appears in the Manager with status **Completed**.

---

## Screenshots

<div align="center">

### Queue Manager
![Queue Manager](AmazeManager.png)

### Job History
![Job History](Jobs.png)

### Queue Configuration
![Queue Config](queueconfig.png)

</div>

---

## Architecture

AmazeLPD deliberately separates a **fast, highly-parallel receive path** from a **throttled processing and dispatch stage**. The on-disk spool folder is the single source of truth for job data; the embedded SQLite database (WAL mode) holds **metadata only**. This one decision — data on disk, not in the database — is what gives it its throughput.

```
LPR Client                          IPP Client
(AS/400 / Linux / Unix /            (CUPS / macOS / Linux
 SAP / Mainframe)                    via lpadmin / lp)
         │                                   │
         │  TCP 515 (RFC 1179)               │  TCP 631 (IPP/1.1)
         ▼                                   ▼
┌─────────────────────────────────────────────────────┐
│                  AmazeLPD Service                    │  Windows service
│   LPD listener + IPP listener → receive workers      │  (NetworkService)
│   → spool file written to disk, THEN acknowledged    │  Acknowledge-after-spool
└──────────────────────────┬──────────────────────────┘
                            │  spool file = source of truth
                            ▼
┌─────────────────────────────────────────────────────┐
│   Ingest (1 FIFO thread) → Supervisor → per-queue    │  SQLite = metadata only
│   workers → global MaxProcessors dispatch semaphore  │  (WAL, single writer)
└──────────────────────────┬──────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────┐
│   Destination:  HTTP/REST endpoint   OR   local      │  per queue,
│   Windows print queue (winspool)     conditioning    │  configurable auth
└─────────────────────────────────────────────────────┘
```

**Components:** the **Service** (listeners, spool processor, dispatch supervisor, per-queue workers, logging); the **Manager** (WPF UI for queue CRUD, live counts, history, retry, settings, licence import); a **Shared library** (SQLite access, domain models, DPAPI credential protection, path resolution); the **LicenseGenerator** (issues RSA-signed licences); and the **Installer** (Inno Setup — registers the service, sets restart-on-failure, ACL-locks the data folders).

**Design highlights:**
- **Receive is decoupled from ingest** — clients keep getting sub-200 ms acknowledgements even while an internal backlog drains, so a slow destination never stalls receiving.
- **Single global write lock + hybrid write model** — a dedicated writer thread batches status/log writes; latency-critical writes (job creation, the Processing claim) run inline. Eliminates `SQLITE_BUSY` job loss; reads use WAL and are lock-free.
- **No double-dispatch** — exactly one worker per queue; a job is claimed as Processing synchronously before the send runs.
- **Fair concurrency** — global `MaxProcessors` slots rotate fairly across all active queues, so one busy queue can't monopolise dispatch.
- **Survives restarts** — pending jobs resume automatically; a reconciliation sweep adopts orphaned spool files without ever racing ingest.

---

## Performance & tuning

All controls are operator settings (in the Manager or config):

| Control | Effect | Default |
|---|---|---|
| `MaxProcessors` | Global cap on concurrent destination dispatches (raise for REST endpoints that absorb concurrency). | 4 (restart to apply) |
| `ReceiveThreadFloor` | Minimum receive-worker floor; `0` auto-sizes from physical RAM and caps to a safe ceiling. | 0 = Auto (restart) |
| `JobTimeoutSeconds` | Per-request REST timeout. | 60 (live) |
| `HistoryRetainDays` | Retention window for completed/failed jobs and their spool files. | 30 (live) |
| `VerboseLogging` | Per-job/per-connection trace logging (keep off on the hot path). | Off (live) |

Measured ceiling: **378 jobs/second** sustained below a 2% failure line at 500 concurrent clients, with a stable ~152 ms median acknowledgement. For realistic LPD traffic (tens of concurrent senders) this is comfortably ahead of demand.

---

## Security

Presented candidly, each posture with its mitigation:

- **Least privilege** — runs as **NetworkService** by default (not LocalSystem); can optionally run under a dedicated domain account or passwordless **gMSA**, selected at install. Ports below 1024 need no elevation.
- **Stored credentials** — REST usernames/passwords/tokens are protected with **Windows DPAPI**; never stored in plaintext.
- **Licence integrity** — licence files are **RSA-signed**, validated against an embedded public key, with optional machine binding and expiry; tampering invalidates the signature.
- **Data at rest** — spool files and the SQLite DB are plaintext on disk; the installer ACL-restricts the data folder — enable BitLocker/volume encryption on the host.
- **Data in transit** — LPD/LPR and IPP are clear-text by design (as is Microsoft LPD); terminate them on a trusted segment, use HTTPS REST destinations, and firewall 515/631 to known senders.

---

## Compatibility

### Sending platforms
| Platform | Method | Notes |
|---|---|---|
| IBM i / AS/400 | OUTQ via LPR | Configure remote system + remote printer queue |
| Linux / Unix / AIX | lpr / CUPS (LPR or IPP) | Standard LPR output, or add as an IPP printer |
| SAP | Spool output via LPR | Configure SAP output device |
| Oracle EBS | Concurrent Manager print | LPR output method |
| Mainframe z/OS | JES spool LPR output | Standard configuration |
| Windows | `lpr.exe` or LPR port | Built in |
| HP-UX / Solaris | lpr | Native |
| macOS / modern Linux / CUPS | IPP (`ipp://server:631/printers/<QueueName>`) | Add as a remote IPP printer — no LPR setup |

### Receiving platforms (what AmazeLPD forwards to)
Any **HTTP/REST** endpoint that accepts document input (AmazeLPD wraps the job with configurable authentication and parameters), or any **local Windows print queue**.

---

## Pricing

| Licence | Price (AUD) | Servers | Includes |
|---|---|---|---|
| **7-day trial** | Free | 1 | Full functionality, no credit card |
| **Perpetual** | **$1,795** one-time | 1 | Own it forever, unlimited queues |
| **Maintenance & Support** | **$429** / year | per server | Software updates + priority support (add-on to a perpetual licence) |
| **Subscription** | **$895** / year | per server | Lower upfront; updates + support included; cancel anytime |

[**Buy a licence at amazelpd.com →**](https://amazelpd.com#pricing)

Each licence is machine-bound and activated with your **order key + the server's Machine ID** at [amazelpd.com/activate.html](https://amazelpd.com/activate.html).

---

## Download & Install

Get the trial or your purchased build from **[amazelpd.com](https://amazelpd.com)**.

**System requirements**
- Windows Server 2016 / 2019 / 2022, or Windows 10 / 11 (x64)
- .NET Framework 4.8 (ships with Windows Update)
- ~50 MB disk
- TCP 515 (LPD) and/or 631 (IPP) available — both configurable

**Installer (recommended)**
1. Run `AmazeLPD_Setup.exe` and accept the licence agreement.
2. The service installs, sets restart-on-failure, and starts automatically.
3. Open **AmazeLPD Manager** from the Start menu; copy your **Machine ID** from the Licence screen to activate.

**Manual installation**
```cmd
cd "C:\Program Files\AmazeLPD\Service"
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\InstallUtil.exe AmazeLPD_Service.exe
sc start AmazeLPD
```

---

## Why AmazeLPD?

| | AmazeLPD | Microsoft LPD | PaperCut | IBM LPD |
|---|---|---|---|---|
| Lifecycle | Actively developed | **Deprecated / removal announced** | Maintained | End-of-life |
| Protocols in | LPD/LPR **and** IPP | LPD/LPR only | LPR + IPP | LPD only |
| Destinations | REST endpoint **or** Windows queue, per queue | Windows queue only | Print management | Queue only |
| Data conditioning (i-series, XML/JSON envelope) | ✅ | ❌ | ❌ | ❌ |
| Management UI | ✅ WPF Manager | ❌ | ✅ (heavy suite) | ❌ |
| Job ledger & audit | ✅ | ❌ | ✅ | ❌ |
| Structured diagnostics | ✅ watchdog + full traces | ❌ minimal | ✅ | ❌ |
| Tunable concurrency / RAM-aware | ✅ | ❌ | Limited | ❌ |
| No SQL Server required | ✅ | ✅ | ❌ | ✅ |
| Demonstrated load (500 concurrent) | **99.8%**, 378 jobs/s | **4.1%**, 0 sustainable | — | — |

---

## FAQ

**Will this work after Microsoft removes LPD from Windows Server?**
Yes — AmazeLPD implements RFC 1179 independently and does not depend on the Microsoft LPD service at all.

**Do I need to change anything on the AS/400 / sending side?**
No. AmazeLPD receives standard LPR jobs — just point your existing output queue or lpr client at the AmazeLPD server IP.

**Can I print from a Mac or modern Linux box without LPR?**
Yes. Every queue is exposed as an IPP printer at `ipp://<server>:631/printers/<QueueName>`; add it with `lpadmin` (or macOS Printers) — jobs land in the same spool as LPR.

**What happens if the destination server is unavailable?**
Jobs are spooled to disk and held Pending, then processed automatically when the destination returns. You can also pause/resume individual queues.

**How many queues / how much load can it handle?**
Each queue has its own destination and parameters; queues run concurrently. In testing it sustained 378 jobs/second at 500 concurrent clients with a ~152 ms median acknowledgement.

**Is the trial fully functional?**
Yes — the 7-day trial is the full product with no feature restrictions.

**What format does AmazeLPD forward jobs in?**
Configurable per queue: raw, or wrapped in an XML/JSON envelope (Base64 payload plus your custom parameters), with i-series stripping and LF/CR normalisation available.

---

## Support

**Maintenance & Support** (updates + priority support) is available as an annual entitlement — see [Pricing](#pricing). Support is provided by email at **support@amazelpd.com**, business hours Australian Eastern time (AEST/AEDT).

---

## Licence

AmazeLPD is proprietary software. See [LICENSE](LICENSE) for full terms.

© 2026 **TopCat Services Pty Ltd** (ABN 89 698 792 960). AmazeLPD is a product of TopCat Services Pty Ltd. All rights reserved.

---

<div align="center">

**[Free Trial](https://amazelpd.com) · [Buy a Licence](https://amazelpd.com#pricing) · [Activate](https://amazelpd.com/activate.html)**

*Solving the Microsoft LPD deprecation — one print queue at a time.*

</div>
