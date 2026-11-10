# Huffman Coding

Huffman coding, invented by David Huffman in 1952 while completing his Ph.D. term paper at MIT, is the universally celebrated standard for **optimal prefix-free variable-length entropy coding**.

---

## 1. Core Intuition & Mathematical Optimality

Huffman coding is provably optimal among all symbol-by-symbol prefix codes:
- **Prefix-Free Property**: No codeword is a prefix of any other codeword (e.g., if `'A'` is `0`, no other code can begin with `0`). This allows deterministic, instantaneous decoding without delimiters.
- **Optimality**: For a given symbol alphabet and discrete probability distribution, Huffman's bottom-up greedy construction produces a prefix code with the **minimum expected codeword length**:
  $$L(C) = \sum_{i=1}^K P(x_i) \cdot l_i$$
  where $l_i$ is the length of code assigned to symbol $x_i$, and:
  $$H(X) \le L(C) < H(X) + 1$$

---

## 2. The Bottom-Up Greedy Min-Heap Algorithm

Instead of partitioning from the top down (like Shannon-Fano), Huffman builds the binary tree from the **leaves up to the root**:

1. **Calculate Frequencies**: Compute the frequency $f(c)$ for every symbol $c \in \Sigma$.
2. **Initialize Min-Heap**: Insert each symbol as a single-node tree into a min-priority queue keyed by frequency.
3. **Iterative Merge**: While the heap contains more than 1 node:
   - Extract the two nodes with the lowest frequencies: $N_1$ and $N_2$.
   - Create an internal node $P$ with children $N_1$ (left, bit `0`) and $N_2$ (right, bit `1`).
   - Set $f(P) = f(N_1) + f(N_2)$.
   - Re-insert $P$ into the min-heap.
4. **Extract Codewords**: The remaining single node is the root. Traverse from root to each leaf to extract the binary codeword.

---

## 3. Step-by-Step Worked Example (Hand Trace)

### Input Data
String: `"ABRACADABRA"` (Length: 11 characters)

### 1. Frequency Table
| Symbol | Frequency |
| :---: | :---: |
| **A** | 5 |
| **B** | 2 |
| **R** | 2 |
| **C** | 1 |
| **D** | 1 |

Total characters: 11. Initial Min-Heap: `[C:1, D:1, B:2, R:2, A:5]`.

### 2. Min-Heap Merge Steps

- **Merge 1**:
  - Pop `C:1` and `D:1`.
  - Create parent `[CD]:2` (Left child: `C`, Right child: `D`).
  - Heap becomes: `[B:2, R:2, [CD]:2, A:5]`.

- **Merge 2**:
  - Pop `B:2` and `R:2`.
  - Create parent `[BR]:4` (Left child: `B`, Right child: `R`).
  - Heap becomes: `[[CD]:2, [BR]:4, A:5]`.

- **Merge 3**:
  - Pop `[CD]:2` and `A:5`? No, heap order: `[CD]:2` and `[BR]:4` are lowest!
  - Create parent `[[CD][BR]]:6` (Left child: `[CD]`, Right child: `[BR]`).
  - Heap becomes: `[A:5, [[CD][BR]]:6]`.

- **Merge 4**:
  - Pop `A:5` and `[[CD][BR]]:6`.
  - Create Root node with weight 11 (Left child: `A`, Right child: internal node).

### 3. Generated Huffman Tree Diagram

```
                 [Root: 11]
                /          \
            0  /            \  1
          [A: 5]          [Node: 6]
                         /         \
                     0  /           \  1
                   [CD: 2]         [BR: 4]
                   /     \         /     \
               0  /     1 \    0  /     1 \
               [C:1]   [D:1]   [B:2]   [R:2]
```

### 4. Resulting Prefix Codebook

| Symbol | Frequency ($f_i$) | Codeword | Bit Length ($l_i$) | Total Bits ($f_i \times l_i$) |
| :---: | :---: | :--- | :---: | :---: |
| **A** | 5 | `0` | 1 | 5 |
| **B** | 2 | `110` | 3 | 6 |
| **R** | 2 | `111` | 3 | 6 |
| **C** | 1 | `100` | 3 | 3 |
| **D** | 1 | `101` | 3 | 3 |
| **Total** | **11** | — | — | **23 bits** |

### 5. Encoded Bitstream
Original message: `A  B   R   A  C   A  D   A  B   R   A`
Encoded bits:     `0 110 111 0 100 0 101 0 110 111 0`

- **Raw 8-bit ASCII size**: $11 \times 8 = 88\text{ bits}$ (11 bytes)
- **Huffman Bitstream size**: 23 bits ($\approx 3\text{ bytes}$)
- **Compression Ratio**: $\frac{88}{23} \approx 3.83\times$ ($73.9\%$ space savings)

---

## 4. Canonical Huffman Representation

In real-world implementations (e.g. DEFLATE / JPEG), storing the full tree pointer structure inside file headers is wasteful. Instead, **Canonical Huffman Codes** are used:
1. All codes of the same bit length are assigned sequentially in numerical order.
2. The decoder only needs a compact list of **code lengths per symbol** (e.g. `A:1, B:3, R:3, C:3, D:3`), slashing header size by up to 80%.

---

## 5. Algorithmic Complexity

| Phase | Time Complexity | Auxiliary Space Complexity |
| :--- | :--- | :--- |
| **Frequency Counting** | $O(N)$ | $O(K)$ |
| **Tree Construction (Min-Heap)** | $O(K \log K)$ | $O(K)$ |
| **Bitstream Encoding** | $O(N)$ | $O(N)$ |
| **Bitstream Decoding** | $O(N)$ (bit-by-bit tree descent) | $O(K)$ |

*(For byte streams, $K \le 256$, making tree construction effectively $O(1)$ constant time overhead).*

---

## 6. Implementation Checklist & Edge Cases

- [ ] **Single Unique Symbol**: Input like `"AAAAAAAA"` (tree contains only 1 leaf; assigning a 1-bit code `0` is necessary).
- [ ] **Empty Input**: `b""` should produce clean empty output without tree construction errors.
- [ ] **Bitstream Padding**: When bit count is not a multiple of 8, pad with zeros and record either the valid bit count or an explicit EOF symbol.
- [ ] **Header Overhead**: On tiny files, the tree header may be larger than the compressed payload (a classic real-world tradeoff).
- [ ] **Round-Trip Test**: Verify `decode(encode(data)) == data` for text, binary, and extreme edge cases.

