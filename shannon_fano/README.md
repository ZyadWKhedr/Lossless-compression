# Shannon-Fano Coding

Developed independently by Claude Shannon (1948) and Robert Fano (1949), Shannon-Fano coding is one of the earliest statistical entropy coding techniques. It assigns variable-length, prefix-free binary codes to symbols based on their observed probabilities.

---

## 1. Core Intuition & Historical Significance

In uncompressed ASCII or raw byte encoding, every character occupies a fixed 8 bits, whether it is the letter `'e'` (appearing ~12% in English text) or `'z'` (appearing ~0.07%). 

Information theory states that an event with probability $P(x)$ carries **self-information**:
$$I(x) = \log_2 \frac{1}{P(x)} = -\log_2 P(x) \text{ bits}$$

Shannon-Fano coding assigns **shorter bit sequences to frequent symbols** and **longer bit sequences to rare symbols**, aiming to approximate the Shannon Entropy limit:
$$H(X) = -\sum_{i=1}^{K} P(x_i) \log_2 P(x_i)$$

---

## 2. The Top-Down Partitioning Algorithm

Unlike Huffman coding (which builds a tree from the bottom up), Shannon-Fano uses a **top-down divide-and-conquer strategy**:

1. **Count Frequencies**: Scan input data and record the occurrence count $f_i$ for each unique symbol.
2. **Sort**: Sort unique symbols in **descending order** of frequency.
3. **Partition**: Split the sorted list into two contiguous subsets such that the sum of frequencies in the left subset is as close as possible to the sum of frequencies in the right subset.
4. **Assign Bits**: Append `'0'` to all codes in the left subset, and `'1'` to all codes in the right subset.
5. **Recurse**: Repeat steps 3 and 4 recursively on each subset until each group contains exactly one symbol.

---

## 3. Step-by-Step Worked Example (Hand Trace)

### Input Alphabet & Frequencies
Suppose an input sample contains 39 total characters across 5 distinct symbols:

| Symbol | Count ($f$) | Probability ($P$) | Target Info $-\log_2 P$ |
| :--- | :--- | :--- | :--- |
| **A** | 15 | $0.3846$ | 1.38 bits |
| **B** | 7 | $0.1795$ | 2.48 bits |
| **C** | 6 | $0.1538$ | 2.70 bits |
| **D** | 6 | $0.1538$ | 2.70 bits |
| **E** | 5 | $0.1282$ | 2.96 bits |

**Total Count**: 39

### Partitioning Trace
- **Step 1 (Root split)**:
  - Left subset: `[A]` (Sum = 15)
  - Right subset: `[B, C, D, E]` (Sum = $7 + 6 + 6 + 5 = 24$)
  - *(Difference = $|15 - 24| = 9$. If we included B in left, Left=22, Right=17, diff=5. Let's choose the minimum difference: Left `[A, B]` = 22, Right `[C, D, E]` = 17, diff=5).*
  - **Left**: `[A, B]` $\rightarrow$ prefix `0`
  - **Right**: `[C, D, E]` $\rightarrow$ prefix `1`

- **Step 2 (Recurse on Left `[A, B]`)**:
  - Left child: `[A]` (Sum = 15) $\rightarrow$ append `0` $\implies$ **A: `00`** (2 bits)
  - Right child: `[B]` (Sum = 7) $\rightarrow$ append `1` $\implies$ **B: `01`** (2 bits)

- **Step 3 (Recurse on Right `[C, D, E]`)**:
  - Total sum = $6 + 6 + 5 = 17$.
  - Best split: Left `[C]` (Sum = 6) vs Right `[D, E]` (Sum = 11, diff = 5). Or Left `[C, D]` (12) vs Right `[E]` (5, diff = 7). Minimum difference is 5.
  - Left child: `[C]` $\rightarrow$ append `0` $\implies$ **C: `10`** (2 bits)
  - Right child: `[D, E]` $\rightarrow$ append `1` (prefix `11`)

- **Step 4 (Recurse on `[D, E]`)**:
  - Left child: `[D]` $\rightarrow$ append `0` $\implies$ **D: `110`** (3 bits)
  - Right child: `[E]` $\rightarrow$ append `1` $\implies$ **E: `111`** (3 bits)

### Final Codebook

| Symbol | Frequency | Code | Bit Length ($l_i$) | Total Bits ($f_i \times l_i$) |
| :--- | :--- | :--- | :--- | :--- |
| **A** | 15 | `00` | 2 | 30 |
| **B** | 7 | `01` | 2 | 14 |
| **C** | 6 | `10` | 2 | 12 |
| **D** | 6 | `110` | 3 | 18 |
| **E** | 5 | `111` | 3 | 15 |
| **Total** | **39** | — | — | **89 bits** |

- **Uncompressed (8-bit ASCII)**: $39 \times 8 = 312\text{ bits}$
- **Compressed Bitstream**: 89 bits (plus header for codebook)
- **Average Code Length**: $\bar{L} = \frac{89}{39} \approx 2.282\text{ bits/symbol}$
- **Shannon Entropy**: $H(X) \approx 2.186\text{ bits/symbol}$

---

## 4. Why Shannon-Fano is Sub-Optimal (vs. Huffman)

Shannon-Fano is a **greedy top-down heuristic**. Because it commits to high-level partitions early without knowing bottom-level compositions, it does not guarantee the minimum expected code length.

### Sub-optimality Demonstration:
Consider frequencies: `A: 35, B: 17, C: 17, D: 16, E: 15` (Sum = 100).
- **Shannon-Fano** split:
  - Left: `[A]` (35) vs Right: `[B, C, D, E]` (65) (diff = 30).
  - Or Left: `[A, B]` (52) vs Right: `[C, D, E]` (48) (diff = 4) $\rightarrow$ Left gets prefix `0`, Right gets prefix `1`.
  - Under Shannon-Fano, `A` receives a 2-bit code (`00`), requiring $35 \times 2 = 70$ bits.
- **Huffman Coding** (bottom-up):
  - Proves that `A` should receive a 1-bit code (`0`), requiring only 35 bits!
  - Huffman's average code length is strictly lower on this distribution.

---

## 5. Algorithmic Complexity

| Phase | Time Complexity | Space Complexity |
| :--- | :--- | :--- |
| **Frequency Analysis** | $O(N)$ | $O(K)$ |
| **Sorting Alphabet** | $O(K \log K)$ | $O(K)$ |
| **Tree Construction (Recursive Splitting)** | $O(K \log K)$ average, $O(K^2)$ worst | $O(K)$ call stack |
| **Encoding Bitstream** | $O(N)$ | $O(N)$ bits |
| **Decoding Bitstream** | $O(N)$ | $O(K)$ prefix tree |

*(where $N$ is input length and $K$ is unique alphabet size, $K \le 256$ for bytes).*

---

## 6. Implementation Checklist & Edge Cases

- [ ] **Single Unique Symbol**: Input like `"AAAAAA"` must not infinite-loop during partitioning.
- [ ] **Equal Frequencies**: Arbitrary tie-breaking must remain deterministic between encoder and decoder.
- [ ] **Bitstream Packing**: Packing variable-length bit strings into raw bytes (8 bits per byte) with valid EOF or bit-padding markers.
- [ ] **Header Serialization**: Storing the frequency table or codebook so the decoder can reconstruct the identical tree.

