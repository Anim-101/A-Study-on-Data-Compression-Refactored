# v0.1 — Shannon-Fano Compression (With Bug)

> **Status:** Buggy / Initial Implementation  
> **Language:** C  
> **Algorithm:** Static Shannon-Fano Coding

---

## Overview

This is the first version of the project — a static (non-adaptive) Shannon-Fano compression system implemented in C. It contains a known bug in the tree-building step where the frequency accumulation for the `secondChild` uses `=+` instead of `+=`, which causes incorrect frequency propagation and corrupted compressed output.

---

## Architecture

```
v0.1(ShannonFano with Bug)/
├── uTypes.h                    # Primitive type aliases (uchar, uint, ulong)
├── BitIO.h                     # BitIO interface declarations
├── BitIO.c                     # Bit-level buffered I/O engine
├── ShannonFano.h               # Shannon-Fano tree struct & function declarations
├── ShannonFano.c               # Shannon-Fano tree construction & encode/decode logic
├── shannonFanoCompressor.c     # Main: compress entry point
└── shannonFanoDeCompressor.c   # Main: decompress entry point
```

---

## Module Breakdown

### `uTypes.h`
Defines short aliases for primitive C types:
- `uchar` → `unsigned char`
- `uint`  → `unsigned int`
- `ulong` → `unsigned long`

---

### `BitIO.h` / `BitIO.c` — Bit-Level Buffered I/O

The I/O engine. All compressed data is written and read bit-by-bit through memory-buffered streams to avoid costly per-byte `fputc`/`fgetc` system calls.

**Key globals:**
| Variable | Role |
|---|---|
| `gIN` / `pOUT` | Global input/output `FILE*` pointers |
| `gbuf` / `pbuf` | Heap-allocated read/write buffers |
| `g_cnt` / `p_cnt` | Bit cursor within current byte (0–7) |
| `pBUFSIZE` / `gBUFSIZE` | Buffer sizes (default 8192 bytes) |

**Key functions:**
| Function | Description |
|---|---|
| `init_putBuffer()` | Allocates output buffer, retries with smaller size on failure |
| `init_getBuffer()` | Allocates and pre-fills input buffer from file |
| `get_bit()` | Reads one bit from the input buffer |
| `put_ZERO()` / `put_ONE()` | Writes a 0 or 1 bit to output buffer |
| `gfgetc()` | Reads a full byte from the buffered input |
| `pfputc(c)` | Writes a full byte to the buffered output |
| `get_nbits(n)` / `put_nbits(k, n)` | Multi-bit read/write (used for header fields) |
| `get_symbol(n)` | Like `get_nbits` but returns `EOF` on end of stream |
| `flush_putBuffer()` | Flushes remaining partial byte + buffer to disk |
| `get_nbytes_out()` | Returns total bytes written so far |

---

### `ShannonFano.h` / `ShannonFano.c` — Tree Engine

Implements the static Shannon-Fano algorithm using a sorted linked-list and a statically-allocated node pool.

**Key data structures:**
```c
typedef struct listnode {
    unsigned long frequency;    // Symbol or group frequency
    int ch;                     // Symbol character (-1 = internal node)
    struct listnode *next;      // Next node in sorted list
    struct listnode *parent;    // Parent in tree
    struct listnode *firstChild;  // Left child (bit = 0)
    struct listnode *secondChild; // Right child (bit = 1)
} listnode_t;

typedef struct {
    unsigned long f;  // Frequency count
    unsigned char b;  // Symbol byte value
} shannonFanoFrequency_Type;
```

**Static node pool:**
```c
listnode_t shannonFanoNodes[ShannonFano_MAX * 3];
// ShannonFano_MAX = 256 symbols × 3 = 768 total node slots
```

**Key functions:**
| Function | Description |
|---|---|
| `init_shannonFanoFrequency()` | Zeros out frequency table for all 256 symbols |
| `init_shannonFanoList()` | Clears symbol→node pointer map |
| `createSymbolList()` | Creates sorted linked-list of symbols with `freq > 0` |
| `createShannonFanoTree(node)` | **Recursively** splits the node's list into two halves, building the tree. **⚠ Contains bug:** `node->secondChild->frequency=+d->frequency` should be `+=` |
| `sfcompress(node)` | Recursively traverses from leaf to root, emitting 0/1 bits |
| `sfdecompress(node)` | Iteratively traverses from root to leaf, consuming bits |

**Known Bug in `createShannonFanoTree`:**
```c
// Line 131 — WRONG:
node->secondChild->frequency =+ d->frequency;
// Should be:
node->secondChild->frequency += d->frequency;
```
This causes the `secondChild`'s frequency to always equal only the last added symbol's frequency, producing an unbalanced tree and corrupt compressed output.

---

### `shannonFanoCompressor.c` — Compressor Entry Point

**Flow:**
1. Opens input and output files from `argv[1]`, `argv[2]`
2. Calls `init_putBuffer()` to set up the output buffer
3. Scans the input file with `readStats()` to build the frequency table
4. Calls `createSymbolList()` → `createShannonFanoTree()` to build the tree
5. Writes a file header:
   - `fileStamp` struct: magic tag `"TSF"` + original file size
   - `shannonFanoCount`: number of distinct symbols
   - Frequency table entries for all symbols with `freq > 0`
6. Re-reads the input file, calling `sfcompress(shannonFanoList[c])` per byte
7. **Bug:** The `while` loop reads all bytes but only encodes the **last** byte (`sfcompress` is called outside the inner loop) — encoding loop bug
8. Calls `flush_putBuffer()` and reports compression ratio

---

### `shannonFanoDeCompressor.c` — Decompressor Entry Point

**Flow:**
1. Opens files, reads and validates the `fileStamp` header
2. Reads `shannonFanoCount` and reconstructs the frequency table from stored entries
3. Rebuilds the exact same Shannon-Fano tree as the compressor
4. Reads the compressed bitstream, calling `sfdecompress(top)` once per original symbol
5. Writes decoded bytes to the output file

---

## Data Flow Diagram

```
Input File
    │
    ▼
[readStats] ──► frequency table ──► [createSymbolList] ──► sorted linked list
                                                                    │
                                                          [createShannonFanoTree]
                                                                    │
                                                              Shannon-Fano Tree
                                                                    │
[File Header Write] ──────────────────────────────────────────────►│
                                                                    │
[Input File Re-read] ──► sfcompress(leaf) ──► BitIO put ──► Output File
```

---

## Known Issues

| Issue | Location | Impact |
|---|---|---|
| `=+` instead of `+=` | `ShannonFano.c:131` | Incorrect tree; corrupted output |
| Encoding loop bug | `shannonFanoCompressor.c:128–136` | Only the last byte of each buffer chunk is encoded |

These issues are both fixed in **v0.1.1**.
