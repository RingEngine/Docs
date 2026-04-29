# Filter Source

本文定义 `filter-src` 源码工程格式。

`filter-src` 是滤镜工程的可编辑作者格式。编译器读取 `filter-src` 工程，并生成 runtime 可消费的 `filter` 包。

## 必需根文件

一个 `filter-src` 工程在工程根目录包含两个必需文件：

```text
filter-src/
  manifest.json
  main.lua
```

### `manifest.json`

`manifest.json` 是必需文件，必须位于工程根目录。

它定义：

- runtime contract 版本
- 输出尺寸决定模式
- 可选工程元信息
- 公共参数
- pass 源文件
- 运行时资源文件

### `main.lua`

`main.lua` 是必需文件，必须位于工程根目录。

它定义滤镜的 Lua 工作流入口。

## Manifest 结构

manifest 根对象包含以下字段：

- `$schema`
- `schemaVersion`
- `runtimeVersion`
- `outputSizeMode`
- `metadata`
- `parameters`
- `passes`
- `assets`

`schemaVersion`、`runtimeVersion` 和 `passes` 是必需字段。

`$schema` 是可选字段。

`outputSizeMode` 缺省时默认是 `passive`。

`metadata`、`parameters` 和 `assets` 是可选字段。

## `$schema`

`$schema` 是可选的 JSON Schema 引用，供编辑器和校验工具使用。

它不参与编译和运行时执行。

示例：

```json
"$schema": "https://raw.githubusercontent.com/RingEngine/Docs/runtime-1/schemas/filter-src.schema.json"
```

本仓库中的 schema：
[schemas/filter-src.schema.json](./schemas/filter-src.schema.json)

## `schemaVersion`

`schemaVersion` 标识 manifest schema 版本。

它使用 `MAJOR.MINOR.PATCH` 字符串形式。

示例：

```json
"schemaVersion": "1.0.0"
```

## `runtimeVersion`

`runtimeVersion` 是单一整数形式的 runtime contract 版本号。

版本为 `N` 的 runtime 支持所有 `runtimeVersion <= N` 的 filter contract。

示例：

```json
"runtimeVersion": 1
```

## `outputSizeMode`

`outputSizeMode` 描述 filter 如何参与输出尺寸决定。

合法值：

- `passive`
- `active`

缺省时为 `passive`。

### `passive`

`passive` 模式下，filter 不在 `onReset` 中请求自己的输出尺寸。

Lua reset 入口是：

```lua
function onReset(ctx)
end
```

### `active`

`active` 模式下，filter 可以在 `onReset` 中请求期望输出尺寸。

Lua reset 入口是：

```lua
function onReset(ctx, outputRequest)
end
```

`outputRequest` 提供：

```lua
outputRequest:setSize(width, height)
```

`width` 和 `height` 是正整数尺寸。

## `metadata`

`metadata` 包含可选的描述性字段。这些字段不属于 runtime 执行 contract。

支持字段：

- `kind`
- `id`
- `name`
- `version`
- `description`
- `authors`
- `tags`

`kind` 如果存在，必须是 `filter-src`。

## `parameters`

`parameters` 声明滤镜的公共参数接口。

参数对宿主应用、作者工具可见，并可在 Lua 中通过 `ctx:getParam(id)` 读取。

参数不直接描述 shader uniform。一个参数可以影响多个 pass、多个 uniform、资源或 Lua 控制流。

每个参数包含：

- `id`
- `type`
- 可选 `label`
- 可选的类型特定字段

`label` 是面向编辑器和作者工具的显示字段，不参与编译和运行时执行。

合法参数类型：

- `float`
- `bool`
- `vec4`
- `mat4`

### `float`

字段：

- `min`
- `max`
- `default`

这些字段都可选。

默认规则：

- 缺省 `min` 为 `0`
- 缺省 `max` 为 `1`
- 缺省 `default` 为解析后的 `min`

如果 `min` 和 `max` 同时存在，必须满足 `min <= max`。

如果 `default` 存在，它必须落在默认解析后的 `[min, max]` 范围内。

### `bool`

字段：

- `default`

`default` 可选，缺省为 `false`。

`bool` 参数不使用 `min`、`max` 或 `semantic`。

### `vec4`

字段：

- `default`
- `semantic`

`default` 如果存在，必须是长度正好为 4 的数字数组。

`vec4` 参数不使用 `min` 或 `max`。

合法 `semantic` 值：

- `color3`
- `color4`
- `ndcPoint2`
- `ndcRect`

#### `color3`

`color3` 表示：

- 第 0、1、2 个分量是 `[0, 1]` 范围内的颜色通道
- 第 3 个分量固定为 `1`

#### `color4`

`color4` 表示：

- 第 0、1、2、3 个分量是 `[0, 1]` 范围内的 RGBA 风格通道

#### `ndcPoint2`

`ndcPoint2` 表示：

- 第 0、1 个分量是归一化设备坐标中的二维点
- 第 0、1 个分量范围是 `[-1, 1]`
- 第 2 个分量固定为 `0`
- 第 3 个分量固定为 `1`

以 `vec4` 表示时即 `[x, y, 0, 1]`。

#### `ndcRect`

`ndcRect` 表示：

- 第 0、1 个分量是归一化设备坐标中矩形的左下角
- 第 2 个分量是矩形宽度，向 x 正方向延伸
- 第 3 个分量是矩形高度，向 y 正方向延伸
- 第 2、3 个分量必须大于或等于 `0`

以 `vec4` 表示时即 `[x, y, w, h]`。右上角是 `[x + w, y + h]`。

### `mat4`

字段：

- `default`

`default` 如果存在，必须是长度正好为 16 的数字数组。

`mat4` 参数不使用 `min`、`max` 或 `semantic`。

## `passes`

`passes` 声明可执行 shader pass。

每个 pass 包含：

- `id`
- `type`

合法 pass 类型：

- `render`
- `compute`

### Render Pass

render pass 包含：

- `id`
- `type: "render"`
- `vertexShader`
- `fragmentShader`

`vertexShader` 和 `fragmentShader` 是指向 GLSL 源文件的工程相对路径。

render pass 使用 runtime fullscreen render model。

render pass 的 vertex shader 必须包含且只包含一个 `in vec2` 顶点输入。这个输入是 fullscreen position input。该输入的反射身份会被记录供 runtime 使用。

Lua 不提供 vertex attribute，也不控制 draw submission。

### Compute Pass

compute pass 包含：

- `id`
- `type: "compute"`
- `computeShader`

`computeShader` 是指向 GLSL 源文件的工程相对路径。

compute pass 通过 `Buffer` 写入数据域输出。

shader 中的 storage buffer 声明有如下限制：

- 允许使用 storage buffer block
- block 内必须且只能声明一个成员
- 这个唯一成员必须是数组

如果 shader 声明了多成员的 storage buffer block，或者唯一成员不是数组，则该声明非法。

compute 的执行次数不写在 manifest 中。Lua 在每次 `ctx:runComputePass(passId, bindings, dispatch)` 调用时提供 dispatch 尺寸。

## `assets`

`assets` 声明运行时资源文件。

每个资源包含：

- `id`
- `path`
- `type`

合法资源类型：

- `image`
- `video`

`path` 是工程相对路径。绝对路径非法。

Lua 通过 `id` 引用资源。

manifest 只声明资源存在和载体类型。它不声明资源在 Lua 或 shader 代码中如何被使用。

## Lua Runtime

Lua 运行在受控嵌入式 runtime 中。

runtime 交互通过 `ctx` 完成。

Lua 工作流入口：

- `onReset(ctx)` 用于 passive 输出尺寸模式
- `onReset(ctx, outputRequest)` 用于 active 输出尺寸模式
- `advance(ctx)` 用于逐帧执行

`onReset` 是 reset 边界。通过 `ctx` 创建的 runtime 对象属于当前 reset scope。

Lua 原生值和 Lua table 可以保存在全局变量或模块级变量中。作者必须在 `onReset(...)` 中按 reset 边界自行初始化或清理这些 Lua 状态。

Lua 无法自行创建、释放或重新绑定的 runtime 对象必须通过 `ctx` 代管，例如 `Target`、`Buffer`、`Input`、`Output`、`Image` 和 `Video`。不要把通过 `ctx` 获取或创建的 runtime 对象保存为跨 reset 使用的 Lua 全局引用；reset 后应重新通过 `ctx` 创建或获取。

## Lua 引用要求

以下形式会在源码中直接写出名字：

- `ctx:runRenderPass("toneRemap", { ... }, output)`
- `ctx:runComputePass("histogramScatter", { ... }, dispatch)`
- 以 Lua table 字面量书写的 uniform block 字段表

当这些名字直接写在源码中时：

- 必需的 Lua 入口函数必须存在
- `ctx` 方法名必须合法
- pass 调用中的 binding 名必须合法
- binding table 中的 uniform block 字段名必须合法

动态 Lua 表达式是允许的。对于动态计算得到的值，本文档不对其施加静态名字要求。

典型形式包括：

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

`ctx` 提供 Lua 可见的 runtime API。

可用函数：

- `ctx:getInput()`
- `ctx:getParam(id)`
- `ctx:getAsset(id)`
- `ctx:createTarget(id, width, height)`
- `ctx:getTarget(id)`
- `ctx:createFloatBuffer(id, shape)`
- `ctx:createUIntBuffer(id, shape)`
- `ctx:getBuffer(id)`
- `ctx:getOutput()`
- `ctx:runRenderPass(passId, bindings, output)`
- `ctx:runComputePass(passId, bindings, dispatch)`

`createTarget`、`createFloatBuffer` 和 `createUIntBuffer` 是 reset-scope 创建 API。它们在 `onReset` 中使用。

`createFloatBuffer(id, shape)` 创建 `float` 数组 buffer。

`createUIntBuffer(id, shape)` 创建 `uint` 数组 buffer。

`shape` 是由正整数维度组成的 Lua 数组。总元素数量是所有维度的乘积。

`getTarget` 和 `getBuffer` 通过 id 取回 reset-scope 对象。

### `ctx:getParam`

`ctx:getParam(id)` 返回当前 parameter 值。

标量 parameter 返回 `number` 或 `boolean`。

数组型 parameter，例如 `vec4` 和 `mat4`，返回数组形式的 Lua table。

对数组型 parameter，runtime 使用当前 Lua 入口调用内的可变快照：

- 每个数组型 parameter 有一个由 runtime 预分配并复用的 snapshot table。
- 每次 Lua 入口函数调用开始前，包括 `onReset(...)` 和 `advance(ctx)`，该 snapshot 标记为未准备。
- 本次调用内第一次 `ctx:getParam(id)` 会把宿主当前 parameter 值复制到 snapshot table，并返回该 table。
- 本次调用内后续对同一 `id` 的 `ctx:getParam(id)` 返回同一个 snapshot table，不会再次复制，也不会刷新已有引用。
- Lua 可以修改这个 snapshot table。修改只影响当前 Lua 入口调用内的该 snapshot，不会回写宿主 parameter，也不会影响下一次 Lua 入口调用。

因此，同一次 `advance(ctx)` 内多次读取同一个数组 parameter 会得到同一个 table：

```lua
function advance(ctx)
  local transform = ctx:getParam("transform")
  mat4.translate(transform, 10, 0, 0)

  local again = ctx:getParam("transform")
  -- again 与 transform 是同一个 table，并包含上面的 translate 结果。
end
```

不要把 `ctx:getParam(id)` 返回的数组 table 当作跨 `onReset(...)` 或 `advance(ctx)` 调用的持久状态保存。

如果需要跨多次 `advance(ctx)` 调用保留一个矩阵或数组，应使用作者自己拥有的 Lua table，并在 `onReset(...)` 中初始化或清理。不要保存 `ctx:getParam(id)` 返回的 snapshot table 本身。

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

上例中的 `drawTransform` 是 Lua 自己拥有的 table。它不是 runtime 对象，因此可以保存在 Lua 变量中；作者负责在 `onReset(...)` 中维护它的 reset 边界。

## `mat4`

`mat4` 是 runtime 注入的 Lua 全局基础库，用于 4x4 矩阵计算。

`mat4` 主要用于在 Lua 中计算传给 shader 的矩阵值，例如输出尺寸变化时的平移、缩放、旋转、像素坐标到 NDC 坐标的正交投影，以及用矩阵转换点。

`mat4` 值是长度为 16 的 number 数组 Lua table。矩阵使用 column-major 布局，与 GLSL `mat4` uniform 兼容：

```text
m[1]  m[5]  m[9]   m[13]
m[2]  m[6]  m[10]  m[14]
m[3]  m[7]  m[11]  m[15]
m[4]  m[8]  m[12]  m[16]
```

`mat4` 库不提供 `mat4.create()`。需要工作矩阵时，作者直接创建 Lua table，并通过 `mat4` 函数填充：

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

上例中的 `params.transform` 是 uniform block 字段。对于 loose uniform binding，也通过 `bindings` 中对应的 binding 名传入。

### 基础函数

- `mat4.identity(m) -> m`
- `mat4.copy(dst, src) -> dst`

`identity` 把 `m` 写成单位矩阵。

`copy` 把 `src` 的 16 个分量复制到 `dst`。

### 矩阵组合

- `mat4.multiply(m, rhs) -> m`
- `mat4.preMultiply(m, lhs) -> m`

`multiply` 的语义是：

```text
m = m * rhs
```

`preMultiply` 的语义是：

```text
m = lhs * m
```

### 追加变换

- `mat4.translate(m, x, y, z) -> m`
- `mat4.scale(m, x, y, z) -> m`
- `mat4.rotateX(m, radians) -> m`
- `mat4.rotateY(m, radians) -> m`
- `mat4.rotateZ(m, radians) -> m`

这些函数在当前矩阵基础上追加变换。例如 `mat4.translate(m, x, y, z)` 的语义是：

```text
m = m * T
```

其中 `T` 是由 `x`、`y`、`z` 定义的平移矩阵。

旋转角度使用弧度。

### 直接设置矩阵

- `mat4.setTranslation(m, x, y, z) -> m`
- `mat4.setScale(m, x, y, z) -> m`
- `mat4.setRotationX(m, radians) -> m`
- `mat4.setRotationY(m, radians) -> m`
- `mat4.setRotationZ(m, radians) -> m`
- `mat4.setOrtho(m, left, right, bottom, top, near, far) -> m`

`set*` 函数覆盖 `m` 的当前内容。它们不会在现有矩阵基础上追加变换。

`setOrtho` 创建正交投影矩阵，把由 `left`、`right`、`bottom`、`top`、`near`、`far` 定义的坐标范围映射到 NDC。

例如把输出画布的像素坐标映射到 NDC，并使用左上角为 `(0, 0)`、y 向下增长的坐标系：

```lua
local proj = {}

function advance(ctx)
  local output = ctx:getOutput()
  mat4.setOrtho(proj, 0, output:getWidth(), output:getHeight(), 0, -1, 1)

  local ndcX, ndcY = mat4.transformPoint2(proj, output:getWidth() * 0.5, output:getHeight() * 0.5)
  -- ndcX 和 ndcY 接近 0。
end
```

### 逆矩阵、转置与点转换

- `mat4.invert(m) -> boolean`
- `mat4.transpose(m) -> m`
- `mat4.transformPoint4(m, x, y, z, w) -> x2, y2, z2, w2`
- `mat4.transformPoint2(m, x, y) -> x2, y2`

`invert` 原地求逆。如果矩阵不可逆，返回 `false`。如果成功，返回 `true`。

`transpose` 原地转置矩阵。

`transformPoint4` 计算：

```text
[x2, y2, z2, w2] = m * [x, y, z, w]
```

`transformPoint2` 是 2D 便捷形式，等价于使用 `z = 0`、`w = 1` 调用 `transformPoint4`，并只返回 `x2`、`y2`。

## Runtime 对象模型

对象通过能力组合描述。

能力关系：

- `Input`: `ReadableImage`
- `Image`: `ReadableImage`
- `Target`: `ReadableImage + WritableOutput`
- `Video`: `Image + VideoControl`
- `Output`: `WritableOutput + SystemOnly`
- `Buffer`: `ReadableData + WritableData`

`Buffer` 是数组数据对象。

在作者约定层面，一个 shader buffer binding 对应一个数组 buffer 对象。GLSL storage buffer block 内部的成员名，不会成为 Lua 顶层 binding 名。

### `Sizable`

方法：

- `getWidth()`
- `getHeight()`

`ReadableImage` 和 `WritableOutput` 都具备尺寸能力。

### `ReadableImage`

可读图像对象可以作为 pass 执行时的图像 binding。

对象：

- `Input`
- `Image`
- `Target`

### `WritableOutput`

可写输出对象可以作为 render pass 输出。

对象：

- `Target`
- `Output`

### `ReadableData`

可读数据对象可以作为 pass 执行时的数据 binding。

对象：

- `Buffer`

### `WritableData`

可写数据对象可以作为 compute pass 输出。

对象：

- `Buffer`

### `VideoControl`

方法：

- `seek(frameIndex)`
- `nextFrame()`

`VideoControl` 由 `Video` 提供。

### `SystemOnly`

`SystemOnly` 对象从 runtime API 获取，不由 Lua 创建。

`Output` 是 `SystemOnly`。

## `runRenderPass`

`ctx:runRenderPass(passId, bindings, output)` 执行一个 render pass。

`bindings` 包含本次调用的输入 binding。

`bindings` 的 key 对应 shader binding 对象名。

如果某个 binding 是 uniform block，那么它对应的值是一个 Lua table，table 的 key 是该 block 的成员名。

`output` 包含本次调用的渲染目标 binding。

render pass 的输出必须是 `Target` 或 `Output`。

示例：

```lua
ctx:runRenderPass("blur", {
  source = ctx:getInput(),
  params = {
    strength = ctx:getParam("strength")
  }
}, ctx:getTarget("tempA"))
```

## `runComputePass`

`ctx:runComputePass(passId, bindings, dispatch)` 执行一个 compute pass。

`bindings` 包含本次调用的全部 shader binding，包括 sampled image、readonly buffer 和 writable buffer。

`bindings` 的 key 对应 shader binding 对象名。

如果某个 binding 是 uniform block，那么它对应的值是一个 Lua table，table 的 key 是该 block 的成员名。

如果某个 binding 是 buffer，那么它对应的值是一个 `Buffer` runtime 对象。

`dispatch` 包含 compute 的 dispatch 尺寸。

`dispatch` 是长度为 1 到 3 的正整数 Lua 数组：

- `{x}`
- `{x, y}`
- `{x, y, z}`

缺省的尾部维度默认是 `1`。

`dispatch` 表示 Lua 请求的逻辑总执行覆盖尺寸。它不是直接传给图形后端的原生 workgroup 数量。编译后的 filter manifest 会记录每个 compute shader 的 `localSize`，runtime 在执行 compute pass 前根据 `dispatch` 和 `localSize` 换算后端实际需要提交的 workgroup 数量。

示例：

```lua
local input = ctx:getInput()

ctx:runComputePass("histogramScatter", {
  source = input,
  histogramPartial = ctx:getBuffer("histogramPartial")
}, { input:getWidth(), input:getHeight() })
```

## Uniform 值

Uniform 值可以是 runtime 对象或纯 Lua 值。

runtime 对象：

- `Input`
- `Image`
- `Video`
- `Target`
- `Buffer`

纯 Lua 值：

- `number`
- `boolean`
- number 数组形式的 Lua table

标量 shader 值使用 `number` 或 `boolean`。

向量和矩阵 shader 值使用数组形式的 Lua table。

uniform block 的值使用 Lua table，table 的 key 是 block 成员名。

`ctx:getParam(id)` 返回纯 Lua 值。Lua 可以在传入 `bindings` 之前复制、编辑、组合和计算这些值。

binding 值必须匹配 pass 的反射元数据。

## 源 manifest 与包 manifest

`filter-src/manifest.json` 描述作者意图。

编译后的 `filter` 包 manifest 描述 runtime 装载事实。

两种 manifest 结构可以相似，但它们是不同格式。

编译器提取 shader 反射信息，并将 runtime-facing 的反射数据写入编译后的 `filter` 包 manifest。

本仓库中的 schema：
[schemas/filter.schema.json](./schemas/filter.schema.json)

如果编译后包包含 `$schema`，它应引用：

```json
"$schema": "https://raw.githubusercontent.com/RingEngine/Docs/runtime-1/schemas/filter.schema.json"
```

编译后包 manifest 拥有独立 schema，因为它由 `filter-runtime` 消费；`filter-src/manifest.json` 则由作者工具和编译器消费。

编译后的 pass 记录会直接在 pass 对象上暴露面向 runtime 的绑定数据：

- `bindings`
- compute pass 使用 `localSize`
- render pass 使用 `vertexInput`

编译后的 pass 记录中没有外层 `reflection` 包装字段。

编译后 manifest 当前支持的 binding `type` 值：

- `sampledImage`
- `buffer`
- `uniformBlock`
- `uniform`

`sampledImage` 将 image-like runtime 对象绑定为 sampled texture 输入。

`buffer` 绑定通过 `ctx:createFloatBuffer` 或 `ctx:createUIntBuffer` 创建的 `Buffer` 对象。

`uniformBlock` 将 Lua table 绑定到结构化 uniform block。table 的 key 是 field 名。

`uniform` 绑定单个标量、向量或矩阵值。

uniform block 和 storage buffer 使用固定 ABI 布局：

- `uniformBlock` 声明编译为 `std140`。
- `buffer` 声明编译为 `std430`。
- 如果 GLSL block 声明没有写布局标准，编译器会在 shader backend 编译前自动注入所需标准。
- 如果作者源码显式声明了不同的 block 布局标准，例如 uniform block 使用 `std430`，或 storage buffer 使用 `std140`，编译失败。

编译后的反射会按这个固定布局记录 uniform block field 的字节 offset 和 size。runtime 必须按这些反射出的字节偏移打包 Lua table 值，不能猜测底层 backend 的原生结构体布局。

uniform block field 当前支持的 `type` 值：

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

buffer 当前支持的 `elementType` 值：

- `float`
- `uint`

对于 storage buffer，编译后的反射是面向 runtime 的数组 buffer 反射。编译后的 manifest 标识 buffer binding 对象本身，以及它的数组元素类型。GLSL storage buffer 内部成员名不属于 Lua 顶层 binding contract。

编译后的反射数据用于 runtime 在加载或 reset 阶段构建直接的绑定计划。逐帧 `advance` 执行时应使用已经准备好的 runtime 结构完成绑定，而不是遍历 manifest。
