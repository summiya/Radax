<div align="center">

# ⚡ RADAX

### Catch the blockers. Protect the event loop.

**A fast async-safety scanner for Python that catches blocking calls, event-loop hazards, and concurrency mistakes before they reach production.**

![Python](https://img.shields.io/badge/Python-3.11%2B-3776AB?style=for-the-badge&logo=python&logoColor=white)
![asyncio](https://img.shields.io/badge/asyncio-ready-7C3AED?style=for-the-badge)
![FastAPI](https://img.shields.io/badge/FastAPI-aware-009688?style=for-the-badge&logo=fastapi&logoColor=white)
![CLI](https://img.shields.io/badge/CLI-arcade%20mode-FF2D95?style=for-the-badge)
![License](https://img.shields.io/badge/license-MIT-00C853?style=for-the-badge)

```text
╔══════════════════════════════════════════════════════╗
║                     RADAX ⚡                         ║
║                                                      ║
║             ASYNC SAFETY SCANNER                     ║
║                                                      ║
║   ██████████████████████████████████████  100%       ║
║                                                      ║
║   BLOCKERS FOUND: 03                                 ║
║   EVENT LOOP HP: 86%                                 ║
║   STATUS: ⚠ NEEDS ATTENTION                          ║
╚══════════════════════════════════════════════════════╝
```

**[Quick Start](#-quick-start) · [What It Finds](#-what-radax-finds) · [How It Works](#-how-it-works) · [Roadmap](#-roadmap)**

</div>

---

## 👾 What is Radax?

Async Python can *look* asynchronous while quietly blocking the event loop.

```python
async def fetch_data():
    response = requests.get("https://example.com")
    return response.json()
```

The function is `async`. The HTTP call is not.

**Radax scans Python code and points directly to operations that can freeze or degrade async workloads.**

It is designed for modern Python applications using **asyncio, FastAPI, SQLAlchemy, HTTP clients, WebSockets, SSE, background jobs, and AI/LLM streaming**.

---

## 🚀 Quick Start

```bash
pip install radax
```

Scan a project:

```bash
radax scan .
```

Or scan a specific directory:

```bash
radax scan src/
```

### 🕹️ The terminal experience

```text
 RADAX // ASYNC SAFETY SCAN
 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 Scanning src/ .................................. DONE

 🔴 PHZ001  Blocking HTTP call inside async function

    src/services/github.py:42

    async def load_profile():
    >   response = requests.get(url)

    requests.get() blocks the event loop.

    FIX → use httpx.AsyncClient or aiohttp


 🟡 PHZ002  Blocking sleep inside async function

    src/workers/sync.py:18

    async def retry():
    >   time.sleep(2)

    FIX → await asyncio.sleep(2)


 🔴 PHZ003  Synchronous database operation

    src/repositories/user.py:71

    >   session.query(User).first()

    FIX → consider SQLAlchemy AsyncSession


 ──────────────────────────────────────────────────
 🏁 SCAN COMPLETE

 Files scanned        184
 Async functions       47
 Errors                 2
 Warnings               1
 Async health        86/100
 ──────────────────────────────────────────────────
```

---

## 🌈 What Radax Finds

| | Detector | Example |
|---|---|---|
| 🌐 | **Blocking HTTP** | `requests.get()` inside `async def` |
| 💤 | **Blocking sleep** | `time.sleep()` in async code |
| 🗄️ | **Sync database access** | synchronous SQLAlchemy in async paths |
| 📁 | **Blocking file I/O** | large synchronous file operations |
| ⚙️ | **Blocking subprocesses** | synchronous process execution |
| 🧵 | **Thread-offload opportunities** | work suited to `asyncio.to_thread()` |
| ⏳ | **Missing await** | coroutine created but never awaited |
| 🔀 | **Concurrency smells** | independent awaits executed sequentially |
| 🧹 | **Lifecycle problems** | tasks/resources not cleaned up correctly |
| 🤖 | **AI workload hazards** | blocking LLM, embedding and streaming calls |

---

## 💥 Before → After

<table>
<tr>
<td width="50%">

### 🔴 Before Radax

```python
import requests

async def get_data():
    response = requests.get(URL)
    return response.json()
```

**The event loop waits.**

</td>
<td width="50%">

### 🟢 After Radax

```python
import httpx

async def get_data():
    async with httpx.AsyncClient() as client:
        response = await client.get(URL)
        return response.json()
```

**Other tasks keep moving.**

</td>
</tr>
</table>

---

## 🧠 Why this matters

An async server may handle many requests on the same event loop. A blocking operation can stop unrelated coroutines from progressing until that operation finishes.

```text
                    EVENT LOOP
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
          Request A           Request B
              │                   │
              ▼                   ▼
         await I/O           await I/O
              │                   │
              └─────────┬─────────┘
                        │
                   💥 BLOCKING
                     OPERATION
                        │
                        ▼
                 ⚡ RADAX FINDS IT
```

---

## 🏗️ How It Works

Radax is planned as a **static-first analyzer**. It inspects Python source without executing the application.

```text
                  ┌─────────────────┐
                  │  Python Project │
                  └────────┬────────┘
                           ▼
                  ┌─────────────────┐
                  │    AST Parser   │
                  └────────┬────────┘
                           ▼
                  ┌─────────────────┐
                  │ Async Context   │
                  │    Analyzer     │
                  └────────┬────────┘
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
        HTTP Rules     DB Rules     Runtime Rules
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                  ┌─────────────────┐
                  │ Finding Engine  │
                  └────────┬────────┘
                           ▼
                Terminal · JSON · SARIF
```

The goal is deterministic analysis that is fast enough for local development, pre-commit hooks, and CI.

---

## 🧩 Rule System

Every finding has a stable rule ID.

| Rule | Meaning |
|---|---|
| `RDX001` | Blocking HTTP call |
| `RDX002` | Blocking sleep |
| `RDX003` | Sync database access in async context |
| `RDX004` | Blocking subprocess |
| `RDX005` | Blocking filesystem operation |
| `RDX006` | CPU-heavy work on the event loop |
| `RDX007` | Suspicious async wrapper |
| `RDX008` | Coroutine created but not awaited |
| `RDX009` | Independent awaits that may run concurrently |
| `RDX010` | Async resource lifecycle issue |

Future configuration:

```toml
[tool.radax]
select = ["RDX001", "RDX002", "RDX003"]
ignore = ["RDX005"]
```

---

## 🛠️ CLI Vision

```bash
# Scan everything
radax scan .

# Strict CI mode
radax scan . --strict

# Machine-readable output
radax scan . --format json

# Select specific rules
radax scan . --select RDX001,RDX003

# Learn why a finding matters
radax explain RDX003

# Generate configuration
radax init
```

---

## ⚙️ CI/CD

Radax is intended to fit naturally into automated pipelines.

```yaml
- name: Radax async safety scan
  run: |
    pip install radax
    radax scan . --strict
```

Planned output formats include **terminal, JSON, SARIF, and GitHub annotations**.

---

## 🧱 Architecture Vision

```text
src/radax/
├── cli/
│   ├── commands/
│   └── output/
├── analysis/
│   ├── parser.py
│   ├── context.py
│   └── scanner.py
├── rules/
│   ├── base.py
│   ├── http.py
│   ├── sleep.py
│   ├── database.py
│   ├── filesystem.py
│   └── subprocess.py
├── findings/
│   ├── models.py
│   └── severity.py
├── reporters/
│   ├── terminal.py
│   ├── json.py
│   └── sarif.py
└── config/
    └── settings.py
```

The architecture is intended to keep individual detectors isolated, testable, and easy to extend.

---

## 🗺️ Roadmap

**🟣 Level 01 — Core Scanner**  
AST parsing · async-function discovery · rule engine · terminal reporter · blocking HTTP/sleep/subprocess detection

**🔵 Level 02 — Framework Intelligence**  
FastAPI awareness · SQLAlchemy checks · filesystem analysis · thread-offloading suggestions · configuration

**🟢 Level 03 — CI Mode**  
JSON · SARIF · GitHub annotations · severity configuration · baselines

**🟠 Level 04 — Advanced Concurrency**  
Missing awaits · task lifecycle · cancellation · resource cleanup · async context managers

**🔴 Level 05 — AI Workloads**  
LLM SDK calls · SSE streaming · embeddings · vector clients · document pipelines · agent/tool execution

---

## 🎯 Design Principles

> ⚡ **Fast** — scans should feel instant.  
> 🔍 **Static-first** — don't execute the user's application.  
> 🎯 **Low noise** — findings should be actionable.  
> 🧠 **Framework-aware** — understand real Python applications.  
> 🧩 **Extensible** — detectors should be easy to add.  
> 🚀 **CI-friendly** — useful from laptop to production pipeline.

---

## 🕹️ Arcade Mode

Because static analysis does not have to look boring.

```text
┌────────────────────────────────────────────┐
│            ⚡ RADAX // LEVEL 01            │
├────────────────────────────────────────────┤
│                                            │
│   EVENT LOOP        ███████████████░  86   │
│                                            │
│   BLOCKERS          👾 👾                   │
│   WARNINGS          ⚡                      │
│   ASYNC HEALTH      ★★★★☆                  │
│                                            │
│             READY PLAYER DEV               │
└────────────────────────────────────────────┘
```

Professional machine-readable output remains available for CI.

---

## 🤝 Contributing

Radax is just entering **Level 01**.

Bug reports, detector ideas, framework integrations, documentation improvements, and pull requests will be welcome as the project evolves.

---

## 📜 License

MIT

---

<div align="center">

# ⚡ RADAX

### Keep the event loop in the fast lane.

```bash
pip install radax
```

**Built for Python developers who refuse to let one blocking call ruin the game.**

</div>
