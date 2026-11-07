# Proposal

## Why

On Linux, oxgl 0.3.0 reaches ANGLE's OpenGL ES backend only by way of a working Vulkan display that offers OpenGL ES 2.0 only. A Raspberry Pi 5 therefore needs Mesa's Vulkan driver (`mesa-vulkan-drivers`) installed just so the probe can fail, even though WebGL 1 and WebGL 2 both render through Mesa's EGL on V3D. Without it, `'gpu'` throws and `'auto'` drops to SwiftShader. Users need a way to name the GPU backend themselves.

## What Changes

- A new environment variable, `OXGL_GPU_BACKEND`, selects which ANGLE display serves GPU requests on Linux: `vulkan` or `gles`. Unset or empty keeps today's automatic choice.
- With `gles`, every GPU request (`'gpu'`, and the GPU attempt of `'auto'`) goes straight to ANGLE's OpenGL ES backend on the system's EGL driver. No Vulkan display is opened for it, so no system Vulkan driver is needed. The CPU-renderer check still applies, so llvmpipe is refused. When the OpenGL ES backend is unavailable or refused, `'auto'` throws `OXGL_GPU_UNAVAILABLE` like `'gpu'` does, instead of falling back to SwiftShader. `backend: 'software'` still uses SwiftShader.
- With `vulkan`, GPU requests use only Vulkan and the system EGL library is never opened. A Vulkan display offering OpenGL ES 2.0 only then fails the request instead of diverting. As with `gles`, `'auto'` throws `OXGL_GPU_UNAVAILABLE` on any failure instead of falling back to SwiftShader.
- The variable is read from the calling thread's `process.env` on every `createContext` call, like `OXGL_BACKEND`, and is validated on every platform. It has no effect outside Linux. An invalid value throws `OXGL_INVALID_BACKEND`.
- A context records the GPU display it was created on, so a restore after context loss reopens the same one.
- The `backend` option keeps its three values. The test-only `OXGL_BACKEND=gles` mode stays as it is and takes precedence over the new variable.
- The rule of one Vulkan device per process still applies, so with `vulkan` set, `'auto'` throws after a `'software'` context instead of falling back.
- README: the backend section, the errors table and the Pi prerequisites describe the variable, and the Pi instructions drop `mesa-vulkan-drivers` when it is set to `gles`.

### Out of scope

- Trying the OpenGL ES backend automatically when no Vulkan driver is installed. That would make the Pi work with no configuration, but it adds detection machinery the explicit variable makes unnecessary. It stays an extension point: one more rule in the GPU candidate list, if wanted later.
- A per-context option for the GPU backend. The variable is process configuration, like `OXGL_BACKEND`; an option can be added later without changing this behaviour.
- Selecting the GPU backend on macOS or Windows, which each have one.
- CI on a Pi. The Pi has no runner; its results are recorded in design.md.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `backend-selection`: adds the `OXGL_GPU_BACKEND` override and its precedence under the test-only mode; makes the automatic Vulkan-to-OpenGL-ES rule apply only when the override is unset; and carves the override out of `'auto'`'s fallback, the GPU unavailable error and the one-Vulkan-device rule.

## Impact

- `src/context.ts`: read and validate `OXGL_GPU_BACKEND`, pass it to the native side, record it with the context's creation parameters for restore.
- `crates/oxgl-node`: a new `createContext` argument; native ABI version 15 → 16.
- `crates/oxgl-core/src/backend.rs`: the GPU attempt takes the chosen display (automatic, Vulkan or OpenGL ES).
- Tests in `oxgl-core` (selection with a fake probe) and Node tests for validation and per-call reading.
- README.
