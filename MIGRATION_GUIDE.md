# Migration Guide: from bitcode to postcard

This guide documents migrating from `bitcode` to `postcard` for binary serde serialization. 

## Quick comparison

| Aspect | bitcode | postcard |
|---|---|---|
| Latest | 0.6.9 | 1.1.3 |
| Downloads (recent/total) | 3.0M/10.9M | 29.7M/74.4M |
| Reverse deps | 210 | 1482 |
| Direct deps | 1 | 2 |
| License | MIT OR Apache-2.0 | MIT OR Apache-2.0 |
| MSRV | 1.70 | (unspecified) |
| Format | Bitwise, optimized for compression; not format-stable across majors | Varint-based postcard wire format; stable since 1.0.0 |

## 1. Dependencies

- Remove: `bitcode = { version = "0.6.9", features = ["serde"] }` (if using serde)
- Add: `postcard = { version = "1.1", default-features = false, features = ["use-std"] }`

Notes:
- bitcode's serde integration is behind `"serde"` feature; its core API uses `Encode`/`Decode` traits.
- postcard is serde-first: use serde derives (`#[derive(Serialize, Deserialize)]`) which you already have.

## 2. Features

| bitcode | postcard | Notes |
|---|---|---|
| `serde` (gated) | enabled via serde integration; choose features on postcard | Use `default-features = false, features = ["use-std"]` for std alloc/Vec |
| `std` (default) | `use-std` (enables alloc/std helpers) | Disabling defaults avoids heapless/cas deps |

## 3. MSRV
- bitcode: 1.70
- postcard: not specified; works fine with modern Rust (1.85+ for edition 2024). No practical blocker.

## 4. License
Both are `MIT OR Apache-2.0` (dual). No compatibility concerns.

## 5. Key Differences

| Concern | bitcode (serde) | postcard |
|---|---|---|
| Serialization | `bitcode::serialize(&T) -> Result<Vec<u8>, Error>` | `postcard::to_allocvec(&T) -> Result<Vec<u8>, Error>` (requires `alloc/use-std`) |
| Deserialization | `bitcode::deserialize<'de, T>(&[u8]) -> Result<T, Error>` | `postcard::from_bytes<'de, T>(&[u8]) -> Result<T, Error>` |
| Traits | `Encode`/`Decode` (non-serde path) | serde `Serialize`/`Deserialize` |
| Wire format | Bitwise packed; not stable across major versions | Varint-based postcard spec; stable since 1.0.0 |
| Error types | bitcode's `Error` (serde::ser::Error + serde::de::Error impl) | postcard's `Error` (serde::ser::Error + serde::de::Error impl) |
| Borrowed data | Supports borrowed (`&str`, `&[u8]`) with careful lifetimes | Supports borrowed with lifetime `'de` tied to input slice |

## 6. Migration Steps

1. **Update Cargo.toml**
   ```toml
   [dependencies]
   serde = { version = "1.0", default-features = false, features = ["derive"] }
   postcard = { version = "1.1", default-features = false, features = ["use-std"] }
   ```

2. **Update imports**
   ```rust
   // replace
   use bitcode::{serialize, deserialize};
   // with
   use postcard::{to_allocvec as serialize, from_bytes as deserialize};
   ```
   Or just rename call sites. If you want minimal diff, aliasing works if signatures are compatible.

3. **Check derive macros** (if using non-serde path)
   - If you used `#[derive(bitcode::Encode, bitcode::Decode)]`, switch to serde: `#[derive(serde::Serialize, serde::Deserialize)]`. Ensure structs implement serde traits.

4. **Update call sites**
   ```rust
   // encode
   let bytes = bitcode::serialize(&value)?;
   // becomes
   let bytes = postcard::to_allocvec(&value)?;
   
   // decode
   let val = bitcode::deserialize::<T>(&bytes)?;
   // becomes
   let val = postcard::from_bytes::<T>(&bytes)?;
   ```

5. **Handle lifetimes** (if deserializing borrowed data)
   - postcard's `from_bytes` requires `T: Deserialize<'de>`; ensure lifetimes match the input slice. If you deserialize owned types, no change.

6. **Test compatibility of persisted data**
   - On-disk format differs: existing bitcode files are **not** readable by postcard. Plan for migration/reset (decode failure path). In the `learning.rs` case, decode failure returns fresh state — acceptable.

## 7. Behavioral/Pitfalls

- **Format incompatibility**: wire formats differ; treat as breaking for persisted data.
- **Size/compression**: bitcode is optimized for bit-packing and post-compression; postcard is varint-based (often smaller/simpler; may compress differently). Measure if size matters.
- **Error semantics**: both implement serde error traits; map accordingly.
- **Stability**: postcard spec is stable since 1.0.0; bitcode explicitly states "Stable format across major versions" is not a goal.
- **MSRV**: bitcode 0.6.9 requires 1.70; postcard unspecified but fine for 1.85+.
