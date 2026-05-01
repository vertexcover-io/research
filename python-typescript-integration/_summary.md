A survey of approaches for calling a Python library from a TypeScript project, using [scenedetect](https://github.com/Breakthrough/PySceneDetect) as a representative stress test (CPU-heavy, OpenCV-dependent, large binary inputs). Nine integration patterns are compared — subprocess, [FastAPI](https://fastapi.tiangolo.com/) sidecar, gRPC, job queues, native embedding, [Pyodide](https://pyodide.org/), serverless, JS-equivalent replacement, and porting — with detailed limitations covering cold-start cost, the GIL, payload size, proxy timeouts, and deployment complexity. The report ends with a decision matrix mapping situations to recommended approaches.

Key takeaways:
- Long-running subprocess workers eliminate Python import overhead for repeated calls; one-shot subprocess is fine for scripts.
- A FastAPI sidecar is the typical production answer, but watch out for the GIL blocking async endpoints and proxy timeouts on long jobs.
- Pyodide is not viable for libraries with C extensions like OpenCV.
- Always check whether a JS equivalent (e.g. `ffmpeg` scene filter) gets you 80% of the way before standing up a Python service.
