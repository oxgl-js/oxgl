# Design

## Context

GPU selection on Linux lives in `Backends::try_gpu` (`crates/oxgl-core/src/backend.rs`). It opens the Vulkan GPU display and, only when that display is a hardware device offering OpenGL ES 2.0 only, diverts to the `gles` slot through `gles_gpu_display`, which applies the CPU-renderer check. The `gles` slot, `DisplayKind::Gles` and the OpenGL ES messages already exist from `linux-gles-backend`. `OXGL_BACKEND` is resolved in `src/context.ts` (`resolveBackend`) and passed to `create_context` in `oxgl-node` as a string; the context's creation parameters are kept for restore.

## Goals / Non-Goals

**Goals:**
- Name the Linux GPU display through configuration, so the OpenGL ES backend works with no Vulkan driver installed.
- Reuse the existing slots, CPU check, failure memory and messages, adding no new display kind.

**Non-Goals:**
- Changing the automatic rule when the variable is unset.
- Changing the test-only `OXGL_BACKEND=gles` mode, which still accepts a CPU renderer for CI on llvmpipe.

## Decisions

**A separate variable, not a fourth `backend` value.** `backend` says *whether* to use a GPU (`'auto'`, `'gpu'`, `'software'`); `OXGL_GPU_BACKEND` says *which* GPU display on Linux. Keeping them orthogonal leaves the TypeScript types and every caller unchanged, and lets a Pi deployment set one variable without touching application code or three.js setups. The alternative, a public `backend: 'gles'`, would be Linux-only, would need its own fallback semantics, and would make applications choose a platform-specific value.

**An explicit GPU backend never falls back.** Naming `gles` or `vulkan` states that SwiftShader is not acceptable for GPU requests, so a silent drop to a CPU renderer would hide a broken setup: Mesa or the Vulkan driver missing, or the user outside `render`. `'auto'` therefore throws `OXGL_GPU_UNAVAILABLE` as `'gpu'` does. `backend: 'software'` is an explicit request for SwiftShader and is unaffected.

**The test-only mode wins.** `OXGL_BACKEND=gles` in a `test-gles` build bypasses GPU selection, so `OXGL_GPU_BACKEND` has no effect on selection there. It is still validated, so a typo is caught on every leg. CI's `gles` legs keep their meaning whatever the variable says.

**Values `vulkan` and `gles`.** `gles` is what the Pi needs. `vulkan` costs one match arm and gives the opposite guarantee: Mesa is never loaded into the process. Unset or empty is the automatic choice.

**Validated everywhere, effective only on Linux.** A typo is caught on every platform, while one deployment configuration can be shared between a Pi and a macOS development machine. macOS and Windows have one GPU display each, so there is nothing to choose there.

**Core shape.** `ContextRequest` gains a `gpu: GpuDisplay` field (`Auto`, `Vulkan`, `Gles`), so it reaches `try_gpu` through the existing `Runtime::create_context` and `Backends::create` signatures. `try_gpu` matches on it:
- `Auto`: today's code.
- `Vulkan`: open the Vulkan GPU display; an ES 2.0-only display fails with its existing cause and does not divert. As for `Gles`, `create` treats `'auto'` like `'gpu'`. The one-Vulkan-device rule still applies, so `'software'` after a Vulkan GPU context is refused as today.
- `Gles` (Linux): call `gles_gpu_display` directly, with the error prefixed `OpenGL ES`. `create` then treats `'auto'` like `'gpu'`: the failure is `CoreError::GpuUnavailable` and SwiftShader is not tried. The Vulkan GPU slot is never touched, so the one-Vulkan-device rule does not arise and `'software'` works in the same process (the OpenGL ES display shares no device with SwiftShader).

Off Linux the argument is ignored. Slots and their failure memory are unchanged: each display still opens at most once per process, so changing the variable between calls switches between already-cached displays, as `OXGL_BACKEND` does.

**Node binding.** `resolveGpuBackend()` in `src/context.ts` reads `process.env.OXGL_GPU_BACKEND` on each call, next to `resolveBackend`, and throws `OXGL_INVALID_BACKEND` for other values. The resolved string is a new `create_context` argument and is stored with the creation parameters, so a restore reopens the same display. The native ABI version goes from 15 to 16.

**Safety rules.** No `unsafe` is added: the change is selection logic in `oxgl-core` and argument plumbing in `oxgl-node`. An unknown string from JS maps to `Auto` with no panic (TS has already validated it). No handles, pointers or `ArrayBuffer`s are involved.

**Alternative considered: try the OpenGL ES backend when Vulkan fails to open.** Zero configuration on the Pi. Rejected because it adds machinery to auto-detect a backend that one variable lets the user name: another fallback rule in `try_gpu`, terminating a refused OpenGL ES display so llvmpipe's threads don't stay for the process, and the tests and messages for both. Runtime cost is not the reason. It is paid once per process and is small: in a `linux/arm64` Ubuntu 24.04 container, opening and refusing llvmpipe took about 17 ms and kept about 20 MB of private memory. It remains possible later as another rule in `try_gpu`, and the variable would still serve to opt out.

## Risks / Trade-offs

- [The Pi needs a variable set to avoid the Vulkan driver] → Documented in the README's Pi prerequisites; the automatic path still works for users who install `mesa-vulkan-drivers`.
- [With `vulkan` set, a `'software'` context makes every later `'auto'` call in the process throw, since the one-Vulkan-device rule refuses the GPU and there is no fallback] → Intended: an explicit backend never falls back. Specified under "One Vulkan device per process on Linux"; a process that needs both leaves the variable unset.
- [A user sets `gles` on a machine whose Vulkan GPU offers ES 3.0] → They get Mesa's GL driver instead of Vulkan. That is what they asked for; on a CPU renderer it is refused as usual.

## Results in a Linux arm64 container

Ubuntu 24.04 `linux/arm64` container, Mesa 25.2.8, Node 24.11.1. Each check ran in a fresh process; "SwiftShader not initialized" and "system EGL not opened" were read from `/proc/self/maps`.

- Task 4.1, Mesa's EGL and GL drivers, no Vulkan driver. With `OXGL_GPU_BACKEND=gles`, WebGL 1 and WebGL 2 `'gpu'` and `'auto'` all throw `OXGL_GPU_UNAVAILABLE`: "OpenGL ES: the GPU backend renders on a CPU (ANGLE (Mesa, llvmpipe (LLVM 20.1.2 128 bits), OpenGL ES 3.2))", naming no Vulkan. `'auto'` leaves SwiftShader unloaded. `'software'` renders on SwiftShader. With the variable unset, `'auto'` renders on SwiftShader and never opens the system EGL. `npm test` with the variable unset: 445 pass, 14 skipped, 1 fail. The failure, the conformance harness's page-runner test ("play rejected: NotSupportedError"), needs ffmpeg, which CI installs and the container lacked; `main` fails it identically there (440 pass, 12 skipped, 1 fail).
- Task 4.2, with `mesa-vulkan-drivers` (lavapipe) added. With `OXGL_GPU_BACKEND=vulkan`, WebGL 2 `'auto'` throws `OXGL_GPU_UNAVAILABLE`: "Vulkan: the GPU backend renders on a CPU (ANGLE (Mesa, Vulkan 1.4.318 (llvmpipe (LLVM 20.1.2 128 bits) (0x00000000)), llvmpipe))", with SwiftShader unloaded and the system EGL never opened. In a fresh process, `'software'` renders on SwiftShader and a following `'auto'` throws `OXGL_GPU_UNAVAILABLE`: "Vulkan: SwiftShader is in use, and a Linux process uses one Vulkan device".

## Verification on the Pi

Raspberry Pi 5, Mesa 26.2.2, Node 26.7.0, user `baton` in the `render` group, with the addon built in the container above. `mesa-vulkan-drivers` stayed installed. Its drivers were hidden from the Vulkan loader with `VK_ICD_FILENAMES` pointing at no file, which ANGLE overrides only while SwiftShader initializes. With the variable unset, that makes WebGL 2 `'gpu'` throw "Vulkan: eglInitialize failed (EGL error 0x3001)", the same failure as with no Vulkan driver installed. Whether V3DV (`libvulkan_broadcom`), the system EGL and SwiftShader were loaded was read from `/proc/self/maps`. (`VK_DRIVER_FILES` was tried first and hides SwiftShader's ICD too, since the loader prefers it over the `VK_ICD_FILENAMES` ANGLE sets.)

- Task 4.3, `OXGL_GPU_BACKEND=gles`, with the drivers visible and hidden alike: WebGL 1 and WebGL 2 `'gpu'`, and WebGL 2 `'auto'`, render on `ANGLE (Broadcom, V3D 7.1.10.2, OpenGL ES 3.1)`, and a `'software'` context in the same process renders on SwiftShader. V3DV is never loaded. A WebGL 2 context created with the variable set, then lost and restored after the variable is unset, comes back unlost on V3D, with V3DV still unloaded. A worker that sets the variable itself gets V3D. `npm test` with the variable set and the drivers hidden: 449 pass, 0 fail, 11 skipped, each skip for this host or a `test-gles` build, including "a Linux process uses one Vulkan device" (task 2.3).
- Task 4.4, `OXGL_GPU_BACKEND=vulkan`, drivers visible: WebGL 2 `'gpu'` and `'auto'` both throw `OXGL_GPU_UNAVAILABLE`: "Vulkan: offers OpenGL ES 2.0 only", naming no OpenGL ES attempt, with the system EGL and SwiftShader never loaded. The rejected Vulkan display is terminated, which unloads V3DV again. With the variable unset, WebGL 2 `'gpu'` and WebGL 1 `'auto'` render on V3D through the OpenGL ES backend, as in 0.3.0.
