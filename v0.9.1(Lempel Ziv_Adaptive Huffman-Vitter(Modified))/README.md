# v0.9.1 — LZ77 + Adaptive Huffman-Vitter (Modified)

> **Status:** Stable  
> **Language:** C  
> **Algorithm:** LZ77 + Adaptive Huffman (Vitter — modified slide encoding, from v0.7.1)

---

## Overview

This version is identical to v0.9 (LZ77 + basic Vitter) but uses the **modified Vitter sliding algorithm** from v0.7.1 instead of the basic Vitter from v0.7. The modification changes `slideNodeVitter` to use strict frequency equality (`==`) rather than the adjusted frequency comparison (`>=`), and adds an extra post-increment slide for internal nodes.

---

## Architecture

```
v0.9.1(Lempel Ziv_Adaptive Huffman-Vitter(Modified))/
├── uTypes.h
├── BitIO.h / BitIO.c
├── Huffman.h / Huffman.c
├── AdaptiveHuffman.c
├── Vitter.c                               # MODIFIED Vitter (from v0.7.1)
├── AdaptiveHuffmanVitter.c                # Same API as v0.9
├── LempelZivList.h / LempelZivList.c
├── LZ_AdaptiveHuffmanVitter_Compressor.c
└── LZ_AdaptiveHuffmanVitter_DeCompressor.c
```

---

## What Changed vs v0.9

The only difference is in `Vitter.c`:

| Aspect | v0.9 (Basic) | v0.9.1 (Modified) |
|---|---|---|
| `slideNodeVitter` comparison | `nodeFrequency >= candidate.freq` | `node.freq == candidate.freq` |
| Post-increment slide for internals | No | Yes (extra `slideNodeVitter` call) |
| `Vitter.c` size | 2862 bytes | 2606 bytes |

**Modified `slideNodeVitter`:**
```c
// v0.9.1 — strict equality, no adjusted frequency
for (i = node->number + 1; i < rootNodeNumber; i++) {
    if (node->frequency == nodeAddress_Number[i]->frequency) {
        if (nodeAddress_Number[i] != node->parent)
            swapNodes(node, nodeAddress_Number[i]);
    } else break;
}
```

**Modified `slideIncrementVitter`:**
```c
slideNodeVitter(*node);            // pre-increment slide
(*node)->frequency++;              // increment
if ((*node)->ch == internalNode) {
    temp = (*node)->parent;
    slideNodeVitter(*node);        // post-increment slide for internal nodes
}
// Move up...
```

---

## Unchanged from v0.9

Everything else in the pipeline is identical:
- LZ77 sliding window and search logic
- 3-tier match classification (literals, 2-byte matches, ≥3-byte matches)
- `AdaptiveHuffmanVitter.c` API (`adaptiveHuffman_initFirstNode`, `encodeSymbol`, `decodeSymbol`)
- `LempelZivList.h/c`
- Header magic `"DEF"`
- Timing measurement via `clock()`

---

## Compression Behavior

The modified slide strategy produces a different tree shape than the basic Vitter. As a result, v0.9 and v0.9.1 compressed files are **not interchangeable** — they must be decoded by their respective decompressors. See v0.9.2 for a combined implementation that supports both.
