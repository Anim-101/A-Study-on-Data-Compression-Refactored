# v0.6 — Adaptive Huffman Compression: FGK Algorithm

> **Status:** Stable  
> **Language:** C  
> **Algorithm:** Adaptive Huffman — Faller-Gallager-Knuth (FGK)

---

## Overview

This version introduces **Adaptive Huffman coding** using the **FGK algorithm** (named after Faller, Gallager, and Knuth). Unlike static Huffman (v0.2), adaptive Huffman **does not require a pre-scan** of the input — the Huffman tree is built and updated on the fly as each symbol is processed. Both encoder and decoder maintain identical trees that evolve in sync as symbols are seen.

The FGK variant maintains the **sibling property**: in the tree numbering schema, nodes with equal weight must be grouped together (siblings), and swaps are performed when incrementing a node's count would violate this ordering.

---

## Architecture

```
v0.6(Adaptive Huffman-FGK)/
├── uTypes.h              # Primitive type aliases
├── BitIO.h               # Bit I/O interface
├── BitIO.c               # Bit-level buffered I/O (with multi-bit support)
├── Huffman.h             # Shared Huffman node struct & declarations
├── Huffman.c             # Base Huffman tree: node pool, sorted list, traversal
├── AdaptiveHuffman.c     # Adaptive layer: zero-node, node swapping, numbering
├── FGK.c                 # FGK-specific tree update algorithm
├── FGK_Compressor.c      # Main: FGK adaptive compressor
└── FGK_DeCompressor.c    # Main: FGK adaptive decompressor
```

---

## Module Breakdown

### `uTypes.h`
Provides `uchar`, `uint`, `ulong` type aliases. Unchanged.

---

### `BitIO.h` / `BitIO.c`
Full bit I/O engine, identical to v0.3. Multi-bit functions (`get_nbits`, `put_nbits`, `get_symbol`) are active since adaptive Huffman needs to emit symbols via their number in the tree.

---

### `Huffman.h` / `Huffman.c` — Base Tree Engine

Provides the foundational data structures and utility functions shared by both static and adaptive Huffman variants.

**Data structure (`listnode_t`):**
```c
typedef struct listnode {
    unsigned long frequency;      // Node weight/frequency
    int ch;                       // Symbol (-1=internal, -2=zero-node)
    int number;                   // Node number in sibling ordering
    struct listnode *next;        // List traversal
    struct listnode *parent;      // Tree parent
    struct listnode *firstChild;  // Left child  (bit = 0)
    struct listnode *secondChild; // Right child (bit = 1)
} listnode_t;
```

**Static node pool:** `listnode_t huffmanNodes[Huffman_MAX * 2]` (512 nodes)  
**Symbol→node map:** `listp huffmanList[Huffman_MAX]` (256 entries)

**Key functions:**
| Function | Description |
|---|---|
| `createNode()` | Allocates next node from the static pool |
| `insert(list, node)` | Inserts into ascending-frequency sorted list |
| `init_huffmanList()` | Clears all symbol→node pointers to NULL |
| `init_huffmanFrequency()` | Zeros the 256-entry frequency table |
| `createSymbolList()` | (Used for static; not central in adaptive mode) |
| `hcompress(node)` | Leaf-to-root bit emission |
| `hdecompress(node)` | Root-to-leaf bit consumption |

---

### `AdaptiveHuffman.c` — Adaptive Layer

Extends `Huffman.c` with the adaptive mechanics shared by both FGK and Vitter.

**Key concepts:**

**Zero-node (`zeroNode`):** A special sentinel leaf that represents "unseen symbols." When a new symbol is first encountered, it expands the zero-node into an internal node with two children: the new symbol node (right) and a new zero-node (left).

**Node numbering:** Every node has a `number` field, arranged so that a higher number means higher position in the sibling-order. The root has `rootNodeNumber = Huffman_MAX * 2 = 512`. An array `nodeAddress_Number[513]` maps node numbers → node pointers for O(1) lookup.

**`createNewZeroNode(c)`:**
```
zeroNode (leaf, ch=-2)
    →  becomes internal node (ch=-1)
       ├── firstChild  = new zeroNode  (ch=-2, number--)
       └── secondChild = new symbol node (ch=c, number--)
```

**`swapNodes(a, b)`:** Swaps two nodes in the tree (updates parent pointers, firstChild/secondChild links) and swaps their entries in `nodeAddress_Number[]` and their `number` values.

---

### `FGK.c` — FGK Update Algorithm

Contains the FGK-specific tree update logic. Included directly into the compressor/decompressor via `#include`.

**Key functions:**

**`getHighestEqualLeafFGK(node)`:**  
Searches `nodeAddress_Number[]` for the highest-numbered node (i.e., closest to root) with the same frequency as `node`, that is also a **leaf** (not an internal node). Used before incrementing a leaf.

**`getHighestEqualNodeFGK(node)`:**  
Same search but accepts any node (leaf or internal). Used during the parent-walk phase.

**`updateTreeFGK(c)`:**  
Main update function called after encoding/decoding each symbol:
1. If symbol `c` is new: call `createNewZeroNode(c)` to expand the zero-node
2. If `c` is a sibling of the zero-node: find the highest equal leaf via `getHighestEqualLeafFGK`, swap if found, increment `c`'s frequency, move to parent
3. Walk from current node up to root: at each node, find `getHighestEqualNodeFGK`, swap if found, increment frequency, move to parent

---

### `FGK_Compressor.c` — Compressor Entry Point

**One-pass adaptive compression flow:**
1. Open files
2. Read file size; write header: magic `"FAHF"` + original file size  
   *(Note: uses a different magic tag to distinguish from static Huffman)*
3. Initialize adaptive Huffman tree: create root (zero-node) with `number = rootNodeNumber`
4. For each input byte `c`:
   - If `c` is new (not yet in tree): emit zero-node's code + raw 8-bit `c`
   - If `c` is known: emit `hcompress(huffmanList[c])`
   - Call `updateTreeFGK(c)` to update the tree
5. Flush output buffer, print statistics

---

### `FGK_DeCompressor.c` — Decompressor Entry Point

**Adaptive decompression flow (mirrors compressor exactly):**
1. Open files, read/validate header
2. Initialize identical tree state (same zero-node start)
3. Loop until original file size bytes decoded:
   - Start from root; traverse tree bit by bit via `get_bit()`
   - If we land on the zero-node: read next 8 bits = new symbol `c`; add to tree
   - Otherwise: `c` = `node->ch` (reached a leaf)
   - Write `c` to output
   - Call `updateTreeFGK(c)`

---

## Key Architectural Difference: Adaptive vs Static

| Property | Static Huffman (v0.2) | Adaptive FGK (v0.6) |
|---|---|---|
| Passes over input | 2 (scan + encode) | 1 (single pass) |
| Header contains | Full frequency table | Just original file size |
| Tree at start | Complete, optimal | Single zero-node |
| Tree evolution | Fixed | Updated per symbol |
| Frequency update | N/A | FGK sibling-swap walk |
| New symbol handling | Already in tree | Expand zero-node + raw bits |

---

## Data Flow

```
Input byte c
      │
      ├── Known symbol?
      │        │ Yes → hcompress(huffmanList[c]) → emit variable-length code
      │        │ No  → hcompress(zeroNode) → emit zero-node code
      │        │        + put_nbits(c, 8) → emit raw 8-bit literal
      │
      └─── updateTreeFGK(c) → swap nodes → increment frequencies → walk to root
```
