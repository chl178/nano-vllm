# AGENTS.md

## What this repo is

nano-vllm is a minimal (~1,200 line) vLLM-style inference engine for offline batch LLM serving. Public API mirrors vLLM: `LLM(path, **kwargs).generate(prompts, sampling_params)`. Only the Qwen3 architecture is implemented. The project's core value is readability/compactness — keep new code terse and avoid adding abstraction layers.

**Keep this file current**: at the end of every task, consider whether anything here needs updating — tools installed, ports/URLs changed, conventions established or broken. Update it in the same session; stale instructions here mislead every future session.

## Running / verifying

- There is **no test suite, linter, typechecker, or CI**. Verification is running the engine end-to-end.
- `pip install -e .` — deps: `torch>=2.4`, `triton>=3.0`, `transformers>=4.51`, `flash-attn`, `xxhash`; Python `>=3.10,<3.13`.
- `python example.py` / `python bench.py` — both hardcode the model path `~/huggingface/Qwen3-0.6B/`; weights must be downloaded separately (see README).
- **CUDA-only**: requires an NVIDIA GPU, flash-attn, triton, and NCCL. On this Windows machine flash-attn/triton are generally unavailable, so end-to-end runs usually aren't possible — rely on careful static verification for edits here.
- Note: `tqdm`, `numpy`, `safetensors` are imported but not declared in `pyproject.toml` (they arrive transitively via transformers).

## Architecture

Request flow: `LLM` (thin alias of `LLMEngine`) → `Scheduler` picks prefill/decode batches → `ModelRunner` runs the forward pass → `Scheduler.postprocess` appends tokens.

- `nanovllm/engine/llm_engine.py` — orchestration loop, tokenizer, tqdm progress, spawns TP worker processes.
- `nanovllm/engine/scheduler.py` — prefill scheduling (chunked prefill only for the first waiting seq), decode scheduling, preemption back to waiting queue.
- `nanovllm/engine/block_manager.py` — paged KV cache with prefix caching via chained xxhash block hashes. `kvcache_block_size` must be a multiple of 256.
- `nanovllm/engine/model_runner.py` — model init/load, KV-cache allocation sized from free GPU memory after a warmup forward, CUDA-graph capture for decode. **Hardcodes `Qwen3ForCausalLM`** and NCCL init at `tcp://localhost:2333`.
- `nanovllm/models/qwen3.py` — model definition. `packed_modules_mapping` maps HF checkpoint names (`q_proj`/`k_proj`/`v_proj`/`gate_proj`/`up_proj`) onto fused params (`qkv_proj`, `gate_up_proj`).
- `nanovllm/layers/` — attention (flash-attn `varlen` for prefill, `with_kvcache` for decode, Triton `store_kvcache` kernel), TP-parallel linear/embedding layers, RMSNorm, RoPE, sampler.
- `nanovllm/utils/context.py` — global attention `Context` (see gotchas).
- `nanovllm/utils/loader.py` — safetensors weight loading via per-param `weight_loader` attributes.

## Key patterns and gotchas

- **Global attention context**: `ModelRunner` calls `set_context(...)` with prefill/decode metadata before each forward; `Attention`/`ParallelLMHead` read it via `get_context()`, and `reset_context()` runs after. Any forward-pass change must keep this bracket intact.
- **Weight loading convention**: linear/embedding params get a `.weight_loader` attribute attached at construction; `load_model` calls it (with a shard id for fused/merged params). New layers must follow this or weights silently won't load.
- **Device/dtype defaults**: the model is constructed under `torch.set_default_device("cuda")` + `hf_config.dtype`, then defaults are restored. Don't create tensors at module import time.
- **Tensor parallelism**: rank 0 is the in-process `ModelRunner`; ranks ≥1 are spawned processes driven by pickled `(method_name, args)` calls over SharedMemory named `nanovllm` plus multiprocessing Events. `Sequence` implements `__getstate__`/`__setstate__` to stay small for pickling — new fields must be added to that state tuple.
- **CUDA graphs**: decode batches ≤512 replay captured graphs at batch sizes `[1,2,4,8] + range(16, max_bs+1, 16)`; prefill, decode >512, and `enforce_eager=True` run eagerly. Changes to decode-path inputs must update `graph_vars` copying in `run_model`.
- **Prefill vs decode logits**: `ParallelLMHead` gathers logits only at each sequence's last position during prefill (reads `cu_seqlens_q` from the context).
- **Sampling**: greedy is intentionally disallowed (`SamplingParams` asserts `temperature > 1e-10`); sampler uses exponential-noise argmax and is `torch.compile`d, as is RoPE forward.
- Adding a new model architecture = new file in `nanovllm/models/` plus wiring the import in `model_runner.py`; nothing else is model-generic.

## Learning canvas (`notes/`)

The user is studying this codebase; concept diagrams are kept as `.excalidraw` files in `notes/` via the globally installed [mcp_excalidraw](https://github.com/yctimlin/mcp_excalidraw) toolkit.

- Canvas UI: **http://127.0.0.1:3333** — port 3333 because Windows WinNAT reserves 2964–3063, so the default 3000 fails with `EACCES`.
- An `excalidraw` MCP server is configured in `~/.zcode/cli/config.json`; prefer its tools when available. CLI fallback (canvas auto-starts on first use): `EXPRESS_SERVER_URL=http://127.0.0.1:3333 mcp-excalidraw-server <cmd>`.
- The canvas is **in-memory only** (v2.1.2 has no restart recovery) — `notes/*.excalidraw` files are the source of truth. After drawing: `export --out notes/<name>.excalidraw`; to continue an old diagram: `import` it first.
- When explaining a concept (scheduler, block manager, attention paths, TP…), offer to draw it on the canvas and save it under `notes/`; verify layout with `screenshot` before exporting. See `notes/CANVAS.md` for details.
- **Keep notes uploaded**: learning notes live on the `learning-notes` branch of the user's fork (remote `fork` = `chl178/nano-vllm`; `origin` is the read-only upstream). After drawing a new diagram or changing anything in `notes/`, commit to `learning-notes` and push to `fork` in the same session — never commit notes to `main`.

## Style

Compact, type-hinted Python with `dataclass(slots=True)` for config-ish classes, few comments, no dead code. Match the existing minimalism.
