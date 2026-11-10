# LZ77 (Sliding Window Dictionary Coding)

Published in 1977 by Abraham Lempel and Jacob Ziv, LZ77 is one of the most influential algorithms in computer science. It introduced the concept of **sliding-window dictionary compression**, forming the foundational engine of DEFLATE, gzip, ZIP, PNG, and 7-Zip.

---

## 1. Core Intuition

Text and binary data frequently repeat words, code snippets, or sequences. Rather than storing the repeated phrase, LZ77 replaces it with a **backwards pointer** to where the identical phrase appeared previously:

```
Original:    "the cat sat on the mat"
             ^-------^       ^-----^
LZ77:        "the cat sat on [distance=16, length=4]mat"
```

Because pointers `(distance, length)` typically consume far fewer bytes than the raw text, substantial compression is achieved.

---

## 2. Sliding Window Architecture

The encoder maintains a window partitioned into two contiguous regions:

```
[ ... Search Buffer ... | ... Lookahead Buffer ... ]
<------- Size W --------><-------- Size L --------->
(Already encoded data)   (Upcoming unencoded data)
```

1. **Search Buffer ($W$)**: History of the most recently processed symbols (e.g. 4KB to 32KB).
2. **Lookahead Buffer ($L$)**: The incoming stream of symbols awaiting compression (e.g. 16 to 258 bytes).
3. **Sliding Mechanism**: After emitting a match or literal, the window slides forward across the input stream.

### Classic Token Format
In original LZ77, every token is a 3-tuple:
$$\mathbf{(\text{offset}, \text{length}, \text{next\_char})}$$
- **$\text{offset}$**: Distance back into the search buffer where the match starts.
- **$\text{length}$**: Number of consecutive matching characters.
- **$\text{next\_char}$**: The first literal character following the match (ensures progress even when length is 0).

### The Overlapping Match Trick (Run-Length capability)
In LZ77, **match length can legally exceed match offset**!
For example, given history `'A'` and lookahead `'AAAAA'`:
- Token `(offset=1, length=5, 'B')` copies 5 bytes from offset 1, repeating `'A'` 5 times!

---

## 3. Step-by-Step Worked Example (Hand Trace)

### Configuration
- Search Buffer Capacity: $W = 7$
- Lookahead Buffer Capacity: $L = 4$
- Input String: `"CABRACADABRARR"` (14 chars)

### Step-by-Step Execution Trace

| Step | Search Buffer | Lookahead Buffer | Match Found | Offset | Length | Next Char | Emitted Token |
| :---: | :--- | :--- | :---: | :---: | :---: | :---: | :---: |
| **1** | `""` | `"CABR"` | None | 0 | 0 | `'C'` | `(0, 0, 'C')` |
| **2** | `"C"` | `"ABRA"` | None | 0 | 0 | `'A'` | `(0, 0, 'A')` |
| **3** | `"CA"` | `"BRAC"` | None | 0 | 0 | `'B'` | `(0, 0, 'B')` |
| **4** | `"CAB"` | `"RACA"` | None | 0 | 0 | `'R'` | `(0, 0, 'R')` |
| **5** | `"CABR"` | `"ACAD"` | `'A'` | 3 | 1 | `'C'` | `(3, 1, 'C')` |
| **6** | `"BRAC"` | `"ADAB"` | `'A'` | 2 | 1 | `'D'` | `(2, 1, 'D')` |
| **7** | `"ACAD"` | `"ABRA"` | `"ABRA"`? Search has `"A"`, `"RA"`. Longest: `"A"` | 4 | 1 | `'B'` | `(4, 1, 'B')` |
| **8** | `"ADAB"` | `"RARR"` | `'R'` | 3 | 1 | `'A'` | `(3, 1, 'A')` |
| **9** | `"ABRA"` | `"RR"` | `'R'` | 1 | 1 | `'R'` | `(1, 1, 'R')` |

### Decompression Trace
The decoder requires no search logic. For each token `(offset, length, char)`:
1. If $\text{length} > 0$, copy $\text{length}$ characters from $\text{current\_pos} - \text{offset}$.
2. Append $\text{next\_char}$.
This makes decompression extraordinarily fast ($O(N)$ memory copies).

---

## 4. Algorithmic Complexity

| Implementation Variant | Compression Time | Decompression Time | Memory Footprint |
| :--- | :--- | :--- | :--- |
| **Naive Linear Search** | $O(N \cdot W \cdot L)$ | $O(N)$ | $O(W + L)$ |
| **Hash-Chain Indexed (Production)** | $O(N \cdot L)$ average | $O(N)$ | $O(W + 2^{16})$ |
| **Suffix Tree / Array Match** | $O(N)$ | $O(N)$ | $O(N)$ |

---

## 5. Implementation Roadmap & Edge Cases

- [ ] **Overlapping Matches**: Handle `offset < length` (e.g. `(1, 10, 'X')`) where each copied character depends on the previously copied character in the same token.
- [ ] **Empty String**: Return empty token list immediately without indexing.
- [ ] **All Unique Characters**: Verify graceful fallback to literal tokens `(0, 0, char)`.
- [ ] **Sliding Window Shift**: Ensure old characters falling outside the window capacity $W$ are purged from search indices.
- [ ] **Round-Trip Test**: Verify `decode(encode(data)) == data` across binary data and repetitive logs.

