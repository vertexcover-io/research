# Investigation Notes — vLLM CLI Parameters for Multimodal LLMs

Goal: build a deep understanding of every CLI parameter relevant when running
vLLM against multimodal models (vision-language, audio-language, video-language,
omni, encoder-decoder).

## How I structured the research

1. Pulled the vLLM source (a shallow, sparse clone of `vllm-project/vllm@main`)
   so I could read the authoritative argument definitions instead of relying on
   docs that are sometimes stale and that I cannot fetch from the rendered site
   (vLLM's docs domain returns 403 to outside fetchers).
2. Read the CLI surface bottom-up from three places:
   - `vllm/engine/arg_utils.py` — `AsyncEngineArgs.add_cli_args`. This is the
     canonical list of engine CLI flags; `vllm serve` re-uses it verbatim.
   - `vllm/entrypoints/openai/cli_args.py` — `FrontendArgs` adds the
     server-only flags (host/port/SSL/CORS/chat-template/tool-call-parser/...).
   - `vllm/config/multimodal.py` — `MultiModalConfig` defines the multimodal
     fields whose CLI names are `--limit-mm-per-prompt`,
     `--mm-processor-kwargs`, `--media-io-kwargs`, etc.
3. Walked the model-family-specific knobs by reading
   `examples/generate/multimodal/{vision_language_offline,audio_language_offline}.py`,
   which contains a `run_<model>()` per supported architecture and shows the
   `EngineArgs(...)` typical for each one. These translate directly into
   `vllm serve` flags.
4. Cross-checked the rendered user docs at
   `docs/features/multimodal_inputs.md` for the tips that are not encoded in
   the argument metadata (security, env vars, frame recovery, etc.).

## Key files & line numbers (for reference)

- `vllm/engine/arg_utils.py:754-1485` — full CLI registration.
  - Multimodal block: lines 1161–1230.
  - Model block (with `--allowed-media-domains`, `--allowed-local-media-path`,
    `--io-processor-plugin`, `--hf-overrides`, `--runner`, `--convert`,
    `--renderer-num-workers`): lines 757–840.
- `vllm/config/multimodal.py:73-339` — `MultiModalConfig` dataclass; each field
  is a CLI flag with its docstring as the `--help` text.
- `vllm/entrypoints/openai/cli_args.py:69-326` — Frontend (server) args.
- `docs/features/multimodal_inputs.md` — security tips
  (`--allowed-media-domains`, `VLLM_MEDIA_URL_ALLOW_REDIRECTS=0`,
  `VLLM_IMAGE_FETCH_TIMEOUT`, `VLLM_VIDEO_FETCH_TIMEOUT`,
  `VLLM_AUDIO_FETCH_TIMEOUT`), frame recovery, RGBA, embeddings.

## Top-level commands

The `vllm` console-script ships these subcommands (see `docs/cli/README.md`):

| Subcommand     | Purpose                                                                  |
| -------------- | ------------------------------------------------------------------------ |
| `vllm serve`   | Starts the OpenAI-compatible HTTP (or gRPC) server.                      |
| `vllm chat`    | Chat-completions client against a running server.                        |
| `vllm complete`| Text-completions client.                                                  |
| `vllm bench {latency,serve,throughput,sweep}` | Benchmarks (needs `vllm[bench]`).                |
| `vllm run-batch` | Batch JSONL inference (OpenAI-batch-style).                            |
| `vllm collect-env` | Diagnostic dump.                                                     |

For multimodal models, the multimodal-specific knobs are exclusively on
`vllm serve` (and the equivalent `EngineArgs` used in offline `LLM(...)`).

## The multimodal-specific CLI flags

(Source: `vllm/config/multimodal.py` + `vllm/engine/arg_utils.py:1161-1230`.)

- `--limit-mm-per-prompt` — JSON dict; legacy ints `{image:4}` or full
  options `{video:{count:1, num_frames:32, width:512, height:512}}`. Default
  999 per modality. Caps how many media items per modality a request may
  contain. Also used during profiling to size encoder activation memory.
- `--language-model-only` (bool, default `False`) — force every modality
  limit to 0; run as a pure LM.
- `--enable-mm-embeds` (bool, default `False`) — accept precomputed media
  embeddings (`image_embeds`, `audio_embeds`). Bad shapes can crash the
  engine; trusted users only.
- `--media-io-kwargs` — JSON keyed by modality, e.g.
  `'{"video":{"frame_recovery":true,"num_frames":40}}'`,
  `'{"image":{"rgba_background_color":[0,0,0]}}'`. Forwarded to the
  media-IO loader before the model processor. Supports `frame_recovery`,
  `rgba_background_color`, and the client-supplied
  `fps/frames_indices/total_num_frames/duration/do_sample_frames` for
  pre-extracted frame sequences.
- `--mm-processor-kwargs` — JSON dict of per-model overrides forwarded to the
  HF processor. Examples: `{"num_crops":16}` (Phi-3.5-V),
  `{"min_pixels":..., "max_pixels":..., "fps":1}` (Qwen2/2.5/3-VL/Omni),
  `{"dynamic_hd":16}` (Phi-4-MM), `{"do_pan_and_scan":true}` (Gemma 3),
  `{"crop_to_patches":true}` (Aya-Vision).
- `--mm-processor-cache-gb` (float ≥ 0, default `4`) — size (GiB) of the
  per-engine multimodal preprocessor cache. **Duplicated** across each API
  server and DP rank: total RAM ≈ `gb × (api_server_count + data_parallel_size)`.
  `0` disables it.
- `--mm-processor-cache-type` (`lru` | `shm`, default `lru`) — `lru` is
  process-local; `shm` is shared-memory FIFO across processes.
- `--mm-shm-cache-max-object-size-mb` (int ≥ 0, default `128`) — only with
  `--mm-processor-cache-type shm`.
- `--mm-encoder-only` (bool, default `False`) — run the vision/audio encoder
  only (disaggregated encoder deployments).
- `--mm-encoder-tp-mode` (`weights` | `data`, default `weights`) — TP strategy
  for the encoder. `weights` splits the ViT weights, `data` replicates them
  and splits the batch. Falls back to `weights` if the encoder doesn't
  implement batch-split.
- `--mm-encoder-attn-backend` (`FLASH_ATTN`, `TRITON`, ...; `None` default) —
  override encoder attention backend. `XFORMERS` is rejected.
- `--mm-encoder-attn-dtype` (`fp8` or `None`, default `None`) — FlashInfer
  cuDNN FP8 attention in the encoder.
- `--mm-encoder-fp8-scale-path` — JSON with per-layer Q/K/V scales; turns
  FP8 encoder attention into static scaling.
- `--mm-encoder-fp8-scale-save-path` — under dynamic FP8, save calibrated
  scales here after the amax warm-up. Mutually exclusive with the load path.
- `--mm-encoder-fp8-scale-save-margin` (float > 0, default `1.5`) — safety
  multiplier applied to auto-saved scales.
- `--interleave-mm-strings` (bool, default `False`) — allow fully interleaved
  text/media chunks when `--chat-template-content-format=string`.
- `--skip-mm-profiling` (bool, default `False`) — skip the worst-case dummy
  profiling step on startup. Faster startup, but you own peak-memory sizing.
- `--video-pruning-rate` (float in `[0, 1)`) — Efficient Video Sampling: drop
  this fraction of video tokens per item.
- `--mm-tensor-ipc` (`direct_rpc` | `torch_shm`, default `direct_rpc`) — IPC
  for multimodal tensors between API server and engine core. `torch_shm` is
  zero-copy.

### JSON CLI syntax sugar

(from `docs/cli/json_tip.inc.md`) nested keys can be passed dotted, and lists
extended with `+`:

```
--limit-mm-per-prompt '{"image":2,"video":1}'
--limit-mm-per-prompt.image 2 --limit-mm-per-prompt.video 1

--mm-processor-kwargs.min_pixels 784 --mm-processor-kwargs.max_pixels 1003520
```

## Adjacent multimodal-relevant flags

(not in the `MultiModalConfig` group but heavily relevant)

- `--allowed-local-media-path PATH` — whitelist a server-side directory so
  clients may pass `file://` URLs. Security risk; leave unset by default.
- `--allowed-media-domains host1 host2 ...` — whitelist HTTP(S) domains for
  remote media; combine with env `VLLM_MEDIA_URL_ALLOW_REDIRECTS=0` to block
  SSRF via redirects.
- `--io-processor-plugin NAME` — load a custom request-input / response-output
  processor plugin at model startup (registered via entrypoints).
- `--renderer-num-workers INT` (default `1`) — thread pool size for async
  tokenisation + chat-template rendering + multimodal preprocessing inside
  the API server.
- `--runner {auto, generate, pooling, transcription, draft}` — runner mode.
  For ASR use `generate` (default path); pooling/embedding models use
  `pooling`.
- `--convert {auto, embed, classify, reward, ...}` — adapter to convert a
  generation model to a pooling task; rarely needed for VLM.
- `--trust-remote-code` — required for many VLMs that ship custom HF
  processors (Phi-3-V, InternVL, MiniCPM-V/o, DeepSeek-VL2, Aria, Molmo,
  Ovis, ...).
- `--hf-overrides '{"architectures":["DeepseekVLV2ForCausalLM"]}'` — patch the
  HF config (e.g. DeepSeek-VL2 needs this).
- `--tokenizer-mode mistral`, `--config-format mistral`, `--load-format mistral` —
  required by Pixtral, Mistral-Small-3.1, Voxtral.
- `--chat-template path.jinja` — required for online serving when the HF repo
  doesn't ship a default and there's no built-in fallback.
- `--chat-template-content-format {auto, string, openai}` — controls how vLLM
  hands message content to the Jinja template; some VLMs need `openai` so
  `image_url`/`video_url` chunks survive; others need `string` plus
  `--interleave-mm-strings`.
- `--disable-chunked-mm-input` (SchedulerConfig) — opt out of splitting a
  multimodal request across scheduler steps; some processors don't handle
  partial inputs well.
- Universal sizing knobs that matter for VLMs: `--max-model-len`,
  `--max-num-seqs`, `--gpu-memory-utilization`, `--tensor-parallel-size`,
  `--enforce-eager`, `--enable-chunked-prefill`.

## Server-only flags worth knowing

(`vllm/entrypoints/openai/cli_args.py`)

- API: `--host`, `--port`, `--uds`, `--api-key`, `--root-path`, `--middleware`,
  `--enable-request-id-headers`, `--disable-fastapi-docs`,
  `--api-server-count` (`-asc`), `--headless`, `--config` (YAML),
  `--grpc`.
- SSL: `--ssl-keyfile`, `--ssl-certfile`, `--ssl-ca-certs`, `--ssl-cert-reqs`,
  `--ssl-ciphers`, `--enable-ssl-refresh`.
- CORS: `--allow-credentials`, `--allowed-origins '[...]'`,
  `--allowed-methods '[...]'`, `--allowed-headers '[...]'`.
- Logging: `--uvicorn-log-level`, `--disable-uvicorn-access-log`,
  `--max-log-len`, `--enable-log-outputs`, `--log-error-stack`,
  `--enable-log-requests`.
- Chat/Tools/Reasoning: `--chat-template`, `--chat-template-content-format`,
  `--trust-request-chat-template`, `--default-chat-template-kwargs`,
  `--response-role`, `--enable-auto-tool-choice`, `--tool-call-parser`,
  `--tool-parser-plugin`, `--exclude-tools-when-tool-choice-none`,
  `--tool-server`, `--reasoning-parser`, `--reasoning-parser-plugin`.
- LoRA-as-named-model: `--lora-modules name=path` or JSON form.

## Env vars (multimodal-adjacent)

- `VLLM_IMAGE_FETCH_TIMEOUT` (default 5s)
- `VLLM_VIDEO_FETCH_TIMEOUT` (default 30s)
- `VLLM_AUDIO_FETCH_TIMEOUT` (default 10s)
- `VLLM_MEDIA_URL_ALLOW_REDIRECTS=0` — pair with `--allowed-media-domains`.

## Model-family-specific notes

Distilled from `examples/generate/multimodal/vision_language_offline.py` and
`audio_language_offline.py`. Each `run_<family>()` builds an `EngineArgs(...)`;
the same keyword names work as `--kebab-case` flags on `vllm serve`.

### Qwen2-VL / Qwen2.5-VL / Qwen3-VL (and Qwen3-VL-MoE)
- `mm_processor_kwargs={"min_pixels": 28*28, "max_pixels": 1280*28*28, "fps": 1}`.
  - Qwen2-VL ignores `fps`; 2.5/3-VL/Omni use it for frame sampling.
  - These bound pre-tile vision tokens — the cheapest knob to keep VRAM in
    check on long videos.
- `limit_mm_per_prompt={"image": N, "video": M}`.
- `--mm-encoder-tp-mode data` is a good default for TP=2/4 with many small
  images.
- Qwen3-VL supports `--media-io-kwargs '{"video":{"frame_recovery":true}}'`.

### Qwen2.5-Omni / Qwen3-Omni (audio+image+video+text)
- Same `min_pixels/max_pixels/fps`. Audio is auto-monoed via the feature
  extractor.
- Set `--limit-mm-per-prompt '{"image":1,"video":1,"audio":1}'`.

### LLaVA family (1.5, NeXT, OneVision, NeXT-Video)
- Minimal: `model`, `max_model_len`, `limit_mm_per_prompt`.
- OneVision is the cheapest video-capable LLaVA; demo uses
  `--max-model-len 8192`.

### Pixtral / Mistral-Small-3.1 / Voxtral
- HF-format Pixtral (`mistral-community/pixtral-12b`): no extras.
- Native (`mistralai/*`): `--tokenizer-mode mistral`, `--config-format mistral`,
  `--load-format mistral`. Voxtral additionally uses `--enforce-eager` and
  disables chunked prefill.
- Mistral-Small-3.1 examples set `--ignore-patterns consolidated.safetensors`
  to force the sharded set.

### Phi-3.5-V / Phi-4-Multimodal
- `--trust-remote-code` always.
- Phi-3.5-V: `mm_processor_kwargs={"num_crops": 16}` (4 for multi-frame).
- Phi-4-MM is LoRA-augmented:
  - `--enable-lora --max-lora-rank 320`
  - `mm_processor_kwargs={"dynamic_hd": 16}`
  - Vision/speech LoRAs are inside the model directory; in `vllm serve`
    expose them as `--lora-modules '{"name":"vision","path":".../vision-lora","base_model_name":"phi4mm"}'`.

### Gemma 3 / Gemma 3N
- `mm_processor_kwargs={"do_pan_and_scan": True}` (or
  `{"pan_and_scan_max_num_crops": N}`).
- Gemma 3N example uses `--enforce-eager`.

### InternVL3 / Mono-InternVL / InternS1
- Always `--trust-remote-code`.
- Some HF processors expose `crop_to_patches`; pass via `mm_processor_kwargs`.

### MiniCPM-V-2.6 / MiniCPM-o-2.6 (image+video+audio)
- `--trust-remote-code`. Defaults `max_num_seqs=2`, `max_model_len=4096`.
- For embeddings: `image_sizes` is a required side-input alongside
  `image_embeds`.

### Llama-4 (Scout / Maverick)
- Large: example uses `--tensor-parallel-size 8`,
  `--gpu-memory-utilization 0.4`, `--max-model-len 8192`.

### Moondream3
- `--tokenizer moondream/starmie-v1`, `--trust-remote-code`,
  `limit_mm_per_prompt={"image":1}`. Task-specific prompt prefixes (`query` /
  `caption`).

### DeepSeek-VL2
- `--hf-overrides '{"architectures":["DeepseekVLV2ForCausalLM"]}'`.

### Aya-Vision
- `mm_processor_kwargs={"crop_to_patches": True}`.

### Whisper (ASR)
- `--max-model-len 448` (Whisper context cap).
- Auto-mono conversion is on for any feature-extractor-mono model.
- Use `vllm.multimodal.audio.split_audio` client-side to chunk >30 s.

### Ultravox / Voxtral / Granite-Speech / Qwen2-Audio / Qwen3-ASR / FunAudioChat / MiDashengLM / MusicFlamingo / AudioFlamingo3
- All accept `limit_mm_per_prompt={"audio": N}`; rely on `mm_processor_kwargs`
  only when there's a model-specific knob (rare).
- Voxtral requires the Mistral trio of flags.

## Gotchas

1. `--limit-mm-per-prompt` defaults to **999** per modality if unspecified,
   which blows up dummy profiling for video models. Set realistic numbers.
2. `--mm-processor-cache-gb` is duplicated across the API process and each DP
   rank. `--data-parallel-size 8` with default `4` GiB costs 32 GiB of CPU RAM
   for the cache alone.
3. `--skip-mm-profiling` is not a no-op: skip it and you own peak-memory
   sizing manually.
4. `--mm-encoder-attn-backend XFORMERS` is explicitly rejected (PR #29262).
5. For embedding inputs over OpenAI API, each item must be its own content
   chunk; `--enable-mm-embeds` *and* an aware chat template are both required.
6. Most VLMs whose HF processor doesn't ship a default chat template will
   fail to start unless you pass `--chat-template path.jinja`. vLLM keeps a
   fallback registry at `vllm/transformers_utils/chat_templates/registry.py`.
7. `--mm-encoder-tp-mode data` falls back to `weights` silently if the model
   doesn't implement batch-split encoder; check startup logs.

## Sources consulted

- vLLM source @ `vllm-project/vllm@main`, paths above
- `docs/features/multimodal_inputs.md`
- `docs/cli/{README.md,serve.md,chat.md,complete.md,run-batch.md,json_tip.inc.md}`
- `docs/configuration/{engine_args.md,serve_args.md,optimization.md}`
- `examples/generate/multimodal/{vision_language_offline,audio_language_offline}.py`
