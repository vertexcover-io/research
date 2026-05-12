# vLLM CLI parameters for multimodal LLMs

A practical map of every CLI flag that matters when you run `vllm serve` (or
build an offline `LLM` / `EngineArgs`) against a multimodal model — vision,
audio, video, or omni. It groups parameters by the layer they belong to,
explains the multimodal-specific ones in depth, and lists the per-model
overrides for the major families.

Pinned to the `main` branch of `vllm-project/vllm` as of May 2026. Defaults
shift between releases, so always confirm with `vllm serve --help=group` on
the version you actually run.

## 1. The shape of the CLI

vLLM ships a single console script, `vllm`, with these subcommands:

| Command            | What it does                                                                  |
| ------------------ | ----------------------------------------------------------------------------- |
| `vllm serve`       | Starts the OpenAI-compatible HTTP server (or gRPC with `--grpc`).             |
| `vllm chat`        | Client that POSTs to a running server's `/v1/chat/completions`.               |
| `vllm complete`    | Client for `/v1/completions`.                                                 |
| `vllm bench {latency,serve,throughput,sweep}` | Benchmarks (`pip install vllm[bench]`).            |
| `vllm run-batch`   | OpenAI-batch-style JSONL inference.                                           |
| `vllm collect-env` | Environment diagnostics.                                                      |

`vllm serve` is the only command that takes engine/model/multimodal flags.
Internally, those flags are the *exact* same fields as `EngineArgs`, registered
in `vllm/engine/arg_utils.py`, plus the frontend (HTTP) flags from
`vllm/entrypoints/openai/cli_args.py`.

The `--help` output is large; use the grouped help:

```
vllm serve --help=listgroup
vllm serve --help=MultiModalConfig
vllm serve --help=ModelConfig
vllm serve --help=SchedulerConfig
vllm serve --help=FrontendArgs
```

JSON-typed flags accept a JSON object *or* a dotted form (and `+` to append
to lists):

```
--limit-mm-per-prompt '{"image":2,"video":1}'
--limit-mm-per-prompt.image 2 --limit-mm-per-prompt.video 1
--mm-processor-kwargs.min_pixels 784 --mm-processor-kwargs.max_pixels 1003520
```

## 2. Where the multimodal flags live

The multimodal-specific group is `MultiModalConfig` (see
`vllm/config/multimodal.py`). It registers the following flags via
`vllm/engine/arg_utils.py:1161-1230`:

### 2.1 Content limits, embeddings, IO

| Flag | Type | Default | Purpose |
| --- | --- | --- | --- |
| `--limit-mm-per-prompt` | JSON dict (`{image:int}` or `{video:{count,num_frames,width,height}}`) | 999 per modality | Caps media items per request *and* sizes the dummy item used during profiling. |
| `--language-model-only` | bool | `False` | Force every limit to 0 — run as a text-only LM. |
| `--enable-mm-embeds` | bool | `False` | Accept precomputed `image_embeds` / `audio_embeds` inputs. Crashes on bad shapes; trust-only. |
| `--media-io-kwargs` | JSON keyed by modality | `{}` | Per-modality knobs for vLLM's media loader: `frame_recovery`, `rgba_background_color`, client-supplied `fps`/`frames_indices`/`total_num_frames`/`duration`/`do_sample_frames`. |
| `--mm-processor-kwargs` | JSON dict | `None` | Per-model overrides forwarded to the HuggingFace processor. The set of keys depends on the model (see §5). |
| `--interleave-mm-strings` | bool | `False` | Allow text/media chunks to interleave when using `--chat-template-content-format=string`. |
| `--video-pruning-rate` | float in [0,1) | `None` | Efficient Video Sampling — fraction of video tokens to drop per item. |

### 2.2 Processor caching

| Flag | Type | Default | Purpose |
| --- | --- | --- | --- |
| `--mm-processor-cache-gb` | float ≥ 0 | `4` | Per-engine preprocessor cache (GiB). *Total RAM* ≈ `gb × (api_server_count + data_parallel_size)`. Set `0` to disable. |
| `--mm-processor-cache-type` | `lru` \| `shm` | `lru` | `lru` is process-local; `shm` is shared-memory FIFO across processes (better for DP). |
| `--mm-shm-cache-max-object-size-mb` | int ≥ 0 | `128` | Only with `--mm-processor-cache-type shm`. |

### 2.3 Encoder (ViT/Audio-tower) optimisation

| Flag | Type | Default | Purpose |
| --- | --- | --- | --- |
| `--mm-encoder-only` | bool | `False` | Skip the language tower; used in disaggregated encoder workers. |
| `--mm-encoder-tp-mode` | `weights` \| `data` | `weights` | TP strategy for the encoder. `data` replicates weights but splits the *batch* — good when you have lots of small inputs. Falls back silently to `weights` if the encoder doesn't support batch-split. |
| `--mm-encoder-attn-backend` | enum (`FLASH_ATTN`, `TRITON`, ...) | `None` | Encoder attention backend override. `XFORMERS` is rejected (PR #29262). |
| `--mm-encoder-attn-dtype` | `fp8` or `None` | `None` | FlashInfer cuDNN FP8 attention in the encoder. |
| `--mm-encoder-fp8-scale-path` | path | `None` | Per-layer FP8 Q/K/V scales for static FP8 attention. |
| `--mm-encoder-fp8-scale-save-path` | path | `None` | Persist calibrated dynamic-FP8 scales after warm-up. Mutually exclusive with `--mm-encoder-fp8-scale-path`. |
| `--mm-encoder-fp8-scale-save-margin` | float > 0 | `1.5` | Safety multiplier on auto-saved scales. |

### 2.4 Profiling and IPC

| Flag | Type | Default | Purpose |
| --- | --- | --- | --- |
| `--skip-mm-profiling` | bool | `False` | Skip worst-case dummy profiling on startup. Faster boot, you own peak-memory sizing. |
| `--mm-tensor-ipc` | `direct_rpc` \| `torch_shm` | `direct_rpc` | IPC for multimodal tensors between API server and engine core; `torch_shm` is zero-copy. |

## 3. Flags adjacent to the MM group but heavily multimodal-relevant

These live in other config groups (`ModelConfig`, `SchedulerConfig`,
`LoadConfig`) but you'll touch them every time you serve a VLM:

- **Security**
  - `--allowed-local-media-path PATH` — whitelists a server-side directory so
    clients can pass `file://` URLs. Off by default.
  - `--allowed-media-domains host1 host2 …` — whitelists HTTP(S) hosts.
    Combine with `VLLM_MEDIA_URL_ALLOW_REDIRECTS=0` (env) to block SSRF via
    redirect chains.
- **Model loading**
  - `--trust-remote-code` — many VLMs ship custom processors (Phi-3-V,
    InternVL, MiniCPM-V/o, DeepSeek-VL2, Aria, Molmo, Ovis, …) and won't
    load without it.
  - `--hf-overrides '{"architectures":["DeepseekVLV2ForCausalLM"]}'` — patch
    the HF config when the model card omits the right architecture.
  - `--tokenizer-mode mistral`, `--config-format mistral`, `--load-format mistral` —
    Pixtral, Mistral-Small-3.1, Voxtral.
  - `--runner {auto, generate, pooling, transcription, draft}` and
    `--convert {auto, embed, classify, reward, ...}` — runner adapter.
    Whisper-style ASR uses `generate`.
  - `--io-processor-plugin NAME` — load a custom request-input /
    response-output processor plugin.
  - `--renderer-num-workers N` — pool size for async tokenisation + chat
    rendering + multimodal preprocessing inside the API server. Bump for
    high-throughput VLM serving.
- **Chat & tools**
  - `--chat-template path.jinja` — required when the model card has no
    template and no built-in fallback exists.
  - `--chat-template-content-format {auto, string, openai}` — controls how
    Jinja sees message content. Some VLMs need `openai`; others need
    `string` + `--interleave-mm-strings`.
- **Scheduling**
  - `--max-model-len`, `--max-num-seqs`, `--max-num-batched-tokens`,
    `--enable-chunked-prefill` (default on).
  - `--disable-chunked-mm-input` — opt out of splitting a multimodal request
    across scheduler steps; needed for some processors.
- **Memory & parallelism**
  - `--gpu-memory-utilization`, `--tensor-parallel-size`,
    `--pipeline-parallel-size`, `--data-parallel-size`,
    `--enable-expert-parallel` (Llama-4-Maverick, MoE VLMs),
    `--cpu-offload-gb`, `--cpu-offload-params`, `--enforce-eager`.
- **Attention / KV cache**
  - `--attention-backend`, `--kv-cache-dtype`, `--enable-prefix-caching`,
    `--prefix-caching-hash-algo`.

## 4. Server-only flags (`FrontendArgs`)

(`vllm/entrypoints/openai/cli_args.py`)

- **API**: `--host`, `--port`, `--uds`, `--api-key`, `--root-path`,
  `--middleware`, `--enable-request-id-headers`, `--disable-fastapi-docs`,
  `--api-server-count` (`-asc`), `--headless`, `--config` (YAML file),
  `--grpc`.
- **SSL**: `--ssl-keyfile`, `--ssl-certfile`, `--ssl-ca-certs`,
  `--ssl-cert-reqs`, `--ssl-ciphers`, `--enable-ssl-refresh`.
- **CORS**: `--allow-credentials`, `--allowed-origins '[...]'`,
  `--allowed-methods '[...]'`, `--allowed-headers '[...]'`.
- **Logging**: `--uvicorn-log-level`, `--disable-uvicorn-access-log`,
  `--max-log-len`, `--enable-log-outputs`, `--log-error-stack`,
  `--enable-log-requests`.
- **Chat / Tools / Reasoning**: `--chat-template`,
  `--chat-template-content-format`, `--trust-request-chat-template`,
  `--default-chat-template-kwargs`, `--response-role`,
  `--enable-auto-tool-choice`, `--tool-call-parser`, `--tool-parser-plugin`,
  `--exclude-tools-when-tool-choice-none`, `--tool-server`,
  `--reasoning-parser`, `--reasoning-parser-plugin`.
- **LoRA-as-named-model**: `--lora-modules name=path` or JSON form.

## 5. Per-family overrides

Distilled from `examples/generate/multimodal/vision_language_offline.py` and
`audio_language_offline.py`. The same keyword names work as kebab-cased
`vllm serve` flags.

### Qwen2-VL / Qwen2.5-VL / Qwen3-VL / Qwen3-VL-MoE

```bash
vllm serve Qwen/Qwen2.5-VL-3B-Instruct \
  --max-model-len 4096 --max-num-seqs 5 \
  --limit-mm-per-prompt '{"image":1,"video":1}' \
  --mm-processor-kwargs '{"min_pixels":784,"max_pixels":1003520,"fps":1}'
```

- `min_pixels` / `max_pixels` cap *pre-tile* vision-token counts — the cheapest
  knob for long-video VRAM.
- `fps` controls server-side frame sampling (2.5/3-VL/Omni only).
- For TP≥2 with many small images, add `--mm-encoder-tp-mode data`.
- Qwen3-VL also handles `--media-io-kwargs '{"video":{"frame_recovery":true}}'`.

### Qwen2.5-Omni / Qwen3-Omni (image+video+audio+text)

Same as Qwen-VL plus `--limit-mm-per-prompt '{"image":1,"video":1,"audio":1}'`.
Audio is auto-mono'd via the feature extractor.

### LLaVA-1.5 / LLaVA-NeXT / LLaVA-OneVision / LLaVA-NeXT-Video

```bash
vllm serve llava-hf/llava-onevision-qwen2-0.5b-ov-hf \
  --runner generate --max-model-len 8192 \
  --limit-mm-per-prompt.image 1 --limit-mm-per-prompt.video 1
```

Nothing exotic; the HF processor doesn't take overrides.

### Pixtral / Mistral-Small-3.1 / Voxtral

```bash
# HF-format Pixtral
vllm serve mistral-community/pixtral-12b --limit-mm-per-prompt.image 1

# Native Mistral repos
vllm serve mistralai/Mistral-Small-3.1-24B-Instruct-2503 \
  --tensor-parallel-size 2 --max-model-len 8192 --max-num-seqs 2 \
  --limit-mm-per-prompt.image 1 \
  --ignore-patterns 'consolidated.safetensors'

vllm serve mistralai/Voxtral-Mini-3B-2507 \
  --tokenizer-mode mistral --config-format mistral --load-format mistral \
  --enforce-eager --no-enable-chunked-prefill \
  --max-model-len 8192 --limit-mm-per-prompt.audio 1
```

### Phi-3.5-V / Phi-4-Multimodal

```bash
vllm serve microsoft/Phi-3.5-vision-instruct \
  --runner generate --trust-remote-code \
  --max-model-len 4096 --limit-mm-per-prompt.image 2 \
  --mm-processor-kwargs '{"num_crops":16}'

vllm serve microsoft/Phi-4-multimodal-instruct \
  --trust-remote-code --enable-lora --max-lora-rank 320 \
  --max-model-len 5120 --max-num-batched-tokens 12800 \
  --mm-processor-kwargs '{"dynamic_hd":16}' \
  --lora-modules '{"name":"vision","path":"<model>/vision-lora","base_model_name":"phi4mm"}'
```

`num_crops`: 16 for single-image, 4 for multi-image.

### Gemma 3 / Gemma 3N

```bash
vllm serve google/gemma-3-4b-it \
  --max-model-len 2048 --limit-mm-per-prompt.image 1 \
  --mm-processor-kwargs '{"do_pan_and_scan":true}'

vllm serve google/gemma-3n-E2B-it --enforce-eager \
  --max-model-len 2048 --limit-mm-per-prompt.image 1
```

`pan_and_scan_max_num_crops` is another knob on Gemma 3.

### InternVL3 / Mono-InternVL / InternS1

```bash
vllm serve OpenGVLab/InternVL3-2B \
  --trust-remote-code --max-model-len 8192 \
  --limit-mm-per-prompt '{"image":1,"video":1}'
```

HF processors expose `crop_to_patches` — pass via `--mm-processor-kwargs`.

### MiniCPM-V-2.6 / MiniCPM-o-2.6

```bash
vllm serve openbmb/MiniCPM-V-2_6 \
  --trust-remote-code --max-model-len 4096 --max-num-seqs 2 \
  --limit-mm-per-prompt '{"image":1,"video":1}'
```

Image-embedding inputs require both `image_embeds` and `image_sizes`.

### Llama-4 (Scout / Maverick)

```bash
vllm serve meta-llama/Llama-4-Scout-17B-16E-Instruct \
  --tensor-parallel-size 8 --gpu-memory-utilization 0.4 \
  --max-model-len 8192 --max-num-seqs 4 --limit-mm-per-prompt.image 1
```

Maverick is MoE — add `--enable-expert-parallel`.

### DeepSeek-VL2

```bash
vllm serve deepseek-ai/deepseek-vl2-tiny --trust-remote-code \
  --hf-overrides '{"architectures":["DeepseekVLV2ForCausalLM"]}' \
  --max-model-len 4096 --limit-mm-per-prompt.image 1
```

### Aya-Vision

```bash
vllm serve CohereForAI/aya-vision-8b \
  --max-model-len 4096 --limit-mm-per-prompt.image 1 \
  --mm-processor-kwargs '{"crop_to_patches":true}'
```

### Moondream3

```bash
vllm serve moondream/moondream3-preview \
  --tokenizer moondream/starmie-v1 --trust-remote-code \
  --max-model-len 2048 --limit-mm-per-prompt.image 1
```

### Whisper (ASR)

```bash
vllm serve openai/whisper-large-v3-turbo \
  --max-model-len 448 --max-num-seqs 5 --limit-mm-per-prompt.audio 1
```

Auto-mono is on. For >30 s audio, call `vllm.multimodal.audio.split_audio`
client-side and chunk.

### Ultravox / Granite-Speech / Qwen2-Audio / Qwen3-ASR / FunAudioChat / MiDashengLM / MusicFlamingo / AudioFlamingo3

Plain audio configs:

```bash
vllm serve fixie-ai/ultravox-v0_5-llama-3_2-1b \
  --max-model-len 4096 --max-num-seqs 5 \
  --trust-remote-code --limit-mm-per-prompt.audio 1
```

## 6. Recipe: secure multimodal server with sane defaults

```bash
export VLLM_MEDIA_URL_ALLOW_REDIRECTS=0
export VLLM_IMAGE_FETCH_TIMEOUT=10
export VLLM_VIDEO_FETCH_TIMEOUT=60

vllm serve Qwen/Qwen2.5-VL-7B-Instruct \
  --host 0.0.0.0 --port 8000 --api-key "$VLLM_API_KEY" \
  --tensor-parallel-size 2 --max-model-len 32768 \
  --gpu-memory-utilization 0.9 \
  --limit-mm-per-prompt '{"image":4,"video":1}' \
  --mm-processor-kwargs '{"min_pixels":784,"max_pixels":1003520,"fps":1}' \
  --mm-processor-cache-gb 8 --mm-processor-cache-type shm \
  --mm-encoder-tp-mode data \
  --allowed-media-domains upload.wikimedia.org cdn.example.com \
  --enable-prefix-caching \
  --enable-auto-tool-choice --tool-call-parser hermes \
  --chat-template-content-format auto
```

What this gives you:

- HTTP+API-key auth, behind a CORS-aware FastAPI.
- TP=2 across 2 GPUs, big enough for Qwen2.5-VL-7B at 32k context.
- 4 images / 1 video per request, with vision-token caps that keep VRAM sane.
- Encoder runs in "data" TP mode → bigger effective ViT batch.
- 8 GiB shared-memory MM preprocessor cache, shared across API/DP workers.
- SSRF-resistant media fetcher: redirect-blocked, whitelisted hosts only.

## 7. Gotchas

- `--limit-mm-per-prompt` defaults to **999** per modality. Profiling allocates
  worst-case dummy items at that count → easy OOM on video models if you
  forget to set it.
- `--mm-processor-cache-gb` is per-process. With `--data-parallel-size 8` the
  default `4` GiB → 32 GiB of CPU RAM just for the cache.
- `--skip-mm-profiling` skips the upfront safety net. Pair with `--enforce-eager`
  and very conservative `--max-num-batched-tokens` if you use it.
- `--mm-encoder-tp-mode data` silently degrades to `weights` if the encoder
  doesn't implement batch-split — verify in startup logs.
- For online serving you almost always need a chat template. If the HF repo
  doesn't ship one and the built-in registry doesn't cover the model, vLLM
  errors at startup. Pass `--chat-template path.jinja` or pick the right
  `--chat-template-content-format` and `--interleave-mm-strings` combo.
- Mistral-format models need *three* flags together
  (`--tokenizer-mode mistral --config-format mistral --load-format mistral`),
  not just one.
- `--enable-mm-embeds` is a trusted-users-only flag. Wrong-shape tensors
  crash the engine.
- `--mm-encoder-attn-backend XFORMERS` is explicitly rejected.

## 8. Cheatsheet by modality

| Need | Flag(s) |
| --- | --- |
| Limit number of images per request | `--limit-mm-per-prompt.image N` |
| Limit number of videos per request | `--limit-mm-per-prompt.video N` |
| Limit number of audios per request | `--limit-mm-per-prompt.audio N` |
| Constrain video frames at profiling | `--limit-mm-per-prompt '{"video":{"count":1,"num_frames":32,"width":512,"height":512}}'` |
| Force one specific FPS / resolution | `--mm-processor-kwargs '{"fps":1,"min_pixels":...,"max_pixels":...}'` (Qwen) |
| Drop video tokens | `--video-pruning-rate 0.3` |
| Recover from a corrupt video frame | `--media-io-kwargs '{"video":{"frame_recovery":true}}'` |
| Black background for RGBA images | `--media-io-kwargs '{"image":{"rgba_background_color":[0,0,0]}}'` |
| Accept precomputed embeddings | `--enable-mm-embeds` |
| Disable MM preprocessor cache | `--mm-processor-cache-gb 0` |
| Share MM cache across workers | `--mm-processor-cache-type shm --mm-processor-cache-gb 8` |
| Run encoder TP in batch-parallel mode | `--mm-encoder-tp-mode data` |
| FP8 ViT attention | `--mm-encoder-attn-dtype fp8` |
| Use local `file://` media | `--allowed-local-media-path /srv/uploads` |
| Whitelist remote hosts | `--allowed-media-domains a.com b.com` (+ `VLLM_MEDIA_URL_ALLOW_REDIRECTS=0`) |
| Skip MM profiling (fast boot) | `--skip-mm-profiling` (only if you've sized memory manually) |

## References

- Source: `vllm-project/vllm@main`
  - `vllm/engine/arg_utils.py` (`AsyncEngineArgs.add_cli_args`)
  - `vllm/entrypoints/openai/cli_args.py` (`FrontendArgs`)
  - `vllm/config/multimodal.py` (`MultiModalConfig`)
  - `vllm/config/model.py` (`ModelConfig` — `runner`, `convert`,
    `allowed_local_media_path`, `allowed_media_domains`, `hf_overrides`,
    `io_processor_plugin`, `renderer_num_workers`)
- Docs:
  - `docs/cli/{README.md,serve.md,chat.md,complete.md,run-batch.md}`
  - `docs/cli/json_tip.inc.md`
  - `docs/configuration/{engine_args.md,serve_args.md,optimization.md}`
  - `docs/features/multimodal_inputs.md`
- Examples:
  - `examples/generate/multimodal/vision_language_offline.py` (one
    `run_<family>()` per supported VLM)
  - `examples/generate/multimodal/audio_language_offline.py`
  - `examples/generate/multimodal/openai_chat_completion_client_for_multimodal.py`
  - `examples/generate/multimodal/qwen2_5_omni/`, `qwen3_omni/`
