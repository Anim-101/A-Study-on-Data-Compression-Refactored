# v1.0 — LZ77 + Adaptive Huffman-FGK + 3-Byte Hash Table

> **Status:** Stable  
> **Language:** C  
> **Algorithm:** LZ77 (hash-accelerated) + Adaptive Huffman (FGK)  
> **Key upgrade:** The LZ77 linked-list index is replaced by a **3-byte rolling hash table** for faster and more accurate match finding

---

## Overview

This is a major architectural upgrade over v0.8. The central innovation is replacing the **character-indexed linked list** (`LempelZivList`) with a **3-byte hash indexed linked list** (`LempelZivHash`). Instead of indexing window positions by their single first byte, each position is hashed from a **3-byte trigram**, dramatically reducing false match candidates and improving compression quality.

Additionally, the sliding window and lookahead buffer sizes are significantly increased:
- Window: **4096 → 16384 bytes** (`NUM_POS_BITS = 14`)
- Pattern buffer: **18 → 259 bytes** (`PAT_BUFFSIZE = (1 << NUM_LEN_BITS) + MIN_LEN = 256 + 3`)
- Position field: **12 → 14 bits**
- Length field: **4 → 8 bits** (adaptive Huffman encoded)

---

## Architecture

```
v1.0(Lempel Ziv_AdaptiveHuffmanFGK_Hash)/
├── uTypes.h
├── BitIO.h / BitIO.c
├── Huffman.h / Huffman.c
├── AdaptiveHuffman.c
├── FGK.c
├── AdaptiveHuffmanFGK.c           # Same high-level API as v0.8
├── LempelZivHash.h                # ✅ NEW: Hash table structure declarations
├── LempelZivHash.c                # ✅ NEW: Dynamically allocated hash-based LZ index
├── LZ_AdaptiveHuffmanFGK_Hash_Compressor.c
└── LZ_AdaptiveHuffmanFGK_Hash_DeCompressor.c
```

---

## Module Breakdown

### New: `LempelZivHash.h` / `LempelZivHash.c` — Hash-Indexed Sliding Window

Replaces the static `LempelZivList` (256 list heads). Instead, window positions are indexed by a **3-byte trigram hash**, giving each position a more unique key.

**Dynamic allocation:**
```c
int *lempelZivHash;      // Hash table: hash_value → first window position
int *lempelZivNext;      // Next pointer per window position
int *lempelZivPrevious;  // Previous pointer per window position
```
All three are heap-allocated at runtime with size = `WIN_BUFFSIZE = 16384`.

**Hash function (3-byte trigram):**
```c
#define hash_w(pos) \
    ((winBuf[(pos) & WIN_MASK] << HASH_SHIFT) \
     ^ (winBuf[((pos)+1) & WIN_MASK] << 1)   \
     ^ (winBuf[((pos)+2) & WIN_MASK] << 4))

#define hash_p(pos) \
    ((pattern[(pos)] << HASH_SHIFT)                     \
     ^ (pattern[((pos)+1) % PAT_BUFFSIZE] << 1)         \
     ^ (pattern[((pos)+2) % PAT_BUFFSIZE] << 4))
```
Where `HASH_SHIFT = NUM_POS_BITS - 8 = 6`. The hash combines bits from three consecutive bytes using bit shifts and XOR.

**Key functions:**
| Function | Description |
|---|---|
| `allocate_lempelZivHash(size)` | Heap-allocates hash, next, previous arrays of `size` ints |
| `free_lempelZivHash()` | Frees all three arrays |
| `insert_lempelZivHashNode(h, i)` | Inserts position `i` at head of `lempelZivHash[h]` |
| `delete_lempelZivHashNode(h, i)` | Removes position `i` from `lempelZivHash[h]` |

---

### Algorithm Constants

| Constant | v0.8 value | v1.0 value | Change |
|---|---|---|---|
| `WIN_BUFFSIZE` | 4096 | 16384 | 4× larger window |
| `NUM_POS_BITS` | 12 | 14 | 2 more bits for position |
| `PAT_BUFFSIZE` | 18 | 259 | 14× larger lookahead |
| `NUM_LEN_BITS` | 4 | 8 | 2× more length precision |
| `MIN_LEN` | 2 | 3 | Min match raised by 1 |

---

### `search()` — Hash-Accelerated Match Finder

The search function is comprehensively redesigned to use the hash table:

1. Compute `hash_p(pat_cnt)` — hash current 3 bytes of pattern buffer
2. Look up `lempelZivHash[hash]` → head of matching trigram list
3. For each candidate position `i`:
   - **Pre-verify**: Check if bytes at the current best match length already match (early exit for non-improvements)
   - **Extend**: Try extending the match byte-by-byte up to `buf_cnt`
   - Track the longest `pos.length` and its `pos.position`
4. Uses labeled `goto skipSearch` to efficiently skip non-improving candidates

This is a **lazy matching** style: confirms the current-best-length suffix before trying to extend, reducing wasted comparisons.

---

### `putCodes()` — Hash Update Protocol

The window update step in `putCodes` is more elaborate because the hash depends on 3 bytes:

1. **Delete old trigrams:** For the range `[win_cnt-2, win_cnt-2+length+2]`, delete `hash_w(j)` entries (since these trigrams span the boundary being overwritten)
2. **Write new bytes** into `winBuf` from `pattern`
3. **Re-insert trigrams:** Re-hash and re-insert the same position range with updated bytes
4. **Read next bytes** from input into pattern buffer to stay full

This 3-step delete→write→reinsert ensures the hash stays consistent even at circular window boundaries.

---

### Encoding Protocol (Same as v0.8 but with larger fields)

| Match class | Prefix | Payload |
|---|---|---|
| Literal (length=1) | `00` | 8-bit raw byte |
| Short match (length=MIN_LEN=3) | `01` | 14-bit position |
| Long match (length>MIN_LEN) | `1` | Adaptive Huffman `(length-4)` + 14-bit position |

---

## Improvements Over v0.8

| Feature | v0.8 (list-based) | v1.0 (hash-based) |
|---|---|---|
| Match index | 256 char-indexed lists | 3-byte hash table |
| Index allocation | Static arrays | Dynamic heap allocation |
| Window size | 4096 bytes | 16384 bytes |
| Lookahead size | 18 bytes | 259 bytes |
| Position bits | 12 | 14 |
| Length bits | 4 | 8 |
| False match rate | Higher (same first byte) | Lower (3-byte trigram) |
| Search efficiency | Linear per-char scan | Hash-partitioned, lazy match |
