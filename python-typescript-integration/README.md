# Integrating a Python library into a TypeScript project

> [!NOTE]
> This is an AI-generated research report. All text and code in this report
> was created by an LLM (Large Language Model) — specifically Claude
> (Opus 4.7) running inside Claude Code. No part of this document was
> written by a human.

## Prompts that produced this report

The report was generated from this conversation with Claude Code:

> Consider you have a python library for example scenedetect and I want to
> use it in a typescript project — help me identify the most optimized way
> to achieve that, along with all the options that exist.

Followed by a refinement:

> Can you add more details about each of these methods and talk about
> limitations a bit more.

The model was then asked to commit the result as a research doc.

---

Research notes on the available approaches, their tradeoffs, and detailed
limitations. Written with `scenedetect` as a running example because it's a
good stress test: it's CPU-heavy, depends on OpenCV, and processes large
binary inputs (video).

---

## Options at a glance

| # | Method | Best for | Avoid when |
|---|---|---|---|
| 1 | Child process (subprocess) | Scripts, CLIs, one-shots | High call frequency, low latency |
| 2 | Local HTTP microservice (FastAPI) | Production sync APIs | Requests longer than proxy timeouts |
| 3 | gRPC | Streaming, strict contracts, high RPS | Browser clients, low traffic |
| 4 | Message / job queue | Long-running jobs, bursty load | Strict request/response semantics |
| 5 | Native embedding (`node-calls-python`) | Electron / desktop apps | Servers, multi-tenancy |
| 6 | WebAssembly (Pyodide) | Pure-Python libs, sandboxed exec | Anything needing C extensions like OpenCV |
| 7 | Serverless functions | Bursty, infrequent workloads | Long jobs, large dependencies |
| 8 | Replace with a JS/TS equivalent | When parity is acceptable | Algorithm-critical workloads |
| 9 | Port the library | Small surface, core to product | Anything large or actively maintained upstream |

---

## 1. Child process (subprocess)

**How it works**: Node spawns `python script.py` (or the `scenedetect` CLI) as
a separate OS process. Data flows through stdin/stdout/stderr or files on disk.

**Two flavors**:
- **One-shot**: `execa('scenedetect', ['-i', 'video.mp4', 'list-scenes'])` —
  fresh Python interpreter per call.
- **Long-running worker**: spawn once, send newline-delimited JSON requests on
  stdin, read responses on stdout. You implement the framing.

**Concrete sketch**:
```ts
const py = spawn('python', ['worker.py']);
py.stdin.write(JSON.stringify({video: 'a.mp4'}) + '\n');
py.stdout.on('data', chunk => /* parse JSON lines */);
```

**Limitations**:
- **Cold start cost is brutal for ML/CV libs**: importing `cv2` + `scenedetect`
  is ~400-800ms. 100 calls = a minute of pure import overhead. Long-running
  worker fixes this but you now own a stateful process.
- **No type safety across the boundary** — you're parsing JSON the Python side
  serialized.
- **Error handling is awkward**: Python tracebacks come over stderr, you have
  to detect non-zero exit codes, and partial stdout output on crash can
  corrupt your parser.
- **Backpressure**: stdout buffers fill up. If your TS side stops reading,
  Python blocks on `print()`. Easy to deadlock without `flush=True` and
  proper stream draining.
- **Binary data**: passing video bytes through stdin works but base64 inflates
  ~33%; better to pass file paths.
- **Concurrency**: one subprocess = one job at a time. You need a pool.
- **Windows quirks**: signal handling, path separators, and shebang lines all
  differ.

---

## 2. Local HTTP microservice (FastAPI / Flask / Litestar)

**How it works**: Python service exposes REST endpoints; TS calls them. The
Python process stays warm, imports happen once at startup.

**Concrete sketch**:
```python
@app.post("/detect")
async def detect(req: DetectRequest) -> SceneList:
    return scenedetect.detect(req.video_url)
```
```ts
const scenes = await fetch('http://localhost:8000/detect', {
  method: 'POST', body: JSON.stringify({video_url})
}).then(r => r.json());
```

**Deployment shapes**:
- Same machine, different process (sidecar pattern)
- Docker compose with two services
- Kubernetes pod with two containers
- Separate fleet behind a load balancer

**Limitations**:
- **The GIL is real**: FastAPI's `async def` doesn't help if `scenedetect` is
  CPU-bound — it'll block the event loop. Use `def` (sync) endpoints +
  uvicorn `--workers N`, or run heavy work in `run_in_executor`. Most people
  get this wrong.
- **Process count tuning**: each worker holds a full copy of OpenCV in memory
  (~200-400MB). 8 workers = 3GB resident before processing anything.
- **Long requests don't fit HTTP well**: scene detection on a 2-hour video
  takes minutes. You'll hit proxy timeouts (nginx default 60s, ALB 60s,
  Cloudflare 100s). Solutions: SSE streaming, polling endpoints, or move to
  a job queue.
- **Payload size**: don't send video bytes in the body. Use signed URLs (S3)
  or shared volumes.
- **Type drift**: TS and Python schemas can diverge silently. Use
  `openapi-typescript` to codegen types from FastAPI's OpenAPI schema in CI.
- **Auth and networking**: now you have a service to secure even if it's
  "internal" — bind to localhost or use mTLS.
- **Observability burden**: you now need logs, metrics, health checks for an
  extra service.

---

## 3. gRPC

**How it works**: Define a `.proto` schema, generate stubs for Python and TS,
communicate over HTTP/2 with binary framing.

**Where it shines for scene detection**: server-streaming RPC — Python yields
scene cuts as they're detected instead of waiting for the whole video.

**Concrete sketch**:
```proto
service SceneDetector {
  rpc Detect(DetectRequest) returns (stream SceneCut);
}
```

**Limitations**:
- **Browser support is poor**: gRPC needs HTTP/2 trailers; browsers can't
  speak raw gRPC. You'd need gRPC-Web + a proxy (Envoy) or switch to
  **Connect-RPC** which works in browsers.
- **Tooling friction**: protoc, buf, generated code in the repo, version
  pinning across two languages. More setup than HTTP.
- **Debugging is harder**: can't `curl` it; need `grpcurl` or a UI like Bloom.
- **Overkill for low traffic**: if you're doing <100 RPS, the wins over JSON
  are negligible.

---

## 4. Message queue / job queue

**How it works**: TS API enqueues a job (with video URL + params); Python
workers pull jobs and process them. Results go back via another queue, a
database row, or a webhook.

**Stack options**:
- **Redis-based**: BullMQ (TS) + RQ or Celery (Python). Same Redis, different
  client libs — they don't share queue formats, so pick one or wrap.
- **Cloud-native**: SQS + Lambda, Pub/Sub + Cloud Run.
- **Heavier**: RabbitMQ, Kafka, Temporal (Temporal is genuinely nice for long
  workflows with retries and durability).

**Concrete shape**:
```
TS API → enqueue(jobId, videoUrl) → Redis
Python worker → pop job → run scenedetect → write results to Postgres
TS API → GET /jobs/:id → reads Postgres
```

**Limitations**:
- **Eventual consistency**: client gets a job ID, must poll or subscribe. No
  more "request → response."
- **Cross-language queue formats**: BullMQ jobs aren't readable by Celery and
  vice versa. If you want both ecosystems' tooling, you need a shared
  neutral format (raw Redis lists with your own envelope) — at which point
  you're building plumbing.
- **Idempotency**: workers can be killed and jobs retried. Your scene
  detection must be safe to run twice (deterministic output keyed by video
  hash, for example).
- **Poison messages**: a video that crashes the worker will get retried
  forever unless you implement DLQs.
- **Visibility timeouts vs job duration**: SQS default visibility is 30s; a
  10-min scene detect will be redelivered to another worker mid-process. You
  have to tune or extend visibility.
- **Operational complexity jumps**: now you have a queue, workers, a results
  store, and a polling/webhook mechanism.

---

## 5. Native embedding (`node-calls-python`, `pythonia`)

**How it works**: Load a Python interpreter inside the Node process via FFI.
Call Python functions like JS functions.

**Concrete sketch**:
```ts
const py = require('node-calls-python').interpreter;
const sd = await py.import('scenedetect');
const scenes = await py.call(sd, 'detect', ['video.mp4']);
```

**Limitations**:
- **One Python interpreter per Node process and the GIL is still there**:
  parallelism is fake. Two Node async calls into Python serialize on the GIL.
- **ABI fragility**: must match Python version, libpython build flags, and
  OpenCV's binary deps. Upgrading Python or Node can break the bridge
  silently.
- **Crash blast radius**: a segfault in OpenCV crashes Node. With
  subprocess/HTTP you lose just the Python process.
- **Deployment nightmare**: your Docker image now needs both `node:X` and
  `python:Y` plus all native libs (libGL for OpenCV, ffmpeg, etc.)
  co-located and compatible.
- **Memory leaks compound**: leaks in either runtime accumulate in the same
  process.
- **No good story for serverless**: Lambda layers don't compose well across
  runtimes.
- **In practice**: only consider this for embedded/desktop apps (Electron)
  where you can't ship a sidecar process. Avoid for servers.

---

## 6. WebAssembly (Pyodide)

**How it works**: CPython compiled to WASM runs inside Node (or browser).
Pure-Python packages work; C extensions need to be ported individually.

**Why it doesn't work for scenedetect**:
- scenedetect depends on **OpenCV**. There's an experimental `opencv-python`
  Pyodide port but it's incomplete and slow.
- ffmpeg/codec access through WASM is limited (you'd need ffmpeg.wasm,
  separate stack).
- Performance is 3-10× slower than native CPython for CV workloads.

**Where it does work**: pure-Python libs (data manipulation, parsing, numpy
and pandas which Pyodide ports). Great for sandboxing untrusted Python.

**Limitations**:
- **No threading** (or limited via SharedArrayBuffer with COOP/COEP headers).
- **File system is virtual** — copying large videos in/out has overhead.
- **Cold start**: loading Pyodide is ~5MB download + a few seconds parse time.
- **Package availability** is the killer — check
  https://pyodide.org/en/stable/usage/packages-in-pyodide.html before
  committing.

---

## 7. Serverless functions

**How it works**: Deploy Python code as Lambda / Cloud Function / Cloud Run;
TS invokes via cloud SDK or HTTP.

**Limitations**:
- **Cold starts on Lambda for OpenCV-bearing code are 3-8 seconds** — first
  request after idle is painful. Provisioned concurrency costs money.
- **Package size limits**: Lambda zip is 250MB unzipped. OpenCV + scenedetect
  + numpy easily blows that. Workarounds: container images (10GB), Lambda
  Layers, or trim deps.
- **Timeout walls**: Lambda 15min hard cap, Cloud Run can go longer but costs
  add up.
- **Ephemeral disk**: `/tmp` is 512MB by default on Lambda (configurable to
  10GB). Big videos won't fit.
- **No GPU on most serverless** (some exist but expensive/limited).
- **Egress costs** for downloading videos from S3 to Lambda repeatedly.
- **Statefulness**: warm containers may reuse imports — design for both cold
  and warm.

**Sweet spot**: short videos (<5 min), unpredictable traffic, you don't want
to manage infra.

---

## 8. Replace with a JS/TS equivalent

**For scenedetect**: `ffmpeg` has built-in scene detection:
```bash
ffmpeg -i in.mp4 -vf "select='gt(scene,0.4)',showinfo" -f null -
```
Parse the `showinfo` output for `pts_time`. Drive it from Node via
`fluent-ffmpeg` (native ffmpeg) or `@ffmpeg/ffmpeg` (WASM).

**Limitations**:
- **Algorithm parity**: ffmpeg's scene filter is a single-threshold pixel-diff;
  scenedetect offers `ContentDetector` (HSV-weighted), `AdaptiveDetector`
  (rolling average), `ThresholdDetector` (fade detection), and `HashDetector`.
  If you depend on the adaptive or content-aware detectors, ffmpeg won't
  reproduce them.
- **Tuning**: scenedetect's defaults are well-researched; ffmpeg's threshold
  takes more trial-and-error.
- **ffmpeg.wasm performance**: 5-10× slower than native ffmpeg, no hardware
  accel.

**General principle**: always check if the JS ecosystem has a "good enough"
equivalent before standing up a Python sidecar. Common matches: `sharp` for
PIL, `tesseract.js` for pytesseract, `onnxruntime-node` for many ML models,
`dayjs` for arrow/pandas datetime work.

---

## 9. Port the library

**When it makes sense**: small surface area, library is core to your product,
performance matters, you want one deployment artifact.

**Limitations**:
- **Maintenance burden forever**: upstream bug fixes don't flow to you.
- **Algorithmic correctness is hard to verify**: subtle differences in pixel
  math, rounding, or color space conversions produce different cut points
  than scenedetect. You need a golden-file test suite comparing against the
  original.
- **Time sink**: scenedetect is ~5K lines of Python with non-trivial CV
  logic. Porting realistically takes weeks plus ongoing maintenance.

---

## Cross-cutting limitations

These hit several options:

1. **Type contracts**: anything with a network/process boundary needs a way to
   keep TS and Python types in sync. Options: OpenAPI codegen (HTTP),
   protobuf (gRPC), JSON Schema + `quicktype`, or hand-written types you'll
   forget to update.

2. **Observability**: traces don't cross language boundaries by default. Use
   OpenTelemetry SDKs in both languages with W3C trace context propagation,
   otherwise debugging "why is this slow?" requires correlating two log
   streams by timestamp.

3. **Dependency hell at deploy time**: any solution shipping Python needs the
   right Python version, native libs (libGL, ffmpeg, libsm6 for OpenCV
   headless), and platform-specific wheels. Always pin via `uv` or
   `pip-tools` and build images on the same arch you deploy to (Mac M-series
   → linux/amd64 mismatch is a classic gotcha).

4. **Resource isolation**: a Python worker with a memory leak in OpenCV can
   OOM the whole host. Run with cgroup limits (Docker `--memory`) and a
   process supervisor that restarts on OOM.

5. **Security posture**: every option that runs Python on user-supplied input
   (video files) inherits Python's CV stack vulnerabilities. Sandbox with
   gVisor/Firecracker or process isolation if input is untrusted.

---

## Decision shortcut

| Situation | Pick |
|---|---|
| Quick script, internal tool | Subprocess (one-shot) |
| Repeated calls, same machine | Subprocess (long-running worker) |
| Production API, sync calls | FastAPI sidecar |
| Heavy/long jobs | Job queue |
| Multiple TS clients, strict typing | gRPC or FastAPI + OpenAPI codegen |
| Bursty, infrequent, cheap | Serverless container |
| Embedded app (Electron) | Subprocess; native embedding only if forced |
| Browser-only execution | Pyodide if pure Python; otherwise rewrite |
| Algorithm has a JS equivalent | Just use the JS version |

---

## Recommendation for the scenedetect example

- **One-shot tool/script**: subprocess to the `scenedetect` CLI, parse its
  CSV/JSON output. Simplest, ships today.
- **Production service**: FastAPI wrapper exposing `/detect` that takes a
  video URL, returns scene cut timestamps as JSON. Containerize with OpenCV
  preinstalled. Generate a TS client from the OpenAPI schema.
- **Large/long videos**: add a job queue so the TS API returns a job ID and
  the Python worker processes asynchronously.
- **Zero Python**: try ffmpeg's scene filter first — it may be enough.
