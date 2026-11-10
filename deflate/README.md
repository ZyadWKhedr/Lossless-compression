# DEFLATE (LZ77 + Huffman Hybrid)

Specified in **RFC 1951** by Phil Katz and Jean-loup Gailly, DEFLATE is the undisputed workhorse of digital compression. It powers **gzip**, **zlib**, **ZIP**, **PNG images**, and **HTTP content-encoding** across the global internet.

---

## 1. Core Intuition: The Capstone Hybrid

No single compression paradigm solves all redundancy:
- **LZ77** excels at finding repeated multi-byte phrases, but leaves individual literal characters and match pointers uncompressed.
- **Huffman Coding** excels at exploiting skewed single-symbol frequencies, but is completely blind to multi-byte phrase repetitions.

DEFLATE synthesizes both methods into a two-stage pipeline:
1. **Stage 1 (LZ77)**: Replaces repeated phrases with `(length, distance)` pointers, outputting a stream of literals, lengths, and distances.
2. **Stage 2 (Huffman)**: Compresses the literals, lengths, and distances using two custom Huffman trees.

```
Raw Bytes ==> [ LZ77 Engine ] ==> Literals / (Len, Dist) ==> [ Dual Huffman Coders ] ==> RFC 1951 Bitstream
```

---

## 2. RFC 1951 Architecture & Block Types

A DEFLATE stream consists of a series of independent blocks. Each block begins with a 3-bit header:
- **`BFINAL` (1 bit)**: `1` if this is the last block of the stream, `0` otherwise.
- **`BTYPE` (2 bits)**:
  - `00`: **Stored / Uncompressed** (raw bytes up to 65,535 bytes; used when data is incompressible to guarantee zero expansion).
  - `01`: **Fixed Huffman** (uses pre-standardized, hardcoded RFC 1951 Huffman tables; zero header overhead).
  - `10`: **Dynamic Huffman** (builds custom Huffman trees optimized for the current block's exact frequency profile).
  - `11`: Reserved / Error.

---

## 3. Dual Alphabet Representation

DEFLATE divides tokens into two separate alphabets:

### Alphabet 1: Literals / Lengths (Symbols 0–285)
- **Codes 0–255**: Literal byte values (`0x00`–`0xFF`).
- **Code 256**: **End-of-Block (EOB)** marker.
- **Codes 257–285**: Match lengths ($3\dots258$ bytes). Codes may be followed by $0\dots5$ extra bits to specify exact length:
  - Code 257 = length 3 (0 extra bits).
  - Code 265 = length $11\dots12$ (1 extra bit).
  - Code 285 = length 258 (0 extra bits).

### Alphabet 2: Distances (Symbols 0–29)
- Represents backwards offsets ($1\dots32,768$ bytes into the 32KB sliding window).
- Codes $0\dots29$ are followed by $0\dots13$ extra bits to identify the exact offset.

### The Meta-Compression (The 3rd Huffman Tree)
In dynamic blocks, the code lengths of the two main trees are themselves run-length encoded and compressed using a **third miniature Huffman tree** (19 symbols). This recursive engineering ensures minimal header size.

---

## 4. Step-by-Step Worked Example (Hand Trace)

### Input String
`"the rain in spain falls mainly on the plain"` (44 bytes)

#### Step 1: LZ77 Parsing Phase
- `"the rain in sp"` $\rightarrow$ Literals.
- `"ain"` $\rightarrow$ Match! Offset 9 (`"ain "` in `"rain "`), Length 3.
- `" falls m"` $\rightarrow$ Literals.
- `"ain"` $\rightarrow$ Match! Offset 18, Length 3.
- `"ly on "` $\rightarrow$ Literals.
- `"the "` $\rightarrow$ Match! Offset 38, Length 4.
- `"plain"` $\rightarrow$ `'p'` literal + `"lain"` Match (Offset 34, Length 4).

#### Step 2: Token Stream Assembly
```
Literals:      't', 'h', 'e', ' ', 'r', 'a', 'i', 'n', ' ', 'i', 'n', ' ', 's', 'p'
Pointer 1:     (Length=3, Distance=9)
Literals:      ' ', 'f', 'a', 'l', 'l', 's', ' ', 'm'
Pointer 2:     (Length=3, Distance=18)
Literals:      'l', 'y', ' ', 'o', 'n', ' '
Pointer 3:     (Length=4, Distance=38)
Literal:       'p'
Pointer 4:     (Length=4, Distance=34)
End of Block:  EOB (Code 256)
```

#### Step 3: Huffman Entropy Pass
1. The literal/length stream contains frequent space characters `' '` and frequent length codes.
2. The distance codes are concentrated around small offsets.
3. High-frequency symbols receive 2-to-4 bit prefix codes.
4. Final bitstream is packed LSB-first into bytes according to RFC 1951 conventions.

---

## 5. Algorithmic Complexity

| Phase | Time Complexity | Memory Requirements |
| :--- | :--- | :--- |
| **LZ77 Hash-Chain Search** | $O(N)$ average | 32 KB sliding window + hash table ($2^{15}$ buckets) |
| **Huffman Tree Construction** | $O(K \log K)$ ($K \le 286$) | $< 2\text{ KB}$ |
| **Bitstream Serialization** | $O(N)$ | Streaming buffer |
| **RFC 1951 Decompression** | $O(N)$ linear | 32 KB history ring buffer |

---

## 6. Implementation Checklist & Edge Cases

- [ ] **Bit Ordering (RFC 1951)**: Huffman codes are packed starting from the **Least Significant Bit (LSB)** of each byte; Huffman code bit patterns are MSB-first.
- [ ] **Stored Block Support (`BTYPE=00`)**: Ensure bit alignment to next byte boundary before reading raw byte counts.
- [ ] **Sliding Window Ring Buffer**: Maintain an efficient 32KB circular buffer for decoding distance pointers.
- [ ] **Max Match Length (258)**: Correctly clamp lookahead matches to RFC 1951's maximum length of 258 bytes.
- [ ] **Round-Trip Test**: Verify against standard Python `zlib.decompress` / `gzip` compatibility.

