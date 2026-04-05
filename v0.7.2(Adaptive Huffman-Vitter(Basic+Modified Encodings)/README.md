# v0.7.2 — Adaptive Huffman: Vitter (Basic + Modified, Asymmetric)

> **Status:** Experimental / Research  
> **Language:** C  
> **Algorithm:** Adaptive Huffman — Vitter's Algorithm with **separate encoding and decoding update strategies**

---

## Overview

This version is the **most complex Vitter variant**, combining both the basic (v0.7) and modified (v0.7.1) sliding strategies into a single implementation — but using them **asymmetrically**: the **encoder uses the original (v0.7) slide strategy**, while the **decoder uses the modified (v0.7.1) slide strategy**.

This asymmetric design is an experimental attempt to test whether such a pairing produces valid (round-trippable) compressed output or whether the encoding/decoding must use identical strategies to produce a lossless codec.

The `Vitter.c` in this version is nearly double the size (4814 bytes vs ~2800 bytes) of previous versions as it contains both full implementations.

---

## Architecture

```
v0.7.2(Adaptive Huffman-Vitter(Basic+Modified Encodings)/
├── uTypes.h                  # Primitive type aliases
├── BitIO.h / BitIO.c         # Full multi-bit buffered I/O
├── Huffman.h / Huffman.c     # Base Huffman tree engine
├── AdaptiveHuffman.c         # Shared adaptive layer (zero-node, swapNodes)
├── Vitter.c                  # Combined Basic+Modified Vitter (asymmetric)
├── Vitter_Compressor.c       # Compressor: uses Encoding functions
└── Vitter_DeCompressor.c     # Decompressor: uses Decoding functions
```

---

## Module Breakdown

All base modules (`uTypes.h`, `BitIO.h/c`, `Huffman.h/c`, `AdaptiveHuffman.c`) are identical to v0.6/v0.7.

---

### `Vitter.c` — Dual-Strategy Implementation

Contains **two complete, parallel implementations** of the Vitter update algorithm:

#### Encoding side (v0.7 — basic):
```c
slideNodeVitterEncoding(node)        // Original slide: >= comparison, adjusted freq
slideIncrementVitterEncoding(&node)  // Original: slide → increment → move up
updateTreeVitterEncoding(c)          // Called by compressor
```

**`slideNodeVitterEncoding`:** Uses `nodeFrequency >= candidate.frequency` where `nodeFrequency = node.frequency + 1` for internal nodes. Includes special exclusion for two internal nodes of equal count.

**`slideIncrementVitterEncoding`:** Save parent → slide → increment → move pointer up.

#### Decoding side (v0.7.1 — modified):
```c
slideNodeVitterDecoding(node)        // Modified slide: strict == comparison
slideIncrementVitterDecoding(&node)  // Modified: slide → increment → extra slide → move up
updateTreeVitterDecoding(c)          // Called by decompressor
```

**`slideNodeVitterDecoding`:** Uses strict equality `node.frequency == candidate.frequency`, no adjusted frequency.

**`slideIncrementVitterDecoding`:** Slide → increment → if internal node: extra `slideNodeVitterDecoding` → move pointer up. This double-slide for internals is the key modification from v0.7.1.

#### Shared:
```c
getBlockLeaderVitter(node)   // Same as v0.7/v0.7.1: used by both sides
```

---

## Function Map

| Function | Encoding | Decoding |
|---|---|---|
| Tree update | `updateTreeVitterEncoding` | `updateTreeVitterDecoding` |
| Slide node | `slideNodeVitterEncoding` (v0.7) | `slideNodeVitterDecoding` (v0.7.1) |
| Slide+increment | `slideIncrementVitterEncoding` | `slideIncrementVitterDecoding` |
| Block leader | `getBlockLeaderVitter` | `getBlockLeaderVitter` (shared) |
| Extra post-increment slide | ❌ | ✅ for internal nodes |

---

## Compressor/Decompressor Split

- `Vitter_Compressor.c` calls `updateTreeVitterEncoding(c)` for each byte processed
- `Vitter_DeCompressor.c` calls `updateTreeVitterDecoding(c)` for each byte decoded

---

## Comparison Across Vitter Variants

| Version | Slide Strategy | Encoder | Decoder | Status |
|---|---|---|---|---|
| v0.7 | Basic (original) | Same | Same | Stable |
| v0.7.1 | Modified (simplified) | Same | Same | Stable |
| v0.7.2 | **Asymmetric** | Basic (v0.7) | Modified (v0.7.1) | Experimental |

---

## Notes

This asymmetric setup is primarily a **research experiment**. For a lossless codec to work correctly, encoder and decoder trees must remain in sync. Using different update strategies on each side will generally cause them to diverge, producing garbled output. This version documents that experimental path as part of the study on compression algorithm behavior.
