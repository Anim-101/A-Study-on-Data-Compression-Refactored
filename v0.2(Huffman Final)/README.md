# v0.2 — Huffman Compression (Final)

> **Status:** Stable  
> **Language:** C  
> **Algorithm:** Static Huffman Coding

---

## Overview

This version replaces Shannon-Fano with a static **Huffman coding** compressor. Huffman coding builds an optimal prefix-free binary tree by repeatedly merging the two lowest-frequency nodes, guaranteeing minimal expected code length. The BitIO and header format remain structurally similar to v0.1.1, but the tree module is entirely new.

---

## Architecture

```
v0.2(Huffman Final)/
├── uTypes.h                  # Primitive type aliases
├── BitIO.h                   # Bit I/O interface
├── BitIO.c                   # Bit-level buffered I/O
├── Huffman.h                 # Huffman node struct & declarations
├── Huffman.c                 # Huffman tree construction & codec logic
├── HuffmanCompressor.c       # Main: compressor entry point
└── HuffmanDeCompressor.c     # Main: decompressor entry point
```

---

## Module Breakdown

### `uTypes.h`
Same as prior versions. Provides `uchar`, `uint`, `ulong` type aliases.

---

### `BitIO.h` / `BitIO.c` — Bit-Level I/O

Functionally equivalent to v0.1.1. Provides buffered single-bit and byte I/O:
- `get_bit()`, `put_ZERO()`, `put_ONE()`
- `gfgetc()`, `pfputc()`
- `init_putBuffer()`, `init_getBuffer()`, `flush_putBuffer()`
- `free_putBuffer()`, `free_getBuffer()`
- `get_nbytes_out()`

---

### `Huffman.h` / `Huffman.c` — Tree Engine

Implements static Huffman coding using a **min-priority sorted linked list** and a **static node pool**.

**Key data structures:**
```c
typedef struct listnode {
    unsigned long frequency;      // Symbol or combined frequency
    int ch;                       // Symbol ASCII value (-1 = internal)
    struct listnode *next;        // Sorted list pointer
    struct listnode *parent;      // Parent in Huffman tree
    struct listnode *firstChild;  // Left child (bit = 0)
    struct listnode *secondChild; // Right child (bit = 1)
} listnode_t;

typedef struct {
    unsigned long f;   // Frequency
    unsigned char b;   // Symbol byte value
} huffmanFrequency_Type;
```

**Static pool:** `listnode_t huffmanNodes[Huffman_MAX * 2]`  
With `Huffman_MAX = 256`, this gives 512 nodes (256 leaves + up to 255 internal nodes + 1 root).

**Key functions:**
| Function | Description |
|---|---|
| `init_huffmanFrequency()` | Zeros frequency table for all 256 symbols |
| `init_huffmanList()` | Clears symbol→node pointer map |
| `createSymbolList()` | Inserts nodes for non-zero symbols into sorted list |
| `insert(list, node)` | Inserts a node maintaining ascending frequency order |
| `createHuffmanTree()` | **Iteratively** merges two lowest-frequency nodes until one root remains |
| `hcompress(node)` | Leaf-to-root traversal; emits 0 for `firstChild` path, 1 for `secondChild` |
| `hdecompress(node)` | Root-to-leaf traversal; consumes bits to find symbol |

**Tree construction (iterative, unlike Shannon-Fano's recursive approach):**
```c
void createHuffmanTree() {
    while (list->next != NULL) {
        // Take two smallest nodes from front of sorted list
        newNode = createNode();
        newNode->firstChild  = list;       // lowest freq
        newNode->secondChild = list->next; // second lowest freq
        newNode->frequency   = list->frequency + list->next->frequency;
        list = list->next->next;           // advance past both
        list->firstChild->parent  = newNode;
        list->secondChild->parent = newNode;
        insert(&list, newNode);            // re-insert merged node
    }
    top = list; // single remaining node = root
}
```

---

### `HuffmanCompressor.c` — Compressor Entry Point

**Flow:**
1. Open input and output files
2. Scan input with `readStats()` → build frequency table
3. Call `createSymbolList()` → `createHuffmanTree()` → Huffman tree ready
4. Write file header:
   - Magic tag `"THF"` + original file size (`fileStamp`)
   - `huffmanCount`: number of distinct symbols
   - Frequency table entries for all symbols with `freq > 0`
5. Re-read input; for each byte call `hcompress(huffmanList[c])`
6. `flush_putBuffer()` → write remaining bits
7. Print compression ratio statistics

---

### `HuffmanDeCompressor.c` — Decompressor Entry Point

**Flow:**
1. Open files
2. Read and validate `fileStamp` header (`"THF"`)
3. Read frequency table, rebuild tree via same construction algorithm
4. Read compressed bitstream; for each original symbol call `hdecompress(top)`
5. Write decoded byte to output file

---

## Data Flow

```
Input File
    │
    ├──[Pass 1]──► readStats ──► Frequency Table ──► Sorted List
    │                                                       │
    │                                              [createHuffmanTree]
    │                                                (iterative merge)
    │                                                       │
    │                                                 Huffman Tree
    │
    └──[Pass 2]──► hcompress(leaf) ──► BitIO ──► Output File
                                                    │
                                           [Header: THF + freqs]
```

---

## Comparison: Shannon-Fano vs. Huffman

| Property | Shannon-Fano (v0.1.1) | Huffman (v0.2) |
|---|---|---|
| Tree construction | Top-down recursive splitting | Bottom-up iterative merging |
| Optimality | Near-optimal | Provably optimal (min avg code length) |
| Node pool size | `256 × 3 = 768` | `256 × 2 = 512` |
| File stamp magic | `"TSF"` | `"THF"` |
| Codec functions | `sfcompress` / `sfdecompress` | `hcompress` / `hdecompress` |
