# Tasks

## 1. Core selection

- [x] 1.1 Add `GpuDisplay` (`Auto`, `Vulkan`, `Gles`) to `oxgl-core` and carry it in `ContextRequest` to `try_gpu`; verify `cargo build` succeeds on macOS
- [x] 1.2 Implement the `Vulkan` and `Gles` arms in `try_gpu` (Linux only; ignored elsewhere); verify with new fake-probe tests in `backend.rs`: `Gles` never opens the GPU slot, refuses a CPU renderer, makes `'auto'` fail with `GpuUnavailable` without trying SwiftShader, and lets `'software'` open afterwards; `Vulkan` never opens the `gles` slot, fails on an ES 2.0-only display, and makes `'auto'` fail with `GpuUnavailable` without trying SwiftShader when the Vulkan display fails, is a CPU, offers ES 2.0 only, or is refused because SwiftShader holds the Vulkan device
- [x] 1.3 Verify `cargo test -p oxgl-core` and `cargo clippy --all-targets --all-features -- -D warnings` (pedantic) pass

## 2. Node binding

- [x] 2.1 Add the `gpu_backend` argument to `create_context` in `oxgl-node`, mapping `vulkan`/`gles` and anything else to `Auto`; bump the native ABI version to 16 in `lib.rs` and `src/load-native.ts`; verify `cargo clippy` passes
- [x] 2.2 Add `resolveGpuBackend()` in `src/context.ts`, read per call, throwing `OXGL_INVALID_BACKEND` naming `OXGL_GPU_BACKEND`; store the result with the creation parameters used for restore; verify with Node tests for an invalid value, per-call reading, a worker setting it, and no effect on macOS, plus a Linux-only test that runs in CI: create an `'auto'` context with the variable unset (SwiftShader), set `OXGL_GPU_BACKEND=gles`, lose and restore it, and check it is restored on SwiftShader rather than lost; the test skips when `OXGL_BACKEND` is set, since under `software` or test-only `gles` it would pass whatever restore does
- [x] 2.3 Make the existing Node test "a Linux process uses one Vulkan device" skip when `OXGL_GPU_BACKEND` is set, since it covers the automatic choice and its child processes inherit the variable (with `gles` the GPU request opens no Vulkan display, so `'gpu'` after `'software'` renders instead of throwing); verify it skips with the variable set and still runs with it unset
- [x] 2.4 Verify `npm test` and `npm run lint` pass on macOS

## 3. Documentation

- [x] 3.1 Document `OXGL_GPU_BACKEND` in the README's backend section and errors table, and change the Pi prerequisites to list `mesa-vulkan-drivers` only for the unset case; verify the documented commands match the variable's accepted values

## 4. Linux verification

- [x] 4.1 In a `linux/arm64` container with Mesa's EGL and no Vulkan driver, verify `OXGL_GPU_BACKEND=gles` with `backend: 'gpu'` throws `OXGL_GPU_UNAVAILABLE` naming only `OpenGL ES` and llvmpipe, `'auto'` throws the same error without initializing SwiftShader, `'software'` still renders, and `npm test` passes with the variable unset. (With it set, `'auto'` throws there by design, so the suite is not run that way.)
- [x] 4.2 In the same container with `mesa-vulkan-drivers` installed, verify `OXGL_GPU_BACKEND=vulkan` with `'auto'` throws `OXGL_GPU_UNAVAILABLE` naming `Vulkan` and lavapipe without initializing SwiftShader, and, in a fresh process that creates a `'software'` context first, that `'auto'` then throws naming SwiftShader in use. lavapipe stands in here for the Vulkan GPU of the scenario "Auto after a software context with the Vulkan override"; the fake-probe test in 1.2 covers the hardware case
- [x] 4.3 On the Raspberry Pi 5 with its Vulkan drivers hidden from the loader (`VK_ICD_FILENAMES` pointing at no file, which leaves the package installed) and V3DV confirmed never loaded, verify `OXGL_GPU_BACKEND=gles` gives WebGL 1 and WebGL 2 `'gpu'` contexts on V3D, `'software'` works afterwards in the same process, a lost-and-restored context stays on V3D, and `npm test` passes with the variable set; record the results in design.md
- [x] 4.4 On the Pi with its Vulkan drivers visible, verify `OXGL_GPU_BACKEND=vulkan` WebGL 2 `'gpu'` and `'auto'` both throw `OXGL_GPU_UNAVAILABLE` naming "offers OpenGL ES 2.0 only" without opening `libEGL.so.1`, and the unset case still renders on V3D; record the results in design.md
