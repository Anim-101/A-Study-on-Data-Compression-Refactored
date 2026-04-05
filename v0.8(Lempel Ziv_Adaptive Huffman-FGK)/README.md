# v0.8 — LZ77 + Adaptive Huffman-FGK (Deflate-style)

> **Status:** Stable  
> **Language:** C  
> **Algorithm:** LZ77 sliding window + Adaptive Huffman (FGK) for match-length encoding  
> **File stamp magic:** `"DEF"` (Deflate-inspired)

---

## Overview

This version combines **LZ77 dictionary compression** (v0.3) with **Adaptive Huffman coding using the FGK algorithm** (v0.6) in a pipeline reminiscent of the **Deflate** algorithm (used in gzip/zlib). The key improvement over standalone LZ77 is that the **match length** values are encoded using an adaptive Huffman code rather than raw fixed-width bits, allowing frequently-occurring lengths to be encoded with fewer bits.

The position field is still encoded as a fixed 12-bit value (since positions are uniformly distributed), but this hybrid approach is more efficient than pure LZ77 for data with regular repetition patterns.

---

## Architecture

```
v0.8(Lempel Ziv_Adaptive Huffman-FGK)/
├── uTypes.h                            # Primitive type aliases
├── BitIO.h / BitIO.c                   # Full multi-bit buffered I/O
├── Huffman.h / Huffman.c               # Base Huffman engine
├── FGK.c                               # FGK adaptive update algorithm
├── AdaptiveHuffman.c                   # Zero-node, swapNodes, numbering
├── AdaptiveHuffmanFGK.c                # ✅ NEW: High-level adaptive Huffman API
├── LempelZivList.h / LempelZivList.c   # Sliding window linked-list index
├── LZ_AdaptiveHuffmanFGK_Compressor.c # Main: compressor
└── LZ_AdaptiveHuffmanFGK_DeCompressor.c # Main: decompressor
```

> **Note:** Compared to pure LZ77 (v0.3), the `LZ77.h/c` modules are renamed to `LempelZivList.h/c` for clarity. The LZ77 function names are also updated (e.g., `insert_lempelZivNode` instead of `insertLZNode`).

---

## Module Breakdown

### Shared Base Modules
- **`uTypes.h`** — type aliases
- **`BitIO.h/c`** — buffered bit I/O (includes `get_nbits`, `put_nbits`)
- **`Huffman.h/c`** — base tree: `createNode`, node pool, `hcompress`, `hdecompress`
- **`AdaptiveHuffman.c`** — `createNewZeroNode`, `swapNodes`, `nodeAddress_Number[]`
- **`FGK.c`** — `updateTreeFGK`, `getHighestEqualLeafFGK`, `getHighestEqualNodeFGK`

---

### `LempelZivList.h` / `LempelZivList.c` — Sliding Window Index

Renamed from `LZ77.h/c` (v0.3). Same linked-list design: 256 character-indexed list heads + `lempelZivPrevious[]` / `lempelZivNext[]` arrays.

**Key functions:**
| Function | Description |
|---|---|
| `init_lempelZivList()` | Set all 256 list heads to `LempelZiv_NULL (-1)` |
| `insert_lempelZivNode(c, i)` | Insert window position `i` at head of `lempelZivList[c]` |
| `delete_lempelZivNode(c, i)` | Remove position `i` from `lempelZivList[c]` |

---

### `AdaptiveHuffmanFGK.c` — High-Level Adaptive Huffman API

A thin wrapper layer above `FGK.c` that provides a clean encode/decode interface for the length values.

**Global state:**
```c
int hmax_symbols;           // Max number of distinct length symbols (PAT_BUFFSIZE - 2)
int huffmanSymbol_bitSize;  // Bits to represent a new symbol (NUM_LEN_BITS = 4)
int hmin;                   // Minimum symbol value (0)
```

**Functions:**
| Function | Description |
|---|---|
| `adaptiveHuffman_initFirstNode(c)` | Initializes the tree: creates root/zero-node, then inserts symbol `c` via `updateTreeFGK` |
| `adaptiveHuffman_encodeSymbol(c)` | If `c` known: `hcompress(huffmanList[c])`. If new: `hcompress(zeroNode)` + `put_nbits(c, huffmanSymbol_bitSize)`. Then `updateTreeFGK(c)` |
| `adaptiveHuffman_decodeSymbol()` | `hdecompress(top)` → if `zeroNodeSymbol`: `get_nbits(huffmanSymbol_bitSize)` as new symbol. Then `updateTreeFGK(c)`. Returns symbol |

---

### `LZ_AdaptiveHuffmanFGK_Compressor.c` — Compressor

**Encoding protocol — 3-tier match classification:**

| Match length | Prefix bits | Payload |
|---|---|---|
| Length == 1 (literal) | `0 0` | 8-bit raw byte |
| Length == 2 (short match) | `0 1` | 12-bit window position |
| Length >= 3 (full match) | `1` | Adaptive Huffman code for `(length-3)` + 12-bit position |

**Flow:**
1. Open files; write header `"DEF"` + original file size
2. `init_lempelZivList()` + initialize window to all zeros (unlike v0.3 which uses spaces)
3. Pre-fill all 4096 window positions in `lempelZivList`
4. Load first `PAT_BUFFSIZE` bytes into pattern buffer
5. `init_getBuffer()`, then initialize Adaptive Huffman with `hmax_symbols = PAT_BUFFSIZE - 2`
6. `adaptiveHuffman_initFirstNode(0)` — prime the tree
7. Loop: `search()` → emit 2-bit prefix → `putCodes()`
8. In `putCodes`: if match ≥ 3 → `adaptiveHuffman_encodeSymbol(length - 3)` + 12-bit position; if match == 2 → just 12-bit position; if literal → 8-bit byte
9. Flush, print timing + ratio

**Timing added:** Uses `clock()` to measure compression time in seconds.

**Window initialization change:** Set to all `0x00` bytes (`memset(winBuf, 0, WIN_BUFFSIZE)`), not spaces like v0.3 (`memset(winBuf, 32, ...)`).

---

### `LZ_AdaptiveHuffmanFGK_DeCompressor.c` — Decompressor

**Flow:**
1. Validate `"DEF"` header
2. Initialize window (all zeros), `lempelZivList`, `adaptiveHuffman_initFirstNode(0)`
3. Loop until original size bytes decoded:
   - Read 2-bit prefix
   - `0 0` → read 8-bit literal byte → copy to output + slide window
   - `0 1` → read 12-bit position → copy 2 bytes from window → slide window
   - `1` → `adaptiveHuffman_decodeSymbol()` = length offset → `+3` = actual length; read 12-bit position; copy `length` bytes from window
4. Write decoded bytes, update window and lzList

---

## Data Flow

```
Input byte stream
       │
       ▼
   search() ─► match or literal?
                │
    ┌───────────┴────────────┐
    │ length >= 3            │ length == 2    │ length == 1 (literal)
    │ prefix: 1              │ prefix: 01     │ prefix: 00
    │ adaptiveHuffman_encode │ 12-bit pos     │ 8-bit literal
    │   (length - 3)         │                │
    │ 12-bit position        │                │
    └───────────┬────────────┘                │
                │                              │
                └──────────────────────────────┘
                                │
                            BitIO output
```

---

## Improvements Over v0.3 (Pure LZ77)

| Feature | v0.3 (LZ77) | v0.8 (LZ + FGK) |
|---|---|---|
| Length encoding | Fixed 4-bit raw | Adaptive Huffman (FGK) |
| Position encoding | Fixed 12-bit raw | Fixed 12-bit raw |
| Match length classes | 1 class | 3-tier (1, 2, ≥3) |
| Window initialization | Spaces (`0x20`) | Zeros (`0x00`) |
| Timing measurement | None | `clock()` output in seconds |
| LZ index naming | `lzList`, `lzNext` | `lempelZivList`, `lempelZivNext` |
| Header magic | `"LZ77"` | `"DEF"` |
