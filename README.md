# mizchi/gfx

Backend-agnostic GPU command-buffer and driver contracts for MoonBit.

Extracted from [mizchi/kagura](https://github.com/mizchi/kagura) so other
MoonBit projects can reuse the rendering primitives without pulling in
the whole game engine.

## At a glance

```
┌─────────────────────────────────────────────────────────┐
│  your renderer                                          │
│  builds DrawTrianglesCommand values                     │
└────────────────┬────────────────────────────────────────┘
                 │ enqueue_draw_triangles
┌────────────────▼────────────────────────────────────────┐
│  CommandQueue (SimpleCommandQueue or your own)          │
│  merges adjacent compatible draws                       │
└────────────────┬────────────────────────────────────────┘
                 │ flush_commands
┌────────────────▼────────────────────────────────────────┐
│  GraphicsDriver (your backend impl)                     │
│  WebGPU / wgpu-native / WebGL2 / offscreen / null       │
└─────────────────────────────────────────────────────────┘
```

Renderers build `DrawTrianglesCommand`s; a `CommandQueue` collects and
optionally merges them; `flush_commands` drives a `GraphicsDriver`
through one render pass. The driver is the only piece that talks to a
real GPU API; the rest is plain data.

## Minimum working example

This uses the bundled `StubGraphicsDriver` (the null backend) so it can
run anywhere `moon test` runs.

```moonbit
test "draw two triangles into a null driver" {
  // 1. Pick a driver. `create_null_graphics` returns a StubGraphicsDriver
  //    that counts begin/end/draw calls but doesn't talk to a GPU.
  let driver = @gfx.create_null_graphics(640, 480)
  driver.initialize()

  // 2. Allocate a render target image and a shader from the driver.
  let dst = driver.new_image(640, 480)
  let shader = driver.new_shader("// dummy WGSL")

  // 3. Build a draw command. Layout is type-erased: vertex_data is a
  //    flat Array[Double], indices are Array[Int].
  let region = @gfx.new_dst_region(0, 0, 640, 480, 6)
  let command = @gfx.new_draw_triangles_command(
    dst,
    shader,
    [region],
    0,
    1,        // pipeline_id (opaque)
    0,        // uniform_hash
    @gfx.BlendMode::Alpha,
    [
      0.0, 0.0, 0.0, 0.0,
      1.0, 0.0, 1.0, 0.0,
      0.0, 1.0, 0.0, 1.0,
      1.0, 1.0, 1.0, 1.0,
    ],
    [0, 1, 2, 1, 3, 2],
    [],       // no source textures
    [],       // no uniforms
  )

  // 4. Enqueue and flush through the driver. The default vertex budget
  //    will keep these two triangles in a single backend draw call.
  let queue = @gfx.new_simple_command_queue()
  queue.enqueue_draw_triangles(command)
  @gfx.flush_commands(driver, queue, false)
}
```

Take the same command, swap the driver for a WebGPU or wgpu-native
impl, and nothing else changes.

## Implementing a backend

A new backend impls `GraphicsDriver` and (if the host wants to negotiate
backends at runtime) `GraphicsBackendFactory`:

```moonbit
struct MyDriver { /* device, queue, swapchain, ... */ }

impl @gfx.GraphicsDriver for MyDriver with initialize(self) {
  // bind a device / swapchain
}

impl @gfx.GraphicsDriver for MyDriver with begin(self, pass) {
  // open a render pass; `pass.clear_color` / `pass.clear_enabled`
}

impl @gfx.GraphicsDriver for MyDriver with end(self, present) {
  // close the pass; swap buffers if `present`
}

impl @gfx.GraphicsDriver for MyDriver with resize(self, w, h) { ... }
impl @gfx.GraphicsDriver for MyDriver with new_image(self, w, h) {
  // allocate a texture and return a handle that wraps your backend id
  @gfx.new_image_handle(/* backend id */ 1, w, h)
}
impl @gfx.GraphicsDriver for MyDriver with new_shader(self, src) { ... }
impl @gfx.GraphicsDriver for MyDriver with draw_triangles(self, cmd) {
  // upload cmd.vertex_data / cmd.indices (or reuse a cached payload
  // when cmd.resource_cache_key != 0) and issue one draw call per
  // cmd.dst_regions entry, respecting cmd.blend, cmd.uniform_dwords, ...
}
impl @gfx.GraphicsDriver for MyDriver with read_pixels(self, x, y, w, h) {
  // None if not supported; otherwise an RGBA8 Array[Int]
}
```

For host shells that want to defer backend choice, also impl
`GraphicsBackendFactory.create(kind, surface, options)` to switch on
`GraphicsBackendKind`.

## What's in here

The package exposes a layered surface that any WebGPU / wgpu-native /
offscreen backend can implement:

- **Command buffer** — `DrawTrianglesCommand`, `DrawCommandDispatch`,
  `DstRegion`, `Color`, `RenderPassDesc`, `dispatch_checksum`,
  `build_draw_command_dispatch`.
- **Handles** — `ImageHandle`, `ShaderHandle`, `PipelineHandle`,
  `FilterMode`.
- **Blend state** — `BlendFactor`, `BlendOperation`, `BlendEquation`,
  `BlendMode`, plus `blend_mode_to_equation`, `blend_mode_to_int`, etc.
- **Driver trait** — `GraphicsDriver` (initialize / begin / end /
  resize / new_image / new_shader / draw_triangles / read_pixels) and
  a `FramebufferSnapshot` / `PixelDiffResult` harness for VRT.
- **Command queue** — `CommandQueue` trait and `SimpleCommandQueue`
  (reference impl with adjacent-batch merging under a 16k-float
  vertex budget) plus `flush_commands` / `clear_screen`.
- **Backend registry** — `StubGraphicsDriver`,
  `create_{webgpu,webgl,wgpu_native,null}_graphics`,
  `NativeGraphicsHooks`, `WebGraphicsHooks`, `GraphicsBackendFactory`,
  `GraphicsBackendKind`, `GraphicsBackendOptions`.
- **Shader plumbing** — `ShaderFrontend`, `UniformCanonicalizer`,
  `BuiltinShaderSourceRepo` traits with `BasicShaderFrontend`,
  `BasicUniformCanonicalizer`, `BasicBuiltinShaderSourceRepo`
  reference implementations. `UniformLayout`, `NamedUniform`,
  `UniformValue`, `PackedUniforms`, `PreservedUniformContext`, plus
  validation helpers and `double_to_f32_bits`.
- **Builtin shader keys** — `BuiltinShaderKey`, `BuiltinShaderKeyEx`,
  `BuiltinShaderFilter`, `BuiltinShaderAddress`, `SamplerSpec`.
- **Surface descriptor** — `SurfaceKind`, `SurfaceToken`,
  `SurfaceProvider` trait, `create_offscreen_surface_token` /
  `create_webgpu_surface_token` / `create_webgl_surface_token`.

Nothing here knows about games, scenes, ECS, assets, or platform
shells; those layers live elsewhere (in kagura, in your engine, ...).

## API tiers

| Tier | Meaning | Examples |
|---|---|---|
| **Stable input** (`pub(all)` struct/enum) | You build these as struct literals | `Color`, `DrawTrianglesCommand`, `DstRegion`, `RenderPassDesc`, blend / filter / uniform enums, `ShaderCompileRequest`, `BuiltinShaderKey`, ... |
| **Stable output** (`pub` struct) | gfx hands these to you; read-only fields, no struct-literal construction | `StubGraphicsDriver`, `Basic{ShaderFrontend,UniformCanonicalizer,BuiltinShaderSourceRepo}`, `BuiltinShaderCacheStats`, `GraphicsResizeStats`, `FramebufferSnapshot`, `PixelDiffResult`, `SimpleCommandQueue`, `{Native,Web}GraphicsHooks` |
| **Backend-return** (`pub(all)` struct) | You build these only from inside a trait impl that returns them | `ImageHandle`, `ShaderHandle`, `PipelineHandle`, `ShaderIR`, `ShaderSourceHash`, `PackedUniforms`, `SurfaceToken` |
| **Open trait** (`pub(open) trait`) | You implement these in your backend / canonicalizer / etc. | `GraphicsDriver`, `CommandQueue`, `ShaderFrontend`, `UniformCanonicalizer`, `BuiltinShaderSourceRepo`, `SurfaceProvider`, `GraphicsBackendFactory` |

For factory-only types there is always a constructor: `new_basic_*`,
`new_*_graphics_hooks{,_full}`, `create_{webgpu,webgl,wgpu_native,null}_graphics`,
`new_simple_command_queue`, etc.

## Layout

```
src/
  handle.mbt              ImageHandle / ShaderHandle / PipelineHandle / FilterMode
  blend.mbt               BlendFactor / Operation / Equation / Mode + conversions
  contracts.mbt           DstRegion / Color / RenderPassDesc / DrawTrianglesCommand /
                          DrawCommandDispatch / dispatch_checksum
  driver.mbt              GraphicsDriver trait + FramebufferSnapshot / PixelDiffResult
  queue.mbt               CommandQueue trait + SimpleCommandQueue + merge logic
  surface.mbt             SurfaceKind / SurfaceToken / SurfaceProvider + factories
  backend_contracts.mbt   StubGraphicsDriver / GraphicsBackendKind / hooks registry
  backend_native_hooks_stub.mbt
  backend_web_hooks_stub.mbt
  shader_contracts.mbt    ShaderIR / ShaderFrontend / Uniform plumbing
```

## Status

- API is **unstable** while the kagura migration settles. Until a 1.0
  line ships, treat the surface as if it could break between any two
  0.x.y versions.
- Tests: 109 whitebox tests covering the command-buffer types, queue
  merge logic, shader IR / uniform packing, and the null driver. Run
  `moon test --target js`.
- Targets: `js` is the primary target; `native` works for the
  contract types (the only target-specific pieces are downstream
  backend impls).

## Using it locally

`moon.mod.json`:

```json
{
  "deps": {
    "mizchi/gfx": { "path": "../gfx-mbt" }
  }
}
```

`moon.pkg`:

```moonbit
import { "mizchi/gfx" @gfx }
```

## License

Apache-2.0. Inherited from kagura.
