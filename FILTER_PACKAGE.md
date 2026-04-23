# Filter Package

This document defines the `filter` package format.

A `filter` package is the publish format of compiled filter output. It carries Lua scripts, compiled shaders, resource files, and the encrypted entry directory required to load them.

The file extension of a `filter` package is `.rfp`.

## Overall Layout

A `filter` package is composed of the following parts in order:

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

`plainMetadata` is a fixed-size header.

`encryptedEntryList` is the encrypted package entry directory.

Each `encryptedEntry_n` is an independently encrypted entry payload.

## `magic`

`magic` is the fixed 4-byte ASCII identifier:

```text
RPKG
```

## `formatVersion`

`formatVersion` is the package format version.

It defines:

- the fixed layout of the plain header
- the signature algorithm
- the key derivation algorithm
- the content encryption algorithm
- the compression algorithm
- the decoding rule of `encryptedEntryList`

An implementation that does not support the current `formatVersion` must reject the package.

## `plainMetadata`

`plainMetadata` is a fixed-size header.

It contains:

- `flags`
- `salt`
- `entryCount`
- `entryListLength`
- `signature`

### `flags`

`flags` is a fixed-width bit field.

Current definition:

- bit 0: the package contains a signature

### `salt`

`salt` is a fixed-length random byte sequence.

It is used for package key derivation.

### `entryCount`

`entryCount` is the number of entries in the package.

### `entryListLength`

`entryListLength` is the byte length of `encryptedEntryList`.

### `signature`

`signature` is a fixed-length signature byte area.

When `flags` indicates that the package is unsigned, this area is filled with zero bytes.

When `flags` indicates that the package is signed, this area contains the signature bytes.

## Signature

The package signature uses the following fixed algorithm set:

- `RSA-PSS`
- `SHA-256`
- 2048-bit key

The signature covers:

- `magic`
- `formatVersion`
- all fields in `plainMetadata` except `signature` itself
- `encryptedEntryList`
- all `encryptedEntry_n`

Signature verification uses only:

- the public key
- the signature bytes
- the signed bytes

Signature verification does not use `salt`.

## Content Key Derivation

Content key derivation uses the following fixed algorithm:

- `HKDF-SHA-256`

The package master key is derived from:

- the caller-provided master key
- `salt`

Each entry content key is derived from:

- the package master key
- `entryId`

`encryptedEntryList` is encrypted with a separately derived manifest key.

## Content Encryption

Content encryption uses the following fixed algorithm:

- `AES-256-GCM`

`encryptedEntryList` and each `encryptedEntry_n` are independent `AES-GCM` ciphertext blocks.

Each encrypted block has its own nonce.

## Compression

Entry content compression uses the following fixed algorithm:

- `DEFLATE RAW`

The original bytes of each entry are compressed first and encrypted after compression.

## `encryptedEntryList`

`encryptedEntryList` is the encrypted entry directory of the whole package.

Its decrypted structure is composed of the following parts in order:

```text
entryListHeader
entryOffsetTable
entryRecord_0
entryRecord_1
entryRecord_2
...
```

### `entryListHeader`

`entryListHeader` contains:

- `entryCount`

### `entryOffsetTable`

`entryOffsetTable` contains the offset of each `entryRecord` within the records area.

The offset order matches the logical entry order.

### `entryRecord`

Each `entryRecord` contains:

- `entryIdLength`
- `entryId`
- `offset`
- `encryptedSize`
- `originalSize`
- `compression`
- `nonce`

#### `entryId`

`entryId` is the logical path identifier of a resource.

In this format, `entryId` is the resource locator.

#### `offset`

`offset` is the offset of the entry ciphertext block relative to the start of the payload area.

#### `encryptedSize`

`encryptedSize` is the byte length of the entry ciphertext block.

#### `originalSize`

`originalSize` is the byte length of the original entry content after decryption and decompression.

#### `compression`

`compression` is the compression method identifier.

The current value is fixed to `DEFLATE RAW`.

#### `nonce`

`nonce` is the fixed-length encryption nonce of the entry.

## `encryptedEntry_n`

Each `encryptedEntry_n` corresponds to one logical file.

Its content is:

- original file bytes
- compressed with `DEFLATE RAW`
- encrypted with `AES-256-GCM`

## Read Order

The package read order is:

1. Read `magic`
2. Read `formatVersion`
3. Read `plainMetadata`
4. Interpret the format according to `formatVersion`
5. Verify the signature when present
6. Derive the package master key from the caller-provided master key and `salt`
7. Decrypt `encryptedEntryList`
8. Locate and decrypt the target entry
9. Decompress the decrypted bytes to obtain the original content

## Plaintext Exposure

The only plaintext-visible package data is:

- `magic`
- `formatVersion`
- `plainMetadata`

Logical file paths, entry layout, and the entry directory are not exposed in plaintext.
