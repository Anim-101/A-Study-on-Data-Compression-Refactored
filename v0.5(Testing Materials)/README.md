# v0.5 — Testing Materials (Initial)

> **Purpose:** Automated batch testing harness  
> **Language:** Java (test runners) + Windows `.exe` binaries  
> **Note:** This folder contains an early version of the testing suite; see v0.5.1 for the finalized version

---

## Overview

This folder is the **initial version of the automated testing framework** for benchmarking the C compression algorithms. It provides Java programs to batch-run compressors and decompressors over a set of test files and collect both compression ratios and decompression timings. It also includes reference compressors (gzip, bzip2) for comparison via the `StandardTools` folder.

---

## Directory Structure

```
v0.5(Testing Materials)/
│
├── CheckRatio.java           # Batch compression: measures compression ratio
├── CheckTime.java            # Batch decompression: measures timing
│
├── HuffmanCompressor.exe     # Pre-compiled Huffman compressor
├── HuffmanDeCompressor.exe   # Pre-compiled Huffman decompressor
├── ShannonFanoCompressor.exe # Pre-compiled Shannon-Fano compressor
├── ShannonFanoDeCompressor.exe
├── LZ77_HuffmanCompressor.exe     # Pre-compiled LZ77+Huffman compressor
├── LZ77_HuffmanDeCompressor.exe
│
├── Dust/                     # Scratch I/O folder
│   ├── CompressorDust/       # Compressed outputs from CheckRatio
│   └── DeCompressorDust/     # Decompressed outputs from CheckTime
│
├── Files/                    # Source test files
├── JavaCompressor/           # Alternative input folder (sorted compressed files)
├── JavaDecompressor/         # Alternative input folder (sorted decompressed files)
│
└── StandardTools/            # Industry reference tools
    ├── CheckTools.java       # Runs gzip.exe / bzip2.exe on test files
    ├── bzip2.exe             # bzip2 binary (Windows)
    ├── gzip.exe              # gzip binary (Windows)
    └── ToCompress/           # Input files for standard tool tests
```

---

## Module Breakdown

### `CheckRatio.java` — Compression Ratio Tester

Scans a folder of input files and compresses each using the chosen `.exe`, printing the compression ratio reported by the compressor.

**Interactive mode selection:**
- `0` → Source files from `Dust/CompressorDust`, outputs to `Dust/DeCompressorDust`
- `1` → Source files from `JavaCompressor/`, outputs to `Dust/DeCompressorDust`

**Default compressor:** `LZ77_HuffmanCompressor.exe`  
*(Switch line `parameters[0]` to select `HuffmanCompressor.exe` or `ShannonFanoCompressor.exe`)*

**Mechanism:** Uses `ProcessBuilder` to invoke the `.exe` on each file and captures both `stdout` and `stderr` (where ratio stats are printed).

---

### `CheckTime.java` — Decompression Timing Tester

Batch-runs decompression over a folder of compressed files, printing timing information output by the decompressor.

**Interactive mode selection:**
- `0` → Source from `Dust/DeCompressorDust`, outputs to `Dust/CompressorDust`
- `1` → Source from `JavaDecompressor/`, outputs to `Dust/CompressorDust`

**Default decompressor:** `LZ77_HuffmanDeCompressor.exe`

---

### `StandardTools/CheckTools.java` — Reference Comparison

Runs **gzip** and **bzip2** on the same test files so their ratio and timing output can be compared against the custom implementations.

**Architecture:**
```java
ProcessBuilder(gzip.exe, file).start()
ProcessBuilder(bzip2.exe, file).start()
→ Reads stderr/stdout → Prints statistics
```

---

## Differences from v0.5.1

| Feature | v0.5 | v0.5.1 |
|---|---|---|
| Unit testing support | ❌ | ✅ (added `UnitTesting/` folder) |
| `CheckRatio.java` size | 3971 bytes | 3627 bytes (cleaner) |
| `CheckTime.java` size | 3975 bytes | 3619 bytes (cleaner) |
| Compiled `.exe` files | Included in root | Removed (run from source) |

---

## Test Flow

```
User selects mode
      │
      ▼
CheckRatio.java (or CheckTime.java)
      │
      ├─ Lists files in source folder
      │
      └─ For each file:
            ProcessBuilder → runs .exe compressor/decompressor
                ├─ stdout → print
                └─ stderr → print (contains ratio/timing stats)
```
