# v0.3 — Lempel-Ziv 77 Compression (Final)

> **Status:** Stable  
> **Language:** C  
> **Algorithm:** LZ77 (Sliding Window / Dictionary Compression)

---

## Overview

This version introduces **LZ77**, a lossless dictionary-based compression algorithm developed by Lempel and Ziv (1977). Instead of building a statistical model of symbol frequencies, LZ77 exploits **repetition** in the data: it maintains a sliding window of previously seen bytes and searches for the longest match between the current input and the window content. Matches are encoded as `(position, length)` pairs; otherwise, literal bytes are emitted.

LZ77 is a **single-pass** compressor (no frequency pre-scan), making it significantly different in architecture from Shannon-Fano and Huffman.

---

## Architecture

```
v0.3(Lempel Ziv 77 Final)/
├── uTypes.h               # Primitive type aliases
├── BitIO.h                # Bit I/O interface (includes multi-bit put_nbits)
├── BitIO.c                # Bit-level buffered I/O
├── LZ77.h                 # LZ77 linked-list structure declarations
├── LZ77.c                 # Sliding-window doubly-linked list management
├── LZ77Compressor.c       # Main: compressor entry point
└── LZ77DeCompressor.c     # Main: decompressor entry point
```

---

## Module Breakdown

### `uTypes.h`
Same as prior versions. Type aliases for `uchar`, `uint`, `ulong`.

---

### `BitIO.h` / `BitIO.c` — Bit I/O with Multi-Bit Support

This version re-introduces the full BitIO from v0.1, since LZ77 needs to write multi-bit fields (position and length) efficiently.

**Key additions over v0.1.1:**
- `get_nbits(n)` — reads `n` bits at once
- `put_nbits(k, n)` — writes `n` bits of value `k`
- `get_symbol(n)` — like `get_nbits` but returns `EOF` at end

**Retained from prior versions:**
- `init_putBuffer()`, `init_getBuffer()`, `flush_putBuffer()`
- `get_bit()`, `put_ZERO()`, `put_ONE()`, `gfgetc()`, `pfputc()`

---

### `LZ77.h` / `LZ77.c` — Sliding Window Index

The LZ77 sliding window is a circular byte buffer of size `WIN_BUFFSIZE = 4096` bytes. To search it efficiently, byte positions are indexed in a **character-indexed doubly-linked list**:

**Data structures:**
```c
int lzList[lzMax];           // 256 list heads, one per byte value
int lzPrevious[lzMaxBuffer]; // previous pointer for each window position
int lzNext[lzMaxBuffer];     // next pointer for each window position
```

Each entry at `lzList[c]` is the head of a linked list of window positions that currently hold byte value `c`. When a position slides out of the window, its node is deleted; when a new byte enters, a node is inserted.

**Key functions:**
| Function | Description |
|---|---|
| `initLZList()` | Sets all 256 list heads to `lzNull (-1)` |
| `insertLZNode(c, i)` | Inserts window position `i` (holding byte `c`) at head of `lzList[c]` |
| `deleteLZNode(c, i)` | Removes position `i` from `lzList[c]`, patching next/prev links |

---

### Algorithm Constants (`LZ77Compressor.c`)

| Constant | Value | Meaning |
|---|---|---|
| `WIN_BUFFSIZE` | 4096 | Sliding window size (bytes) |
| `NUM_POS_BITS` | 12 | Bits to encode match position (log2 4096) |
| `PAT_BUFFSIZE` | 18 | Lookahead (pattern) buffer size |
| `NUM_LEN_BITS` | 4 | Bits to encode match length offset |
| `MIN_MATCH_LEN` | 3 | Minimum match length to emit a `(pos, len)` token |
| `WIN_MASK` | 4095 | Mask for circular window indexing (`& WIN_MASK = % 4096`) |

---

### `LZ77Compressor.c` — Compressor Entry Point

**One-pass compression flow:**
1. Open files; write header: magic `"LZ77"` + original file size
2. Initialize sliding window (`winBuf`) to all spaces (`0x20`)
3. Pre-insert all 4096 window positions into `lzList`
4. Load initial `PAT_BUFFSIZE (18)` bytes into the pattern buffer
5. Loop: `search()` → `putCodes()` until input exhausted
6. `flush_putBuffer()`, report ratio

**`search(w, p)` — Match Finder:**
- Looks up `lzList[p[pat_cnt]]` to find all window positions with the same first byte
- Extends each candidate match byte-by-byte up to `buf_cnt` (remaining input)
- Keeps track of the longest match found
- Returns `pos_t { position, length }`

**`putCodes(pos)` — Token Emitter:**
- If `length < MIN_MATCH_LEN`: emit flag bit `1` + 8-bit literal byte
- If `length >= MIN_MATCH_LEN`: emit flag bit `0` + 4-bit `(length - 3)` + 12-bit `position`
- After encoding, slides the window forward: removes old positions, inserts new ones
- Reads next bytes from input into pattern buffer to stay full

---

### `LZ77DeCompressor.c` — Decompressor Entry Point

**Flow:**
1. Read and validate `"LZ77"` header
2. Initialize window (all spaces) — must match compressor initialization
3. Read flag bit:
   - `1` → next 8 bits = literal byte; copy to output and window
   - `0` → next 4 bits = length offset + next 12 bits = position; copy `length + MIN_MATCH_LEN` bytes from window position to output and slide window
4. Repeat until original file size bytes decoded

---

## Data Flow Diagram

```
Input File
    │
    ▼
[Initialize Window (4096 spaces)] + [lzList index]
    │
    ▼
┌──────────────────────────────────┐
│  search(window, pattern)         │   ← Finds longest match
│    → pos_t { position, length }  │
└───────────┬──────────────────────┘
            │
            ▼
┌──────────────────────────────────┐
│  putCodes(pos)                   │   ← Encodes token
│  if match: 0 + len(4b) + pos(12b)│
│  if literal: 1 + byte(8b)        │
└───────────┬──────────────────────┘
            │
            ▼
        BitIO Output Buffer
            │
            ▼
        Output File
```

---

## Comparison With Prior Versions

| Property | Shannon-Fano (v0.1.1) | Huffman (v0.2) | LZ77 (v0.3) |
|---|---|---|---|
| Algorithm type | Statistical | Statistical | Dictionary |
| Passes over input | 2 (scan + encode) | 2 (scan + encode) | 1 (single pass) |
| Model | Frequency table | Frequency table | Sliding window |
| Match encoding | N/A | N/A | `(position, length)` pairs |
| Literal encoding | Huffman code | Huffman code | 8-bit raw byte |
| Good for | Skewed distributions | Skewed distributions | Repetitive data |
