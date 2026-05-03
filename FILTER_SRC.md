# Filter Source

This document defines the `filter-src` source project format.

`filter-src` is the editable authoring form of a filter project. A compiler consumes a `filter-src` project and produces a runtime-consumable `filter` package.

## Required Root Files

A `filter-src` project contains two required root files at the project root:

```text
filter-src/
  manifest.json
  main.lua
```

### `manifest.json`

`manifest.json` is required and must be located at the project root.

It defines:

- the runtime contract version
- the output size decision mode
- optional project metadata
- public parameters
- pass source files
- runtime asset files

### `main.lua`

`main.lua` is required and must be located at the project root.

It defines the Lua workflow entry points for the filter.

## Manifest Structure

The manifest root object has these fields:

- `$schema`
- `schemaVersion`
- `runtimeVersion`
- `outputSizeMode`
- `metadata`
- `parameters`
- `passes`
- `assets`

`schemaVersion`, `runtimeVersion`, and `passes` are required.

`$schema` is optional.

`outputSizeMode` defaults to `passive` when omitted.

`metadata`, `parameters`, and `assets` are optional.

## `$schema`

`$schema` is an optional JSON Schema reference for editors and validation tools.

It does not participate in compilation or runtime execution.

Example:

```json
"$schema": "https://raw.githubusercontent.com/RingEngine/Docs/runtime-1/schemas/filter-src.schema.json"
```

Schema in this repository:
[schemas/filter-src.schema.json](./schemas/filter-src.schema.json)

## `schemaVersion`

`schemaVersion` identifies the manifest schema version.

It uses `MAJOR.MINOR.PATCH` string form.

Example:

```json
"schemaVersion": "1.0.0"
```

## `runtimeVersion`

`runtimeVersion` is a single integer runtime contract version.

A runtime with version `N` supports all filter contracts with `runtimeVersion <= N`.

Example:

```json
"runtimeVersion": 1
```

## `outputSizeMode`

`outputSizeMode` describes how the filter participates in output size determination.

Valid values:

- `passive`
- `active`

When omitted, the value is `passive`.

### `passive`

In `passive` mode, the filter does not request its own output size during `onReset`.

The Lua reset entry point is:

```lua
function onReset(ctx)
end
```

### `active`

In `active` mode, the filter can request its preferred output size during `onReset`.

The Lua reset entry point is:

```lua
function onReset(ctx, outputRequest)
end
```

`outputRequest` provides:

```lua
outputRequest:setSize(width, height)
```

`width` and `height` are positive integer dimensions.

## `metadata`

`metadata` contains optional descriptive fields. These fields are not part of the runtime execution contract.

Supported fields:

- `kind`
- `id`
- `name`
- `version`
- `description`
- `authors`
- `tags`

`kind`, when present, must be `filter-src`.

## `parameters`

`parameters` declares the public parameter interface of the filter.

Parameters are visible to host applications, authoring tools, and Lua through `ctx:getParam(id)`.

Parameters do not describe shader uniforms directly. A single parameter can affect multiple passes, multiple uniforms, resources, or Lua control flow.

Each parameter has:

- `id`
- `type`
- optional `label`
- optional type-specific fields

`label` is an editor-facing display field. It is not part of compilation or runtime execution.

Valid parameter types:

- `float`
- `bool`
- `vec4`
- `mat4`

### `float`

Fields:

- `min`
- `max`
- `default`

All fields are optional.

Defaulting rules:

- missing `min` defaults to `0`
- missing `max` defaults to `1`
- missing `default` defaults to the resolved `min`

If `min` and `max` are both present, `min <= max`.

If `default` is present, it must fall within `[min, max]` after default resolution.

### `bool`

Fields:

- `default`

`default` is optional and defaults to `false`.

`bool` parameters do not use `min`, `max`, or `semantic`.

### `vec4`

Fields:

- `default`
- `semantic`

`default`, when present, is an array of exactly 4 numbers.

`vec4` parameters do not use `min` or `max`.

Valid `semantic` values:

- `color3`
- `color4`
- `ndcPoint2`
- `ndcRect`

#### `color3`

`color3` means:

- components 0, 1, and 2 are color channels in `[0, 1]`
- component 3 is fixed to `1`

#### `color4`

`color4` means:

- components 0, 1, 2, and 3 are RGBA-like channels in `[0, 1]`

#### `ndcPoint2`

`ndcPoint2` means:

- components 0 and 1 are a 2D point in normalized device coordinates
- components 0 and 1 are in `[-1, 1]`
- component 2 is fixed to `0`
- component 3 is fixed to `1`

In `vec4` form this is `[x, y, 0, 1]`.

#### `ndcRect`

`ndcRect` means:

- components 0 and 1 are the lower-left corner of a rectangle in normalized device coordinates
- component 2 is the rectangle width, extending toward positive x
- component 3 is the rectangle height, extending toward positive y
- components 2 and 3 must be greater than or equal to `0`

In `vec4` form this is `[x, y, w, h]`. The upper-right corner is `[x + w, y + h]`.

### `mat4`

Fields:

- `default`

`default`, when present, is an array of exactly 16 numbers.

`mat4` parameters do not use `min`, `max`, or `semantic`.

## `passes`

`passes` declares executable shader passes.

Each pass has:

- `id`
- `type`

Valid pass types:

- `render`
- `compute`

### Render Pass

A render pass has:

- `id`
- `type: "render"`
- `vertexShader`
- `fragmentShader`

`vertexShader` and `fragmentShader` are project-relative paths to GLSL source files.

A render pass uses the runtime fullscreen render model.

The vertex shader for a render pass must contain exactly one `in vec2` vertex input. That input is the fullscreen position input. The reflected input identity is recorded for runtime use.

Lua does not provide vertex attributes and does not control draw submission.

### Compute Pass

A compute pass has:

- `id`
- `type: "compute"`
- `computeShader`

`computeShader` is a project-relative path to a GLSL source file.

A compute pass writes data-domain output through `Buffer`.

Storage buffer declarations in shader code are restricted:

- a storage buffer block is allowed
- the block must declare exactly one member
- that member must be an array

If a shader declares a storage buffer block with multiple members, or with a single non-array member, that declaration is invalid.

Compute execution count is not declared in the manifest. Lua provides compute dispatch dimensions for each invocation through `ctx:runComputePass(passId, bindings, dispatch)`.

## `assets`

`assets` declares runtime asset files.

Each asset has:

- `id`
- `path`
- `type`

Valid asset types:

- `image`
- `video`

`path` is a project-relative path. Absolute paths are invalid.

Lua references assets by `id`.

The manifest declares asset existence and carrier type. It does not declare how a resource is used internally by Lua or shader code.

## Lua Runtime

Lua runs as a controlled embedded runtime.

Runtime interaction is performed through `ctx`.

Lua workflow entry points:

- `onReset(ctx)` for passive output size mode
- `onReset(ctx, outputRequest)` for active output size mode
- `advance(ctx)` for per-frame execution

`onReset` is the reset boundary. Runtime objects created through `ctx` belong to the current reset scope.

Lua primitive values and Lua tables may be stored in global or module-level variables. Authors are responsible for initializing or clearing this Lua state in `onReset(...)` according to the reset boundary.

Runtime objects that Lua cannot create, release, or rebind by itself must be managed through `ctx`, such as `Target`, `Buffer`, `Input`, `Output`, `Image`, and `Video`. Do not keep runtime objects acquired or created through `ctx` as Lua global references across resets. Create or retrieve them again through `ctx` after reset.

### Available Lua Libraries

The runtime exposes a fixed Lua library surface. Scripts must not depend on globals outside this list.

Available standard Lua libraries:

- `_G`
- `math`
- `string`
- `table`

Available runtime-injected globals:

- `ctx`
- `mat4`
- `time`

The `math` library includes Lua-style `math.random(...)` and `math.randomseed(seed)`. Random output is not repeatable unless the script explicitly calls `math.randomseed(seed)`.

The `os`, `io`, `package`, and `debug` libraries are not part of the runtime contract.

## Lua Reference Requirements

The following forms use names written directly in source:

- `ctx:runRenderPass("toneRemap", { ... }, output)`
- `ctx:runComputePass("histogramScatter", { ... }, dispatch)`
- `ctx:clearOutput(output, color)`
- `time.now()`
- `time.parts(out, timestamp, options)`
- `time.fromDate(date, options)`
- uniform block field tables written as Lua table literals

When these names are written directly in source:

- required Lua entry functions must be present
- `ctx` method names must be valid
- binding names for pass calls must be valid
- uniform block field names for binding tables must be valid

Dynamic Lua expressions are allowed. This document does not assign static name requirements to values that are computed dynamically.

Examples include:

```lua
local passId = choosePass()
ctx:runRenderPass(passId, bindings, output)

local assetId = prefix .. suffix
ctx:getAsset(assetId)

local bindings = {}
bindings[name] = value
ctx:runComputePass("histogramScatter", bindings, dispatch)
```

## `ctx`

`ctx` provides the runtime API visible to Lua.

Available functions:

- `ctx:getInput()`
- `ctx:getParam(id)`
- `ctx:getAsset(id)`
- `ctx:createTarget(id, width, height)`
- `ctx:getTarget(id)`
- `ctx:createFloatBuffer(id, shape)`
- `ctx:createUIntBuffer(id, shape)`
- `ctx:getBuffer(id)`
- `ctx:getOutput()`
- `ctx:clearOutput(output, color)`
- `ctx:runRenderPass(passId, bindings, output)`
- `ctx:runComputePass(passId, bindings, dispatch)`

`createTarget`, `createFloatBuffer`, and `createUIntBuffer` are reset-scope creation APIs. They are used from `onReset`.

`createFloatBuffer(id, shape)` creates a `float` array buffer.

`createUIntBuffer(id, shape)` creates a `uint` array buffer.

`shape` is an array-like Lua table of positive integer dimensions. The total element count is the product of all dimensions.

`getTarget` and `getBuffer` retrieve reset-scope objects by id.

### `ctx:getParam`

`ctx:getParam(id)` returns the current parameter value.

Scalar parameters return `number` or `boolean`.

Array parameters, such as `vec4` and `mat4`, return array-like Lua tables.

For array parameters, the runtime uses a mutable snapshot for the current Lua entry call:

- Each array parameter has a runtime-allocated snapshot table that is reused.
- Before each Lua entry function call, including `onReset(...)` and `advance(ctx)`, that snapshot is marked as not prepared.
- The first `ctx:getParam(id)` call for that parameter during the current entry call copies the host-current parameter value into the snapshot table and returns that table.
- Later `ctx:getParam(id)` calls for the same `id` during the same entry call return the same snapshot table. They do not copy again and do not refresh existing references.
- Lua may mutate this snapshot table. The mutation only affects that snapshot during the current Lua entry call. It does not write back to the host parameter and does not affect later Lua entry calls.

Therefore, multiple reads of the same array parameter during one `advance(ctx)` call return the same table:

```lua
function advance(ctx)
  local transform = ctx:getParam("transform")
  mat4.translate(transform, 10, 0, 0)

  local again = ctx:getParam("transform")
  -- again is the same table as transform and includes the translate above.
end
```

Do not keep an array table returned by `ctx:getParam(id)` as persistent state across `onReset(...)` or `advance(ctx)` calls.

If a matrix or array must be preserved across multiple `advance(ctx)` calls, use an author-owned Lua table and initialize or clear it in `onReset(...)`. Do not keep the snapshot table returned by `ctx:getParam(id)` itself.

```lua
local drawTransform = {}

function onReset(ctx)
  mat4.identity(drawTransform)
end

function advance(ctx)
  local sourceTransform = ctx:getParam("transform")

  mat4.copy(drawTransform, sourceTransform)
  mat4.translate(drawTransform, 10, 0, 0)

  ctx:runRenderPass("draw", {
    source = ctx:getInput(),
    params = {
      transform = drawTransform
    }
  }, ctx:getOutput())
end
```

In this example, `drawTransform` is an author-owned Lua table. It is not a runtime object, so it may be stored in a Lua variable. The author is responsible for maintaining its reset boundary in `onReset(...)`.

## `time`

`time` is a runtime-injected Lua global library for wall-clock time and calendar conversion.

`time` is separate from the render timeline. Frame time, delta time, and frame index are runtime execution concepts and belong on `ctx` APIs rather than in `time`.

Available functions:

- `time.now() -> timestamp`
- `time.parts(out, timestamp [, options]) -> out`
- `time.fromDate(date [, options]) -> timestamp`

### `time.now`

`time.now()` returns the current Unix timestamp as seconds.

The return value is a Lua `number`. It may include a fractional part and must provide at least millisecond precision.

### `time.parts`

`time.parts(out, timestamp [, options])` decomposes a timestamp into calendar fields.

`timestamp` is Unix time in seconds. It may include a fractional part.

`out` must be a Lua table. The runtime writes fields into `out` and returns `out`, so authors can reuse the same table across frames.

`options` is an optional Lua table. Supported fields:

- `utc`: when `true`, use UTC. Otherwise, use the runtime's local time zone.

Fields written to `out`:

- `year`: full year
- `month`: `1` to `12`
- `day`: `1` to `31`
- `hour`: `0` to `23`
- `min`: `0` to `59`
- `sec`: `0` to `59`
- `millis`: `0` to `999`
- `wday`: `1` to `7`, Sunday is `1`
- `yday`: `1` to `366`, January 1 is `1`

Example:

```lua
local wall = {}

function advance(ctx)
  time.parts(wall, time.now(), { utc = true })

  ctx:runRenderPass("draw", {
    params = {
      clock = { wall.hour, wall.min, wall.sec, wall.millis }
    }
  }, ctx:getOutput())
end
```

### `time.fromDate`

`time.fromDate(date [, options])` converts date fields into a Unix timestamp in seconds.

`date` is a Lua table. `time.fromDate` reads only these fields:

- `year`: required
- `month`: required
- `day`: required
- `hour`: optional, defaults to `0`
- `min`: optional, defaults to `0`
- `sec`: optional, defaults to `0`
- `millis`: optional, defaults to `0`

The input date fields are normalized by calendar rules before conversion. Derived fields such as `wday` and `yday` are not read by `time.fromDate`.

`options` is an optional Lua table. Supported fields:

- `utc`: when `true`, interpret the input as UTC. Otherwise, interpret it in the runtime's local time zone.

Example:

```lua
local launchAt = time.fromDate({
  year = 2026,
  month = 5,
  day = 3,
  hour = 20,
  min = 30,
  sec = 0,
  millis = 250
}, { utc = true })
```

`time.parts` and `time.fromDate` use separate table shapes. Some field names are shared because they describe the same calendar concepts, but `time.parts` output is not the required input structure for `time.fromDate`.

## `mat4`

`mat4` is a runtime-injected Lua global library for 4x4 matrix calculation.

Authors use `mat4` to compute matrix values for shaders, such as translation, scale, rotation, pixel-coordinate to NDC orthographic projection when output size changes, and point transformation through a matrix.

A `mat4` value is a Lua table containing exactly 16 numbers. Matrices use column-major layout, matching GLSL `mat4` uniforms:

```text
m[1]  m[5]  m[9]   m[13]
m[2]  m[6]  m[10]  m[14]
m[3]  m[7]  m[11]  m[15]
m[4]  m[8]  m[12]  m[16]
```

The `mat4` library does not provide `mat4.create()`. Authors create ordinary Lua tables for working matrices and fill them through `mat4` functions:

```lua
local transform = {}

function advance(ctx)
  mat4.identity(transform)
  mat4.translate(transform, 100, 50, 0)
  mat4.scale(transform, 200, 100, 1)

  ctx:runRenderPass("draw", {
    source = ctx:getInput(),
    params = {
      transform = transform
    }
  }, ctx:getOutput())
end
```

In this example, `params.transform` is a uniform block field. For a loose uniform binding, pass the matrix through the corresponding binding name in `bindings`.

### Basic Functions

- `mat4.identity(m) -> m`
- `mat4.copy(dst, src) -> dst`

`identity` writes the identity matrix into `m`.

`copy` copies the 16 components of `src` into `dst`.

### Matrix Composition

- `mat4.multiply(m, rhs) -> m`
- `mat4.preMultiply(m, lhs) -> m`

`multiply` means:

```text
m = m * rhs
```

`preMultiply` means:

```text
m = lhs * m
```

### Appending Transforms

- `mat4.translate(m, x, y, z) -> m`
- `mat4.scale(m, x, y, z) -> m`
- `mat4.rotateX(m, radians) -> m`
- `mat4.rotateY(m, radians) -> m`
- `mat4.rotateZ(m, radians) -> m`

These functions append a transform to the current matrix. For example, `mat4.translate(m, x, y, z)` means:

```text
m = m * T
```

where `T` is the translation matrix defined by `x`, `y`, and `z`.

Rotation angles are in radians.

### Setting Matrices

- `mat4.setTranslation(m, x, y, z) -> m`
- `mat4.setScale(m, x, y, z) -> m`
- `mat4.setRotationX(m, radians) -> m`
- `mat4.setRotationY(m, radians) -> m`
- `mat4.setRotationZ(m, radians) -> m`
- `mat4.setOrtho(m, left, right, bottom, top, near, far) -> m`

`set*` functions overwrite the current contents of `m`. They do not append a transform to the existing matrix.

`setOrtho` creates an orthographic projection matrix that maps the coordinate range defined by `left`, `right`, `bottom`, `top`, `near`, and `far` into NDC.

For example, this maps output-canvas pixel coordinates to NDC, using a coordinate system whose upper-left corner is `(0, 0)` and whose y axis grows downward:

```lua
local proj = {}

function advance(ctx)
  local output = ctx:getOutput()
  mat4.setOrtho(proj, 0, output:getWidth(), output:getHeight(), 0, -1, 1)

  local ndcX, ndcY = mat4.transformPoint2(proj, output:getWidth() * 0.5, output:getHeight() * 0.5)
  -- ndcX and ndcY are near 0.
end
```

### Inversion, Transpose, And Point Transformation

- `mat4.invert(m) -> boolean`
- `mat4.transpose(m) -> m`
- `mat4.transformPoint4(m, x, y, z, w) -> x2, y2, z2, w2`
- `mat4.transformPoint2(m, x, y) -> x2, y2`

`invert` inverts the matrix in place. It returns `false` if the matrix is not invertible. It returns `true` on success.

`transpose` transposes the matrix in place.

`transformPoint4` computes:

```text
[x2, y2, z2, w2] = m * [x, y, z, w]
```

`transformPoint2` is the 2D convenience form. It is equivalent to calling `transformPoint4` with `z = 0` and `w = 1`, then returning only `x2` and `y2`.

## Runtime Object Model

Objects are described by capability composition.

Capability relations:

- `Input`: `ReadableImage`
- `Image`: `ReadableImage`
- `Target`: `ReadableImage + WritableOutput`
- `Video`: `Image + VideoControl`
- `Output`: `WritableOutput + SystemOnly`
- `Buffer`: `ReadableData + WritableData`

`Buffer` is an array data object.

At the authoring contract level, a shader buffer binding corresponds to one array buffer object. The internal member name inside the GLSL storage buffer block does not become a top-level Lua binding name.

### `Sizable`

Methods:

- `getWidth()`
- `getHeight()`

`ReadableImage` and `WritableOutput` are sizable.

### `ReadableImage`

Readable image objects can be used as image bindings for pass execution.

Objects:

- `Input`
- `Image`
- `Target`

### `WritableOutput`

Writable output objects can be used as render pass outputs.

Objects:

- `Target`
- `Output`

Writable output objects can also be cleared through `ctx:clearOutput(output, color)`.

### `ReadableData`

Readable data objects can be used as data bindings for pass execution.

Objects:

- `Buffer`

### `WritableData`

Writable data objects can be used as compute pass outputs.

Objects:

- `Buffer`

### `VideoControl`

Methods:

- `seek(frameIndex)`
- `nextFrame()`

`VideoControl` is provided by `Video`.

### `SystemOnly`

`SystemOnly` objects are acquired from runtime APIs and are not created by Lua.

`Output` is `SystemOnly`.

## `clearOutput`

`ctx:clearOutput(output, color)` clears a writable output to a known color.

`output` must be `Target` or `Output`.

`color` is a Lua table with normalized color components in the `0.0` to `1.0` range. It may be array-like or keyed:

```lua
ctx:clearOutput(ctx:getOutput(), { 0.1, 0.1, 0.12, 1.0 })
ctx:clearOutput(ctx:getTarget("tempA"), { r = 0, g = 0, b = 0, a = 1 })
```

If alpha is omitted, it defaults to `1.0`.

Clearing is an explicit runtime command. Authors should call it before render passes when the initial contents of an output matter.

## `runRenderPass`

`ctx:runRenderPass(passId, bindings, output)` executes a render pass.

`bindings` contains the input bindings for the invocation.

Binding keys are shader binding object names.

If a binding is a uniform block, the binding value is a Lua table keyed by the block's member names.

`output` contains the render target binding for the invocation.

Render pass output must be `Target` or `Output`.

Example:

```lua
ctx:runRenderPass("blur", {
  source = ctx:getInput(),
  params = {
    strength = ctx:getParam("strength")
  }
}, ctx:getTarget("tempA"))
```

## `runComputePass`

`ctx:runComputePass(passId, bindings, dispatch)` executes a compute pass.

`bindings` contains all shader bindings for the invocation, including sampled images, readonly buffers, and writable buffers.

Binding keys are shader binding object names.

If a binding is a uniform block, the binding value is a Lua table keyed by the block's member names.

If a binding is a buffer, the binding value is a `Buffer` runtime object.

`dispatch` contains compute dispatch dimensions.

`dispatch` is an array-like Lua table of 1 to 3 positive integers:

- `{x}`
- `{x, y}`
- `{x, y, z}`

Missing trailing dimensions default to `1`.

`dispatch` represents the logical total execution coverage requested by Lua. It is not the native workgroup count passed directly to the graphics backend. The compiled filter manifest records each compute shader's `localSize`, and the runtime converts `dispatch` into backend workgroup counts before executing the compute pass.

Example:

```lua
local input = ctx:getInput()

ctx:runComputePass("histogramScatter", {
  source = input,
  histogramPartial = ctx:getBuffer("histogramPartial")
}, { input:getWidth(), input:getHeight() })
```

## Uniform Values

Uniform values are runtime objects or pure Lua values.

Runtime objects:

- `Input`
- `Image`
- `Video`
- `Target`
- `Buffer`

Pure Lua values:

- `number`
- `boolean`
- array-like Lua tables of numbers

Scalar shader values use `number` or `boolean`.

Vector and matrix shader values use array-like Lua tables.

Uniform block values use Lua tables keyed by block member name.

`ctx:getParam(id)` returns pure Lua values. Lua can copy, edit, combine, and compute new values before passing them into `bindings`.

Binding values must match reflected pass metadata.

## Source Manifest And Package Manifest

`filter-src/manifest.json` describes authoring intent.

The compiled `filter` package manifest describes runtime loading facts.

The two manifest structures can be similar, but they are distinct formats.

The compiler extracts shader reflection data and writes runtime-facing reflection data into the compiled `filter` package manifest.

Schema in this repository:
[schemas/filter.schema.json](./schemas/filter.schema.json)

When present, the compiled package `$schema` should reference:

```json
"$schema": "https://raw.githubusercontent.com/RingEngine/Docs/runtime-1/schemas/filter.schema.json"
```

The compiled package manifest has its own schema because it is consumed by `filter-runtime`, while `filter-src/manifest.json` is consumed by authoring tools and the compiler.

Compiled pass records expose runtime-facing binding data directly on the pass object:

- `bindings`
- `localSize` for compute passes
- `vertexInput` for render passes

There is no outer `reflection` wrapper in the compiled pass record.

Binding `type` values currently supported by the compiled manifest are:

- `sampledImage`
- `buffer`
- `uniformBlock`
- `uniform`

`sampledImage` binds an image-like runtime object as a sampled texture input.

`buffer` binds a `Buffer` object created with `ctx:createFloatBuffer` or `ctx:createUIntBuffer`.

`uniformBlock` binds a Lua table to a structured uniform block. The table keys are field names.

`uniform` binds a single scalar, vector, or matrix value.

Uniform blocks and storage buffers use fixed ABI layouts:

- `uniformBlock` declarations are compiled as `std140`.
- `buffer` declarations are compiled as `std430`.
- If a GLSL block declaration omits the layout standard, the compiler injects the required standard before shader backend compilation.
- If author source explicitly declares a different block layout standard, such as `std430` on a uniform block or `std140` on a storage buffer, compilation fails.

The compiled reflection records uniform block field offsets and sizes using this fixed layout. Runtimes must pack Lua table values according to those reflected byte offsets instead of relying on backend-native struct layout guesses.

Uniform block field `type` values currently supported are:

- `float`
- `bool`
- `int`
- `uint`
- `vec2`
- `vec3`
- `vec4`
- `ivec2`
- `ivec3`
- `ivec4`
- `uvec2`
- `uvec3`
- `uvec4`
- `mat2`
- `mat3`
- `mat4`

Buffer `elementType` values currently supported are:

- `float`
- `uint`

For storage buffers, compiled reflection is runtime-facing array buffer reflection. The compiled manifest identifies the buffer binding object and its array element type. The internal GLSL storage buffer member name is not part of the top-level Lua binding contract.

Compiled reflection is intended for runtime load/reset planning. A runtime can turn this manifest data into direct backend binding tables before frame execution. Per-frame `advance` execution should bind by the already prepared runtime structures instead of traversing the manifest.
