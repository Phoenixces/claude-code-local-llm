# claude-code-local-llm

Run Claude Code with local LLMs on Apple Silicon — real tool execution, real agentic loops, fully offline.

Most tutorials tell you to point Claude Code at Ollama. None work. The model generates text that *looks like* tool calls but nothing executes. This project uses [vllm-mlx](https://github.com/waybarrios/vllm-mlx) — the only backend that speaks Claude Code's native Anthropic Messages API with real `tool_use` content blocks. When the model writes code, it lands on disk. The agentic loop actually works.

No API key. No cloud. No subscription.

## Requirements

- Apple Silicon Mac (M1/M2/M3/M4/M5)
- 16GB+ unified memory (24GB+ recommended)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- [Homebrew](https://brew.sh)

## Quick start

```bash
git clone https://github.com/Phoenixces/claude-code-local-llm.git
cd claude-code-local-llm
./bash/install.sh
cclocal
```

First run downloads the default model (~5GB, one-time). Then starts vllm-mlx and launches Claude Code.

### Verify it works

In Claude Code, type:

```
create a file called /tmp/test_tools.txt with "hello world"
```

**Working**: Claude Code calls the Write tool, creates the file, confirms.  
**Broken**: Claude Code generates text saying it created the file, but nothing exists on disk.

## Models

| Flag | Model | Size | RAM needed | Notes |
|------|-------|------|-----------|-------|
| *(default)* `--gemma-light` | Gemma-4-E4B | ~5GB | 16GB+ | Clean tool calling, verified end-to-end |
| `--gemma` | Gemma-4-26B-A4B MoE | ~16GB | 24GB+ | Google MoE, 3.8B active params |
| `--review` | GLM-4.7-Flash | ~17GB | 24GB+ | Stronger reasoning |
| `--coder` | Qwen3-Coder-30B-A3B | ~18GB | 24GB+ | Heavier code model |
| `--qwen3` | Qwen3.5-9B | ~5GB | 16GB+ | General reasoning — leaks plain-text thinking [1] |
| `--coder7b` | Qwen2.5-Coder-7B | ~5GB | 16GB+ | Code analysis — tool calls unreliable [2] |
| `--light` | *(alias)* | | | Back-compat alias for `--gemma-light` |
| `--model ID` | Any MLX model | varies | varies | Custom HuggingFace model ID (not tested) |

[1] Qwen3.5 ignores `enable_thinking=false` at the template level and emits plain-text "Thinking Process:" preamble outside `<think>` tags. Known upstream issue; see [vllm-project/vllm#35574](https://github.com/vllm-project/vllm/issues/35574) and [QwenLM/Qwen3#1625](https://github.com/QwenLM/Qwen3/issues/1625).

[2] Qwen2.5-Coder-7B hallucinates an XML tool-call format (`<Write path="..." content="..."/>`) that no parser handles. Good for non-agentic code analysis, not Claude Code's tool loop.

```bash
cclocal                # Interactive menu: pick model, see what's cached, manage cache
cclocal --gemma-light  # Direct launch, Gemma-4-E4B (default)
cclocal --gemma        # Direct launch, Gemma-4-26B MoE
cclocal --review       # Direct launch, GLM-4.7-Flash
cclocal --coder        # Direct launch, Qwen3-Coder-30B-A3B
cclocal --list         # List cached models on disk
cclocal --rm           # Manage/delete cached models (interactive)
cclocal --server       # Start server only, connect Claude Code separately
cclocal -h             # Show all options

# Operational flags (combine with any model flag)
cclocal --gemma --out-tokens 16384   # Bigger output budget for large file writes (default 8192)
cclocal --gemma --safe               # Force the memory-safeguard menu
cclocal --gemma --no-mem-check       # Skip GPU-headroom preflight

# Remote backend
cclocal --dgx-active                 # DGX Spark preset (MoE, faster)
cclocal --dgx-idle                   # DGX Spark preset (dense, steadier)
cclocal --remote http://host:8000    # Any remote vLLM endpoint (model auto-detected)
```

`cclocal` with no arguments opens an interactive menu showing all supported models, cache status, and a cache management screen.

### What `cclocal` handles automatically

Full root-cause writeups in [Why this is hard](#why-this-is-hard-and-how-we-solved-it) (#16–#18) and the [field report](field-report.md).

- **Memory preflight.** Estimates model footprint vs the GPU budget (`iogpu.wired_limit_mb` cap, or ~75% of RAM). If headroom is tight, offers to shrink server context and/or raise the GPU wired limit via `sudo` (auto-reverted on exit, never persisted). `--safe` forces the menu; `--no-mem-check` skips it.
- **Output budget.** `CLAUDE_CODE_MAX_OUTPUT_TOKENS` defaults to 8192; override with `--out-tokens N`.
- **No classifier stall.** 8 built-in tools pre-allowed (`--allowedTools`) — auto mode skips the slow per-action safety-classifier call.
- **Write-in-parts hint.** System-prompt line tells the model to build large files incrementally.
- **Fail-loud truncation notice.** `--tool-call-truncation-notice`: a truncated tool call returns an explicit "write it in smaller parts" message instead of silent text.
- **Diagnosable logs.** `server.log` rotated to `server.log.1` on each launch.
- **Pinned ML runtime.** `bash/install.sh` pins `mlx==0.31.1` / `mlx-lm==0.31.1` (newer versions crash generation from a worker thread).

### Server-only mode

```bash
cclocal --server
```

Then connect Claude Code from any terminal:

```bash
ANTHROPIC_BASE_URL=http://127.0.0.1:8000 \
ANTHROPIC_API_KEY=not-needed \
ANTHROPIC_MODEL=mlx-community/gemma-4-e4b-it-4bit \
claude --strict-mcp-config --mcp-config /path/to/claude-code-local/mcp-local.json \
  --tools "Bash,Read,Edit,Write,Glob,Grep,WebSearch,WebFetch"
```

> Replace `/path/to/claude-code-local` with the repo path. `cclocal --server` prints the full command.

### Remote backend (DGX Spark or any vLLM box)

Point Claude Code at a remote box running plain vLLM. Recent vLLM ships a native Anthropic Messages API (`/v1/messages` with real `tool_use` blocks, Anthropic SSE streaming, `count_tokens`) — the same wiring works. `bash/run.sh` skips the local server lifecycle entirely.

```bash
cclocal --dgx-active                      # preset: MoE box (faster)
cclocal --dgx-idle                        # preset: dense box (steadier reasoning)
cclocal --remote http://host:8000         # any remote vLLM endpoint
cclocal --remote http://host:8000 --remote-model Qwen/Qwen3.6-35B-A3B
```

- **Auto-detect.** Without `--remote-model`, the launcher reads the model id from `/v1/models`.
- **Presets.** Edit `DGX_ACTIVE` / `DGX_IDLE` addresses at the top of `bash/run.sh` to match your boxes.
- **Nothing local runs.** vllm-mlx not required in remote mode.
- **Reachability.** Unreachable endpoint gives a clear error (with Tailscale hint) instead of a hang.

> **Reasoning models emit `thinking`.** Qwen3-class models return `thinking` blocks; recent vLLM wraps them as structured Anthropic `thinking` content. No request parameter disables thinking on the `/v1/messages` endpoint — disable it **server-side on the remote box** if needed.

---

## Why this is hard (and how we solved it)

Running Claude Code with a local model isn't just "point it at localhost". 15+ problems break the experience. Full field report: [`field-report.md`](field-report.md).

### 1. Ollama can't produce real tool calls

**Problem**: Ollama's Anthropic adapter generates text that *looks like* tool calls but never emits real `tool_use` blocks. Tested with qwen3.5:9b, qwen3.5:35b-a3b, glm-4.7-flash — all produce fake tool calls.

**Solution**: Use vllm-mlx — native Anthropic Messages API with real `tool_use` / `tool_result` blocks.

### 2. `end_turn` vs `stop` (the loop killer)

**Problem**: Claude Code needs `stop_reason: "end_turn"`. Backends returning `"stop"` (OpenAI convention) cause Claude Code to stop after one response.

**Solution**: vllm-mlx's native `/v1/messages` returns correct Anthropic stop reasons.

### 3. Reasoning/thinking tokens (garbage output)

**Problem**: Qwen 3.x and Gemma 4 emit thinking tokens that Claude Code doesn't expect — causes garbage output and misparses tool calls.

**Solution**: `bash/run.sh` sets `VLLM_MLX_ENABLE_THINKING=false`, suppressing thinking tokens at the template level.

### 4. KV cache invalidation (90% slowdown)

**Problem**: Claude Code's attribution header changes every request, invalidating the KV cache. Follow-up responses go from 2s to 30s+.

**Solution**: `CLAUDE_CODE_ATTRIBUTION_HEADER=0` (set by `bash/run.sh`).

### 5. Background Haiku model calls (crash)

**Problem**: Claude Code calls `claude-haiku-4-5-20251001` for background tasks. Local server returns 404 → hang.

**Solution**: All model tier env vars (`ANTHROPIC_DEFAULT_OPUS_MODEL`, `ANTHROPIC_DEFAULT_SONNET_MODEL`, `ANTHROPIC_DEFAULT_HAIKU_MODEL`, `CLAUDE_CODE_SUBAGENT_MODEL`) set to the same local model.

### 6. Token counting endpoint (silent failure)

**Problem**: Claude Code calls `/v1/messages/count_tokens`. Most local servers don't implement it.

**Solution**: vllm-mlx supports it. `DISABLE_PROMPT_CACHING=1` reduces dependence on it.

### 7. Concurrent requests OOM

**Problem**: Claude Code fires concurrent requests (main + background + subagents). Two concurrent 24K+ token prompts exceed the Metal GPU buffer on 24GB and crash the server.

**Solution**: Run in single-request mode (no `--continuous-batching`). `--kv-cache-quantization` halves KV cache memory, adding headroom before OOM.

### 8. Streaming format mismatches

**Problem**: Claude Code expects Anthropic SSE events. OpenAI-format streaming shows only the last token.

**Solution**: vllm-mlx uses native Anthropic SSE streaming.

### 9. Tool flooding (259 tools overwhelm local models)

**Problem**: Claude Code sends ALL tool definitions every request. With plugins enabled, 200+ tools crammed into the system prompt. Even 30B models choke.

**Solution**:
```
--strict-mcp-config --mcp-config mcp-local.json    # strips all plugin/MCP tools
--tools "Bash,Read,Edit,Write,Glob,Grep,WebSearch,WebFetch"  # 8 built-in tools only
```

### 10. Real API key leaking to local server

**Problem**: Real `ANTHROPIC_API_KEY` (`sk-ant-...`) may be sent to the local server.

**Solution**: `env -u ANTHROPIC_API_KEY -u ANTHROPIC_AUTH_TOKEN` in `bash/run.sh` unsets real keys before setting the dummy.

### 11. Autoupdater and telemetry

**Problem**: Startup update checks and telemetry can hang local-only sessions.

**Solution**:
```
DISABLE_AUTOUPDATER=1
DISABLE_TELEMETRY=1
DISABLE_ERROR_REPORTING=1
```

### 12. Memory pressure on 24GB

| Model | Size | Free RAM | Status |
|-------|------|----------|--------|
| **Gemma-4-E4B** | **~5GB** | **~19GB** | **Default — verified tool loop** |
| Qwen3.5-9B | ~5GB | ~19GB | Works but leaks plain-text thinking |
| Qwen2.5-Coder-7B | ~5GB | ~19GB | Code analysis only — tool calls unreliable |
| Gemma-4-26B-A4B MoE | ~16GB | ~8GB | Fast inference, tight on 24GB |
| GLM-4.7-Flash | ~16.9GB | ~7GB | Works single-request only |

### 13. vllm-mlx missing `return` statement (historical)

**Problem**: Earlier versions crashed on startup: `TypeError: cannot unpack non-iterable NoneType object`. Missing `return` in `vllm_mlx/utils/tokenizer.py` `load_model_with_fallback()`.

**Solution**: Fixed upstream. `bash/install.sh` installs from [vitorallo/vllm-mlx@claude-code-local-patches](https://github.com/vitorallo/vllm-mlx/tree/claude-code-local-patches) which includes the fix plus Gemma 4 channel-token cleanup patches.

### 14. Health endpoint mismatch

**Problem**: Scripts polling readiness grep for `"ok"` but vllm-mlx returns `"status":"healthy"`.

**Solution**: `bash/run.sh` greps for `"healthy"`.

### 15. Model name `default` not recognized

**Problem**: `ANTHROPIC_MODEL=default` causes 404 — vllm-mlx requires the full HuggingFace model ID.

**Solution**: `bash/run.sh` passes the full model ID (e.g., `mlx-community/gemma-4-e4b-it-4bit`).

### 16. First-run model download looks like a hang

**Problem**: First use downloads 5–18GB before the server comes up. The old fixed-duration poll with no output looked like a frozen launch.

**Solution**: `bash/run.sh` watches the model's HuggingFace cache directory and prints live progress:

```
⬇ Downloading model 8.4GB (42.3 MB/s)
⏳ Model cached (16.0GB) — loading into memory... 12s
```

The timeout (`STALL_LIMIT`, 240s) only aborts if there's **no** download progress — a slow-but-progressing download never times out. Partial downloads are preserved and resume on restart.

### 17. OOM crash under agentic load — memory preflight

**Problem**: On 24GB, large models survive short prompts but the KV cache grows each turn. Once context passes ~24K tokens, Metal throws:

```
libc++abi: terminating due to uncaught exception of type std::runtime_error:
[METAL] Command buffer execution failed: Insufficient Memory
(kIOGPUCommandBufferCallbackErrorOutOfMemory)
```

This kills the entire vllm-mlx process, leaving Claude Code retrying a dead backend.

**Solution**: `bash/run.sh` runs `memory_preflight` before starting the server. The binding constraint is **not** total RAM — macOS only makes ~75% GPU-addressable (`iogpu.wired_limit_mb` cap). The preflight estimates model footprint against the effective GPU budget and when headroom is tight shows:

```
⚠ Tight memory  ~15GB model, GPU budget ~18GB → ~3GB for KV cache.
Safeguards:
  1) Shrink context   --max-tokens 32768 → 16384   (recommended)
  2) Raise GPU limit  iogpu.wired_limit_mb 0 → 21504  (sudo, until reboot)
  3) Both
  c) Continue as-is (risky)     q) Quit
Choose [1/2/3/c/q] (Enter = 1):
```

- **Option 1** lowers `--max-tokens` (biggest KV-cache saver).
- **Option 2** raises the Metal wired-memory limit via `sudo sysctl` — per-session only, auto-reverted on exit, resets on reboot.
- 5GB models with ample headroom skip the prompt entirely. `--safe` forces the menu; `--no-mem-check` skips it.

### 18. Write/Edit tool call silently does nothing

**Symptom**: The model "calls" Write — Claude Code shows the invocation — then silence. No file, no error, HTTP 200. Short-arg tools (`Bash ls`) work; large `Write`/`Edit` calls don't.

**Cause** — three compounding factors:

1. **Output-token truncation.** A `Write` serializes the entire file body as output tokens. With the old 4096 cap, generation is cut mid-`content`, JSON never closes, no `tool_use` block can be built — silent HTTP 200.
2. **Fork channel-filter destroys tool calls.** The Gemma channel cleaner, on an unclosed `<|channel>thought`, deletes everything to end-of-text — including any following tool call. Also matches channel markers *inside* a file body, corrupting otherwise-valid calls.
3. **Invalid JSON from weak model.** A 4-bit model may fail to `\"`-escape quotes in long content; `json.loads` fails; tool input is dropped.

**Fixed in the fork** (`vitorallo/vllm-mlx`, branch `fix/gemma4-toolcall-safe-and-faildloud`):
- **Pinned MLX stack.** `pyproject.toml` pins `mlx==0.31.1` / `mlx-lm==0.31.1`, caps `mlx-vlm<0.5.0`. `mlx 0.31.2` breaks GPU streams in worker threads.
- **Tool-call-span-safe channel cleaning.** Channel stripping applied only *outside* `<|tool_call>…<tool_call|>` spans. A truncated thought no longer deletes the following tool call; channel markers inside a file body no longer corrupt the call. Covered by `tests/test_gemma4_toolcall_safety.py`.
- **Fail-loud `--tool-call-truncation-notice`.** When a tool call is truncated by the token cap, server returns an explicit "write the file in smaller parts" message instead of silent HTTP 200.

**Mitigations in `bash/run.sh`**:
- `CC_OUTPUT_TOKENS` default raised 4096 → 8192; per-run override `--out-tokens N`.
- `--append-system-prompt` tells the model to build files >~150 lines in sections.
- `server.log` rotated to `server.log.1` instead of truncated.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| vllm-mlx crashes on startup (TypeError: NoneType) | Using unpatched upstream | `./bash/install.sh` installs from our fork |
| Model generates text about tools but nothing executes | Using Ollama | Switch to vllm-mlx — Ollama can't produce real tool_use blocks |
| Metal GPU OOM crash (`kIOGPUCommandBufferCallbackErrorOutOfMemory`) | Large model + growing agentic context exceeds RAM | Take the `memory_preflight` prompt, or use a 5GB model — see #17 |
| First run hangs at "Waiting for server..." | Multi-GB model downloading | Not hung — live progress now shows; partial downloads resume — see #16 |
| Write/Edit shows then silently does nothing | Large tool-call output truncated | `--out-tokens 16384`, inspect `server.log.1` — see #18 |
| Claude Code asks about "detected custom API key" | Real API key leaking | Use `cclocal` which unsets real keys |
| "Model does not exist" (404) | Wrong model name | Must use full HuggingFace ID |
| Slow responses (30-60s) | Normal for local inference | Context grows each turn — 24K+ tokens at ~8 tok/s |

---

## Configuration reference

### Environment variables (set by bash/run.sh per-session)

| Variable | Value | Purpose |
|----------|-------|---------|
| `ANTHROPIC_BASE_URL` | `http://127.0.0.1:8000` or remote URL | Point Claude Code at the server |
| `ANTHROPIC_API_KEY` | `not-needed` | Dummy key (real key explicitly unset) |
| `ANTHROPIC_MODEL` | Full HuggingFace ID | Model identifier |
| `ANTHROPIC_DEFAULT_*_MODEL` | Same as above | Route all tiers (Opus/Sonnet/Haiku) locally |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Same as above | Route subagent calls locally |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | `8192` default | Output cap; must fit a whole Write/Edit body (see #18) |
| `CLAUDE_CODE_ATTRIBUTION_HEADER` | `0` | Prevents KV cache invalidation |
| `DISABLE_PROMPT_CACHING` | `1` | Local server doesn't support Anthropic caching |
| `DISABLE_AUTOUPDATER` | `1` | No update checks |
| `DISABLE_TELEMETRY` | `1` | No telemetry |
| `DISABLE_ERROR_REPORTING` | `1` | No error reporting |
| `DISABLE_NON_ESSENTIAL_MODEL_CALLS` | `1` | Reduce background model calls |

### vllm-mlx server flags (set by bash/run.sh)

| Flag | Purpose |
|------|---------|
| `VLLM_MLX_ENABLE_THINKING=false` | Disable thinking/reasoning tokens |
| `--kv-cache-quantization` | 8-bit KV cache — halves cache memory |
| `--cache-memory-percent 0.35` | 35% of RAM for cache (~8.4GB on 24GB) |
| `--prefill-step-size 4096` | Faster time-to-first-token on large prompts |
| `--stream-interval 4` | Batch 4 tokens before streaming |
| `--timeout 600` | 10 min timeout (default 300s caused disconnects) |
| `--max-tokens` | Server context window: 32768, or 16384 if preflight shrinks it |
| `--enable-auto-tool-choice --tool-call-parser auto` | Parse model output into structured tool_use blocks |
| `--tool-call-truncation-notice` | On truncated tool call, return explicit error instead of silent text |

### Memory preflight flags

| Flag / env | Purpose |
|------------|---------|
| `--safe` / `CCLOCAL_FORCE_MEMCHECK=1` | Always show memory-safeguard menu |
| `--no-mem-check` / `CCLOCAL_NO_MEMCHECK=1` | Skip GPU-headroom preflight |
| `--out-tokens N` | Max output tokens (default 8192; raise to 16384 for large writes) |
| `iogpu.wired_limit_mb` | Raised via `sudo sysctl` by preflight option 2; per-session only, reverted on exit |

### Claude Code flags (set by bash/run.sh)

| Flag | Purpose |
|------|---------|
| `--strict-mcp-config` | Ignore global plugins |
| `--mcp-config mcp-local.json` | Empty config — no plugin tools |
| `--tools "Bash,Read,..."` | 8 essential built-in tools only |
| `--allowedTools "Bash,Read,..."` | Pre-approve 8 tools so auto mode skips the slow safety-classifier call |
| `--append-system-prompt "..."` | Tells model to build large files in incremental calls |

---

## File structure

```
claude-code-local/
  bash/
    run.sh                  # Launcher — starts vllm-mlx + Claude Code
    install.sh              # Setup — creates .venv, installs vllm-mlx, creates cclocal
  mcp-local.json            # Empty MCP config (strips plugins for local sessions)
  field-report.md           # Full field report: every problem, root cause, fix
  .venv/                    # Local Python venv with vllm-mlx (created by install.sh)
  .gitignore
  README.md
```

---

## Links

- [vllm-mlx](https://github.com/waybarrios/vllm-mlx) — Anthropic-compatible MLX inference server
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — Anthropic's CLI for Claude
- [Why Claude Code Fails with Local LLMs](https://explore.n1n.ai/blog/why-claude-code-fails-local-llm-inference-2026-02-19) — Detailed failure analysis
- [Claude Code tool flooding issue](https://github.com/anthropics/claude-code/issues/25857) — 259 tools sent to local models
- [Ollama Anthropic Compatibility](https://docs.ollama.com/api/anthropic-compatibility) — Confirmed broken for tool_use

---

## Citation

```bibtex
@software{vllm_mlx2025,
  author = {Barrios, Wayner},
  title = {vLLM-MLX: Apple Silicon MLX Backend for vLLM},
  year = {2025},
  url = {https://github.com/waybarrios/vllm-mlx},
  note = {Native GPU-accelerated LLM and vision-language model inference on Apple Silicon}
}
```
