# Advanced Developer Guide

This document is for readers who want to keep changing the project, open PRs, or build on top of it.

Language versions:

- [简体中文](advanced.md)
- English (this page)

If you only want to configure and use the project, start with:

- [README](../README.en.md)
- [Development Log (2026-04-22)](development-log-2026-04-22.en.md)
- [GitHub Actions Guide](../.github/workflows/README.en.md)

---

## Project Structure

| File | Purpose |
| --- | --- |
| [`app/deploy.py`](../app/deploy.py) | Main runtime entry, responsible for browser startup, login, claiming, and scheduling |
| [`app/services/epic_authorization_service.py`](../app/services/epic_authorization_service.py) | Login, login-result listeners, and post-login validation |
| [`app/services/epic_games_service.py`](../app/services/epic_games_service.py) | Weekly freebie discovery, product-page entry, add-to-cart, checkout, and checkout verification handling |
| [`app/settings.py`](../app/settings.py) | Environment variables, model routing, and defaults |
| [`app/extensions/llm_adapter.py`](../app/extensions/llm_adapter.py) | Gemini / AiHubMix / GLM compatibility adapter |
| [`.github/workflows/epic-gamer.yml`](../.github/workflows/epic-gamer.yml) | GitHub Actions workflow entry |

---

## Local Development

```bash
uv sync
uv run black . -C -l 100
uv run ruff check --fix
```

Notes:

1. This repository currently does not recommend adding extra test runs.
2. When changing the captcha chain, preserve logs and screenshots first.
3. When changing the checkout flow, prioritize "do not report success unless success is actually confirmed."

---

## Real Pitfalls Encountered During This Adaptation

These are not hypothetical issues. They all happened during real development and were explicitly fixed.

### 1. GLM is not a simple Base URL replacement

`hcaptcha-challenger` internally depends on a `google-genai`-style multimodal interface.

That means you cannot support GLM by only changing `GEMINI_BASE_URL` to Zhipu's endpoint.

The actual work is to preserve the upper-layer call pattern while converting images, messages, and structured outputs into a format GLM accepts in the adapter layer.

---

### 2. Challenge types really do change across phases

The challenge type during login is not guaranteed to match the challenge type during checkout.

| Phase | Challenge type |
| --- | --- |
| Login | `image_drag_single` |
| Checkout | `image_label_multi_select` |

If the adapter only handles drag challenges, the flow can still die on the second verification step at checkout.

---

### 3. GLM output format is not stable

The following response forms were seen in real runs:

| Response form | Meaning |
| --- | --- |
| `Source Position: (...)` | Coordinate text |
| `{"source": [...], "target": [...]}` | Structured drag coordinates |
| `{"answer":"..."}` | A string wrapped inside `answer` |
| `image_label_multi_select` | Only the challenge type name |
| Semi-structured JSON | Incomplete or malformed responses |

That is why [`llm_adapter.py`](../app/extensions/llm_adapter.py) now contains a lot of fallback logic that unwraps content and remaps it into the schema expected by the challenger.

---

### 4. Epic checkout can show more than hCaptcha

The following states were all confirmed during checkout:

| Scenario | Seen in real runs |
| --- | --- |
| `Device not supported` | Yes |
| `One more step` | Yes |
| An extra checkout iframe | Yes |
| The page still sitting on `Place Order` | Yes |

Because of that, [`epic_games_service.py`](../app/services/epic_games_service.py) now does all of the following:

1. Detects and tries to dismiss the device-not-supported dialog.
2. Detects checkout security checks explicitly.
3. Loops after `Place Order` to observe the actual result instead of assuming success.
4. Refuses to report success until success is confirmed.

---

### 5. Ownership detection cannot scan the whole page loosely

At one point, copyright text like `owned by ...` was incorrectly interpreted as "already owned."

The fix was:

1. Look at the purchase button and checkout state first.
2. Only accept high-confidence success markers.

---

### 6. Artifacts are critical

Checkout problems were not diagnosed from console output alone. These files were essential:

| File | Why it matters |
| --- | --- |
| `purchase_debug/*.png` extracted from `epic-runtime-<run_id>` | Shows the actual rendered page |
| `purchase_debug/*.txt` extracted from `epic-runtime-<run_id>` | Shows page text and iframe text |
| Log files extracted from `epic-logs-<run_id>` | Shows the full execution chain |

Without those artifacts, many checkout failures would still be guesswork.

---

## Maintenance Priorities

If you continue maintaining this project, keep watching these classes of change first:

1. Whether Epic changes the captcha type on the login page.
2. Whether product-page button labels change.
3. Whether checkout iframe behavior or `Place Order` behavior changes.
4. Whether GLM / Gemini response formats change again.
5. Whether the GitHub Actions runtime environment changes.
