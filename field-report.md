# Field Report: Running Claude Code on a Local LLM

Problems encountered making Claude Code genuinely usable against a local model served by `vllm-mlx` on Apple Silicon, and the fixes that went into `cclocal` and the `vllm-mlx` star.

Getting a cloud-grade agent to drive a quantized local model is a *stack of compounding problems* across the harness, inference server, ML runtime, and model itself. Each layer has a failure mode that masquerades as another layer's bug. Most fail **silently**.

---

## Environment

- Apple Silicon, 24 GB unified memory
- Inference: `vllm-mlx` from the `vitorallo/vllm-mlx` star
- Driver: `cclocal` (`bash/run.sh`) — session-scoped env, restricted tool set, no MCP
- Model under test: Gemma-4-26B-A4B (4-bit MoE, ~10–23 tok/s)

---

## Problems, Root Causes, Fixes

### 1. First-run download looked like a hang

- **Symptom:** launch sat at "waiting for server" for minutes, then timed out.
- **Root cause:** blind fixed-duration `/health` poll. Multi-GB HuggingFace download legitimately exceeded it.
- **Fix:** live progress from HF cache directory size (downloaded GB + rate), stall-based timeout that only aborts on genuine no-progress. Partial downloads resume.
- **Residual:** none.

### 2. OOM crashes under agentic load

- **Symptom:** hard crash mid-generation — `[METAL] Insufficient Memory (kIOGPUCommandBufferCallbackErrorOutOfMemory)`. Claude Code left retrying a dead backend.
- **Root cause:** ~15 GB model + growing KV cache (every tool output fed back) exceeds Metal budget. macOS only makes ~75% of RAM GPU-addressable by default — "free RAM" overstates what's usable.
- **Fix:** `memory_preflight` estimates model footprint against effective GPU budget (`iogpu.wired_limit_mb` cap). When tight, offers: (1) shrink server context window, (2) raise GPU wired limit via `sudo` for the session (auto-reverted on exit), (3) both. Silent when ample headroom; `--safe` forces it.
- **Residual:** single artifact larger than the budget still can't be produced in one pass. Hardware/model size is the real lever.

### 3. Write/Edit tool calls that silently did nothing — the central problem

- **Symptom:** model "calls" Write; invocation shown; silence. No file, no error, HTTP 200. Short tools (`ls`) worked.
- **Root cause 3a — output-token truncation.** `Write` serializes the entire file body as output tokens. Old 4096 cap cuts mid-`content`; JSON never closes; no `tool_use` block built; silent HTTP 200.
- **Root cause 3b — fork channel-cleaner destroyed tool calls.** Gemma channel cleaner, on unclosed `<|channel>thought` (from truncated stream), deleted everything to end-of-text — including following tool calls. Also matched channel markers *inside* a file body.
- **Root cause 3c — streaming parser fragility.** Tool-call completion detected by substring-in-delta heuristics; large multi-line content defeats them.
- **Root cause 3d — invalid JSON from weak model.** 4-bit model frequently fails to `\"`-escape quotes in long content; `json.loads` fails; tool input dropped.
- **Fixes:**
  - Output budget raised to tunable knob (`--out-tokens`, default 8192); `server.log` rotated for post-mortems.
  - **Fork D1/D2:** channel cleaner now tool-call-span-safe — stripping only outside `<|tool_call>…<tool_call|>` spans. Covered by `tests/test_gemma4_toolcall_safety.py`.
  - **Fork fail-loud:** `--tool-call-truncation-notice` — truncated tool call returns explicit "write the file in smaller parts" instead of silent HTTP 200.
  - **Proactive guidance:** `cclocal` passes a system-prompt hint to write large files incrementally.
- **Residual:** 3d is a model-capability limit. Intermittent on large quote-dense content, not absolute. Mitigation: stronger model or chunked writes.

### 4. ML runtime dependency fragility

- **Symptom:** every request 500'd with `RuntimeError: There is no Stream(gpu, 1) in current thread`.
- **Root cause:** `mlx 0.31.2` broke GPU streams in `ThreadPoolExecutor` worker threads. `mlx-lm 0.31.2` broke `BatchGenerator` API. `mlx-vlm 0.5.0` hard-requires the broken `mlx`. Fork's dependency floors were loose (`>=`), so reinstall silently resolved to the broken stack.
- **Fix:** `pyproject.toml` pins `mlx==0.31.1`, `mlx-lm==0.31.1`, caps `mlx-vlm<0.5.0`. Reproducible installs.
- **Residual:** pin freezes out later runtime improvements. Bisection plan documented; exit criteria require a real generation test.

### 5. Permission classifier vs. a slow serialized model

- **Symptom:** in auto mode, Write blocked — "model temporarily unavailable, so auto mode cannot determine the safety of Write".
- **Root cause:** auto mode makes a separate model call to classify each tool action. A local model serializes generation and is slow; the classifier call can't return in time.
- **Fix:** `cclocal` pre-allows its restricted tool set (`--allowedTools`). Pre-approved tools need no safety classification.
- **Residual:** none for the scoped tool set.

### 6. Standard harness behaviours that punish weak models

- **Read-before-write:** Claude Code refuses to `Write`/`Edit` an existing file until it has been `Read`. A weak model flails against it (`ls` instead of `Read`). Not a bug — mitigated by fresh paths or a stronger model. Symptom ("Error writing file") looks like infrastructure failure.
- **"Prompt caching disabled" banner:** benign. Local server doesn't implement server-side caching. No token cost applies locally.

---

## The Fundamental Limit

After truncation, channel-destruction, OOM, runtime crash, and classifier are all solved, the remaining wall is **a weak, quantized local model reliably emitting valid structured output (JSON tool calls) for large, complex actions**. This is a capability limit, not an engineering one. The correct response: fail *loudly*, pre-empt where possible, let the agent chunk the work or use a stronger model.

---

## Scaling

- **Auto-adapts:** memory preflight is RAM-relative and stays silent with headroom; fork fixes and dependency pin are memory-independent correctness fixes.
- **Does not auto-scale:** output-token cap is an explicit conservative default — more RAM doesn't raise it. Raise manually with `--out-tokens`.
- **The biggest lever is the model.** Large memory's real value is enabling a *stronger* model that produces valid JSON for big tool calls and recovers from standard guards without flailing.

---

## Summary

| Area | Improvement |
|---|---|
| Visibility | Live download progress; stall-based readiness timeout; rotated `server.log` |
| Memory | GPU-budget-aware preflight; interactive shrink / GPU-limit raise; session-scoped auto-reverted `sudo` |
| Tool calls | Tunable output budget; tool-call-span-safe channel cleaning; fail-loud truncation notice; proactive write-in-parts guidance |
| Runtime | Pinned, reproducible ML dependency stack |
| Permissions | Pre-allowed tool set so auto mode skips slow classifier call |
| Honesty | Silent failures → explicit signals; limitations documented not papered over |

## Takeaway

Every fix exists because a layer failed *silently* and looked like a different layer's fault. Making a cloud-grade agent usable on a local LLM is: (a) make every failure loud and diagnosable, (b) right-size budgets to the hardware, (c) pin a fragile runtime, (d) be honest about the model-capability ceiling. That is the actual cost of "local Claude Code".
