# Adaptive Huffman Coding (FGK / Vitter)

Adaptive Huffman coding (dynamic Huffman) is a **single-pass** entropy coding algorithm that builds and updates the Huffman tree dynamically as data is processed. Both the encoder and decoder maintain identical, synchronized trees without requiring a pre-computed frequency table or file header.

---

## 1. Why Adaptive Huffman?

Static Huffman coding suffers from two distinct disadvantages:
1. **Two-Pass Requirement**: The entire input must be scanned once to count symbol frequencies, and a second time to encode the bits. This prevents real-time streaming over networks.
2. **Header Overhead**: The frequency table or tree topology must be transmitted in the file header, which can significantly expand small files.

Adaptive Huffman eliminates both problems: transmission begins immediately on byte 1, and the codebook continuously adapts to changing statistics (**non-stationary sources**).

---

## 2. Key Theoretical Concepts

### The Sibling Property (Gallager, 1978)
A binary tree with weights possesses the **Sibling Property** if its nodes can be arranged in a sequence of non-increasing weights, such that each node in the sequence is adjacent to its sibling. 

A tree is an optimal Huffman tree **if and only if** it satisfies the Sibling Property.

### The NYT (Not Yet Transmitted) Symbol
Because the alphabet is not known in advance, the tree contains a special leaf node called **NYT** (also called the 0-node, weight 0):
- When an **unseen symbol** arrives:
  1. The encoder transmits the current path to the **NYT** node.
  2. The encoder transmits the raw, uncompressed symbol (e.g. 8 bits).
  3. The NYT node splits into a new parent, a new NYT leaf (weight 0), and a new symbol leaf (weight 1).
- When an **already-seen symbol** arrives:
  1. The encoder transmits the current path from the root to that symbol's leaf.
  2. The symbol's weight is incremented, and the tree is rebalanced via node swaps if the Sibling Property is threatened.

### FGK vs. Vitter Algorithm
- **FGK (Faller, Gallager, Knuth)**: Traverses from the updated node to the root, swapping any node that exceeds the highest-numbered node of the same weight before incrementing.
- **Vitter (1987)**: Introduces invariant properties that minimize external path length, guaranteeing fewer total node swaps and slightly improved compression ratios.

---

## 3. Step-by-Step Worked Example (Hand Trace)

### Input Message: `"BOOK"`
Alphabet: standard 8-bit ASCII.

```
Initial State:
   [NYT (wt: 0, id: 512)]
```

#### Step 1: Process `'B'` (First appearance)
1. Message sent: **(Empty NYT path)** + raw ASCII `'B'` (`01000010`).
2. Tree update: The initial NYT splits into:
   - Internal Node (weight 1)
   - Left Child: `NYT` (weight 0)
   - Right Child: `'B'` (weight 1)

```
        (Node: 1)
       /         \
    0 /           \ 1
   [NYT: 0]      ['B': 1]
```

#### Step 2: Process `'O'` (First appearance)
1. Transmit path to NYT: bit `0`.
2. Transmit raw ASCII `'O'` (`01001111`).
3. NYT splits into new internal node, child `'O'`, and new NYT:

```
              (Root: 2)
             /         \
          0 /           \ 1
      (Node: 1)       ['B': 1]
      /       \
   0 /         \ 1
  [NYT: 0]    ['O': 1]
```

#### Step 3: Process `'O'` (Second appearance)
1. `'O'` is already in the tree! Transmit current code: `01`.
2. Weight of `'O'` increases from 1 to 2.
3. **Sibling Property Check**: Node `'O'` (weight 2) now outweighs its parent's sibling `'B'` (weight 1).
4. **Tree Swap**: Node `'O'` swaps places with the highest node in weight block 1 (node `'B'`).
5. Propagate weight update to root (Root weight becomes 3).

```
              (Root: 3)
             /         \
          0 /           \ 1
      ['B': 1]        ['O': 2]   <-- 'O' promoted to shorter 1-bit code!
      /       \
   0 /         \ 1
  [NYT: 0]   (Unused/Rebalanced)
```

#### Step 4: Process `'K'` (First appearance)
1. Transmit current path to NYT: `00`.
2. Transmit raw ASCII `'K'` (`01001011`).
3. Split NYT and rebalance.

---

## 4. Algorithmic Complexity

| Operation | Time Complexity (per symbol) | Total Time ($N$ symbols) | Space Complexity |
| :--- | :--- | :--- | :--- |
| **Lookup Symbol Node** | $O(1)$ (via direct array map) | $O(N)$ | $O(K)$ |
| **Code Generation** | $O(D)$ where $D \le K$ is tree depth | $O(N \cdot D)$ | $O(K)$ |
| **Tree Swap & Rebalance** | $O(D)$ | $O(N \cdot D)$ | $O(K)$ |
| **Overall** | $O(D)$ | $O(N \cdot \text{depth})$ | $O(K)$ nodes |

For an 8-bit alphabet ($K = 256$), the maximum tree depth $D \le 256$, making adaptive Huffman run in linear time $O(N)$ with fixed memory overhead ($\le 513$ tree nodes).

---

## 5. Implementation Roadmap & Edge Cases

- [ ] **Dual-Tree Synchronization**: The decoder must perform the exact same node swaps in the exact same sequence as the encoder.
- [ ] **Block Leader Identification**: Proper identification of the highest-numbered node in a weight block during swap procedures.
- [ ] **Escape Sequence (NYT)**: Handling 8-bit literal byte reading immediately after encountering an NYT code.
- [ ] **Zero-Length Stream**: Gracefully terminating if the stream is empty.
- [ ] **Alphabet Saturation**: When all 256 byte values have been transmitted, NYT is never emitted again.

