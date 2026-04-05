# v0.7 — Adaptive Huffman Compression: Vitter Algorithm (Basic)

> **Status:** Stable  
> **Language:** C  
> **Algorithm:** Adaptive Huffman — Vitter's Algorithm (basic encoding)

---

## Overview

This version implements **Vitter's Adaptive Huffman algorithm**, an improvement over FGK (v0.6) that reduces the number of node swaps needed during tree updates. Vitter's algorithm introduces the concept of **sliding** — instead of finding a swap partner, a node is slid past other nodes of equal weight rather than directly swapped.

The key behavioral difference from FGK:
- **FGK:** Finds the highest equal node and swaps once
- **Vitter:** Uses `slideNodeVitter` to slide a node past all equal-weight nodes before incrementing, maintaining optimal ordering with fewer incremental swaps

This is the **basic encoding** variant (v0.7.1 introduces a modified/alternative encoding).

---

## Architecture

```
v0.7(Adaptive Huffman-Vitter)/
├── uTypes.h                  # Primitive type aliases
├── BitIO.h                   # Bit I/O interface
├── BitIO.c                   # Multi-bit buffered I/O
├── Huffman.h                 # Base Huffman node struct
├── Huffman.c                 # Base tree engine
├── AdaptiveHuffman.c         # Zero-node, swapNodes, node numbering
├── Vitter.c                  # Vitter-specific tree update algorithm
├── Vitter_Compressor.c       # Main: Vitter adaptive compressor
└── Vitter_DeCompressor.c     # Main: Vitter adaptive decompressor
```

---

## Module Breakdown

### Shared Modules (Identical to v0.6)

- **`uTypes.h`** — type aliases
- **`BitIO.h` / `BitIO.c`** — full bit I/O engine with multi-bit support
- **`Huffman.h` / `Huffman.c`** — base tree data structures, `createNode`, `insert`, `hcompress`, `hdecompress`
- **`AdaptiveHuffman.c`** — `createNewZeroNode`, `swapNodes`, `nodeAddress_Number[]` array

---

### `Vitter.c` — Vitter Tree Update Algorithm

The core algorithmic difference from FGK. All three functions implement Vitter's "slide-and-increment" update strategy.

**`getBlockLeaderVitter(node)`:**  
Finds the highest-numbered node with the same frequency as `node` that is not its direct parent. Unlike FGK's version, this checks for a special exclusion: if `node` is a leaf (`ch > internalNode`) and the candidate is internal (`ch == internalNode`), the candidate is skipped. This preserves Vitter's block ordering rule where internal nodes precede leaves of the same weight.

```c
// Exclusion: don't swap a leaf with an internal node of same weight
if (!(node->ch > internalNode && nodeAddress_Number[i]->ch == internalNode)) {
    high = nodeAddress_Number[i];
}
```

**`slideNodeVitter(node)`:**  
Slides `node` forward (toward higher numbers = closer to root) past all nodes whose frequency is ≥ `node`'s frequency+1 (for leaves) or = `node`'s frequency (for internal nodes). Each slide is done via `swapNodes`. This effectively positions the node at the correct place in the ordering **before** incrementing.

**`slideIncrementVitter(node*)`:**  
Combines slide + increment in one operation:
1. Save `parent = node->parent`
2. Call `slideNodeVitter(node)` — slide node to its correct position
3. Increment `node->frequency`
4. If `node` is a leaf: move pointer to `parent` for continued upward walk
5. If `node` is internal: move pointer to `node->parent` (post-slide position)

**`updateTreeVitter(c)`:**  
Main update, called once per encoded/decoded symbol:
1. If `c` is new: `createNewZeroNode(c)` → set `node = zeroNode->parent`, `increment = node->secondChild` (the new symbol)
2. If `c` is known:
   - Find block leader via `getBlockLeaderVitter`
   - Swap with block leader if found
   - If `c` is sibling of zero-node: set `increment = node`, `node = node->parent`
3. Walk from `node` to `top`: call `slideIncrementVitter(&node)` at each step
4. Apply `slideIncrementVitter(&increment)` for the new symbol if needed

---

### `Vitter_Compressor.c` — Compressor Entry Point

**Flow (one-pass adaptive):**
1. Open files; write header: magic `"VAHF"` + original file size
2. Initialize tree: zero-node at `rootNodeNumber`, `nodeAddress_Number[rootNodeNumber] = top`
3. For each input byte `c`:
   - If new: emit zero-node code + 8-bit raw `c`
   - If known: `hcompress(huffmanList[c])`
   - `updateTreeVitter(c)`
4. Flush, print statistics

---

### `Vitter_DeCompressor.c` — Decompressor Entry Point

**Flow (mirrors compressor):**
1. Validate `"VAHF"` header
2. Initialize same tree state
3. Until original size bytes decoded:
   - Traverse tree via bits from `get_bit()`
   - If zero-node reached: read 8-bit literal = new symbol
   - Else: `c = node->ch`
   - Write `c`, call `updateTreeVitter(c)`

---

## FGK vs Vitter: Algorithm Comparison

| Property | FGK (v0.6) | Vitter (v0.7) |
|---|---|---|
| Update strategy | Find highest equal → swap once | Slide node past all equals, then increment |
| Swap frequency | One swap per level per symbol | Multiple small slides, fewer total swaps |
| Internal/leaf ordering | Not enforced | Leaves and internals of same weight kept ordered |
| Block leader rule | Highest equal any node | Highest equal, excluding parent, with leaf/internal ordering |
| Key function | `getHighestEqualNodeFGK` | `slideNodeVitter` + `slideIncrementVitter` |
| Magic header | `"FAHF"` | `"VAHF"` |

---

## Data Flow

```
Input byte c
      │
      ├── Known? → hcompress(huffmanList[c]) → variable code bits
      │   New?  → hcompress(zeroNode) + put_nbits(c, 8)
      │
      └── updateTreeVitter(c):
               getBlockLeaderVitter → swap if needed
               slideIncrementVitter (walk to root)
                   └── slideNodeVitter → reposition
                       + increment frequency
```
