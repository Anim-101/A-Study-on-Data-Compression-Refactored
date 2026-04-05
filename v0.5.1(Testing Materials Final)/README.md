# v0.5.1 — Testing Materials (Final)

> **Purpose:** Final automated testing + unit testing framework  
> **Language:** Java (test runners)  
> **Status:** Stable (no compiled .exe files; run source directly or use external executables)

---

## Overview

This is the **finalized testing suite**, building on v0.5 with two key improvements:
1. `.exe` binaries are removed from the root (cleaner repo)
2. A new **`UnitTesting/`** directory provides structured unit tests for each algorithm

The Java test harnesses (`CheckRatio.java`, `CheckTime.java`) are also slightly cleaned up from v0.5.

---

## Directory Structure

```
v0.5.1(Testing Materials Final)/
│
├── CheckRatio.java           # Batch compression ratio tester
├── CheckTime.java            # Batch decompression timing tester
│
├── Dust/                     # Scratch I/O working directories
│   ├── CompressorDust/       # Compressed test outputs
│   └── DeCompressorDust/     # Decompressed test outputs
│
├── Files/                    # Input test file set
├── JavaCompressor/           # Alternative compressed input files
├── JavaDecompressor/         # Alternative decompressed input files
│
├── StandardTools/            # Reference tool binaries + test runner
│   ├── CheckTools.java
│   ├── bzip2.exe
│   ├── gzip.exe
│   └── ToCompress/
│
└── UnitTesting/              # ✅ NEW: Structured unit tests per algorithm
    ├── Main/                 # Unit tests using algorithm's main functions
    │   ├── Huffman/
    │   ├── Lempel Ziv/
    │   ├── Lempel Ziv_Adaptive Huffman/
    │   └── Shannon Fano/
    │
    ├── Sub/                  # Unit tests for sub-functions/modules
    └── Tables/               # Unit tests for data structures (trees, hash tables)
```

---

## Module Breakdown

### `CheckRatio.java` — Compression Ratio Tester
_(Cleaned up from v0.5)_

Automates batch compression of test files using a user-selected compressor `.exe` and prints each file's compression ratio.

**Mode selection:**
- `0` → Files from `Dust/CompressorDust` → `Dust/DeCompressorDust`
- `1` → Files from `JavaCompressor/` → `Dust/DeCompressorDust`

**Default exe:** `LZ77_HuffmanCompressor.exe`  
*(Swap `parameters[0]` for Huffman or ShannonFano)*

---

### `CheckTime.java` — Decompression Timing Tester
_(Cleaned up from v0.5)_

Automates batch decompression and collects decompressor timing/ratio stats from `stderr`.

**Mode selection:**
- `0` → Files from `Dust/DeCompressorDust` → `Dust/CompressorDust`
- `1` → Files from `JavaDecompressor/` → `Dust/CompressorDust`

---

### `UnitTesting/` — Algorithm Unit Tests

The `UnitTesting/` directory is organized into three tiers:

| Subdirectory | Purpose |
|---|---|
| `Main/` | End-to-end tests for each algorithm's full compress/decompress cycle |
| `Sub/` | Focused tests on individual sub-functions (e.g., tree building, symbol lists) |
| `Tables/` | Tests for core data structures (linked lists, hash tables, node pools) |

**`Main/` sub-algorithm folders:**

| Folder | Tests |
|---|---|
| `Shannon Fano/` | Full Shannon-Fano compress → decompress round-trip |
| `Huffman/` | Full Huffman compress → decompress round-trip |
| `Lempel Ziv/` | Full LZ77 compress → decompress round-trip |
| `Lempel Ziv_Adaptive Huffman/` | Full LZ77 + Adaptive Huffman round-trip |

---

## Differences from v0.5

| Feature | v0.5 | v0.5.1 |
|---|---|---|
| Compiled `.exe` in root | ✅ Present | ❌ Removed |
| Unit testing directory | ❌ | ✅ Added |
| `CheckRatio.java` size | 3971 bytes | 3627 bytes |
| `CheckTime.java` size | 3975 bytes | 3619 bytes |

---

## Test Flow Overview

```
Batch Testing (CheckRatio / CheckTime):
┌─────────────────────┐
│ Java test runner     │
│ ┌─────────────────┐ │
│ │ ProcessBuilder  │ │  → compressor.exe inputFile outputFile
│ │ Capture stderr  │ │  → print ratio/timing statistics
│ └─────────────────┘ │
└─────────────────────┘

Unit Testing (UnitTesting/):
┌──────────────────────────────┐
│ Main/ → full compress+decomp │
│ Sub/  → individual functions │
│ Tables/ → data structures    │
└──────────────────────────────┘
```
