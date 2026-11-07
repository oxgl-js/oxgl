# Spec Delta

## ADDED Requirements

### Requirement: OXGL_GPU_BACKEND environment override
On Linux, the `OXGL_GPU_BACKEND` environment variable SHALL choose the display that serves a GPU request, from `backend: 'gpu'` or the GPU attempt of `'auto'`: `gles` uses only ANGLE's OpenGL ES backend on the system's EGL driver and opens no Vulkan GPU display, and `vulkan` uses only Vulkan and never opens the system EGL library. The CPU-renderer check applies to either. Unset or empty keeps the automatic choice.

#### Scenario: WebGL 2 on the Pi without a Vulkan driver
- **WHEN** `OXGL_GPU_BACKEND=gles` and `createContext({ width: 64, height: 64, webgl2: true, backend: 'gpu' })` runs on a Raspberry Pi 5 with Mesa's EGL and V3D GL driver, no Mesa Vulkan driver installed, and the user in the `render` group
- **THEN** a WebGL 2 context is returned whose `GL_RENDERER` names V3D and names neither Vulkan nor llvmpipe

#### Scenario: WebGL 1 on the Pi without a Vulkan driver
- **WHEN** `OXGL_GPU_BACKEND=gles` and `createContext({ width: 4, height: 4, backend: 'gpu' })` runs on the same machine
- **THEN** the context's `GL_RENDERER` names V3D and does not name Vulkan

#### Scenario: OpenGL ES backend on a CPU renderer
- **WHEN** `OXGL_GPU_BACKEND=gles` and a context is created with `backend: 'gpu'` on a Linux machine whose EGL driver renders on llvmpipe
- **THEN** `OXGL_GPU_UNAVAILABLE` is thrown whose message names `OpenGL ES` and the refused renderer and does not name `Vulkan`

#### Scenario: Software alongside the OpenGL ES backend
- **WHEN** `OXGL_GPU_BACKEND=gles`, a context has been created with `backend: 'gpu'` on the Pi, and `createContext({ width: 4, height: 4, backend: 'software' })` is called
- **THEN** a SwiftShader context is returned, and the first context still renders on V3D

#### Scenario: Vulkan only on a GPU whose Vulkan driver gives OpenGL ES 2.0 only
- **WHEN** `OXGL_GPU_BACKEND=vulkan` and `createContext({ width: 4, height: 4, webgl2: true, backend: 'gpu' })` runs on the Pi with Mesa's Vulkan driver
- **THEN** `OXGL_GPU_UNAVAILABLE` is thrown whose message names `Vulkan` with the cause "offers OpenGL ES 2.0 only", and the system EGL library is never opened

#### Scenario: Software request ignores the GPU override
- **WHEN** `OXGL_GPU_BACKEND=gles` and `createContext({ width: 4, height: 4, backend: 'software' })` is called on a machine without Mesa
- **THEN** a SwiftShader context is returned

### Requirement: No fallback from an explicit GPU backend
When `OXGL_GPU_BACKEND` is `gles` or `vulkan` on Linux, `backend: 'auto'` SHALL NOT fall back to SwiftShader. A failure to open the chosen display, a refused CPU renderer, a Vulkan display offering OpenGL ES 2.0 only, or a failed context creation SHALL throw an `Error` with `code` `'OXGL_GPU_UNAVAILABLE'`, as for `backend: 'gpu'`. `backend: 'software'` SHALL still create the context on SwiftShader.

#### Scenario: No fallback from the OpenGL ES backend
- **WHEN** `OXGL_GPU_BACKEND=gles` and `createContext({ width: 4, height: 4, backend: 'auto' })` runs on a Linux machine without Mesa's EGL, or whose EGL driver renders on llvmpipe
- **THEN** an `Error` with `code === 'OXGL_GPU_UNAVAILABLE'` is thrown whose message names `OpenGL ES` and its cause, no context is returned, and SwiftShader is not initialized

#### Scenario: No fallback from the Vulkan backend
- **WHEN** `OXGL_GPU_BACKEND=vulkan` and `createContext({ width: 4, height: 4, backend: 'auto' })` runs on a Linux machine with no Vulkan driver, or whose only Vulkan device is Mesa's lavapipe
- **THEN** an `Error` with `code === 'OXGL_GPU_UNAVAILABLE'` is thrown whose message names `Vulkan` and its cause, no context is returned, and SwiftShader is not initialized

#### Scenario: No fallback from a Vulkan display offering OpenGL ES 2.0 only
- **WHEN** `OXGL_GPU_BACKEND=vulkan` and `createContext({ width: 4, height: 4, webgl2: true, backend: 'auto' })` runs on the Pi with Mesa's Vulkan driver
- **THEN** an `Error` with `code === 'OXGL_GPU_UNAVAILABLE'` is thrown whose message names `Vulkan` with the cause "offers OpenGL ES 2.0 only", and SwiftShader is not initialized

### Requirement: OXGL_GPU_BACKEND is read per call and validated
`OXGL_GPU_BACKEND` SHALL be read from the calling thread's `process.env` on every `createContext` call. On every platform, a value other than `vulkan`, `gles` or empty SHALL make `createContext` throw an `Error` with `code` `'OXGL_INVALID_BACKEND'` naming the variable and the value. Outside Linux a valid value SHALL have no effect. A context restored after context loss SHALL use the GPU display it was created on.

#### Scenario: Invalid GPU override value
- **WHEN** `OXGL_GPU_BACKEND=opengl` and `createContext` is called
- **THEN** an `Error` with `code === 'OXGL_INVALID_BACKEND'` is thrown, mentioning `OXGL_GPU_BACKEND` and `opengl`

#### Scenario: GPU override set inside a worker
- **WHEN** a worker thread on the Pi sets `process.env.OXGL_GPU_BACKEND = 'gles'` and calls `createContext({ width: 4, height: 4, webgl2: true, backend: 'gpu' })` with no Mesa Vulkan driver installed
- **THEN** the context's `GL_RENDERER` names V3D

#### Scenario: GPU override outside Linux
- **WHEN** `OXGL_GPU_BACKEND=gles` and `createContext({ width: 4, height: 4, backend: 'gpu' })` is called on macOS
- **THEN** the context renders through Metal

#### Scenario: Restore keeps the GPU display
- **WHEN** a context created with `OXGL_GPU_BACKEND=gles` on the Pi is lost, `OXGL_GPU_BACKEND` is then unset, and the context is restored through `WEBGL_lose_context.restoreContext()`
- **THEN** the restored context's `GL_RENDERER` names V3D and no Vulkan display is opened

### Requirement: Test-only OpenGL ES mode takes precedence
In an addon built with `test-gles`, `OXGL_BACKEND=gles` SHALL take precedence over `OXGL_GPU_BACKEND`, which SHALL then have no effect on selection. An invalid `OXGL_GPU_BACKEND` value SHALL still throw `OXGL_INVALID_BACKEND`.

#### Scenario: Test-only mode with the Vulkan override
- **WHEN** an addon built with `test-gles` runs on Linux with `OXGL_BACKEND=gles` and `OXGL_GPU_BACKEND=vulkan`, and `createContext({ width: 4, height: 4 })` is called
- **THEN** the context is created on ANGLE's OpenGL ES backend, as with `OXGL_BACKEND=gles` alone

#### Scenario: Invalid GPU override under the test-only mode
- **WHEN** an addon built with `test-gles` runs with `OXGL_BACKEND=gles` and `OXGL_GPU_BACKEND=opengl`, and `createContext({ width: 4, height: 4 })` is called
- **THEN** an `Error` with `code === 'OXGL_INVALID_BACKEND'` is thrown, mentioning `OXGL_GPU_BACKEND` and `opengl`

## MODIFIED Requirements

### Requirement: OpenGL ES backend on Linux
On Linux, when `OXGL_GPU_BACKEND` is unset or empty, a request SHALL use ANGLE's OpenGL ES backend on the system's EGL driver, through EGL's surfaceless platform (`EGL_MESA_platform_surfaceless`), when the Vulkan display opened, was not rejected as a CPU, and offers no OpenGL ES 3.0 config. The Vulkan display SHALL then serve no context. Whether it offers OpenGL ES 3.0 SHALL be determined once, when it is opened.

#### Scenario: WebGL 2 on a GPU whose Vulkan driver gives OpenGL ES 2.0 only
- **WHEN** `createContext({ width: 64, height: 64, webgl2: true, backend: 'gpu', attributes: { antialias: false } })` runs on a Raspberry Pi 5 with Mesa's V3D GL driver installed and the user in the `render` group
- **THEN** a WebGL 2 context is returned whose `GL_RENDERER` names V3D and names neither Vulkan nor llvmpipe, and `RED_BITS`, `GREEN_BITS`, `BLUE_BITS` and `ALPHA_BITS` are each `8`

#### Scenario: Antialiased WebGL 2 through the OpenGL ES backend
- **WHEN** the same call is made with `antialias: true`
- **THEN** `getParameter(SAMPLES)` is `4` and `getContextAttributes().antialias` is `true`

#### Scenario: WebGL 1 on a GPU whose Vulkan driver gives OpenGL ES 2.0 only
- **WHEN** `createContext({ width: 4, height: 4, backend: 'gpu' })` runs on the same machine
- **THEN** the context's `GL_RENDERER` names V3D and does not name Vulkan

#### Scenario: WebGL 1 without the OpenGL ES backend
- **WHEN** the same call runs on the same machine with `OXGL_GPU_BACKEND` unset, without Mesa's EGL or with Mesa's GL driver falling back to llvmpipe
- **THEN** `OXGL_GPU_UNAVAILABLE` is thrown, and with `backend: 'auto'` the context's `GL_RENDERER` names SwiftShader

#### Scenario: Sampler array indexed by a loop variable
- **WHEN** a WebGL 1 fragment shader on the same machine with Mesa's EGL samples `uniform sampler2D uni[2]` as `uni[i]` in a `for` loop over `i`
- **THEN** it renders the sampled colors and the process is unaffected (cf. Khronos `conformance/glsl/bugs/sampler-array-using-loop-index.html`)

#### Scenario: Vulkan with OpenGL ES 3.0 keeps Vulkan
- **WHEN** a WebGL 2 context is created with `backend: 'gpu'` on a Linux machine whose Vulkan display offers an OpenGL ES 3.0 config
- **THEN** the context renders through Vulkan and the system EGL library is never opened

### Requirement: Automatic selection with fallback
`backend: 'auto'` (the default) SHALL first attempt the platform GPU backend and, if display initialization or context creation fails for any reason, SHALL create the context on SwiftShader instead, except when `OXGL_GPU_BACKEND` names a GPU backend on Linux ("No fallback from an explicit GPU backend"). A GPU failure that is followed by a successful fallback SHALL NOT be reported as an error.

#### Scenario: No GPU available
- **WHEN** `createContext({ width: 4, height: 4 })` runs on a machine with no usable GPU with `OXGL_GPU_BACKEND` unset
- **THEN** a context is returned that renders with SwiftShader, and `getError()` returns `NO_ERROR`

#### Scenario: GPU available
- **WHEN** `backend: 'auto'` is used on a machine with a usable GPU
- **THEN** the context renders through the GPU backend

### Requirement: GPU unavailable error
When `backend: 'gpu'` is requested (by option or override), or `backend: 'auto'` with `OXGL_GPU_BACKEND` naming a GPU backend on Linux, and no GPU backend can create the context, `createContext` SHALL throw an `Error` with `code` `'OXGL_GPU_UNAVAILABLE'` whose message names each GPU backend attempted (`Vulkan`, `OpenGL ES`, `Metal` or `D3D11`) and the cause of each failure. It SHALL NOT fall back to software.

#### Scenario: Forced GPU without a GPU
- **WHEN** `backend: 'gpu'` is used on a machine with no usable GPU
- **THEN** an `Error` with `code === 'OXGL_GPU_UNAVAILABLE'` is thrown

#### Scenario: Both Linux GPU backends named
- **WHEN** a WebGL 2 request with `backend: 'gpu'` on a Linux machine reaches the OpenGL ES backend and it fails, for example because `libEGL.so.1` is not installed
- **THEN** the message names `Vulkan` with the cause "offers OpenGL ES 2.0 only" and `OpenGL ES` with its own cause

### Requirement: One Vulkan device per process on Linux
On Linux, where the Vulkan GPU and SwiftShader share one ANGLE display, the first of them to initialize on an accepted device SHALL hold it for the process. A later request for the other SHALL fail with a message naming the backend in use, not render on the held device. `'auto'` SHALL treat a refused GPU as unavailable, which with `OXGL_GPU_BACKEND=vulkan` makes it throw `OXGL_GPU_UNAVAILABLE`.

Accepted trade-off: a Linux process cannot render on a Vulkan GPU and on SwiftShader at once.

#### Scenario: Software after a Vulkan GPU context
- **WHEN** a context has been created with `backend: 'gpu'` on a Linux machine whose Vulkan GPU offers OpenGL ES 3.0, and `createContext({ width: 4, height: 4, backend: 'software' })` is called
- **THEN** an `Error` with `code === 'OXGL_CONTEXT_CREATION_FAILED'` is thrown whose message names the Vulkan GPU in use, and no context is returned

#### Scenario: GPU after a software context
- **WHEN** a context has been created with `backend: 'software'` on a Linux machine with a Vulkan GPU with `OXGL_GPU_BACKEND` unset, and `createContext({ width: 4, height: 4, backend: 'gpu' })` is called
- **THEN** an `Error` with `code === 'OXGL_GPU_UNAVAILABLE'` is thrown whose message names SwiftShader in use, and `backend: 'auto'` returns a SwiftShader context

#### Scenario: WebGL 2 without Mesa's EGL on the Pi
- **WHEN** Mesa's EGL is not installed on the Pi and `createContext({ width: 4, height: 4, webgl2: true })` is called with `OXGL_GPU_BACKEND` unset
- **THEN** the context's `GL_RENDERER` names SwiftShader

#### Scenario: Auto after a software context with the Vulkan override
- **WHEN** `OXGL_GPU_BACKEND=vulkan`, a context has been created with `backend: 'software'` on a Linux machine with a Vulkan GPU, and `createContext({ width: 4, height: 4, backend: 'auto' })` is called
- **THEN** an `Error` with `code === 'OXGL_GPU_UNAVAILABLE'` is thrown whose message names SwiftShader in use, and no context is returned
