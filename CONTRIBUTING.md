# Contributing to GuardLayer

Welcome — and thank you for your interest in contributing! This document is written for contributors of all experience levels, including those who are new to security tooling or risk-based systems. Read it fully before opening a PR.

---

## 📌 Project Scope — Read This First

GuardLayer has a deliberately fixed scope. The goal is to build a *focused, well-engineered* tool — not a tool that does everything. Before building something, check that it fits.

**We welcome contributions in these areas:**

* New signal extractor implementations (behavioral, contextual, AI-specific, content)
* Tests for the scoring engine, decision engine, and signal extractors
* CLI improvements — new panels, keyboard shortcuts, better display
* AI provider integrations (new Ollama models, new cloud providers)
* Documentation, examples, integration guides, tutorials
* Bug fixes across any module
* New YAML configuration options

**These are out of scope — PRs in these areas will not be merged:**

* Web UI or browser-based dashboard (the CLI is intentional, not a limitation)
* Full ML model training pipelines
* Distributed system architecture
* Deep framework integrations beyond FastAPI middleware
* Auto-adjusting weights at runtime (weights are human-adjusted via YAML — this is a design decision, not a gap)

If you're unsure whether your idea fits, open a GitHub issue and ask *before* building. This saves everyone time.

---

## 🛠️ Setup Guide

### Prerequisites

You'll need Python 3.10 or higher, and Git. No other tools required for basic development.

### Step-by-Step Setup

```bash
# Step 1: Fork the repo on GitHub, then clone your fork
git clone https://github.com/YOUR_USERNAME/guardlayer.git
cd guardlayer

# Step 2: Create a virtual environment so dependencies stay isolated
python -m venv venv
source venv/bin/activate        # Linux / Mac
venv\Scripts\activate           # Windows

# Step 3: Install the project in editable mode with dev + detection extras
# "editable mode" (-e) means code changes take effect without reinstalling
pip install -e ".[dev,detection]"

# Step 4: Run the test suite to confirm your setup is working
pytest tests/
```

If all tests pass, your environment is ready.

---

## 🧠 Architecture — Understanding the System Before You Touch It

This is the most important section for contributors. **Please read it carefully.** Understanding why the architecture is designed this way will help you write contributions that fit naturally into the codebase — and avoid the common mistakes that cause PRs to be rejected.

### The Core Philosophy

GuardLayer is built around one principle: **no single signal should be able to block a request on its own.**

Most security tools use rules: `if X → block`. GuardLayer rejects this. Instead, it collects many weak signals, combines them into a weighted risk score, and makes a *proportional* decision. This approach is more intelligent, fairer to legitimate users, and much harder to game.

### The Full Pipeline

When a request arrives, it passes through these stages in order:

```
Event
  │
  ▼
Fast Layer          ← sync, < 5ms, handles obvious cases only
  │
  ▼
Signal Extraction   ← all extractors run, each returns a value 0.0–1.0
  │
  ▼
Context Enrichment  ← loads the user's rolling history from the context store
  │
  ▼
Risk Scoring        ← combines signals into a single weighted score
  │
  ▼
Decision Engine     ← maps score to an action: allow/flag/throttle/block
  │
  ▼
Action + Feedback   ← executes the decision, logs everything async
```

Here is what each stage actually does and why it exists:

**Fast Layer** (`layers/fast.py`) runs synchronously and must complete in under 5ms. It handles only the most obvious, cheap-to-check cases — known-bad IP addresses, payloads that exceed a size limit, hard rate limit violations. If a request fails here it exits immediately without going through signal extraction. This exists because running full NLP detection on a 50MB payload is wasteful when you can reject it in 1ms based on size alone.

**Signal Extraction** (`signals/`) runs all signal extractors against the event. Each extractor is independent and produces a `Signal` object with a `value` between 0.0 and 1.0. Extractors can use specialized detection libraries (like `rebuff` for injection, `presidio` for PII) or simple logic. They should *not* know about each other, and they should *not* make decisions — they only measure.

**Context Enrichment** (`core/context.py`) loads the `IdentityContext` for the user making the request. This context contains their known IP addresses, recent risk scores, request count, and session age. A signal extractor that needs context (like the new-IP detector) reads from this store during extraction. This stage is what makes the system "memory-aware" — the same score from a new user and a trusted long-term user means very different things.

**Risk Scorer** (`core/scorer.py`) takes all the signals and computes a single risk score using the weighted additive formula. It also applies interaction bonuses — if two or more high-scoring signals come from *different* signal categories at the same time, the score gets a small amplification. This reflects real-world threat analysis: a rate spike alone is mildly suspicious, but a rate spike *combined* with a prompt injection attempt is much more serious than either alone.

**Decision Engine** (`core/decision.py`) maps the score to one of four actions using configurable threshold bands. It also constructs the reason object — the full explanation of which signals drove the decision and by how much.

### The Score Formula

Every signal extractor outputs a `value` in `[0.0, 1.0]`. The scorer multiplies each value by its configured weight and sums the results:

```
risk = Σ (signal.value × signal.weight)
risk = min(risk + interaction_bonus, 1.0)   # capped at 1.0
```

The decision bands:

| Score        | Decision                                     |
| ------------ | -------------------------------------------- |
| < 0.30       | ALLOW — passes through                      |
| 0.30 – 0.55 | FLAG — passes through but logged for review |
| 0.55 – 0.75 | THROTTLE — delayed or rate-limited          |
| > 0.75       | BLOCK — request rejected with reason        |

**Critical rule: all signal values must be normalized to `[0.0, 1.0]` before being returned.** If your extractor computes a raw count (like number of pattern matches), you must normalize it. The scorer assumes every input is in range — if you return a value of 3.5, the math breaks.

### Key Module Map

| Module                    | What It Does                                                  |
| ------------------------- | ------------------------------------------------------------- |
| `core/engine.py`        | The orchestrator — runs each pipeline stage in order         |
| `core/scorer.py`        | The weighted score formula + interaction bonuses              |
| `core/decision.py`      | Threshold evaluation → action + reason object                |
| `core/context.py`       | Per-identity rolling history store                            |
| `layers/fast.py`        | Sync fast layer — regex, size limits, rate limits            |
| `layers/async_layer.py` | Async pipeline for signal extraction and scoring              |
| `signals/base.py`       | `BaseSignalExtractor`— the interface all signals implement |
| `signals/content.py`    | Content-based signals: injection, jailbreak, toxicity         |
| `signals/pii.py`        | PII and secrets detection signals                             |
| `signals/behavioral.py` | Behavioral signals: rate, burst, entropy                      |
| `signals/contextual.py` | Contextual signals: geo, device, IP history                   |
| `signals/temporal.py`   | Temporal signals: timing gaps, burst patterns                 |
| `providers/base.py`     | `BaseAIProvider`— the interface for AI fallback            |
| `cli/dashboard.py`      | The three-tab blessed terminal UI                             |
| `storage/sqlite.py`     | SQLite-backed persistence layer for local state and records   |
| `feedback/store.py`     | False positive logging and feedback tracking                  |

---

## ✍️ Writing a Signal Extractor

Signal extractors are the most common type of contribution, and they're well-scoped for contributors who are new to the codebase. Each extractor is one class that inherits from `BaseSignalExtractor`.

Here is the complete interface you need to implement:

```python
from guardlayer.signals.base import BaseSignalExtractor, Signal
from guardlayer.core.context import IdentityContext
from guardlayer.models import Event

class ExampleSignal(BaseSignalExtractor):
    # name: unique identifier for this signal, used in logs and decision output
    name = "example_signal"

    # weight: how much this signal contributes to the final score.
    # This is the DEFAULT — it can always be overridden in guardlayer.yaml.
    # Choose a weight that reflects the signal's reliability and severity.
    # A signal that almost never false-positives can have a higher weight.
    weight = 0.30

    # category: which of the 5 signal categories this belongs to.
    # Use: behavioral | content | contextual | temporal | ai_content
    category = "content"

    def extract(self, event: Event, context: IdentityContext) -> Signal:
        """
        Analyze the event and return a Signal.

        Args:
            event: the current request/chat message being evaluated
            context: the identity's rolling history (recent scores, known IPs, etc.)

        Returns:
            Signal with value in [0.0, 1.0]
            0.0 = no concern at all
            1.0 = maximum concern
        """

        # --- your detection logic goes here ---
        raw_count = 0
        patterns = ["suspicious_pattern_a", "suspicious_pattern_b"]
        for pattern in patterns:
            if pattern in event.content.lower():
                raw_count += 1

        # IMPORTANT: normalize to [0.0, 1.0] before returning.
        # In this example, matching all patterns = score of 1.0.
        # Matching none = score of 0.0.
        # Never return a value outside this range.
        normalized_value = min(raw_count / len(patterns), 1.0)

        return Signal(value=normalized_value, weight=self.weight, name=self.name)
```

Once written, place your extractor in the appropriate `signals/` file and register it:

```python
# if it detects content threats → signals/content.py
# if it measures behavior patterns → signals/behavioral.py
# if it reads context/history → signals/contextual.py
# if it measures timing → signals/temporal.py
# if it's AI-specific → signals/content.py (ai_content category)
```

**The single most important rule:** your `extract()` method must never add directly to a risk score or make a decision. It only measures and returns a value. The scorer handles everything else.

---

## 🧪 Writing Tests

Every new signal extractor requires tests. Tests live in `tests/signals/` and follow this pattern:

```python
# tests/signals/test_content.py

def test_xss_signal_detects_script_tag():
    signal = XSSPatternSignal()
    event = Event(content="<script>alert('xss')</script>")
    result = signal.extract(event, context=mock_context())

    # value should be greater than 0 — the signal should have fired
    assert result.value > 0.0

def test_xss_signal_clean_input_returns_zero():
    signal = XSSPatternSignal()
    event = Event(content="What is the weather today?")
    result = signal.extract(event, context=mock_context())

    # clean input should score 0.0
    assert result.value == 0.0

def test_xss_signal_value_is_normalized():
    signal = XSSPatternSignal()
    event = Event(content="<script>" * 100)   # extreme input
    result = signal.extract(event, context=mock_context())

    # value must never exceed 1.0, no matter the input
    assert result.value <= 1.0
```

Run your tests with:

```bash
pytest tests/                                     # run everything
pytest tests/signals/test_content.py -v           # run one file with verbose output
pytest tests/signals/test_content.py::test_name   # run one specific test
```

---

## 🔀 Pull Request Process

Follow this process to give your PR the best chance of being merged quickly:

1. **Branch from `main`** and give it a descriptive name — `feature/signal-xss-detection`, `fix/scorer-normalization-cap`, `docs/signal-tutorial`.
2. **Make focused changes** — one PR should do one thing. A PR that adds a signal extractor, refactors the scorer, and improves the CLI all at once is very hard to review.
3. **Add tests** — new signal extractors require tests before the PR will be reviewed.
4. **Check normalization** — if your signal returns raw counts or floats from a library, make sure you normalize to `[0.0, 1.0]` before returning.
5. **Open the PR** and fill out the template completely.
6. **Expect review within 48–72 hours** — if you haven't heard back, leave a comment to check in.

### PR Checklist

Before submitting, verify that all of these are true:

* [ ] Signal value is normalized to `[0.0, 1.0]`
* [ ] Tests added and all passing locally
* [ ] Follows the `BaseSignalExtractor` interface exactly
* [ ] No direct `risk +=` or decision logic inside the extractor
* [ ] Signal weight is defined as a class attribute (configurable via YAML)
* [ ] Signal is placed in the correct `signals/` file for its category
* [ ] CONTRIBUTING.md scope check passed — this fits within the project's defined scope

---

## 💬 Communication

The best way to communicate depends on what you need:

* **Architecture questions or design discussions** → open a GitHub Discussion before writing any code
* **Bug reports** → open a GitHub Issue using the bug report template
* **New signal ideas** → open a GitHub Issue using the signal extractor template
* **Scope questions** → comment on any related issue and ask before building
* **PR feedback** → respond directly in the PR review thread

---

## 🏷️ Issue Labels

| Label                | What It Means                                                      |
| -------------------- | ------------------------------------------------------------------ |
| `good-first-issue` | Safe for new contributors — no changes to core engine, clear spec |
| `intermediate`     | Requires understanding the signal/scorer pipeline                  |
| `advanced`         | Core engine work — discuss with maintainer before starting        |
| `documentation`    | Docs, examples, tutorials                                          |
| `testing`          | Writing or improving tests                                         |
| `signal-extractor` | Implementing a new signal extractor                                |
| `cli`              | Terminal dashboard improvements                                    |
| `ai-provider`      | New AI fallback provider implementation                            |

---

## 🧭 Design Rules — Non-Negotiable

These rules define the architecture. A PR that violates any of them will not be merged, regardless of how well the code is written.

**1. No blocking on a single signal.** The scorer always combines multiple signals. A single signal, no matter how high it scores, cannot by itself cause a block.

**2. Everything is event-based.** The system processes events. Signals are extracted from events. Decisions are made on events. There is no concept of a "banned user" or persistent block state.

**3. Fast layer does lightweight work only.** Regex, size checks, and rate limit lookups. No NLP, no library calls, no async operations. If it can't run in under 5ms, it doesn't belong in the fast layer.

**4. Signal values are always normalized.** Every extractor outputs a float in `[0.0, 1.0]`. This is the contract the scorer depends on.

**5. Rules are configurable via YAML.** Weights and thresholds live in config files, not hardcoded in Python. A developer should be able to adjust the system's behavior without touching the source code.

**6. All scoring goes through `scorer.py`.** No extractor, middleware, or provider should add directly to a risk score. Everything flows through the scorer so the math stays consistent and auditable.

---

Thank you for contributing to GuardLayer. Every signal extractor, test, and documentation improvement makes the system more useful for the developers who depend on it. 🛡️
