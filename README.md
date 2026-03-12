# NAVIGATOR — Production-Grade Distributed AI Computer-Use Agent
### A Complete Technical Reference: v5 → v6 → v6.1 → v7 Series

Creator: Tassie Cow Herder
---

## Table of Contents

1. [What Is Navigator?](#1-what-is-navigator)
2. [Architecture Overview](#2-architecture-overview)
3. [Version History & Graft Lineage](#3-version-history--graft-lineage)
   - [v5.0 — The Foundation](#v50--the-foundation)
   - [v6.0 — Grounding, Undo & Cost Routing](#v60--grounding-undo--cost-routing)
   - [v6.1 — Closed-Loop Visual Feedback](#v61--closed-loop-visual-feedback)
   - [v7.0 — Full Graft: Complete & Executable](#v70--full-graft-complete--executable)
4. [Feature Matrix](#4-feature-matrix)
5. [Requirements](#5-requirements)
6. [Installation](#6-installation)
7. [Configuration](#7-configuration)
8. [Deployment](#8-deployment)
   - [Single-Node](#single-node)
   - [Distributed Multi-Node](#distributed-multi-node)
   - [Docker](#docker)
9. [CLI Reference](#9-cli-reference)
10. [Component Deep-Dives](#10-component-deep-dives)
11. [Strengths](#11-strengths)
12. [Weaknesses & Known Limitations](#12-weaknesses--known-limitations)
13. [Suggested Improvements](#13-suggested-improvements)
14. [Future Enhancements](#14-future-enhancements)
15. [Security Considerations](#15-security-considerations)
16. [Troubleshooting](#16-troubleshooting)

---

## 1. What Is Navigator?

Navigator is a **production-grade, distributed AI computer-use agent**. It observes a live desktop screen, reasons about what it sees using a large vision-language model (LLM), and autonomously executes mouse clicks, keyboard input, scrolling, hotkeys, and clipboard operations to accomplish natural-language tasks — without any GUI framework hooks, accessibility APIs, or browser extensions.

**In plain terms:** you describe a task in English (`"Open Excel, find the Q3 sheet, and export it as CSV"`), and Navigator takes over the keyboard and mouse to do it, watching the screen the whole time to verify progress.

### Primary Use Cases

| Domain | Examples |
|---|---|
| **Desktop Automation** | File management, batch renaming, data entry across legacy apps |
| **QA & Testing** | End-to-end UI testing for desktop software without test frameworks |
| **Research Assistance** | Web scraping, document gathering, cross-application data collection |
| **Enterprise Workflows** | Multi-step processes spanning email, spreadsheets, ERPs, and browsers |
| **Robotic Process Automation (RPA)** | Drop-in replacement for fragile selector-based RPA bots |
| **Accessibility** | Assisting users by automating repetitive computer tasks |

Navigator's key differentiator from traditional RPA tools is its **vision-first approach**: it sees the screen like a human does, making it robust to UI changes that break selector-based automation.

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        NavigatorAgent                           │
│                                                                 │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────────┐  │
│  │ ScreenBuffer │   │  ThinkAndAct │   │  ActionExecutor    │  │
│  │              │   │              │   │                    │  │
│  │ • Async cap  │──▶│ • Vision LLM │──▶│ • PyAutoGUI        │  │
│  │ • Frame dedup│   │ • OCR hints  │   │ • UndoStack (v6)   │  │
│  │ • Dyn count  │   │ • Grounding  │   │ • dHash verify(v6.1│  │
│  │ • phash sim  │   │ • Reflection │   │ • Recovery trigger │  │
│  └──────────────┘   │ • History    │   └────────────────────┘  │
│                     │   compress   │                            │
│  ┌──────────────┐   │ • Memory ret │   ┌────────────────────┐  │
│  │  SafetyGuard │   └──────────────┘   │   AuditLog         │  │
│  │              │                      │                    │  │
│  │ • Semantic   │   ┌──────────────┐   │ • HMAC-signed JSONL│  │
│  │   classifier │   │ LongTermMem  │   │ • Integrity verify │  │
│  │ • Pattern    │   │              │   └────────────────────┘  │
│  │   blocking   │   │ • SQLite/Redis│                          │
│  │ • Human gate │   │ • Embeddings │   ┌────────────────────┐  │
│  └──────────────┘   │ • Cosine sim │   │DistributedTaskQueue│  │
│                     └──────────────┘   │                    │  │
│  ┌──────────────┐                      │ • Redis pub/sub    │  │
│  │ CircuitBreaker   ┌──────────────┐   │ • Worker heartbeat │  │
│  │ + RateLimiter│   │GroundingEngine   │ • Multi-node coord │  │
│  │              │   │ Florence-2   │   └────────────────────┘  │
│  │ • Jitter back│   │ (v6, opt.)   │                           │
│  │ • Throttle   │   └──────────────┘                           │
│  └──────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────▼────────┐
                    │  xAI Grok API    │
                    │  (vision + embed)│
                    └──────────────────┘
```

### Data Flow Per Step

```
Screen capture → phash dedup → dynamic frame selection
     → OCR text extraction  →  Florence-2 grounding (optional)
          → LLM call (history + memory + reflection hint)
               → Action JSON parsed → Safety check
                    → Execute (pyautogui) → dHash verify
                         → Recovery trigger if no change
                              → History update → Audit log
```

---

## 3. Version History & Graft Lineage

### v5.0 — The Foundation

v5.0 established the complete production scaffold that all subsequent versions build upon. It introduced seven major upgrades over the earlier v4.0 prototype:

| # | Upgrade | Description |
|---|---|---|
| 1 | **Real Embeddings** | Replaced bag-of-words with xAI / Voyage / Nomic API embeddings (1024-d fixed vectors) with automatic fallback chain |
| 2 | **Lightweight OCR** | pytesseract or EasyOCR extracts screen text and injects it as a grounding hint in every LLM message |
| 3 | **Dynamic Frame Count** | 1–2 frames on static screens, 4–6 on moving screens; ~30–50% token reduction during idle phases |
| 4 | **History Compression** | Every N steps, a compact LLM-generated summary replaces raw turn history, keeping context bounded |
| 5 | **Coarse MOVE_TO Action** | New action type moves cursor to approximate UI element centre before a precise click, tolerating ±15 px noise |
| 6 | **Reflection / Self-Critique** | Every `reflection_interval` steps, a second LLM call evaluates progress and detects repetitive loops |
| 7 | **Screenshot Embedding Cache** | LRU dict caches phash → embedding pairs; duplicate frames skip the API call entirely |

**Carried over from v4.0:** safety/security (semantic checker, prompt-injection defence, HMAC audit log, action signing), memory/state (Redis-backed LTM, SQLite session store with WAL and connection pool), error handling (structured retry with jitter, circuit breaker, graceful degradation, signal handlers), performance (async frame pipeline, connection pooling, adaptive rate limiter), and distribution (worker discovery via Redis pub/sub, distributed task queue, node heartbeat).

---

### v6.0 — Grounding, Undo & Cost Routing

v6.0 was derived from v5.0 by grafting three first-principles improvements. Each was designed to be modular and opt-in, with heavy dependencies guarded by `try/except`.

#### Upgrade 1 — Florence-2 On-Device Visual Grounding

The LLM previously had to guess pixel coordinates from vision alone. v6.0 introduced an **optional on-device object detection step** using Microsoft Florence-2-base (or Florence-2-large for production). Florence-2 runs locally (CPU or GPU) and outputs labelled bounding boxes for every detected UI element. These are injected into the LLM message as:

```
DETECTED UI ELEMENTS (Florence-2 grounding):
• Login button @ center (1423, 847) bbox [1380, 830, 1466, 864]
• Username field @ center (760, 312) bbox [520, 295, 1000, 329]
```

The LLM is instructed to **prefer these exact centres** for MOVE_TO and CLICK actions. This eliminates coordinate guesswork and dramatically improves click accuracy on complex UIs.

**Trade-off:** Florence-2-base adds ~500 MB VRAM / ~2 GB RAM and ~200–800 ms per frame on CPU. It is disabled by default (`grounding_enabled: false`) and activated with `--grounding`.

#### Upgrade 2 — UNDO Action Type + UndoStack

Before every mutating action (click, type, hotkey, move), the executor snapshots:
- Current mouse position
- Current clipboard contents

These are stored in a LIFO `UndoStack`. The LLM can now output `{"type": "undo"}` to restore the most recent snapshot, enabling rudimentary reversal of accidental or failed actions.

#### Upgrade 3 — Cost-Aware Model Routing

In v5.0, the expensive `grok-4-vision-preview` model was used for **every** API call: main actions, reflection, history compression, and fact extraction. v6.0 routes auxiliary calls (reflection, compression) to a cheaper mini model (`grok-3-mini` by default), yielding approximately **60% cost reduction** on non-vision steps with no measurable accuracy loss.

New config fields:
- `reflection_model` (default: `grok-3-mini`)
- `compression_model` (default: `grok-3-mini`)

---

### v6.1 — Closed-Loop Visual Feedback

v6.1 addressed a subtle but critical failure mode: **silent action failures**. When the agent clicks a button that is disabled, types into a field that doesn't accept focus, or scrolls in the wrong region, the screen remains unchanged — but v5/v6 had no mechanism to detect this. The agent would simply proceed, assuming success, leading to confusing multi-step failures.

v6.1 introduced a **perceptual feedback loop** using difference hashing (dHash):

1. Before every mutating action, capture the screen's dHash.
2. After execution, capture again.
3. If `pre_hash == post_hash`: the screen did **not change**.
4. Inject a **RECOVERY TRIGGER** message into history:

```
RECOVERY TRIGGER: No visual change detected after the last action.
The screen is identical to the previous frame. The action had ZERO effect.
Re-plan a completely different strategy or verify the element is visible and clickable.
```

The LLM receives this on the very next step and is forced to reconsider its strategy rather than repeat the same failing action.

Additionally, v6.1 hardened the Florence-2 parser:
- More robust regex handles punctuation, hyphens, and non-ASCII labels
- Centre deduplication within ±5 px prevents near-duplicate detections
- Hard cap of 20 elements prevents context bloat
- Full error resilience: malformed coordinate values are skipped gracefully

---

### v7.0 — Full Graft: Complete & Executable

v7.0 is the **definitive merge** of v5.0 (complete runnable base), v6.0 (grounding, undo, cost routing), and v6.1 (dHash feedback loop, hardened parser). It also fixes a bug present in all prior versions:

**v7 Bug Fix:** Fact extraction (`_extract_and_store_fact`) was using `self.cfg.model` (the expensive vision model) for the short 15-word key-fact extraction task. v7.0 routes this to `cfg.fact_extract_model` (default: `grok-3-mini`), completing the cost-aware routing applied to all auxiliary LLM calls.

New config field:
- `fact_extract_model` (default: `grok-3-mini`)

v7.0 guarantees:
- Zero stubs or truncated class bodies
- All `NavigatorAgent.run()` loop improvements fully integrated
- Full CLI with all flags from v5, v6, and v6.1
- Backward-compatible configuration (all new fields have sensible defaults)

#### Complete v5 → v7 Improvement Summary

| Feature | v5 | v6 | v6.1 | v7 |
|---|---|---|---|---|
| Real embeddings (API) | ✅ | ✅ | ✅ | ✅ |
| OCR text grounding | ✅ | ✅ | ✅ | ✅ |
| Dynamic frame count | ✅ | ✅ | ✅ | ✅ |
| History compression | ✅ (vision model) | ✅ (mini model) | ✅ | ✅ |
| Self-reflection | ✅ (vision model) | ✅ (mini model) | ✅ | ✅ |
| MOVE_TO coarse action | ✅ | ✅ | ✅ | ✅ |
| Embedding LRU cache | ✅ | ✅ | ✅ | ✅ |
| Florence-2 grounding | ❌ | ✅ (opt-in) | ✅ (hardened) | ✅ (hardened) |
| UNDO action type | ❌ | ✅ | ✅ | ✅ |
| dHash visual feedback | ❌ | ❌ | ✅ | ✅ |
| Recovery trigger injection | ❌ | ❌ | ✅ | ✅ |
| Fact extraction (mini model) | ❌ | ❌ | ❌ | ✅ (bug fix) |
| Fully executable (no stubs) | ✅ | ⚠️ partial | ⚠️ partial | ✅ |

---

## 4. Feature Matrix

### Core Capabilities

| Capability | Implementation |
|---|---|
| Screen capture | `mss` — zero-copy memory-mapped frame grab |
| Frame deduplication | DCT-based perceptual hash (phash), configurable similarity threshold |
| Vision LLM | xAI Grok-4-vision-preview via OpenAI-compatible API |
| OCR grounding | pytesseract (preferred) or EasyOCR (fallback) |
| Visual grounding | Florence-2-base/large (optional, local inference) |
| Action execution | PyAutoGUI (mouse, keyboard, scroll, hotkey, clipboard) |
| Safety classification | Separate safety LLM + deterministic pattern check + optional human gate |
| Long-term memory | SQLite (default) or Redis; real embedding retrieval with cosine similarity |
| Session persistence | SQLite WAL with thread-local connection pool |
| Audit logging | Append-only HMAC-signed JSONL |
| Distribution | Redis pub/sub task queue, multi-node heartbeat, worker-mode poller |
| Rate limiting | Token-bucket with adaptive throttle multiplier feedback loop |
| Fault tolerance | Circuit breaker (open/half-open/closed), jitter backoff, graceful degradation |
| History management | Compression with mini LLM + configurable rolling window |
| Self-reflection | Periodic loop detection and strategy correction |
| Undo | Mouse position + clipboard snapshot before every mutating action |
| Visual change detection | dHash pre/post comparison; recovery trigger on no-change |

---

## 5. Requirements

### Python

- **Python 3.10+** (required for `match`/`case` structural pattern matching in `ActionExecutor`)
- **Python 3.11+** recommended (built-in `tomllib`, improved type hint resolution)

### Required Python Packages

```
openai>=1.0.0          # API client (OpenAI-compatible, used for xAI)
numpy>=1.24.0          # Array operations (embeddings, frame hashing)
opencv-python>=4.8.0   # Frame processing, phash, OCR preprocessing
mss>=9.0.0             # Cross-platform screen capture
pyautogui>=0.9.54      # Mouse and keyboard control
```

### Optional Python Packages

```
# OCR (install at least one for OCR grounding)
pytesseract>=0.3.10    # Tesseract wrapper (recommended — faster)
easyocr>=1.7.0         # Pure-Python OCR (heavier, no system dependency)

# Clipboard support
pyperclip>=1.8.2       # Clipboard read/write for UNDO and CLIPBOARD actions

# Redis (for distributed mode and Redis memory backend)
redis>=5.0.0           # Redis client

# TOML config file support
tomli>=2.0.0           # Python < 3.11 only (3.11+ has built-in tomllib)

# Florence-2 visual grounding (optional, GPU recommended)
transformers>=4.40.0   # HuggingFace Transformers
torch>=2.1.0           # PyTorch
Pillow>=10.0.0         # PIL for image conversion

# Alternative embedding providers
# (keys set via env var, no additional packages needed beyond openai)
```

### System Dependencies

| Dependency | Purpose | Platform |
|---|---|---|
| **Tesseract OCR** | Backend for pytesseract | Linux: `apt install tesseract-ocr` / macOS: `brew install tesseract` / Windows: installer |
| **Redis** | Distributed task queue and memory backend | Optional; local or remote |
| **CUDA-capable GPU** | Florence-2 inference acceleration | Optional; CPU works but is slower |
| **Display server** | Screen capture target | X11 / Wayland / macOS Quartz / Windows GDI |

### API Keys

| Variable | Purpose | Required |
|---|---|---|
| `XAI_API_KEY` | xAI Grok vision + embedding API | **Yes** |
| `VOYAGE_API_KEY` | Voyage AI embeddings | No (alternative) |
| `NOMIC_API_KEY` | Nomic Atlas embeddings | No (alternative) |
| `NAVIGATOR_HMAC_KEY` | Audit log signing key | No (defaults insecurely) |
| `NAVIGATOR_NODE_ID` | Distributed node identity | No (auto-generated) |

---

## 6. Installation

```bash
# 1. Clone / copy the navigator script
cp UnifiedSystemv7.py navigator.py

# 2. Create and activate a virtual environment
python3.11 -m venv nav-env
source nav-env/bin/activate   # Windows: nav-env\Scripts\activate

# 3. Install required dependencies
pip install openai numpy opencv-python mss pyautogui

# 4. Install optional dependencies (recommended)
pip install pytesseract pyperclip redis tomli

# 5. Install Tesseract system binary (Linux example)
sudo apt-get install -y tesseract-ocr

# 6. (Optional) Install Florence-2 dependencies for visual grounding
pip install transformers torch Pillow
# GPU support: pip install torch --index-url https://download.pytorch.org/whl/cu121

# 7. Set required environment variables
export XAI_API_KEY="your-xai-api-key-here"
export NAVIGATOR_HMAC_KEY="$(python3 -c 'import secrets; print(secrets.token_hex(32))')"

# 8. Verify installation
python navigator.py --help
```

---

## 7. Configuration

Navigator supports three configuration methods, applied in this priority order:
**CLI flags > TOML file > Environment variables > Code defaults**

### TOML Config File

```toml
# navigator.toml
model = "grok-4-vision-preview"
safety_model = "grok-4-safety-preview"
reflection_model = "grok-3-mini"
compression_model = "grok-3-mini"
fact_extract_model = "grok-3-mini"

capture_fps = 10
max_steps = 50
step_pause = 0.4

embedding_provider = "xai"
embedding_dim = 1024
embed_cache_size = 64

enable_ocr = true
ocr_backend = "auto"   # "tesseract" | "easyocr" | "auto" | "none"
ocr_max_chars = 1200

grounding_enabled = false
grounding_model = "microsoft/Florence-2-large"   # large for production

reflection_enabled = true
reflection_interval = 5
history_compress_enabled = true
history_compress_every = 10
history_compress_keep = 4
history_limit = 30

memory_backend = "sqlite"   # "sqlite" | "redis"
session_db_path = "navigator_sessions.db"
memory_ttl_seconds = 86400
memory_top_k = 5

rate_limit_calls_per_min = 25
circuit_breaker_threshold = 5
circuit_breaker_reset_sec = 60

audit_log_path = "navigator_audit.jsonl"
enable_audit_log = true
```

```bash
python navigator.py --config navigator.toml "Open the browser and search for quarterly reports"
```

### Environment Variables

Every `Config` field can be overridden with `NAV_<FIELD_NAME_UPPERCASE>`:

```bash
export NAV_MAX_STEPS=100
export NAV_GROUNDING_ENABLED=true
export NAV_MEMORY_BACKEND=redis
export NAV_REDIS_URL=redis://myredis:6379/0
export NAV_REFLECTION_MODEL=grok-3-mini
export NAV_EMBEDDING_PROVIDER=voyage
```

### Key Configuration Parameters

| Parameter | Default | Description |
|---|---|---|
| `model` | `grok-4-vision-preview` | Main vision + action model |
| `safety_model` | `grok-4-safety-preview` | Safety classification model |
| `reflection_model` | `grok-3-mini` | Self-critique model (cheap) |
| `compression_model` | `grok-3-mini` | History summarisation model (cheap) |
| `fact_extract_model` | `grok-3-mini` | Memory fact extraction model (cheap) |
| `max_steps` | `30` | Hard cap on agent steps per task |
| `reflection_interval` | `5` | Steps between self-critique calls |
| `history_compress_every` | `10` | Steps between history compression |
| `grounding_enabled` | `false` | Activate Florence-2 visual grounding |
| `embedding_provider` | `xai` | `xai` / `voyage` / `nomic` / `bow` |
| `memory_backend` | `sqlite` | `sqlite` / `redis` |
| `stuck_consecutive` | `3` | Frames before stuck detection fires |
| `max_stuck_recoveries` | `4` | Auto-recovery attempts before giving up |

---

## 8. Deployment

### Single-Node

Basic single-task execution:

```bash
# Set API key
export XAI_API_KEY="xai-..."

# Run a task
python navigator.py "Open Notepad, type 'Hello World', and save to Desktop as hello.txt"

# With OCR grounding disabled (faster, less accurate)
python navigator.py --no-ocr "Click the Start button"

# With Florence-2 grounding enabled (slower, more accurate clicks)
python navigator.py --grounding "Find and click the Log
