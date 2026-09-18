# Changelog

All notable changes to `mizchi/gfx` will be documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project follows [semver](https://semver.org/), but because the
surface is still pre-1.0, **every minor bump may break compatibility**.

## [Unreleased]

## [0.1.1] — 2026-09-18

### Changed

- **BREAKING** — conversion / query helpers are now methods on the
  associated type instead of free functions:

  | Old (free fn) | New (method) |
  |---|---|
  | `blend_mode_to_int(m)` | `BlendMode::to_int(m)` / `m.to_int()` |
  | `blend_mode_from_int(i)` | `BlendMode::from_int(i)` |
  | `blend_mode_to_equation(m)` | `BlendMode::to_equation(m)` |
  | `blend_factor_*` | `BlendFactor::to_int` / `from_int` |
  | `blend_operation_*` | `BlendOperation::to_int` / `from_int` |
  | `filter_mode_*` | `FilterMode::to_int` / `from_int` |
  | `builtin_filter_*` | `BuiltinShaderFilter::to_int` / `from_int` |
  | `builtin_address_*` | `BuiltinShaderAddress::to_int` / `from_int` |
  | `builtin_key_to_ex(k)` | `BuiltinShaderKey::to_ex(k)` |
  | `estimated_draw_call_count(c)` | `DrawTrianglesCommand::estimated_draw_call_count(c)` |
  | `estimated_total_index_count(c)` | `DrawTrianglesCommand::estimated_total_index_count(c)` |
  | `build_draw_command_dispatch(c)` | `DrawTrianglesCommand::build_dispatch(c)` |
  | `dispatch_checksum(d)` | `DrawCommandDispatch::checksum(d)` |
  | `pixel_diff_ratio(r)` | `PixelDiffResult::diff_ratio(r)` |
  | `compare_framebuffer_snapshots(a, b, t)` | `FramebufferSnapshot::compare_with(a, b, t)` |
  | `graphics_resize_stats(d)` | `StubGraphicsDriver::resize_stats(d)` |
  | `builtin_shader_cache_stats(r)` | `BasicBuiltinShaderSourceRepo::cache_stats(r)` |
  | `builtin_shader_source_cache_size(r)` | `BasicBuiltinShaderSourceRepo::cache_size(r)` |
  | `builtin_shader_source_cache_limit(r)` | `BasicBuiltinShaderSourceRepo::cache_limit(r)` |
  | `clear_builtin_shader_source_cache(r)` | `BasicBuiltinShaderSourceRepo::clear_cache(r)` |
  | `validate_uniform_layout(l)` | `UniformLayout::validate(l)` |
  | `validate_shader_compile_request(r)` | `ShaderCompileRequest::validate(r)` |
  | `validate_preserved_uniform_context(c)` | `PreservedUniformContext::validate(c)` |
  | `compute_expected_preserved_dwords(c)` | `PreservedUniformContext::expected_dwords(c)` |
  | `shader_unit_eq(a, b)` | `ShaderUnit::eq(a, b)` / `a.eq(b)` |
  | `shader_hash_eq(a, b)` | `ShaderSourceHash::eq(a, b)` / `a.eq(b)` |

  Migration is a mechanical rename; each helper is callable either as
  a static method (`Type::method(value, ...)`) or as an instance method
  (`value.method(...)`).

- Removed `compile_shader_ir` and `calc_shader_source_hash` free
  functions; call `frontend.compile_ir(request)` and
  `frontend.calc_source_hash(request)` directly via the `ShaderFrontend`
  trait. `build_canonical_uniforms` is kept because it composes three
  trait methods and a private normalization step.

### Added

- `FramebufferSnapshot::from_pixels(x, y, w, h, pixels)` constructor
  for wrapping an externally-captured RGBA8 buffer (golden fixture,
  manual decode, etc.) without going through a `GraphicsDriver`.
- 7 blackbox tests in `src/public_api_test.mbt` exercising the public
  surface as a downstream consumer would see it (null-driver round
  trip, BlendMode int round-trip, build_dispatch / checksum,
  FramebufferSnapshot diff, validation errors, ShaderFrontend hash
  stability). Total tests now 116/116.

### Performance

- Command-buffer hot paths:
  - `DrawTrianglesCommand::build_dispatch` now walks `dst_regions` once
    instead of up to three times (~6% faster on native).
  - `SimpleCommandQueue` merging of explicit-payload commands is two
    passes with pre-sized buffers instead of three.
  - `FramebufferSnapshot::compare_with` hoists the per-pixel bounds
    check out of the loop (~4% faster on native).

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
