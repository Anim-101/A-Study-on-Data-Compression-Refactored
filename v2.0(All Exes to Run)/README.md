# v2.0 — All Compiled Executables

> **Purpose:** Binary distribution archive  
> **Content:** Pre-compiled Windows `.exe` files for all algorithm variants  
> **Usage:** Run each `.exe` with `inputFile outputFile` arguments

---

## Overview

This folder contains all final compiled Windows executables from the entire study — every algorithm variant across all versions, ready to run without compilation. It serves as the **distribution package** for all compressor/decompressor tools developed through v0.1.1 → v1.2.

---

## Directory Contents

```
v2.0(All Exes to Run)/
├── ShannonFanoCompressor.exe
├── ShannonFanoDeCompressor.exe
│
├── HuffmanCompressor.exe
├── HuffmanDeCompressor.exe
│
├── LZ77Compressor.exe
├── LZ77DeCompressor.exe
│
├── FGK_Compressor.exe
├── FGK_DeCompressor.exe
│
├── Vitter_Basic_Compressor.exe
├── Vitter_Basic_DeCompressor.exe
│
├── Vitter_Mod_Compressor.exe
├── Vitter_Mod_DeCompressor.exe
│
├── Vitter_BasMod_Compressor.exe            ← Asymmetric (v0.7.2 style)
├── Vitter_BasMod_DeCompressor.exe
│
├── LZ_AdaptiveHuffmanFGK_Compressor.exe
├── LZ_AdaptiveHuffmanFGK_DeCompressor.exe
│
├── LZ_AdaptiveHuffmanFGK_Hash_Compressor.exe
├── LZ_AdaptiveHuffmanFGK_Hash_DeCompressor.exe
│
├── LZ_AdaptiveHuffmanVitte_Basicr_Compressor.exe   ← (typo in filename)
├── LZ_AdaptiveHuffmanVitter_Basic_DeCompressor.exe
│
├── LZ_AdaptiveHuffmanVitter_Modified_Compressor.exe
├── LZ_AdaptiveHuffmanVitter_Modified_DeCompressor.exe
│
├── LZ_AdaptiveHuffmanVitter_Basic_Hash_Compressor.exe
├── LZ_AdaptiveHuffmanVitter_Basic_Hash_DeCompressor.exe
│
├── LZ_AdaptiveHuffmanVitter_Modified_Hash_Compressor.exe
└── LZ_AdaptiveHuffmanVitter_Modified_Hash_DeCompressor.exe
```

---

## Executable → Version Mapping

| Executable | Source Version | Algorithm |
|---|---|---|
| `ShannonFanoCompressor/DeCompressor.exe` | v0.1.1 | Static Shannon-Fano |
| `HuffmanCompressor/DeCompressor.exe` | v0.2 | Static Huffman |
| `LZ77Compressor/DeCompressor.exe` | v0.3 | LZ77 |
| `FGK_Compressor/DeCompressor.exe` | v0.6 | Adaptive Huffman (FGK) |
| `Vitter_Basic_Compressor/DeCompressor.exe` | v0.7 | Adaptive Huffman (Vitter Basic) |
| `Vitter_Mod_Compressor/DeCompressor.exe` | v0.7.1 | Adaptive Huffman (Vitter Modified) |
| `Vitter_BasMod_Compressor/DeCompressor.exe` | v0.7.2 | Adaptive Huffman (Vitter Asymmetric) |
| `LZ_AdaptiveHuffmanFGK_Compressor/DeCompressor.exe` | v0.8 | LZ77 + FGK (list) |
| `LZ_AdaptiveHuffmanVitter_Basic*.exe` | v0.9 | LZ77 + Vitter Basic (list) |
| `LZ_AdaptiveHuffmanVitter_Modified*.exe` | v0.9.1 | LZ77 + Vitter Modified (list) |
| `LZ_AdaptiveHuffmanFGK_Hash_Compressor/DeCompressor.exe` | v1.0 | LZ77 + FGK + Hash |
| `LZ_AdaptiveHuffmanVitter_Basic_Hash*.exe` | v1.1 | LZ77 + Vitter Basic + Hash |
| `LZ_AdaptiveHuffmanVitter_Modified_Hash*.exe` | v1.2 | LZ77 + Vitter Modified + Hash |

---

## Usage

All executables follow the same calling convention:
```
<CompressorName>.exe  <inputFile>  <outputFile>
<DeCompressorName>.exe <compressedFile> <outputFile>
```

Statistics (compression ratio, file sizes, timing) are printed to `stderr`.

---

## Notes

- All executables are **Windows PE32 binaries** compiled for x86 targets
- Compressor/decompressor pairs must be matched — e.g., `LZ_AdaptiveHuffmanFGK_Hash_Compressor.exe` output can only be decoded by `LZ_AdaptiveHuffmanFGK_Hash_DeCompressor.exe`
- The `Vitter_BasMod_*.exe` pair uses asymmetric encoder/decoder update strategies and **will not correctly round-trip**
- Use the Java test harnesses in v0.5.1 to batch-run these executables against test file sets
