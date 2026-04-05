# v0.9.2 — LZ77 + Adaptive Huffman-Vitter (Basic + Modified, With Bugs)

> **Status:** Buggy / Experimental  
> **Language:** C  
> **Algorithm:** LZ77 + Adaptive Huffman using asymmetric Vitter encoding/decoding (from v0.7.2)

---

## Overview

This version mirrors v0.7.2 at the LZ pipeline level — it uses the **asymmetric** Vitter strategy where the encoder uses the **basic (v0.7) slide** and the decoder uses the **modified (v0.7.1) slide**. Since both sides use different tree update strategies, the encoder and decoder trees diverge after the first few symbols, producing **incorrect decompressed output**.

This exists as a documented research artifact showing the impact of mismatched encode/decode strategies.

---

## Architecture

```
v0.9.2(Lempel Ziv_Adaptive HuffmanVitter(Basic and Modified(With Bugs)))/
├── uTypes.h
├── BitIO.h / BitIO.c
├── Huffman.h / Huffman.c
├── AdaptiveHuffman.c
├── Vitter.c                              # Dual strategy (from v0.7.2)
├── AdaptiveHuffmanVitter.c               # MODIFIED: asymmetric encode/decode API
├── LempelZivList.h / LempelZivList.c
├── LZ_AdaptiveHuffmanVitter_Compressor.c
└── LZ_AdaptiveHuffmanVitter_DeCompressor.c
```

---

## Key Difference: `AdaptiveHuffmanVitter.c`

The high-level API is **split into separate encoder and decoder init functions**, using different Vitter update functions for each side:

```c
void adaptiveHuffman_encodeSymbol(int c) {
    // encode...
    updateTreeVitterEncoding(c);   // ← Basic Vitter (v0.7)
}

int adaptiveHuffman_decodeSymbol(void) {
    // decode...
    updateTreeVitterDecoding(c);   // ← Modified Vitter (v0.7.1)
}

void adaptiveHuffman_initFirstNodeEncoder(int c) {
    // init...
    updateTreeVitterEncoding(c);   // ← Basic Vitter
}

void adaptiveHuffman_initFirstNodeDecoder(int c) {
    // init...
    updateTreeVitterDecoding(c);   // ← Modified Vitter
}
```

**Note:** The single `adaptiveHuffman_initFirstNode` from v0.9/v0.9.1 is split into two separate entry points for encoder and decoder.

---

### `Vitter.c` — Dual Strategy (from v0.7.2)

Contains both `slideNodeVitterEncoding`/`slideIncrementVitterEncoding`/`updateTreeVitterEncoding` (the old basic slide from v0.7) **and** `slideNodeVitterDecoding`/`slideIncrementVitterDecoding`/`updateTreeVitterDecoding` (the modified slide from v0.7.1).

See [v0.7.2 README](../../v0.7.2(Adaptive%20Huffman-Vitter(Basic%2BModified%20Encodings)/README.md) for the full function breakdown.

---

## Bug Summary

Because `updateTreeVitterEncoding` (basic) and `updateTreeVitterDecoding` (modified) produce different tree orderings after symbols are processed:

1. After a few symbols, encoder's `huffmanList[c]` and decoder's `huffmanList[c]` map to **different tree nodes**
2. The bit codes emitted by the encoder no longer correspond to valid paths in the decoder tree
3. Decompressed output becomes garbled after initial bytes

---

## Comparison: v0.9, v0.9.1, v0.9.2

| Version | Encoder strategy | Decoder strategy | Round-trip valid? |
|---|---|---|---|
| v0.9 | Vitter Basic | Vitter Basic | ✅ Yes |
| v0.9.1 | Vitter Modified | Vitter Modified | ✅ Yes |
| v0.9.2 | Vitter Basic | Vitter Modified | ❌ No (bug) |
