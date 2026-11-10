# Benchmark Test Data Corpus

This directory contains standardized datasets designed to test, benchmark, and stress lossless compression algorithms across distinct statistical characteristics and data regimes.

---

## Dataset Overview

| File Name | Size (Bytes) | Primary Characteristic | Target Algorithms | Expected Behavior |
| :--- | :--- | :--- | :--- | :--- |
| `repeated_runs.txt` | ~2,960 B | Massive contiguous runs of identical characters | **RLE**, Deflate | Extreme compression (>90% space savings). |
| `english_text.txt` | ~3,350 B | Natural language with skewed symbol frequencies and common English n-grams | **Huffman**, **Arithmetic**, **LZ77**, **LZW**, **Deflate** | Moderate-to-high compression (~40–60% ratio). |
| `structured_log.log` | ~3,200 B | Highly predictable structural grammar with repeating tokens/paths | **LZ77**, **LZ78**, **LZW**, **Deflate** | Very high dictionary match rate; token repetitions compress heavily. |
| `sensor_readings.csv` | ~3,100 B | Slowly drifting numerical time-series telemetry | **Delta / Predictive**, **Golomb-Rice** | Raw differences are small integers centered at 0; high delta-residual compression. |
| `random_bytes.bin` | 2,048 B | High entropy (pseudo-random bytes $0\dots255$, uniform distribution) | *All algorithms* | **Cannot compress** (slight file expansion due to header/framing overhead). |

---

## Detailed Data Profiles

### 1. `repeated_runs.txt`
- **Pattern**: Long contiguous blocks of single characters (e.g., $45 \times \text{'A'}$, $80 \times \text{'C'}$, $150 \times \text{'E'}$).
- **Entropy**: Very low global and local entropy.
- **Pedagogical Purpose**: Demonstrates where Run-Length Encoding (RLE) excels and why simple algorithms remain standard in image masks, fax machines (CCITT T.4), and sparse bitmap compression.

### 2. `english_text.txt`
- **Pattern**: English prose excerpt from *Alice's Adventures in Wonderland*.
- **Entropy**: Letter frequencies mirror natural language statistics (`e`, `t`, `a`, `o` are common; `q`, `z`, `x` are rare).
- **Pedagogical Purpose**: Classic benchmark for statistical coders (Huffman, Shannon-Fano, Arithmetic) and phrase-substitution methods (LZ77, LZW).

### 3. `structured_log.log`
- **Pattern**: ISO timestamps, standard HTTP methods (`GET`, `POST`), repeating endpoint URIs, and status codes.
- **Entropy**: Intermediate frequency skew, but high dictionary redundancy across repeated phrases and structural templates.
- **Pedagogical Purpose**: Demonstrates the strength of dictionary algorithms over pure character-entropy coders.

### 4. `sensor_readings.csv`
- **Pattern**: Timestamped numeric readings (temperature, relative humidity, atmospheric pressure).
- **Entropy**: Absolute values vary across large ranges (e.g., 1013.2 hPa), but consecutive differences ($\Delta = x_i - x_{i-1}$) are minuscule ($[-0.2, +0.2]$).
- **Pedagogical Purpose**: Highlights why transforming the domain via **Delta / Predictive coding** drastically reduces entropy before passing data to an entropy coder like Golomb-Rice or Huffman.

### 5. `random_bytes.bin`
- **Pattern**: 2,048 uniformly distributed random bytes generated from a cryptographic PRNG.
- **Entropy**: Maximum entropy ($H \approx 8.0 \text{ bits/symbol}$).
- **Pedagogical Purpose**: **The Shannon Limit and Pigeonhole Principle Demonstration**. Proof that no lossless algorithm can compress all possible inputs. Every algorithm tested on this file is expected to either fail to compress or slightly expand the file due to metadata and tree overhead.

---

## Verification & Integrity

All sample files are tracked directly in git for reproducible benchmarking across environments.

