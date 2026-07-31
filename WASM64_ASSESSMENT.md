# Feasibility assessment: wasm64 (Memory64) support in duckdb-wasm

*Status: analysis only — no implementation. Written against upstream `main` (v1.5.5-dev).*

## Why

All current bundles are wasm32 and linked with `-s MAXIMUM_MEMORY=4GB`
(`lib/CMakeLists.txt`), so a query workload can never address more than 4 GB.
The Memory64 proposal (part of Wasm 3.0) lifts that: browsers currently allow
up to **16 GB** of wasm memory via the JS API.

## Ecosystem readiness (as of mid-2026)

| Piece | Status |
| --- | --- |
| Chrome / Edge | Shipped, default-on since Chrome 133 (Feb 2025), up to 16 GB |
| Firefox | Shipped, default-on since Firefox 134 (Jan 2025) |
| Safari | **Not supported** (through 26.x) |
| Node.js | Supported in current LTS (V8 ≥ 13.x) |
| Emscripten | `-sMEMORY64=1` non-experimental; the pinned toolchain (3.1.71 in `.github/workflows/main.yml`) supports it |
| Performance | 64-bit addressing disables the guard-page trick; engines emit explicit bounds checks. Expect roughly 10–30% slowdown depending on workload (see the SpiderMonkey "Is Memory64 actually worth using?" write-up) |

Two consequences fall out immediately:

1. wasm64 can only ever be an **additional opt-in bundle**, never a replacement —
   Safari has no support at all. The existing bundle-selection machinery
   (`packages/duckdb-wasm/src/platform.ts`, `wasm-feature-detect` already has a
   `memory64()` probe) makes this straightforward to slot in.
2. The prize is 16 GB, not "unlimited" — a 4× improvement, and the wasm32
   builds keep their performance edge for everyone who doesn't need it.

## What has to change

### 1. Build system — easy

Everything is compiled from source in-tree (DuckDB, Arrow, utf8proc, re2, …),
so an ABI-incompatible flag is just another build flavor:

- Add `-sMEMORY64=1` to `CMAKE_CXX_FLAGS` *and* `WASM_LINK_FLAGS` plus a raised
  `-sMAXIMUM_MEMORY` (e.g. 16GB) in `lib/CMakeLists.txt`, behind a
  `WITH_WASM_MEMORY64` option mirroring `WITH_WASM_THREADS`.
- New `DUCKDB_PLATFORM` string (e.g. `wasm_mem64`) and new targets in
  `Makefile` / CI, new dist entries (`duckdb-mem64.wasm`, worker, `.d.ts`).
- DuckDB core itself compiles cleanly on 64-bit pointers everywhere else, so
  few if any source patches are expected; the in-tree patches
  (`patches/duckdb`, `patches/arrow`) are not pointer-size specific.

Cost driver is mostly CI time (each flavor is a full DuckDB+Arrow build) and
one more entry in the release matrix.

### 2. JS/Wasm boundary — moderate, and the design already helps

Under `MEMORY64`, wasm pointers surface as `i64`/`BigInt` at raw boundaries.
Three things keep this tractable here:

- **The response protocol is double-packed.** `callSRet`
  (`packages/duckdb-wasm/src/bindings/runtime.ts`) and the C++ `WASMResponse`
  pack status/pointer/size into `HEAPF64` doubles — valid for addresses up to
  2^53, so the core call protocol survives wasm64 unchanged.
- **JS library imports are already signature-annotated.** `lib/js-stubs.js`
  uses `__sig: '…p…'` pointer annotations, so Emscripten auto-converts
  BigInt↔Number at those import boundaries.
- **Heap views still work.** `HEAPU8.subarray(begin, begin + len)` with number
  indices is fine below 2^53; engines back a 16 GB memory with large
  ArrayBuffers.

What does need auditing (~a day or two of focused work, low risk):

- `ccall` argument/return types: pointer args currently declared as
  `'number'` must become `'pointer'` where they cross as raw i64
  (≈50 call sites in `bindings_base.ts`, `udf_runtime.ts`).
- 32-bit bitwise pointer arithmetic: patterns like `(response >> 3)` in
  `runtime.ts` / `udf_runtime.ts` break for addresses ≥ 4 GB (JS bitwise ops
  truncate to 32 bits) — replace with `/ 8` or `Math.floor`.
- Any place that reads a **pointer stored in wasm memory** via a 32-bit view
  (UDF data-view / validity buffers in `udf_runtime.ts`,
  `lib/src/json_dataview.cc` counterparts) needs an 8-byte read path.
- `stackAlloc`/`stackSave`/`stackRestore` returns under MEMORY64 (Emscripten
  wraps these, but the `.d.ts` typings in `bindings/duckdb-*.d.ts` need
  `number | bigint` review).

The Rust shell (`packages/duckdb-wasm-shell`) is a separate wasm32 module that
talks to duckdb-wasm through the JS API only — unaffected.

### 3. Loadable extensions — the hard part

The flagship builds use Emscripten dynamic linking (`-s MAIN_MODULE=1`,
`lib/CMakeLists.txt`) and load extensions (parquet, json, httpfs, spatial, …)
as side modules served per-platform (`wasm_mvp` / `wasm_eh` / `wasm_threads`)
from `extensions.duckdb.org`. wasm64 breaks this in two ways:

- **Toolchain**: Emscripten's dynamic-linking support under `MEMORY64` exists
  but is far less battle-tested than wasm32 dylinking; expect to find and
  upstream bugs.
- **Ecosystem**: a `wasm64` platform must be added to DuckDB's
  `extension-ci-tools` build matrix, every extension rebuilt, signed and
  distributed for it. That is an upstream/organizational effort across the
  DuckDB extension ecosystem, not something this repo can do alone.

A pragmatic v1 sidesteps this entirely: ship the wasm64 bundle **statically
linked** with parquet + json (the non-`DUCKDB_WASM_LOADABLE_EXTENSIONS` path
that still exists in `lib/CMakeLists.txt`), with extension loading disabled.
Users who need >4 GB usually need memory, not the long tail of extensions.

## Difficulty summary

| Work item | Effort | Risk |
| --- | --- | --- |
| CMake/Makefile/CI flavor + dist packaging | ~1 week | Low |
| TS bindings pointer audit + typings | ~2–4 days | Low–medium |
| Bundle selection + docs (`platform.ts`, feature detect) | ~1 day | Low |
| Testing across Chrome/Firefox/Node, >4 GB workloads | ~1 week | Medium |
| Loadable extensions under wasm64 | Months (upstream coordination) | High |

**Bottom line:** a static-linked, opt-in `wasm_mem64` bundle giving 16 GB in
Chrome, Firefox and Node is a realistic **2–4 week** effort with the main cost
in build-matrix plumbing and a careful pointer audit — the double-packed
response protocol and `__sig`-annotated stubs mean the bindings were written
in a wasm64-friendly style already. Full parity (loadable extensions, COI/
threads variant) is a much larger, ecosystem-wide effort. Safari users keep
the wasm32 bundles either way, and the wasm32 default should remain because of
the Memory64 bounds-check performance penalty.
