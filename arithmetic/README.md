# Arithmetic Coding

Arithmetic coding is widely considered the most mathematically elegant and efficient entropy coding algorithm in data compression. Unlike Huffman coding, which must assign an integer number of bits to each symbol, arithmetic coding encodes an entire message into a **single arbitrary-precision fractional number** in the interval $[0.0, 1.0)$.

---

## 1. Why Arithmetic Coding Beats Huffman

In Huffman coding, every symbol must consume at least **1 full bit**. Consider a symbol that appears with $95\%$ probability ($P = 0.95$):
- Theoretical information content: $-\log_2(0.95) \approx 0.074\text{ bits}$.
- Huffman coding must assign at least **1 bit**, wasting over **$92\%$** of transmission capacity on that symbol!

Arithmetic coding breaks the integer-bit barrier. It allocates fractional bits per symbol, allowing it to approach the Shannon Entropy limit arbitrarily closely:
$$\lim_{N \to \infty} \frac{\text{Bits}(N)}{N} = H(X)$$

---

## 2. Theoretical Mechanics (The Continuous Model)

1. The message starts with the full interval $[0.0, 1.0)$.
2. Each unique symbol in the alphabet $\Sigma$ is assigned a sub-interval proportional to its probability:
   $$\text{Range}(c) = [\text{Low}(c), \text{High}(c))$$
3. As each symbol $s$ in the message arrives, the current interval $[L, H)$ is narrowed:
   $$\text{Current Range} = H - L$$
   $$L_{\text{new}} = L + \text{Current Range} \times \text{Low}(s)$$
   $$H_{\text{new}} = L + \text{Current Range} \times \text{High}(s)$$
4. After processing the final symbol, any single number within the final interval $[L, H)$ unambiguously identifies the original message.

---

## 3. Step-by-Step Worked Example (Hand Trace)

### Alphabet & Probability Model
| Symbol | Probability ($P$) | Cumulative Range $[\text{Low}, \text{High})$ |
| :---: | :---: | :---: |
| **A** | $0.50$ | $[0.00, 0.50)$ |
| **B** | $0.30$ | $[0.50, 0.80)$ |
| **C** | $0.20$ | $[0.80, 1.00)$ |

### Input Message: `"BAC"`

```
Initial State:  [0.0, 1.0),  Range = 1.0
```

#### Symbol 1: `'B'`
- Symbol range: $[0.5, 0.8)$
- $L_1 = 0.0 + 1.0 \times 0.50 = 0.50$
- $H_1 = 0.0 + 1.0 \times 0.80 = 0.80$
- **New Interval**: $[0.50, 0.80)$, $\text{Range} = 0.30$

#### Symbol 2: `'A'`
- Symbol range: $[0.0, 0.5)$
- $L_2 = 0.50 + 0.30 \times 0.00 = 0.50$
- $H_2 = 0.50 + 0.30 \times 0.50 = 0.65$
- **New Interval**: $[0.50, 0.65)$, $\text{Range} = 0.15$

#### Symbol 3: `'C'`
- Symbol range: $[0.8, 1.0)$
- $L_3 = 0.50 + 0.15 \times 0.80 = 0.62$
- $H_3 = 0.50 + 0.15 \times 1.00 = 0.65$
- **Final Interval**: $[0.62, 0.65)$

### Selecting Final Codeword
Any number $x \in [0.62, 0.65)$ can be chosen. In binary:
- $0.625 = \frac{5}{8} = 0.101_2$
- The binary fraction `0.101` falls squarely inside $[0.62, 0.65)$.
- The transmitted bitstream is simply: **`101`** (only 3 bits for 3 symbols!).

### Decoding Verification (Hand Trace)
Decoder knows the cumulative ranges and receives code $v = 0.625$:
1. $0.625 \in [0.50, 0.80) \implies$ Emit `'B'`. Rescale $v \leftarrow \frac{0.625 - 0.50}{0.30} \approx 0.4167$.
2. $0.4167 \in [0.00, 0.50) \implies$ Emit `'A'`. Rescale $v \leftarrow \frac{0.4167 - 0.00}{0.50} \approx 0.8333$.
3. $0.8333 \in [0.80, 1.00) \implies$ Emit `'C'`. Message complete!

---

## 4. Finite Integer Arithmetic & Renormalization

Standard 64-bit floating-point numbers exhaust precision after ~15 symbols. Practical implementations (e.g. Witten, Neal, and Cleary 1987) use **fixed-precision integer registers** (e.g., 32-bit or 64-bit integers) and stream out bits using **renormalization**:

- Let integer bounds be $[0, 2^{32}-1]$.
- **$E_1$ Scaling (Upper half output)**: When both `low` and `high` reside in $[0, \text{Half})$, output bit `0` and shift left.
- **$E_2$ Scaling (Lower half output)**: When both `low` and `high` reside in $[\text{Half}, \text{Full})$, output bit `1` and shift left.
- **$E_3$ Scaling (Underflow prevention)**: When `low` is in $[\text{Quarter}, \text{Half})$ and `high` is in $[\text{Half}, \text{ThreeQuarter})$, the interval straddles the midpoint. Increment an `underflow_bits` counter and rescale from the center.

---

## 5. Algorithmic Complexity

| Phase | Time Complexity | Auxiliary Space Complexity |
| :--- | :--- | :--- |
| **Frequency Model Construction** | $O(N)$ | $O(K)$ |
| **Streaming Encoding** | $O(N)$ | $O(1)$ |
| **Streaming Decoding** | $O(N \log K)$ (binary search on cumulative table) | $O(K)$ |

---

## 6. Implementation Checklist & Edge Cases

- [ ] **Stream Termination**: Arithmetic codes need either an explicit **EOF symbol** or a 32-bit uncompressed length prefix in the header.
- [ ] **Underflow Straddle**: Ensure $E_3$ underflow expansion correctly emits delayed inverted bits.
- [ ] **Adaptive Model Support**: Frequency counts can increment dynamically during streaming, eliminating the need to send frequency tables upfront.
- [ ] **Rounding & Overflow**: Prevent 32-bit register overflow during intermediate range multiplications:
  $$\text{range} \times \text{freq} \implies \text{requires 64-bit intermediate arithmetic}.$$

