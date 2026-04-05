# v0.9 — LZ77 + Adaptive Huffman-Vitter (Basic)

> **Status:** Stable  
> **Language:** C  
> **Algorithm:** LZ77 sliding window + Adaptive Huffman (Vitter — basic encoding)

---

## Overview

This version replaces the FGK adaptive Huffman back-end (v0.8) with **Vitter's algorithm** (from v0.7). The LZ77 front-end and the overall pipeline architecture are identical to v0.8. The only significant change is swapping `AdaptiveHuffmanFGK.c` for `AdaptiveHuffmanVitter.c`, which calls `updateTreeVitter` instead of `updateTreeFGK`.

---

## Architecture

```
v0.9(Lempel Ziv_Adaptive Huffman-Vitter)/
├── uTypes.h
├── BitIO.h / BitIO.c
├── Huffman.h / Huffman.c
├── AdaptiveHuffman.c                      # Same as v0.8
├── Vitter.c                               # Basic Vitter (from v0.7)
├── AdaptiveHuffmanVitter.c                # ✅ NEW: Vitter-backed adaptive API
├── LempelZivList.h / LempelZivList.c      # Same as v0.8
├── LZ_AdaptiveHuffmanVitter_Compressor.c
└── LZ_AdaptiveHuffmanVitter_DeCompressor.c
```

---

## Module Breakdown

### Unchanged Modules
All base modules are identical to v0.8:
- `uTypes.h`, `BitIO.h/c`, `Huffman.h/c`, `AdaptiveHuffman.c`, `LempelZivList.h/c`

---

### `Vitter.c` — Vitter Tree Update
Same as v0.7 (basic Vitter). Contains:
- `getBlockLeaderVitter(node)` — finds block leader
- `slideNodeVitter(node)` — slides node with `>=` adjusted-freq comparison
- `slideIncrementVitter(&node)` — slide + increment + move up
- `updateTreeVitter(c)` — full per-symbol update

---

### `AdaptiveHuffmanVitter.c` — High-Level Vitter API

Structurally identical to `AdaptiveHuffmanFGK.c` but calls Vitter's update function:

```c
void adaptiveHuffman_initFirstNode(int c) {
    init_huffmanList();
    top = zeroNode = createNode();
    assignedNumber = rootNodeNumber;
    updateTreeVitter(c);     // ← Vitter instead of FGK
}

void adaptiveHuffman_encodeSymbol(int c) {
    if (huffmanList[c]) hcompress(huffmanList[c]);
    else { hcompress(zeroNode); put_nbits(c, huffmanSymbol_bitSize); }
    updateTreeVitter(c);     // ← Vitter update
}

int adaptiveHuffman_decodeSymbol(void) {
    int c = hdecompress(top);
    if (c == zeroNodeSymbol) c = get_nbits(huffmanSymbol_bitSize);
    updateTreeVitter(c);     // ← Vitter update
    return c;
}
```

---

### `LZ_AdaptiveHuffmanVitter_Compressor.c` / `_DeCompressor.c`

**Identical pipeline to v0.8:**

| Step | Same as v0.8? |
|---|---|
| LZ77 sliding window (4096 bytes, zeros init) | ✅ |
| Pattern buffer (18 bytes) | ✅ |
| 3-tier match classification (1, 2, ≥3) | ✅ |
| Match length: adaptive Huffman encoded | ✅ |
| Match position: fixed 12-bit | ✅ |
| Literal: fixed 8-bit | ✅ |
| Header magic | `"DEF"` ✅ |
| Timing via `clock()` | ✅ |

---

## FGK vs Vitter Back-end

| Property | v0.8 (LZ + FGK) | v0.9 (LZ + Vitter Basic) |
|---|---|---|
| Adaptive Huffman variant | FGK | Vitter (basic) |
| Update function | `updateTreeFGK` | `updateTreeVitter` |
| Swap strategy | Highest equal, then swap | Block leader swap + slide |
| High-level API file | `AdaptiveHuffmanFGK.c` | `AdaptiveHuffmanVitter.c` |
| Everything else | Same | Same |
