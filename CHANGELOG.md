# Changelog

All notable changes to `mizchi/gfx` will be documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project follows [semver](https://semver.org/), but because the
surface is still pre-1.0, **every minor bump may break compatibility**.

## [Unreleased]

## [0.1.0] — initial extraction

Extracted from `mizchi/kagura` at the point it was `modules/kagura_engine/src/gfx/`.

### Added

- `GraphicsDriver`, `CommandQueue`, `ShaderFrontend`,
  `UniformCanonicalizer`, `BuiltinShaderSourceRepo`, `SurfaceProvider`,
  `GraphicsBackendFactory` traits, all `pub(open)` so downstream code
  can implement them.
- `DrawTrianglesCommand`, `DrawCommandDispatch`, `DstRegion`, `Color`,
  `RenderPassDesc`, `BlendMode` / `BlendFactor` / `BlendOperation` /
  `BlendEquation`, `FilterMode`, `ImageHandle`, `ShaderHandle`,
  `PipelineHandle` — the core type-erased command-buffer surface.
- `SimpleCommandQueue` (reference impl) with adjacent batch-merge under
  a 16384-float vertex budget.
- `StubGraphicsDriver` family: `create_webgpu_graphics`,
  `create_webgl_graphics`, `create_wgpu_native_graphics`,
  `create_null_graphics`.
- `NativeGraphicsHooks` / `WebGraphicsHooks` registry with
  `set_*_graphics_hooks` / `reset_*_graphics_hooks` for runtime
  backend plugging.
- `ShaderIR`, `ShaderSourceHash`, `ShaderCompileRequest`,
  `ShaderEntrypoints`, `ShaderUnit`, `UniformValue`, `UniformLayout`,
  `NamedUniform`, `PackedUniforms`, `PreservedUniformContext`,
  `IntSize`, `FloatRect`, `SamplerSpec`, `BuiltinShaderKey` / `*KeyEx`,
  `BuiltinShaderFilter` / `BuiltinShaderAddress`.
- `BasicShaderFrontend`, `BasicUniformCanonicalizer`,
  `BasicBuiltinShaderSourceRepo` — reference implementations of the
  shader plumbing traits.
- `FramebufferSnapshot` + `PixelDiffResult` + the
  `create_framebuffer_snapshot` / `compare_framebuffer_snapshots` /
  `pixel_diff_ratio` helpers for visual regression testing on top of
  `GraphicsDriver.read_pixels`.
- `SurfaceKind`, `SurfaceToken`, `create_offscreen_surface_token`,
  `create_webgpu_surface_token`, `create_webgl_surface_token`. These
  moved here from kagura's `platform` package so gfx no longer depends
  on it.
- 109 whitebox tests covering the command-buffer types, queue merge
  logic, shader IR / uniform packing, and the null driver.

### Changed vs. the in-kagura version

- `pub trait` → `pub(open) trait` for all 7 traits, so external packages
  can implement them.
- 11 implementation-detail types demoted from `pub(all)` to `pub`:
  `StubGraphicsDriver`, `Basic{ShaderFrontend,UniformCanonicalizer,BuiltinShaderSourceRepo}`,
  `{Native,Web}GraphicsHooks`, `BuiltinShaderCacheStats`,
  `GraphicsResizeStats`, `FramebufferSnapshot`, `PixelDiffResult`,
  `SimpleCommandQueue`. Callers must use the matching factory function
  to construct them.
- Data-input types stay `pub(all)` so callers can compose them as
  struct literals (`Color`, `DrawTrianglesCommand`, `DstRegion`, the
  blend / filter / uniform enums and structs, `ShaderCompileRequest`,
  ...).
- Backend-output types stay `pub(all)` so external trait impls can
  return them: `ImageHandle`, `ShaderHandle`, `PipelineHandle`,
  `ShaderIR`, `ShaderSourceHash`, `PackedUniforms`, `SurfaceToken`.
- Added `derive(Debug)` to `DrawTrianglesCommand`, `SimpleCommandQueue`,
  `StubGraphicsDriver`, `Basic{ShaderFrontend,UniformCanonicalizer,BuiltinShaderSourceRepo}`,
  and the private `BuiltinShaderSourceCacheEntry` / `*EntryEx`.
- Documented the 7 public traits inline and added field-level docs to
  `DrawTrianglesCommand`, `Color`, `RenderPassDesc`, `DstRegion`,
  `FilterMode`, and the three handle types.
- `dispatch_checksum` is now `pub` in gfx (previously duplicated as
  internal helpers in kagura's sprite2d / text bench files).

### Removed

- The dependency on `mizchi/kagura_engine/platform`. `moon.pkg` only
  imports `moonbitlang/core/cmp`.
