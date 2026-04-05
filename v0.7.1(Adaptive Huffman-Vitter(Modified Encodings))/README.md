# v0.7.1 — Adaptive Huffman: Vitter (Modified Encoding)

> **Status:** Stable  
> **Language:** C  
> **Algorithm:** Adaptive Huffman — Vitter's Algorithm (modified slide encoding)

---

## Overview

This version contains an alternative implementation of Vitter's Adaptive Huffman algorithm with a **modified `slideNodeVitter` and `slideIncrementVitter`** compared to v0.7. The modification changes how the sliding step handles internal nodes after incrementing, applying an additional slide after the increment to refine the tree ordering.

---

## Architecture

```
v0.7.1(Adaptive Huffman-Vitter(Modified Encodings))/
├── uTypes.h                  # Primitive type aliases
├── BitIO.h / BitIO.c         # Full multi-bit buffered I/O
├── Huffman.h / Huffman.c     # Base Huffman tree engine
├── AdaptiveHuffman.c         # Zero-node, swapNodes, numbering (same as v0.6/v0.7)
├── Vitter.c                  # MODIFIED Vitter update algorithm
├── Vitter_Compressor.c       # Compressor entry point
└── Vitter_DeCompressor.c     # Decompressor entry point
```

---

## Module Breakdown

All modules except `Vitter.c` are identical to v0.7:
- `uTypes.h`, `BitIO.h/c`, `Huffman.h/c`, `AdaptiveHuffman.c` — unchanged
- `Vitter_Compressor.c`, `Vitter_DeCompressor.c` — same flow, same magic header `"VAHF"`

---

### `Vitter.c` — Modified Algorithm (Key Change)

The modification is in `slideNodeVitter` and `slideIncrementVitter`:

**v0.7 `slideNodeVitter`** (original):
```c
// Compares with frequency+1 for leaves, frequency for internals
unsigned long nodeFrequency;
if (node->ch == internalNode)
    nodeFrequency = node->frequency + 1;
else
    nodeFrequency = node->frequency;

for (i = node->number + 1; ...) {
    if (nodeFrequency >= nodeAddress_Number[i]->frequency) {
        // Complex conditions...
        swapNodes(node, nodeAddress_Number[i]);
    }
}
```

**v0.7.1 `slideNodeVitter`** (modified — simplified):
```c
// Directly compares equal frequencies only
for (i = node->number + 1; i < rootNodeNumber; i++) {
    if (node->frequency == nodeAddress_Number[i]->frequency) {
        if (nodeAddress_Number[i] != node->parent) {
            swapNodes(node, nodeAddress_Number[i]);
        }
    } else {
        break;
    }
}
```
The modified version uses **strict equality** (`==`) rather than the `>=` comparison with adjusted frequency. This is a simpler sliding rule.

**v0.7.1 `slideIncrementVitter`** (modified):
```c
void slideIncrementVitter(huffmanNode_t **node) {
    huffmanNode_t *temp;
    slideNodeVitter(*node);        // slide before increment

    (*node)->frequency++;          // increment

    if ((*node)->ch == internalNode) {
        temp = (*node)->parent;
        slideNodeVitter(*node);    // ← EXTRA slide AFTER increment for internal nodes
    }

    if ((*node)->ch > internalNode)
        *node = (*node)->parent;
    else
        *node = temp;
}
```

The key addition is the **second `slideNodeVitter(*node)` call after incrementing** for internal nodes. In v0.7, the slide only happens before the increment. This modified version also slides internal nodes after their count increases, potentially producing a tighter ordering at the cost of more swaps.

---

## Behavioral Difference: v0.7 vs v0.7.1

| Aspect | v0.7 (Basic) | v0.7.1 (Modified) |
|---|---|---|
| `slideNodeVitter` comparison | `nodeFrequency >= candidate.freq` (adjusted) | `node.freq == candidate.freq` (strict equal) |
| Post-increment slide for internals | No | Yes |
| Leaf/internal ordering in slide | Complex exclusion logic | Simpler: skip only direct parent |
| Compression ratio | Standard Vitter | Slightly different (may vary per input) |
| File size (`Vitter.c`) | 2862 bytes | 2606 bytes (simpler logic) |

---

## Use Case

This version exists as an **experimental alternative** to understand how different sliding strategies affect compression ratio and tree stability. Both v0.7 and v0.7.1 are combined in v0.7.2 to enable side-by-side comparison.
