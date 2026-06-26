# Running Heavy ML Inference in a Worker Subprocess

> When a plugin needs heavy machine-learning inference -- monocular depth,
> segmentation, super-resolution, matting -- running the model on the host's
> render thread is risky: a long, memory-hungry, or crash-prone inference call
> blocks the UI and can take the whole host down with it. The robust pattern is
> to run inference in a **separate worker process** and talk to it over a simple
> request/response channel, with a graceful fallback when the worker can't start.

This is a *different* mechanism from sharing state between several plugins in one
host process (PlugPlug, `dlopen` global symbols, AEGP stream params) -- see
[inter-plugin communication](../advanced/ipc.md) for that. Here the goal is
**isolation**: push fragile, heavyweight work out of the host's address space.

## Why a separate process, not a thread

| Concern | In-process thread | Separate worker process |
|---------|-------------------|--------------------------|
| A crash in the model/runtime | Takes down the host | Killed and respawned; host survives |
| Heavy native deps (ONNX Runtime, CUDA, large models) loaded into the host | Bloats host, version conflicts | Isolated; host stays lean |
| A multi-second inference call | Blocks the render thread / UI | Runs out-of-band; host can show progress or fall back |
| Native runtime that calls `exit()`/`abort()` on error | Fatal to host | Contained |

The hard requirement that motivates this pattern is **never crash the host**. A
subprocess boundary is the only way to guarantee that a third-party inference
runtime's failure is recoverable.

## Architecture

```
Host render thread
  |
  |  WorkerManager (process-wide singleton)
  |    - lazily spawns the worker exe on first use
  |    - holds one IPC client connection
  v
IPC client  ----[ request / response over named pipe or stdio ]---->  Worker process
                                                                        |
                                                                        |  loads model once
                                                                        v
                                                                     ML runtime (e.g. ONNX)
```

- **WorkerManager** -- a process-wide singleton on the plugin side. It owns the
  worker's lifetime and a single client connection, and spawns the worker lazily
  the first time inference is requested.
- **IPC client** -- a thin RPC wrapper: serialize a request (e.g. a frame plus
  parameters), send it, block for the response (or poll asynchronously),
  deserialize the result.
- **Worker process** -- a standalone executable that loads the model once, then
  serves requests in a loop until told to exit or its parent dies.

## Transport: named pipe or stdio

Two simple, dependency-free transports cover both platforms:

- **Named pipe / local socket** -- a named pipe on Windows
  (`\\.\pipe\<name>`), a Unix domain socket or FIFO on macOS/Linux. Bidirectional
  and easy to frame.
- **stdio** -- spawn the worker with its stdin/stdout redirected and exchange
  length-prefixed messages over the pipes. Zero naming/permission concerns; the
  channel dies automatically when either side exits.

Either way, frame messages explicitly (a length prefix or a sentinel) so partial
reads reassemble correctly, and pick a payload format: raw bytes for pixel
buffers (fast), plus a small header (JSON or a packed struct) for parameters and
result metadata.

```cpp
// Request/response contract (illustrative):
struct InferRequest {
    uint32_t width, height, channels;
    // followed by width*height*channels bytes of pixel data
    // plus any model parameters
};

struct InferResult {
    bool      ok;
    uint32_t  width, height, channels;
    // followed by the result buffer (e.g. a single-channel depth map)
};
```

## Lazy spawn on first use

Spawn the worker the first time inference is actually needed, not at plugin load
-- many sessions never use the ML feature, and an unused worker wastes memory and
startup time.

```cpp
class WorkerManager {
public:
    static WorkerManager& instance() { static WorkerManager h; return h; }

    // Returns a connected client, or nullptr if the worker can't be started.
    IpcClient* client() {
        std::lock_guard<std::mutex> lock(m_mutex);
        if (m_client && m_client->alive()) return m_client.get();
        if (!spawnWorker()) return nullptr;   // graceful failure: caller falls back
        return m_client.get();
    }

private:
    bool spawnWorker() {
        std::string exe = locateWorkerExecutable();   // see below
        if (exe.empty()) return false;
        try {
            m_proc   = launchProcess(exe, /* with pipe handles */);
            m_client = std::make_unique<IpcClient>(m_proc.channel());
            return m_client->handshake();              // confirm it answered
        } catch (...) {
            return false;
        }
    }

    std::unique_ptr<IpcClient> m_client;
    ProcessHandle              m_proc;
    std::mutex                 m_mutex;
};
```

## Graceful fallback when the worker can't start

The worker is an *enhancement*, never a hard dependency of the render. If it
fails to spawn, the connection drops, or it returns an error, the plugin must
fall back to a degraded-but-working path (a non-ML approximation) and keep
rendering:

```cpp
IpcClient* client = WorkerManager::instance().client();
if (!client) {
    // Worker unavailable -> use the built-in fallback (e.g. derive depth from
    // luma instead of from the ML model). Never throw; never crash the host.
    params.source = SOURCE_FALLBACK;
} else {
    InferResult r = client->infer(frame, w, h);
    if (r.ok) useResult(r);
    else      params.source = SOURCE_FALLBACK;   // model failed this frame
}
```

This also makes the feature naturally optional per build/tier: a build that ships
no worker simply always takes the fallback path.

## Locating the worker executable

The worker must be found **relative to the plugin's own binary**, not via a
fixed absolute path or `PATH` -- plugins install to host-specific locations and
the working directory is the host's, not yours.

1. Resolve the path of the loaded plugin binary at runtime:
   - macOS: `dladdr()` on a symbol in the plugin, then walk up to
     `Contents/MacOS/`.
   - Windows: `GetModuleHandleEx`/`GetModuleFileName` with a plugin symbol.
2. Join the known relative location of the worker (see embedding below).
3. Verify the file exists and is executable before spawning; if not, return
   "no worker" so the caller falls back.

## Embedding the worker in the package

Ship the worker alongside the plugin so it always travels with it:

- **macOS** -- place the worker inside the plugin bundle at
  `Contents/MacOS/<Worker>` (next to the plugin binary). It is signed and
  notarized as part of the bundle.
- **Windows** -- place `<Worker>.exe` next to the `.aex`/`.ofx` (or in a
  `Contents/Win64`-style subfolder) and include it in the installer.

Either way the worker resolves from the plugin's own directory, so a user who
copies the plugin gets the worker too.

## Lifetime and cleanup

- Keep the worker alive across frames -- reloading a large model per frame is
  prohibitively slow. Load once, serve many requests.
- Have the worker exit when its parent dies (poll the parent PID, or rely on the
  pipe closing) so it never lingers after the host quits.
- Serialize access to a single worker connection (mutex), or run a small pool if
  the host renders frames concurrently and the model is thread-safe enough to
  warrant it.

## Checklist

- [ ] Inference runs in a separate process, never on the host render thread
- [ ] Transport is framed (length-prefixed) over a named pipe or stdio
- [ ] Worker spawned lazily on first use, behind a process-wide singleton
- [ ] Every failure path (no exe, spawn fail, dropped connection, model error)
      falls back gracefully -- the render never throws or crashes the host
- [ ] Worker located relative to the plugin binary, not an absolute path
- [ ] Worker embedded in the bundle (mac `Contents/MacOS`) / shipped alongside
      (Windows) and signed with it
- [ ] Worker loads the model once and exits when the parent dies

## See also

- [Inter-plugin communication](../advanced/ipc.md) -- the *other* IPC problem:
  sharing state between plugins in one host process (distinct from this
  out-of-process worker pattern).
- [Protecting Shipped ML Model Weights](model-encryption.md) -- if the worker
  ships an embedded model, obfuscate the weights.

*Tags: `architecture`, `ipc`, `subprocess`, `worker`, `ml`, `inference`, `onnx`, `fallback`*
