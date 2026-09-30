# Independent Bench: MLX-Serve 26.9.7-dev · GLM-5.3-Flash Q4_K GGUF (ds4 engine) · Mac Studio M5 Ultra

Independent llmprobe run of [mlx-serve](https://github.com/ddalcu/mlx-serve) built from `main` on 2026-09-30, serving [`antirez/glm-5.3-flash-gguf`](https://huggingface.co/antirez/glm-5.3-flash-gguf) `GLM-5.3-Flash-Q4_K.gguf` (320B-A18B) through mlx-serve's bundled **ds4** GGUF engine on a single Mac Studio M5 Ultra 256 GB. Context sweep from 2k to 256k, with a 4-stream concurrency burst at every rung.

Same harness, rungs and machine as the companion [Qwen 3.8 Flash-Next run](https://github.com/drowzeys/keys-Independent-Bench-MLX-Serve-26.9.7-dev-Qwen3.8-Flash-Next-4bit-8bit-mixed), so the two are directly comparable.

**Full credit to [ddalcu](https://github.com/ddalcu)** for mlx-serve and to **[antirez](https://github.com/antirez)** for the ds4 engine and the GLM-5.3-Flash GGUF quants. This repo only publishes an independent measurement of their work. Dev build, not a release.

## Setup

| | |
|---|---|
| Machine | Mac Studio M5 Ultra (Mac17,15), 256 GB, macOS 27.0.1 |
| Engine | mlx-serve **26.9.7-dev**, commit [`933566c`](https://github.com/ddalcu/mlx-serve/commit/933566c), ReleaseFast · ds4 `9139e2a` (GGUF backend) · ggml 0.23.0 · llama.cpp b10809 |
| Stack | MLX 0.32.3 · mlx-c `56b2d39` · NAX on (M5 neural accelerators) · Metal 4 tensor API on · Xcode 27.0 / Metal Toolchain 27A266a |
| Model | GLM-5.3-Flash **Q4_K** GGUF, 177.8 GiB (text only; vision encoder GGUF not loaded) |
| Server flags | `--serve --host 127.0.0.1 --port 11234 --max-concurrent 4 --ctx-size 262144` |
| Effective | ctx 262,144 (manual) · **concurrency 1** (ds4 overrides `--max-concurrent`) · compact DSA KV · embedded MTP, 2 draft tokens |
| Harness | [llmprobe](https://www.npmjs.com/package/llmprobe) 0.6.13, `--bench-only` |
| Sampling | greedy (t=0), thinking on, reasoning medium |
| Runs | warmup + median of 3 per scenario; decode forced to 192 tokens |
| Rungs | `--rungs 2k,4k,8k,16k,32k,64k,128k,250k --concurrency 4` (custom, not comparable to default llmprobe runs) |
| Quiet machine | Ollama quit, no other model servers, no builds or downloads during the run |

## Headline

| Metric | Result |
|---|---|
| Decode | **35.8 tok/s** (35.6–37.5) |
| Prefill (9,368-token prompt) | **438.8 tok/s** |
| TTFT (short prompt) | **427 ms** |
| Speculative decode | **1.57×** (predictable 41.0 · novel 26.2 tok/s · 1.96 tok/step) |
| Prefix cache | not detected (3.0 s cold → 3.0 s warm, although usage reports 1,411/1,411 cached) |
| Concurrency ×4 (short prompts) | serialized: 26.7 tok/s aggregate vs 28.7 alone (0.23 efficiency) |
| Sustained | 35.8 → 35.9 tok/s over 142m 21s (+0.3%, steady) |

## Context sweep

| Context | Decode tok/s | Prefill tok/s | TTFT | tok/step | ×4 total tok/s | ×4 each tok/s | ×4 worst TTFT |
|---|---|---|---|---|---|---|---|
| 2k | 26.4 | 471 | 4.5 s | 1.68 | 16.9 | 27.5 | 38.7 s |
| 4k | 30.2 | 453 | 9.0 s | 1.81 | 12.3 | 29.0 | 56.1 s |
| 8k | 27.6 | 433 | 19.1 s | 1.75 | 7.3 | 26.9 | 98.0 s |
| 16k | 26.9 | 426 | 38.2 s | 1.73 | 4.2 | 26.4 | 174.3 s |
| 33k | 27.1 | 418 | 78.3 s | 1.76 | 2.3 | 27.2 | 334.3 s |
| 66k | 24.4 | 409 | 160.5 s | 1.71 | 1.1 | 24.8 | 665.7 s |
| 131k | 23.3 | 394 | 332.8 s | 1.76 | 0.6 | 24.0 | 1,355.1 s |
| **256k** | **19.0** | **370** | **692.3 s** | 1.71 | 0.3 | 20.0 | 2,797.6 s |

## Side by side with Qwen 3.8 Flash-Next (same build, machine, harness)

| | GLM-5.3-Flash Q4_K (ds4) | Qwen 3.8 Flash-Next mixed 4/8 (MLX) |
|---|---|---|
| Size / active | 320B-A18B · 177.8 GiB | 125B-A6B · 100 GiB |
| Decode | 35.8 tok/s | 154.5 tok/s |
| Prefill | 439 tok/s | 4,183 tok/s |
| Decode @ 256k | 19.0 tok/s | 114.4 tok/s |
| TTFT @ 256k | 11.5 min | 62 s |
| tok/step (spec) | 1.96 | 7.84 |
| Concurrency | serialized (1 slot) | batched (4 slots) |

## Observations

- Decode stays at 26–30 tok/s from 2k through 33k with thinking on, and falls to 19 tok/s at 256k (about −30%).
- Prefill is roughly flat at 370–470 tok/s across the whole ladder, so TTFT scales linearly and a full 256k prompt takes about 11.5 minutes.
- **MTP is active even though the server says it isn't.** The startup log prints `has_mtp=false`, and `/v1/models` reports `mtp_loaded:false`, because `ds4_engine_has_mtp()` only checks for a sidecar MTP file. GLM's MTP head is embedded in the GGUF and is tracked separately (`glm_mtp`). The scheduler engages it through `mtpDraftTokens()`, which returns 2, and llmprobe measured 1.68–1.96 tok/step. [`patches/ds4-embedded-mtp-reporting.patch`](patches/ds4-embedded-mtp-reporting.patch) is a 6-line fix on the Zig side (ds4 itself untouched). With it applied, the log shows `has_mtp=true, draft_tokens=2` and `/v1/models` shows `mtp_loaded:true`. It only corrects the reporting: speed is unchanged (41.6 tok/s on a predictable prompt, versus 41.0 in the bench).
- ds4 caps GLM speculation at **2 draft tokens** (the GLM spec cycle is written for `limit <= 2`). That caps it at 3 tok/step. The measured ~1.7 tok/step means roughly 35% of drafts are accepted.
- **Concurrency:** ds4 forces one slot, so every ×4 burst is marked `serialized` (0.25 efficiency). Aggregate throughput under 4 concurrent requests is about a quarter of a single stream, because the requests wait in a queue.
- **Auto-context misestimates this GGUF.** Without `--ctx-size`, mlx-serve's memory sizer picked **13,312 tokens** ("85% of the 16,730-token memory ceiling"). It appears to bill a dense KV cache, but ds4's compact DSA cache is only ~0.37 GiB per 32k rows. The explicit `--ctx-size 262144` loaded cleanly and ran the full ladder with RSS steady at ~178 GB.

## Caveats

- Every context rung reported `answer never used RETRY_BUDGET_MS — context may be unread`. This was also true of the Qwen run. With thinking on and decode capped at 192 tokens, the answer never gets past reasoning, so this run does not verify long-context recall. Speed numbers are unaffected.
- In a single pre-bench smoke request at temperature 1.0 (not part of the bench), the reasoning text contained a stray `<|user|>` turn marker. The chat template or stop tokens may need a look for sampled use. All bench traffic was greedy.

## Files

- [`raw/llmprobe-output.txt`](raw/llmprobe-output.txt): full console output
- [`raw/llmprobe-report.json`](raw/llmprobe-report.json): llmprobe run record
- [`raw/llmprobe-report-card.html`](raw/llmprobe-report-card.html): llmprobe report card (download and open locally)
- [`raw/server-startup.log`](raw/server-startup.log): mlx-serve startup log for this run (engine, context, concurrency, MTP lines)
- [`patches/ds4-embedded-mtp-reporting.patch`](patches/ds4-embedded-mtp-reporting.patch): MTP status reporting fix against `933566c`

## Recipe: re-create for validation

Pinned inputs:

| Input | Pin |
|---|---|
| mlx-serve | `933566c4107f34d459652cd24ca0fd0f4bbf68c6` (main, 2026-09-30) |
| Model | `antirez/glm-5.3-flash-gguf` @ `b2fa29d7a6b410db11221c904973967b80b760f5`, file `GLM-5.3-Flash-Q4_K.gguf` (190,875,526,464 bytes) |
| Zig | `0.17.0-dev.2248+3f6a02acd` (staged by `scripts/fetch-zig.sh`) |
| Toolchain | Xcode 27.0 (27A266a) + Metal Toolchain, cmake 4.4, libwebp 1.6.0 |
| Harness | llmprobe 0.6.13 on Node 24 |

### 1. Build mlx-serve

Same as the [Qwen Flash-Next recipe](https://github.com/drowzeys/keys-Independent-Bench-MLX-Serve-26.9.7-dev-Qwen3.8-Flash-Next-4bit-8bit-mixed#recipe-re-create-for-validation), steps 1–2:

```bash
git clone --recurse-submodules https://github.com/ddalcu/mlx-serve && cd mlx-serve
git checkout 933566c4107f34d459652cd24ca0fd0f4bbf68c6
git submodule update --init --recursive
./scripts/fetch-zig.sh && export PATH="$PWD/.zig-toolchain:$PATH"
./scripts/fetch-llama.sh && ./scripts/build-mlx.sh
zig build -Doptimize=ReleaseFast
./zig-out/bin/mlx-serve --version      # expect 26.9.7-dev · ds4 9139e2a · "nax on" on M5
```

### 2. Fetch the model

```bash
hf download antirez/glm-5.3-flash-gguf GLM-5.3-Flash-Q4_K.gguf \
  --revision b2fa29d7a6b410db11221c904973967b80b760f5 \
  --local-dir ~/.mlx-serve/models/antirez/glm-5.3-flash-gguf
```

### 3. Serve

Quiet machine: no other model servers loaded (this needs ~178 GB resident), no builds or large downloads running.

```bash
./zig-out/bin/mlx-serve \
  --model ~/.mlx-serve/models/antirez/glm-5.3-flash-gguf/GLM-5.3-Flash-Q4_K.gguf \
  --serve --host 127.0.0.1 --port 11234 --max-concurrent 4 --ctx-size 262144
# log should show: "[gguf] engine: ds4 (arch=glm5-next ...)", "Context size: 262144 tokens (manual)"
# and "Concurrency: --max-concurrent=1" (ds4 override). Tensor-page warmup takes ~1 min.
```

`--ctx-size` is required. Without it, auto-context settles at ~13k and the upper rungs fail.

### 4. Bench

```bash
npx -y llmprobe@0.6.13 127.0.0.1:11234 \
  -m GLM-5.3-Flash-Q4_K --bench-only \
  --rungs 2k,4k,8k,16k,32k,64k,128k,250k --concurrency 4 \
  --label "GLM-5.3-Flash Q4_K GGUF (ds4) | mlx-serve 26.9.7-dev 933566c | <your machine>" --no-color \
  | tee llmprobe-output.txt
```

This took about **2 h 23 min** on an M5 Ultra and sent ~3.15M prompt tokens across 76 requests. Most of that time is the serialized ×4 bursts at 128k and 256k. On an M5 Ultra, expect decode near 36 tok/s and prefill around 370–470 tok/s. Please open an issue with your numbers and machine.

## Related

- [Qwen 3.8 Flash-Next mixed 4/8 on the same mlx-serve build](https://github.com/drowzeys/keys-Independent-Bench-MLX-Serve-26.9.7-dev-Qwen3.8-Flash-Next-4bit-8bit-mixed)

## Credits

- [**ddalcu**](https://github.com/ddalcu): [mlx-serve](https://github.com/ddalcu/mlx-serve), the MLX-Serve app, the GGUF/ds4 integration and M5 Ultra support in this build.
- [**antirez**](https://github.com/antirez): the ds4 inference engine and the [GLM-5.3-Flash GGUF quants](https://huggingface.co/antirez/glm-5.3-flash-gguf).
- **Z.ai**: GLM-5.3-Flash.
- [llmprobe](https://www.npmjs.com/package/llmprobe): benchmark harness.
- Apple MLX / ggml: the underlying frameworks.
