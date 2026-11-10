# Compression Benchmarks & Performance Harness

This directory contains the benchmarking harness, evaluation metrics, and comparative analysis tools for evaluating all implemented lossless compression algorithms under identical conditions.

---

## Benchmark Metrics & Formulas

When comparing lossless data compression algorithms, single metrics can be misleading. We evaluate each algorithm using four complementary dimensions:

### 1. Compression Ratio ($CR$)
Measures the factor by which the input was reduced in size:
$$CR = \frac{\text{Uncompressed Size (bytes)}}{\text{Compressed Size (bytes)}}$$
- $CR > 1.0$: Compression achieved (higher is better).
- $CR < 1.0$: Negative compression (file expansion).

### 2. Space Savings ($SS$)
The percentage of the original data eliminated:
$$SS = \left(1 - \frac{\text{Compressed Size}}{\text{Uncompressed Size}}\right) \times 100\%$$

### 3. Bits Per Character / Symbol ($bpc$)
The average number of bits required to store each input symbol:
$$bpc = \frac{\text{Compressed Size (bytes)} \times 8}{\text{Original Symbol Count}}$$
- Theoretical lower bound for static memoryless sources is the **Shannon Entropy**:
  $$H(X) = -\sum_{i=1}^{K} P(x_i) \log_2 P(x_i)$$

### 4. Throughput (Speed)
- **Encoding Speed**: $\text{Throughput}_{\text{enc}} = \frac{\text{Uncompressed Size (MB)}}{\text{Encoding Time (seconds)}}$
- **Decoding Speed**: $\text{Throughput}_{\text{dec}} = \frac{\text{Uncompressed Size (MB)}}{\text{Decoding Time (seconds)}}$

---

## Expected Performance Profile Matrix

Below is the theoretical and empirical comparison profile across all algorithms when tested against standard data profiles:

| Algorithm | Category | Repeated Data | English Prose | Structured Logs | Telemetry / Audio | Random Data | Memory Overhead |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **RLE** | Run-length | **Extreme** ($CR > 10\times$) | Poor / Expands | Moderate | Poor | Expands | $O(1)$ |
| **Shannon-Fano** | Entropy (Top-Down) | Poor | Good (~1.5–1.7$\times$) | Moderate | Poor | Expands | $O(K)$ |
| **Huffman** | Entropy (Bottom-Up) | Moderate | Very Good (~1.7–1.9$\times$) | Good | Poor | Expands | $O(K)$ |
| **Adaptive Huffman** | One-Pass Entropy | Moderate | Very Good | Good | Poor | Expands | $O(K)$ |
| **Arithmetic** | Exact Entropy | Moderate | **Optimal Entropy** | Good | Poor | Expands | $O(K)$ |
| **Golomb / Rice** | Geometric Entropy | Good (runs) | Poor | Poor | **Optimal for residuals** | Expands | $O(1)$ |
| **LZ77** | Sliding Window Dict | Very Good | Very Good (~2.0–2.5$\times$) | **Exceptional** | Moderate | Expands | $O(W)$ |
| **LZ8** | Phrase Tree Dict | Very Good | Good | Very Good | Moderate | Expands | $O(\text{Dict})$ |
| **LZW** | Alphabet Dict | Very Good | Very Good | Very Good | Moderate | Expands | $O(\text{Dict})$ |
| **Delta Coding** | Domain Transform | Moderate | Poor | Poor | **Massive Entropy Drop** | Neutral | $O(1)$ |
| **BWT + MTF** | Block Sort Transform | Extreme | **Exceptional** | **Exceptional** | Moderate | Expands | $O(N)$ |
| **DEFLATE** | Hybrid (LZ77+Huffman) | **Extreme** | **Exceptional** (~2.8–3.5$\times$) | **State-of-the-Art** | Good | Expands | $O(W + K)$ |

---

## The Key Teaching Moment: Random Data & The Shannon Limit

A cornerstone insight demonstrated by these benchmarks is that **no lossless compression algorithm can compress all possible inputs**.

### Mathematical Proof via the Pigeonhole Principle:
1. Suppose an algorithm could compress all inputs of length $N$ bytes into files of length $< N$ bytes.
2. The number of possible inputs of length $N$ over an 8-bit alphabet is $2^{8N}$.
3. The total number of all possible outputs of length $< N$ bytes is:
   $$\sum_{k=0}^{N-1} 2^{8k} = \frac{2^{8N} - 1}{255} < 2^{8N}$$
4. By the Pigeonhole Principle, there are strictly fewer possible compressed bitstrings than uncompressed inputs. Therefore, at least two different original files would map to the exact same compressed representation, making deterministic lossless decompression mathematically impossible.

When our benchmark suite runs on `random_bytes.bin`, every single algorithm must demonstrate a compression ratio $CR \le 1.0$, illustrating this fundamental law of information theory.

---

## Planned Benchmark CLI Specification

Once implementations are complete, the benchmark suite will be invoked via:

```bash
# Run all algorithms against all corpus files
python -m benchmarks.run

# Run benchmark for a specific algorithm
python -m benchmarks.run --algo huffman

# Run benchmark on a specific file
python -m benchmarks.run --input data/english_text.txt

# Export results to Markdown table or CSV
python -m benchmarks.run --format markdown --output benchmarks/results.md
```

