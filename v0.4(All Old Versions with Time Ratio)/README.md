# v0.4 — All Old Versions with Timing Benchmarks

> **Purpose:** Benchmarking / Historical Archive  
> **Language:** Java (test harness) + Windows `.exe` binaries (C compiled)  
> **Contains:** Pre-compiled executables + Java timing/ratio test runners

---

## Overview

This folder is a **benchmarking archive** that bundles compiled executables from Shannon-Fano, Huffman, LZ77, and an early LZ77+Huffman deflate-style implementation, alongside Java programs that automate batch compression and timing analysis across multiple test files.

The C source code for each algorithm is present within the named subdirectories, organized by algorithm family.

---

## Directory Structure

```
v0.4(All Old Versions with Time Ratio)/
│
├── CheckRatio.java              # Java: batch-compress files and compare ratios
├── CheckTime.java               # Java: batch-decompress and measure timings
│
├── HuffmanCompressor.exe        # Static Huffman compressor (from v0.2)
├── HuffmanDeCompressor.exe      # Static Huffman decompressor
├── ShannonFanoCompressor.exe    # Shannon-Fano compressor (from v0.1.1)
├── ShannonFanoDeCompressor.exe  # Shannon-Fano decompressor
├── LZ77_HuffmanCompressor.exe   # LZ77 + Huffman (Deflate-style) compressor
├── LZ77_HuffmanDeCompressor.exe # LZ77 + Huffman decompressor
│
├── Dust/                        # Scratch I/O folders for batch tests
│   ├── CompressorDust/          # Compressed output files land here
│   └── DeCompressorDust/        # Decompressed output files land here
│
├── Files/                       # Input test files for benchmarking
├── JavaCompressor/              # Java-based reference compressor outputs
├── JavaDecompressor/            # Java-based reference decompressor outputs
│
├── StandardTools/               # Reference tools for comparison
│   ├── CheckTools.java          # Runs gzip/bzip2 on same files for comparison
│   ├── bzip2.exe                # bzip2 reference compressor
│   ├── gzip.exe                 # gzip reference compressor
│   └── ToCompress/              # Input files for standard tool tests
│
├── Adaptive Huffman/            # C source for Adaptive Huffman (FGK and Vitter)
│   ├── Adaptive Huffman Coding-FGK/
│   └── Adaptive Huffman Coding-Vitter/
│
├── Huffman Coding/              # C source for static Huffman (same as v0.2)
├── Lempel Ziv 77/               # C source for LZ77 (same as v0.3)
└── Shannon Fano Coding/         # C source for Shannon-Fano (same as v0.1.1)
```

---

## Module Breakdown

### `CheckRatio.java` — Compression Ratio Tester

A Java program that automates batch compression across a folder of test files, using the compiled `.exe` tools.

**Modes (user selects at runtime):**
- `0` — Compress files from `Dust/CompressorDust` → `Dust/DeCompressorDust`
- `1` — Compress files from `JavaCompressor/` → `Dust/DeCompressorDust`

**How it works:**
- Uses `ProcessBuilder` to invoke the C `.exe` compressor on each file
- Reads both `stdout` and `stderr` from each process and prints to console
- The compressor `.exe` programs themselves print ratio statistics to `stderr`
- By default configured to use `LZ77_HuffmanCompressor.exe`; can be switched to `HuffmanCompressor.exe` or `ShannonFanoCompressor.exe` by changing the `parameters[0]` line

**Architecture:**
```java
ProcessBuilder(compressor.exe, inputFile, outputFile).start()
→ Reads: stdout + stderr streams
→ Prints: compression statistics output
```

---

### `CheckTime.java` — Decompression Timing Tester

Sister program to `CheckRatio.java`. Automates batch decompression and measures timing via the compressor's own stderr output.

**Modes:**
- `0` — Decompress files from `Dust/DeCompressorDust` → `Dust/CompressorDust`
- `1` — Decompress files from `JavaDecompressor/` → `Dust/CompressorDust`

**Default exe:** `LZ77_HuffmanDeCompressor.exe` (swap-able to Huffman or Shannon-Fano)

---

### `StandardTools/CheckTools.java` — Reference Tool Comparison

Runs the industry-standard **gzip** and **bzip2** compressors on the same test files and prints their output for side-by-side comparison.

---

### Compiled Executables

| File | Algorithm | From Version |
|---|---|---|
| `ShannonFanoCompressor.exe` | Static Shannon-Fano | v0.1.1 |
| `ShannonFanoDeCompressor.exe` | Static Shannon-Fano | v0.1.1 |
| `HuffmanCompressor.exe` | Static Huffman | v0.2 |
| `HuffmanDeCompressor.exe` | Static Huffman | v0.2 |
| `LZ77_HuffmanCompressor.exe` | LZ77 + Huffman (Deflate-like) | Internal |
| `LZ77_HuffmanDeCompressor.exe` | LZ77 + Huffman | Internal |

---

### Algorithm Source Subdirectories

Each subdirectory contains the full C source for the named algorithm (equivalent to its dedicated version folder):

| Subdirectory | Algorithm |
|---|---|
| `Shannon Fano Coding/` | Same as v0.1.1 |
| `Huffman Coding/` | Same as v0.2 |
| `Lempel Ziv 77/` | Same as v0.3 |
| `Adaptive Huffman/Adaptive Huffman Coding-FGK/` | Early FGK (same as v0.6) |
| `Adaptive Huffman/Adaptive Huffman Coding-Vitter/` | Early Vitter (same as v0.7) |

---

## Test Harness Flow

```
User selects mode (0 or 1)
      │
      ▼
CheckRatio.java / CheckTime.java
      │
      ├─ Lists all files in selected source folder
      │
      └─ For each file:
           ProcessBuilder → run .exe compressor/decompressor
               │
               ├─ Read stdout → print
               └─ Read stderr → print (contains ratio/timing stats)
```
