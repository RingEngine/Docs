# Filter 资源包

本文定义 `filter` 资源包格式。

`filter` 资源包是编译后滤镜产物的发布格式。

它承载 Lua 脚本、编译后的 shader、资源文件，以及读取这些内容所需的加密 entry 清单。

`filter` 资源包的文件扩展名是 `.rfp`。

## 总体结构

一个 `filter` 资源包按顺序由以下部分组成：

```text
magic
formatVersion
plainMetadata
encryptedEntryList
encryptedEntry_0
encryptedEntry_1
encryptedEntry_2
...
```

`plainMetadata` 是固定长度头。

`encryptedEntryList` 是加密后的资源包 entry 清单。

每个 `encryptedEntry_n` 是一个独立加密的 entry 内容块。

## `magic`

`magic` 是固定的 4 字节 ASCII 标识：

```text
RPKG
```

## `formatVersion`

`formatVersion` 是资源包格式版本号。

它决定：

- 明文头的固定布局
- 签名算法
- 密钥派生算法
- 内容加密算法
- 压缩算法
- `encryptedEntryList` 的解码规则

不支持该 `formatVersion` 的实现必须拒绝加载该资源包。

## `plainMetadata`

`plainMetadata` 是固定长度头。

它包含：

- `flags`
- `salt`
- `entryCount`
- `entryListLength`
- `signature`

### `flags`

`flags` 是固定宽度位字段。

当前定义：

- bit 0：资源包包含签名

### `salt`

`salt` 是固定长度的随机字节序列。

它用于资源包内容密钥派生。

### `entryCount`

`entryCount` 是资源包内的 entry 数量。

### `entryListLength`

`entryListLength` 是 `encryptedEntryList` 的字节长度。

### `signature`

`signature` 是固定长度签名字节区。

当 `flags` 表示资源包未签名时，该区域全为零字节。

当 `flags` 表示资源包已签名时，该区域包含签名字节。

## 签名

资源包签名使用以下固定算法组合：

- `RSA-PSS`
- `SHA-256`
- 2048-bit key

签名覆盖以下字节：

- `magic`
- `formatVersion`
- `plainMetadata` 中除 `signature` 自身之外的所有字段
- `encryptedEntryList`
- 所有 `encryptedEntry_n`

签名验证只使用：

- 公钥
- 签名字节
- 被签名字节

签名验证不使用 `salt`。

## 内容密钥派生

内容密钥派生使用以下固定算法：

- `HKDF-SHA-256`

资源包主密钥由以下内容共同派生：

- 调用方提供的主密钥
- `salt`

每个 entry 的内容密钥由以下内容共同派生：

- 资源包主密钥
- `entryId`

`encryptedEntryList` 使用独立派生出的清单密钥加密。

## 内容加密

内容加密使用以下固定算法：

- `AES-256-GCM`

`encryptedEntryList` 和每个 `encryptedEntry_n` 都是独立的 `AES-GCM` 密文块。

每个加密块都有自己的 nonce。

## 压缩

entry 内容压缩使用以下固定算法：

- `DEFLATE RAW`

每个 entry 的原始字节先压缩，再加密。

## `encryptedEntryList`

`encryptedEntryList` 是整个资源包的加密 entry 清单。

它解密后的结构按顺序由以下部分组成：

```text
entryListHeader
entryOffsetTable
entryRecord_0
entryRecord_1
entryRecord_2
...
```

### `entryListHeader`

`entryListHeader` 包含：

- `entryCount`

### `entryOffsetTable`

`entryOffsetTable` 包含每个 `entryRecord` 在 records 区域中的偏移量。

偏移量顺序与 entry 的逻辑顺序一致。

### `entryRecord`

每个 `entryRecord` 包含：

- `entryIdLength`
- `entryId`
- `offset`
- `encryptedSize`
- `originalSize`
- `compression`
- `nonce`

#### `entryId`

`entryId` 是资源的逻辑路径标识。

在本格式中，`entryId` 就是资源定位符。

#### `offset`

`offset` 是该 entry 密文块相对于 payload 区起始位置的偏移量。

#### `encryptedSize`

`encryptedSize` 是该 entry 密文块的字节长度。

#### `originalSize`

`originalSize` 是该 entry 解密并解压后的原始内容字节长度。

#### `compression`

`compression` 是压缩方法标识。

当前值固定为 `DEFLATE RAW`。

#### `nonce`

`nonce` 是该 entry 的固定长度加密 nonce。

## `encryptedEntry_n`

每个 `encryptedEntry_n` 对应一个逻辑文件。

它的内容是：

- 原始文件字节
- 经 `DEFLATE RAW` 压缩
- 再经 `AES-256-GCM` 加密

## 读取顺序

资源包的读取顺序如下：

1. 读取 `magic`
2. 读取 `formatVersion`
3. 读取 `plainMetadata`
4. 按 `formatVersion` 规则解释格式
5. 如存在签名，则验证签名
6. 使用调用方提供的主密钥与 `salt` 派生资源包主密钥
7. 解密 `encryptedEntryList`
8. 定位并解密目标 entry
9. 解压解密后的字节，得到原始内容

## 明文暴露范围

资源包中仅以下信息以明文可见：

- `magic`
- `formatVersion`
- `plainMetadata`

逻辑文件路径、entry 布局以及 entry 清单都不以明文暴露。
