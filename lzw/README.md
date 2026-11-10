# LZW (Lempel-Ziv-Welch)

Published in 1984 by Terry Welch, LZW is a landmark refinement of LZ78. By pre-populating the dictionary with all single-character alphabet values, LZW completely eliminates literal characters from its output stream—emitting **purely integer dictionary codes**.

---

## 1. Why LZW Changed the Industry

In LZ78, every token was a pair: `(dictionary_index, next_char)`. Terry Welch asked: *What if the dictionary already contains all possible 1-character strings from the start?*

- **The Breakthrough**: Because every 1-byte character (0–255) is pre-registered, the matching prefix $W$ is guaranteed to never be empty.
- **Pure Code Stream**: The output consists entirely of integer indices. No raw characters are ever emitted.
- **Historical Impact**: LZW became the core algorithm of the **GIF image format** (1987), TIFF, the Unix `compress` utility, and the Adobe PDF standard.

---

## 2. Algorithm Mechanics

### Encoder Algorithm
1. **Initialize Dictionary**: Map every single byte value $0\dots255$ to its corresponding integer index:
   $$\text{Dict}[i] = \text{bytes}([i]) \quad \text{for } 0 \le i \le 255$$
   Set `next_code` = 256.
2. Let $W$ be an empty byte string.
3. For each byte $C$ in the input:
   - If $W + C$ exists in the dictionary:
     $$W \leftarrow W + C$$
   - Else:
     1. Emit code for $W$.
     2. Add $W + C$ to the dictionary at `next_code`.
     3. Increment `next_code`.
     4. $W \leftarrow C$.
4. At end of input, emit code for $W$.

---

## 3. Step-by-Step Worked Example (Hand Trace)

### Input Message: `"TOBEORNOTTOBEORTOBEORNOT"` (24 characters)
*(For illustration, assume alphabet codes: `B:1, E:2, N:3, O:4, R:5, T:6`, starting `next_code` = 7).*

#### Detailed Hand Trace

| Step | Current $W$ | Next $C$ | Combined $W + C$ | In Dict? | Output Code | New Dict Entry | New Code |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **1** | `""` | `'T'` | `"T"` | Yes (6) | — | — | — |
| **2** | `"T"` | `'O'` | `"TO"` | No | **6** (`'T'`) | `"TO"` | 7 |
| **3** | `"O"` | `'B'` | `"OB"` | No | **4** (`'O'`) | `"OB"` | 8 |
| **4** | `"B"` | `'E'` | `"BE"` | No | **1** (`'B'`) | `"BE"` | 9 |
| **5** | `'E'` | `'O'` | `"EO"` | No | **2** (`'E'`) | `"EO"` | 10 |
| **6** | `"O"` | `'R'` | `"OR"` | No | **4** (`'O'`) | `"OR"` | 11 |
| **7** | `"R"` | `'N'` | `"RN"` | No | **5** (`'R'`) | `"RN"` | 12 |
| **8** | `"N"` | `'O'` | `"NO"` | No | **3** (`'N'`) | `"NO"` | 13 |
| **9** | `"O"` | `'T'` | `"OT"` | No | **4** (`'O'`) | `"OT"` | 14 |
| **10** | `"T"` | `'T'` | `"TT"` | No | **6** (`'T'`) | `"TT"` | 15 |
| **11** | `"T"` | `'O'` | `"TO"` | Yes (7) | — | — | — |
| **12** | `"TO"` | `'B'` | `"TOB"` | No | **7** (`"TO"`) | `"TOB"` | 16 |
| **13** | `"B"` | `'E'` | `"BE"` | Yes (9) | — | — | — |
| **14** | `"BE"` | `'O'` | `"BEO"` | No | **9** (`"BE"`) | `"BEO"` | 17 |
| **15** | `"O"` | `'R'` | `"OR"` | Yes (11) | — | — | — |
| **16** | `"OR"` | `'T'` | `"ORT"` | No | **11** (`"OR"`) | `"ORT"` | 18 |
| **17** | `"T"` | `'O'` | `"TO"` | Yes (7) | — | — | — |
| **18** | `"TO"` | `'B'` | `"TOB"` | Yes (16) | — | — | — |
| **19** | `"TOB"`| `'E'` | `"TOBE"` | No | **16** (`"TOB"`)| `"TOBE"` | 19 |
| **20** | `"E"` | `'O'` | `"EO"` | Yes (10) | — | — | — |
| **21** | `"EO"` | `'R'` | `"EOR"` | No | **10** (`"EO"`)| `"EOR"` | 20 |
| **22** | `"R"` | `'N'` | `"RN"` | Yes (12) | — | — | — |
| **23** | `"RN"` | `'O'` | `"RNO"` | No | **12** (`"RN"`)| `"RNO"` | 21 |
| **24** | `"O"` | `'T'` | `"OT"` | Yes (14) | — | — | — |
| **End**| `"OT"` | EOF | — | — | **14** (`"OT"`)| — | — |

**Output Codes**: `[6, 4, 1, 2, 4, 5, 3, 4, 6, 7, 9, 11, 16, 10, 12, 14]` (16 integers vs 24 raw chars).

---

## 4. The Infamous "KwKwK" Decoder Edge Case

There is a subtle, notorious scenario where the encoder emits a code that the decoder **does not yet have in its dictionary**.

### How it Happens:
Suppose input contains the repeating pattern `"ABABABA"`.
- When the encoder processes the final `"ABA"`, it adds `"ABA"` to the dictionary and immediately emits the new code.
- When the decoder reads this code, it has not yet completed the addition of `"ABA"`!
- The condition occurs whenever the input matches the pattern:
  $$\text{String } W = K + \text{phrase} + K \quad (\text{starts and ends with the same character } K).$$

### The Universal Fix in the Decoder:
```python
if code in dictionary:
    entry = dictionary[code]
elif code == next_code:
    # KwKwK special case!
    entry = previous_entry + previous_entry[0:1]
else:
    raise ValueError(f"Corrupt LZW code: {code}")
```
This single check solves the edge case and guarantees 100% round-trip fidelity.

---

## 5. Algorithmic Complexity

| Operation | Time Complexity | Space Complexity |
| :--- | :--- | :--- |
| **Encoder** | $O(N)$ with hash map / trie | $O(\text{Max Codes})$ (e.g. 4,096 entries for 12-bit) |
| **Decoder** | $O(N)$ direct array lookups | $O(\text{Max Codes})$ |

---

## 6. Implementation Checklist & Edge Cases

- [ ] **Pre-populated 0–255 Alphabet**: Ensure byte dictionary indices are properly initialized.
- [ ] **The KwKwK Test**: Specifically test inputs like `"AAAAAAA"` or `"abababa"` to trigger and verify the KwKwK branch.
- [ ] **Bit-Packing Strategy**:
  - Variable-width bit packing (start at 9 bits; when `next_code == 512`, expand to 10 bits, up to 12 bits max 4096).
- [ ] **Table Overflow Policy**: Freeze dictionary updates once 4096 codes are reached, or emit a `CLEAR_CODE` to reset.
- [ ] **Round-Trip Test**: `decode(encode(data)) == data`.

