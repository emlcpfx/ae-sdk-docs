# Protecting Shipped ML Model Weights

> If your plugin bundles trained model weights (an `.onnx` file, a `.bin`, a
> tensor blob), shipping them as plain files lets anyone copy the model straight
> out of your package. A lightweight defense is to **encrypt the weights at build
> time** with a build secret, **embed** the encrypted bytes into the binary, and
> **decrypt on load** at runtime. This raises the bar significantly without a
> heavy crypto dependency.

This pattern pairs naturally with the
[ML inference worker subprocess](ml-inference-worker-subprocess.md): the worker
is the component that embeds and decrypts the model.

## Threat model -- what this does and does not do

This is **obfuscation, not unbreakable encryption.** It defeats casual copying
("grab the `.onnx` from the bundle"). It does **not** defeat a determined
reverse engineer: the decryption key has to be present in the shipped binary (or
derivable by it), so anyone willing to read your code or dump runtime memory can
recover the weights. Treat it as a deterrent that makes extraction inconvenient,
not as a guarantee.

For real secrecy you would keep the model server-side and never ship it. The
build-time XOR approach below is the practical middle ground for a model that
must run locally.

## The pipeline

```
build time                                  runtime (in the worker process)
----------                                  ---------------------------------
model.onnx  --XOR with MODEL_KEY-->          embedded bytes  --XOR with key-->
            embed as C++ byte array          decrypt in memory
            (model_data.h)                   feed buffer to the ML runtime
```

Three moving parts:

1. A **build secret** (a key string) supplied to the build, never committed to
   source control.
2. A **build script** that XORs the model file with the key and emits a C++
   header containing the encrypted bytes as an array.
3. **Runtime decryption** in the worker that XORs the embedded bytes back with
   the same key and hands the plaintext buffer to the inference runtime --
   without ever writing a decrypted file to disk.

## 1. The build secret

Supply the key as a build-time definition (from a CI secret or local
environment variable), e.g. `MODEL_KEY`. Do not hardcode it in tracked source.

```cmake
# Key comes from the environment / CI secret -- never checked in.
if(NOT DEFINED MODEL_KEY)
    set(MODEL_KEY "$ENV{MODEL_KEY}")
endif()

# Make it available to the code that does the runtime decrypt:
target_compile_definitions(worker PRIVATE MODEL_KEY="${MODEL_KEY}")
```

## 2. Build-time XOR + embed

A small script reads the model, XORs it with the key, and writes a C++ header.
Run it as a custom build step so the encrypted header is regenerated whenever the
model or key changes.

```python
#!/usr/bin/env python3
# embed_model.py model.onnx --key "$MODEL_KEY" -o model_data.h
import argparse

ap = argparse.ArgumentParser()
ap.add_argument("model")
ap.add_argument("--key", required=True)
ap.add_argument("-o", "--out", required=True)
args = ap.parse_args()

data = bytearray(open(args.model, "rb").read())
key  = args.key.encode("utf-8")
for i in range(len(data)):
    data[i] ^= key[i % len(key)]          # repeating-key XOR

with open(args.out, "w") as f:
    f.write("#pragma once\n#include <cstddef>\n#include <cstdint>\n")
    f.write(f"static const size_t kModelLen = {len(data)};\n")
    f.write("static const uint8_t kModelData[] = {\n")
    for i in range(0, len(data), 16):
        f.write("  " + ",".join(str(b) for b in data[i:i+16]) + ",\n")
    f.write("};\n")
```

Wire it into CMake so the encrypted header is a build artifact (the plaintext
model never gets copied into the package):

```cmake
add_custom_command(
    OUTPUT  "${CMAKE_CURRENT_BINARY_DIR}/model_data.h"
    COMMAND python3 "${CMAKE_CURRENT_SOURCE_DIR}/embed_model.py"
            "${CMAKE_CURRENT_SOURCE_DIR}/model.onnx"
            --key "${MODEL_KEY}"
            -o "${CMAKE_CURRENT_BINARY_DIR}/model_data.h"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/model.onnx"
    COMMENT "Encrypting + embedding model weights")

target_sources(worker PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/model_data.h")
target_include_directories(worker PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
```

## 3. Runtime decrypt (in the worker)

Decrypt into a heap buffer at load time and hand that buffer to the inference
runtime. Most runtimes (including ONNX Runtime) can create a session from an
in-memory byte buffer, so you never write a plaintext file to disk.

```cpp
#include "model_data.h"   // kModelData[], kModelLen
#include <cstring>
#include <vector>

static std::vector<uint8_t> decryptModel() {
    std::vector<uint8_t> buf(kModelData, kModelData + kModelLen);
    const char* key = MODEL_KEY;
    const size_t kl = std::strlen(key);
    for (size_t i = 0; i < buf.size(); ++i)
        buf[i] ^= (uint8_t)key[i % kl];   // same repeating-key XOR
    return buf;                            // plaintext model bytes, in memory
}

// Example: create an ONNX Runtime session from the in-memory buffer.
// std::vector<uint8_t> model = decryptModel();
// Ort::Session session(env, model.data(), model.size(), sessionOptions);
```

Because decryption is symmetric, the same key and the same repeating-XOR
recovers the original bytes exactly.

## Practical notes

- **Keep the plaintext model out of the shipped package.** Only the encrypted
  `model_data.h` (compiled into the binary) ships. Confirm the build never copies
  the raw model into the bundle.
- **Never write the decrypted bytes to disk** -- decrypt into memory and feed the
  buffer directly to the runtime. A temp file undoes the whole point.
- **Rotate the key per release if it leaks**, and keep it only in CI secrets /
  local env, never in tracked files or logs.
- **Do this in the worker process**, so the key and decrypted weights live in the
  isolated worker rather than the host's address space (see
  [ML inference worker subprocess](ml-inference-worker-subprocess.md)).
- **Want a stronger bar?** Swap the XOR for a real symmetric cipher (e.g. AES via
  a small crypto library) and derive the key rather than embedding it verbatim.
  The structure -- build-time encrypt, embed, runtime decrypt-in-memory -- stays
  the same; only the transform changes. It is still ultimately bounded by the
  fact that the key ships with the binary.

## Checklist

- [ ] Key supplied from a build secret / env var, never committed
- [ ] Build script XORs the model and emits a C++ byte-array header
- [ ] Custom build step regenerates the header when model or key changes
- [ ] Only the encrypted header (compiled in) ships -- not the raw model
- [ ] Runtime decrypts into memory and feeds the buffer to the runtime directly
- [ ] No plaintext model ever written to disk
- [ ] Documented internally as obfuscation, not unbreakable protection

## See also

- [ML inference worker subprocess](ml-inference-worker-subprocess.md) -- where
  the embedded, decrypted model is loaded and run.

*Tags: `architecture`, `ml`, `model`, `encryption`, `obfuscation`, `onnx`, `build`, `security`*
