# v0.1.1 — Shannon-Fano Compression (Final / Bug Fixed)

> **Status:** Stable  
> **Language:** C  
> **Algorithm:** Static Shannon-Fano Coding  
> **Fixes:** Both bugs present in v0.1 are resolved here

---

## Overview

This is the corrected and final version of the Shannon-Fano compressor. It fixes the two critical bugs from v0.1:
1. The `=+` typo in tree construction is corrected to `+=`.
2. The compression loop now properly encodes **every** byte read from the buffer, not just the last one.

The `BitIO.c` module was also significantly refactored: the large multi-bit I/O functions (`get_nbits`, `put_nbits`, `get_symbol`) were removed since Shannon-Fano only needs single-bit output (`put_ZERO`, `put_ONE`, `get_bit`), resulting in a much smaller and focused I/O layer.

---

## Architecture

```
v0.1.1(ShannonFano Final)/
├── uTypes.h                     # Primitive type aliases
├── BitIO.h                      # Bit I/O interface (simplified)
├── BitIO.c                      # Bit-level buffered I/O (streamlined version)
├── ShannonFano.h                # Shannon-Fano tree declarations
├── ShannonFano.c                # Shannon-Fano tree logic (bug fixed)
├── ShannonFanoCompressor.c      # Main: compressor (loop bug fixed)
└── ShannonFanoDeCompressor.c    # Main: decompressor
```

---

## Module Breakdown

### `uTypes.h`
Unchanged from v0.1. Provides `uchar`, `uint`, `ulong` type aliases.

---

### `BitIO.h` / `BitIO.c` — Streamlined Bit I/O

The BitIO module was simplified. The multi-bit functions (`get_nbits`, `put_nbits`, `get_symbol`) that were present in v0.1 are **removed** in this version, since Shannon-Fano only requires single-bit I/O.

**Retained functions:**
| Function | Description |
|---|---|
| `init_putBuffer()` | Allocates write buffer with retry logic |
| `init_getBuffer()` | Allocates and fills read buffer |
| `free_putBuffer()` / `free_getBuffer()` | Releases heap memory |
| `flush_putBuffer()` | Writes remaining bits/bytes to disk |
| `get_bit()` | Reads one bit from the input buffer |
| `put_ZERO()` / `put_ONE()` | Writes a single 0 or 1 bit |
| `gfgetc()` | Reads a full byte from input buffer |
| `pfputc(c)` | Writes a full byte to output buffer |
| `get_nbytes_out()` | Returns bytes written so far |

**Buffer sizes:** Default 8192 bytes for both read and write.

---

### `ShannonFano.h` / `ShannonFano.c` — Tree Engine (Fixed)

Identical in structure to v0.1 but with both bugs corrected.

**Data structures:** Same as v0.1 — `listnode_t` tree nodes and `shannonFanoFrequency_Type` frequency entries, backed by a static pool of `ShannonFano_MAX * 3 = 768` nodes.

**Fixed in `createShannonFanoTree`:**
```c
// v0.1 (BUGGY):
node->secondChild->frequency =+ d->frequency;

// v0.1.1 (FIXED):
node->secondChild->frequency += d->frequency;
```
Both children now correctly accumulate the total frequency of their subtrees, producing a properly balanced Shannon-Fano tree.

**Key functions (unchanged interface):**
| Function | Description |
|---|---|
| `init_shannonFanoFrequency()` | Zero-initializes the 256-entry frequency table |
| `init_shannonFanoList()` | Clears the symbol→node pointer array |
| `createSymbolList()` | Builds sorted linked list from non-zero frequency entries |
| `createShannonFanoTree(node)` | Recursively divides list into two frequency-balanced subtrees |
| `sfcompress(node)` | Leaf-to-root traversal emitting bits |
| `sfdecompress(node)` | Root-to-leaf traversal consuming bits |

---

### `ShannonFanoCompressor.c` — Compressor Entry Point (Fixed)

**Loop bug corrected:** The inner loop now calls `sfcompress` for every byte:
```c
// v0.1 (BUG — encodes only last byte of buffer):
while(in_i < nread) {
    c = (uchar)*(inputBuffer + in_i);
    ++in_i;
}
sfcompress(shannonFanoList[c]);  // ← outside loop!

// v0.1.1 (FIXED — encodes every byte):
while(in_i < nread) {
    c = (uchar)*(inputBuffer + in_i);
    ++in_i;
    sfcompress(shannonFanoList[c]);  // ← inside loop
}
```

**Compression flow:**
1. Open input/output files
2. Scan input → build frequency table
3. Build Shannon-Fano tree
4. Write file header: magic `"TSF"`, original file size, symbol count, frequency table
5. Re-read input; call `sfcompress(leaf)` per byte → bits go to output buffer
6. Flush and report ratio

---

### `ShannonFanoDeCompressor.c` — Decompressor Entry Point

**Flow:**
1. Open files
2. Read and validate `fileStamp` (magic = `"TSF"`)
3. Read frequency table and reconstruct tree using same algorithm
4. Read bitstream → call `sfdecompress(top)` once per original symbol
5. Write decoded byte to output

---

## Data Flow

```
Input File
    │
    ├──[Pass 1: readStats]──► Frequency Table ──► Sorted List ──► Shannon-Fano Tree
    │
    └──[Pass 2: encode]────► sfcompress(leaf) ──► BitIO ──► Output File
                                                              │
                                                     [Header: TSF + freqs]
```

---

## Changes from v0.1

| Change | v0.1 | v0.1.1 |
|---|---|---|
| Tree frequency bug | `=+` (wrong) | `+=` (fixed) |
| Encoding loop | `sfcompress` outside inner loop | `sfcompress` inside inner loop |
| BitIO complexity | Includes `get_nbits`, `put_nbits`, `get_symbol` | Simplified, single-bit only |
| File size (`BitIO.c`) | 7518 bytes (335 lines) | 2679 bytes (~110 lines) |
