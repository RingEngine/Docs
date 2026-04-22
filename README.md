# Ring Engine Docs

[中文](#中文) | [English](#english)

## 中文

这是 Ring Engine 主要文档与标准的发布仓库。

当前仓库的目标，是先把 Ring Engine 生态中最基础、最需要稳定下来的文档资产整理出来，并以可发布、可引用、可逐步演进的方式维护。现阶段我们优先补充 filter 相关内容，后续会继续把更多模块、格式、接口约定和工程规范逐步补齐。

### 当前成果

目前仓库已经包含一批围绕 filter 的基础文档和 schema：

- `FILTER_SRC.md`：`filter-src` 源码工程格式说明（英文）
- `FILTER_SRC.zh-CN.md`：`filter-src` 源码工程格式说明（中文）
- `schemas/filter-src.schema.json`：`filter-src` manifest schema
- `schemas/filter.schema.json`：运行时 `filter` 包 schema

这些内容为后续的编译器、运行时、编辑器支持、校验工具和第三方生态接入提供统一依据。

### 仓库定位

这个仓库会作为 Ring Engine 文档与标准的对外发布位置，主要承载：

- 面向作者、工具链和 runtime 的格式规范
- 可机读的 schema 与约束定义
- 面向实现者的行为约定与兼容性说明
- 随版本演进维护的设计说明、示例和迁移文档

### 后续规划

接下来会继续围绕以下方向完善：

- 补全 filter 运行时包格式、编译产物约束与兼容性规则
- 增加更多示例、约定说明和最佳实践
- 逐步补齐除 filter 之外的 Ring Engine 其他核心模块文档
- 建立更完整的版本化发布方式，让文档、schema 和实现演进保持同步

整体上，这个仓库会从“先把关键标准写清楚”出发，逐步发展为 Ring Engine 的主文档中心。

## English

This repository is the main publishing home for Ring Engine documentation and specifications.

Its current purpose is to organize the most foundational documentation assets in the Ring Engine ecosystem and maintain them in a form that is publishable, referenceable, and easy to evolve over time. For now, we are focusing on filter-related materials first. Additional modules, formats, interface contracts, and engineering conventions will be added incrementally afterward.

### Current Progress

The repository already includes an initial set of filter-related documents and schemas:

- `FILTER_SRC.md`: the `filter-src` authoring project format specification in English
- `FILTER_SRC.zh-CN.md`: the `filter-src` authoring project format specification in Chinese
- `schemas/filter-src.schema.json`: the manifest schema for `filter-src`
- `schemas/filter.schema.json`: the schema for the runtime `filter` package

These assets provide a shared foundation for compilers, runtimes, editor tooling, validation tools, and third-party ecosystem integrations.

### Repository Role

This repository will serve as the public release location for Ring Engine docs and standards, including:

- format specifications for authors, toolchains, and runtimes
- machine-readable schemas and constraints
- behavioral contracts and compatibility notes for implementers
- versioned design notes, examples, and migration guidance

### Roadmap

The next steps are to continue expanding in these areas:

- complete the runtime filter package format, compiler outputs, and compatibility rules
- add more examples, conventions, and best practices
- document additional Ring Engine core modules beyond filters
- establish a clearer versioned publishing workflow so docs, schemas, and implementations can evolve together

Over time, this repository is intended to grow from a focused standards seed into the primary documentation hub for Ring Engine.
