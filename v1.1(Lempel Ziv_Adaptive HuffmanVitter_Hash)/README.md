# v1.1 — LZ77 + Adaptive Huffman-Vitter + 3-Byte Hash Table (Basic)

> **Status:** Stable  
> **Language:** C  
> **Algorithm:** LZ77 (hash-accelerated) + Adaptive Huffman (Vitter — basic)

---

## Overview

This version applies the same **hash-based LZ77 acceleration** from v1.0 but replaces the FGK back-end with **Vitter's basic adaptive Huffman algorithm** (from v0.7). The architecture is identical to v1.0 with the single difference of using `AdaptiveHuffmanVitter.c` (calling `updateTreeVitter`) instead of `AdaptiveHuffmanFGK.c`.

---

## Architecture

```
v1.1(Lempel Ziv_Adaptive HuffmanVitter_Hash)/
├── uTypes.h
├── BitIO.h / BitIO.c
├── Huffman.h / Huffman.c
├── AdaptiveHuffman.c
├── Vitter.c                               # Basic Vitter (from v0.7)
├── AdaptiveHuffmanVitter.c                # Vitter-backed adaptive API
├── LempelZivHash.h / LempelZivHash.c      # 3-byte hash table (from v1.0)
├── LZ_AdaptiveHuffmanVitter_Hash_Compressor.c
└── LZ_AdaptiveHuffmanVitter_Hash_DeCompressor.c
```

---

## Module Breakdown

All modules are identical to v1.0 except the adaptive Huffman back-end:

| Module | Source |
|---|---|
| `uTypes.h` | Unchanged |
| `BitIO.h/c` | Same as v1.0 |
| `Huffman.h/c` | Same as v1.0 |
| `AdaptiveHuffman.c` | Same as v1.0 |
| `LempelZivHash.h/c` | Same as v1.0 |
| `Vitter.c` | Basic Vitter (v0.7) |
| `AdaptiveHuffmanVitter.c` | Calls `updateTreeVitter` |

---

### `AdaptiveHuffmanVitter.c` — Vitter-Backed API

```c
void adaptiveHuffman_initFirstNode(int c) {
    init_huffmanList();
    top = zeroNode = createNode();
    assignedNumber = rootNodeNumber;
    updateTreeVitter(c);     // ← Vitter
}

void adaptiveHuffman_encodeSymbol(int c) {
    if (huffmanList[c]) hcompress(huffmanList[c]);
    else { hcompress(zeroNode); put_nbits(c, huffmanSymbol_bitSize); }
    updateTreeVitter(c);     // ← Vitter
}

int adaptiveHuffman_decodeSymbol(void) {
    int c = hdecompress(top);
    if (c == zeroNodeSymbol) c = get_nbits(huffmanSymbol_bitSize);
    updateTreeVitter(c);     // ← Vitter
    return c;
}
```

---

## Algorithm Parameters (Same as v1.0)

| Constant | Value |
|---|---|
| `WIN_BUFFSIZE` | 16384 bytes |
| `NUM_POS_BITS` | 14 bits |
| `PAT_BUFFSIZE` | 259 bytes |
| `NUM_LEN_BITS` | 8 bits |
| `MIN_LEN` | 3 |
| `HASH_SHIFT` | 6 |

---

## Encoding Protocol (Same as v1.0)

| Match class | Prefix | Payload |
|---|---|---|
| Literal | `00` | 8-bit byte |
| Min match (=3) | `01` | 14-bit position |
| Long match (>3) | `1` | AH(Vitter) for `(len-4)` + 14-bit position |

---

## FGK vs Vitter Back-end at Scale

| Property | v1.0 (FGK Hash) | v1.1 (Vitter Basic Hash) |
|---|---|---|
| Adaptive Huffman | FGK | Vitter (basic slide) |
| Update call | `updateTreeFGK` | `updateTreeVitter` |
| Rest of architecture | Identical | Identical |
| Expected ratio difference | Similar; Vitter may vary | Input-dependent |
