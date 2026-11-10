# Run-Length Encoding (RLE)

Run-Length Encoding (RLE) is the simplest form of lossless data compression. It replaces consecutive sequences of identical data elements (called **runs**) with a single data value and a count of how many times that value repeats.

---

## 1. Core Intuition & Theory

Data often exhibits high **spatial or temporal locality**. In graphics (icons, black-and-white faxes, bitmap masks) or tabular records, long contiguous sequences of identical bytes occur frequently:

```
Original:    W W W W W W W W W W W W B B B
Compressed:  (12, W) (3, B)
```

Instead of storing 15 individual characters, we store 2 pairs (4 bytes total), achieving a $3.75\times$ compression ratio.

### Real-World Applications
- **TIFF & BMP Graphics**: Used for binary masks and palette images (PackBits variant).
- **Facsimile (Fax) Transmission**: ITU-T T.4 and T.6 standard for 1-bit black/white scanned documents.
- **JPEG / MPEG**: Compresses zero-valued high-frequency DCT transform coefficients.

---

## 2. Encoding Schemes

### Scheme A: Byte-Pair RLE (Simple)
Every run is serialized as two bytes: `[Count (1 byte), Value (1 byte)]`.
- **Maximum run length per byte**: 255.
- **Run overflow**: If a run exceeds 255 bytes (e.g., 300 `A`s), it is split into multiple runs: `(255, 'A')` followed by `(45, 'A')`.
- **Tradeoff**: Extremely fast, but expands data when runs are of length 1 (`"ABC"` expands from 3 bytes to 6 bytes).

### Scheme B: PackBits / Flagged RLE (Production Grade)
Uses a signed header byte $n$ to distinguish literal runs from repeating runs:
- If $0 \le n \le 127$: The next $n + 1$ bytes are **uncompressed literals**.
- If $-127 \le n \le -1$: The next single byte is **repeated** $1 - n$ times (up to 128 times).
- $n = -128$: Reserved / NOP.
- **Tradeoff**: Eliminates catastrophic expansion on uncompressible data.

---

## 3. Step-by-Step Worked Example (Hand Trace)

### Input Data
String: `"AAAAABBBCCDAAAAA"` (Length: 16 bytes)

### Encoding Trace (Scheme A: Count-Byte Pairs)
1. Scan `'A'`: count reaches 5. Next character is `'B'` $\rightarrow$ Emit `(5, 'A')`.
2. Scan `'B'`: count reaches 3. Next character is `'C'` $\rightarrow$ Emit `(3, 'B')`.
3. Scan `'C'`: count reaches 2. Next character is `'D'` $\rightarrow$ Emit `(2, 'C')`.
4. Scan `'D'`: count reaches 1. Next character is `'A'` $\rightarrow$ Emit `(1, 'D')`.
5. Scan `'A'`: count reaches 5. End of input $\rightarrow$ Emit `(5, 'A')`.

### Output Bytestream
| Pair # | Run Count (Byte) | Symbol (Byte) | Hex Representation |
| :--- | :--- | :--- | :--- |
| 1 | 5 | `'A'` | `0x05 0x41` |
| 2 | 3 | `'B'` | `0x03 0x42` |
| 3 | 2 | `'C'` | `0x02 0x43` |
| 4 | 1 | `'D'` | `0x01 0x44` |
| 5 | 5 | `'A'` | `0x05 0x41` |

- **Original Size**: 16 bytes
- **Compressed Size**: 10 bytes
- **Compression Ratio**: $\frac{16}{10} = 1.6\times$ ($37.5\%$ space savings)

### Pathological Worst-Case (Data Expansion)
Input: `"ABCDEFGH"` (8 distinct bytes)
- Emitted: `(1, 'A'), (1, 'B'), (1, 'C'), (1, 'D'), (1, 'E'), (1, 'F'), (1, 'G'), (1, 'H')`
- Output: 16 bytes ($200\%$ size of original, $CR = 0.5\times$).

---

## 4. Algorithmic Complexity

| Phase | Time Complexity | Auxiliary Space Complexity | Streamable? |
| :--- | :--- | :--- | :--- |
| **Encoding** | $O(N)$ | $O(1)$ | Yes |
| **Decoding** | $O(M)$ (where $M$ is uncompressed length) | $O(1)$ | Yes |

---

## 5. Implementation Roadmap & Edge Cases

When implementing `encoder.py` and `decoder.py`, ensure round-trip correctness:
$$\text{decode}(\text{encode}(x)) \equiv x$$

### Crucial Edge Cases to Test:
- [ ] **Empty Input**: `b""` should produce `b""`.
- [ ] **Single Byte**: `b"A"` should encode and decode cleanly.
- [ ] **Run Overflow**: Sequence of 300 consecutive `b"X"` symbols must split across 255-byte boundaries.
- [ ] **No Repeating Symbols**: Distinct bytes `b"abcdefg123456"` must not crash or corrupt.
- [ ] **Binary Values**: Raw byte values `0x00`, `0xFF`, and control characters.
- [ ] **Corrupt Stream**: Truncated compressed byte stream (odd byte count in Scheme A) should raise an informative `ValueError`.

