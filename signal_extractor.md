---
name: New Signal Extractor
about: Propose or implement a new signal extractor
title: "[SIGNAL] "
labels: signal-extractor, good-first-issue
assignees: ''
---

## 📡 Signal Name

`signal_name` (snake_case)

## 📂 Category

- [ ] `behavioral` — rate, burst, endpoint repetition
- [ ] `content` — injection, toxicity, PII, secrets
- [ ] `contextual` — IP history, geo, device
- [ ] `temporal` — timing gaps, burst patterns
- [ ] `ai_content` — prompt injection, jailbreak, token abuse

## 🔍 What Does This Signal Detect?

Describe what behavior or pattern this signal measures.

## 📥 Input

What data does the extractor need?
- [ ] `event.content` (the request body / prompt text)
- [ ] `event.metadata` (IP, headers, timestamps)
- [ ] `context` (identity history, recent scores)

## 📤 Output

How will you normalize the value to `[0.0, 1.0]`?

Example:
```python
# raw count of pattern matches → normalize against max expected
value = min(match_count / 5.0, 1.0)
```

## 🧪 Detection Library Used (If Any)

- [ ] `rebuff` (prompt injection)
- [ ] `llm-guard` (jailbreak)
- [ ] `detoxify` (toxicity)
- [ ] `presidio` (PII)
- [ ] `detect-secrets` (API keys / secrets)
- [ ] `re` (regex patterns)
- [ ] `langdetect` (language)
- [ ] `GeoIP2` (geo location)
- [ ] Custom logic (describe below)

## ⚖️ Suggested Weight

Default weight for this signal (0.0–1.0). Remember this is configurable via YAML.

Example: `0.35`

## 📋 Notes

Any edge cases, normalization decisions, or dependencies to flag.
