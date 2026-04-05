# v1.2 — LZ77 + Adaptive Huffman-Vitter + 3-Byte Hash Table (Modified)

> **Status:** Stable  
> **Language:** C  
> **Algorithm:** LZ77 (hash-accelerated) + Adaptive Huffman (Vitter — modified slide)

---

## Overview

This is the **final version of the core compression algorithm** in this study. It combines:
- **LZ77** with the 3-byte hash-accelerated sliding window (from v1.0)
- **Adaptive Huffman with Vitter's modified sliding algorithm** (from v0.7.1 / v0.9.1)

The only difference from v1.1 is that `Vitter.c` uses the modified slide strategy (strict equality comparison, extra post-increment slide for internal nodes) instead of the basic one.

---

## Architecture

```
v1.2(Lempel Ziv_Adaptive HuffmanVitter_Hash(Modified)/
├── uTypes.h
├── BitIO.h / BitIO.c
├── Huffman.h / Huffman.c
├── AdaptiveHuffman.c
├── Vitter.c                               # MODIFIED Vitter (from v0.7.1)
├── AdaptiveHuffmanVitter.c                # Same API as v1.1
├── LempelZivHash.h / LempelZivHash.c      # Same 3-byte hash (from v1.0)
├── LZ_AdaptiveHuffmanVitter_Hash_Compressor.c
└── LZ_AdaptiveHuffmanVitter_Hash_DeCompressor.c
```

---

## Module Breakdown

All modules identical to v1.1 except `Vitter.c`:

| Module | Source |
|---|---|
| `uTypes.h`, `BitIO.h/c` | Unchanged |
| `Huffman.h/c`, `AdaptiveHuffman.c` | Unchanged |
| `LempelZivHash.h/c` | Same as v1.0/v1.1 |
| `AdaptiveHuffmanVitter.c` | Same API as v1.1 (calls `updateTreeVitter`) |
| **`Vitter.c`** | **Modified (v0.7.1 style)** |

---

### `Vitter.c` — Modified Slide (Key Difference)

**`slideNodeVitter` — modified:**
```c
// Strict equality, no adjusted-frequency comparison
for (i = node->number + 1; i < rootNodeNumber; i++) {
    if (node->frequency == nodeAddress_Number[i]->frequency) {
        if (nodeAddress_Number[i] != node->parent)
            swapNodes(node, nodeAddress_Number[i]);
    } else break;
}
```

**`slideIncrementVitter` — modified (extra post-increment slide for internals):**
```c
slideNodeVitter(*node);           // pre-increment slide
(*node)->frequency++;             // increment
if ((*node)->ch == internalNode) {
    temp = (*node)->parent;
    slideNodeVitter(*node);       // ← EXTRA slide for internal nodes
}
if ((*node)->ch > internalNode)
    *node = (*node)->parent;
else
    *node = temp;
```

---

## Algorithm Parameters (Same as v1.0 and v1.1)

| Constant | Value |
|---|---|
| `WIN_BUFFSIZE` | 16384 bytes |
| `NUM_POS_BITS` | 14 bits |
| `PAT_BUFFSIZE` | 259 bytes |
| `NUM_LEN_BITS` | 8 bits |
| `MIN_LEN` | 3 |

---

## Version Lineage Summary

| Version | LZ Index | Adaptive Huffman Variant |
|---|---|---|
| v0.8 | Linked list (4096 window) | FGK |
| v0.9 | Linked list (4096 window) | Vitter Basic |
| v0.9.1 | Linked list (4096 window) | Vitter Modified |
| v1.0 | Hash table (16384 window) | FGK |
| v1.1 | Hash table (16384 window) | Vitter Basic |
| **v1.2** | **Hash table (16384 window)** | **Vitter Modified** |

---

## Notes

- v1.1 and v1.2 produce **different compressed files** and must be decompressed by their respective decompressors
- v1.2 is the most refined version: largest window, hash-accelerated matching, and the post-increment slide refinement
- All pre-compiled `.exe` versions are bundled in `v2.0(All Exes to Run)/`
