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
- 第 2、3 个分量固定为 `0`

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

Lua 全局变量可以存在，但 reset-safe 的 runtime 状态必须使用通过 `ctx` 创建的 runtime-managed 对象。

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
