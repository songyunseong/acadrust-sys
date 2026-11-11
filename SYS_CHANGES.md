# SYS CAD changes to acadrust

This directory is a fork of the `acadrust` crate, used by SYS CAD as a path dependency.

- **Original crate:** acadrust 0.6.6 (crates.io)
- **crates.io checksum:** `8923c99ea1f151cb7fc17ce2fc14f8b1e330fa22611e535602d985c0cc3c89a2`
- **Upstream git commit (`.cargo_vcs_info.json`):** `5d5c482b3fcb3dc77e8281474461760f32959436`
- **Upstream repository:** https://github.com/hakanaktt/acadrust
- **License:** MPL-2.0 (see `LICENSE`). Modified files keep their MPL-2.0 headers.

SYS CAD stays on 0.6.6 rather than 0.6.7/0.6.8 because those releases can read some damaged DWGs as empty with no note (see `crates/dwg` comments).

## Changes

### Faster ASCII DXF writing (SYS CAD 0.2.7)

COEX mall (98 MB DWG) spent ~12.1 s of a ~12.5 s DWG→DXF conversion inside `DxfWriter`, producing a 332 MB ASCII DXF. DWG reading itself was only ~0.4 s. The cost is millions of small group-code writes and `f64` formatting (`write!(cursor, "{}", value)`).

In `src/io/dxf/writer/text_writer.rs`:

- Format ordinary finite doubles with `ryu` (already a crate dependency; previously unused by the ASCII writer) instead of `std::fmt` via `Cursor`. Non-finite values still become `0.0`; magnitudes `< 1e-15` or `>= 1e15` still use `{:.16e}` as before. Integer values still carry a decimal point (`15.0`).
- Batch writes through a 256 KiB `write_buf`, flushed on capacity, on `flush()`, on `into_inner()`, and on `Drop`. Output is the same sequence of bytes aside from any shorter scientific form `ryu` may choose; those values parse back to the same `f64`.

- In `src/io/dxf/writer/section_writer.rs` `write_ole2frame`: encode each 127-byte OLE payload chunk with `DxfStreamWriter::write_binary` instead of `chunk.iter().map(|b| format!("{:02X}", b)).collect::<String>()` plus `write_string`. The old path allocated one temporary `String` per payload byte. On the COEX mall drawing this was ~12 s of a 12.5 s DWG→DXF conversion (47 OLE2FRAME entities, ~150+ MB of payload written as ASCII hex). Hex digits remain uppercase; group codes and chunking are unchanged.

An MPL-2.0 Exhibit A notice was added at the top of each modified file (the upstream sources in 0.6.6 did not carry per-file headers). Temporary `ACADRUST_DXF_TIMES` section/entity timing is also present in the writer and is silent unless that environment variable is set.

### Workspace lock hygiene (SYS CAD 0.2.7)

[dev-dependencies] (criterion, proptest, serde_json) are commented out in this Cargo.toml so that using the crate as a path dependency does not rewrite the SYS CAD Cargo.lock with acadrust's test-only crates. The upstream entries remain in Cargo.toml.orig. Isolated acadrust tests can still be built from a standalone project that restores those dev-dependencies.
