# LZ78 (Dictionary-Based Compression)

Published in 1978 by Abraham Lempel and Jacob Ziv, LZ78 departs from the sliding-window model of LZ77 by constructing an **explicit, growing dictionary of phrases**.

---

## 1. Core Intuition & Architecture

While LZ77 searches backwards through a sliding window of raw bytes, LZ78 builds a **trie of phrases** observed in the input stream. Each token emitted references a previously registered phrase index and appends exactly one new character:

$$\mathbf{(\text{dictionary\_index}, \text{next\_char})}$$

By pairing an existing dictionary index with a new character, every single token automatically registers a new phrase of length $L + 1$ into the dictionary without transmitting extra data.

---

## 2. Algorithm Mechanics

### Encoder
1. **Initialize Dictionary**: Index `0` points to the empty string `""`.
2. **Track Current Phrase**: Maintain a matching prefix string $P$ (starts empty).
3. **Scan Next Symbol $c$**:
   - If $P + c$ exists in the dictionary, set $P \leftarrow P + c$ and continue scanning.
   - If $P + c$ does **not** exist in the dictionary:
     1. Emit `(Index(P), c)`.
     2. Add $P + c$ to the dictionary at the next available index.
     3. Reset $P \leftarrow ""$.
4. **End of Input**: If $P$ is not empty, emit `(Index(P), EOF)` or `(Index(P[:-1]), P[-1])`.

### Decoder
1. **Initialize Dictionary**: Index `0` points to `""`.
2. For each received token `(index, char)`:
   - Reconstruct phrase: $\text{phrase} = \text{Dictionary}[index] + char$.
   - Output $\text{phrase}$.
   - Add $\text{phrase}$ to next available dictionary slot.

---

## 3. Step-by-Step Worked Example (Hand Trace)

### Input String: `"BABAABRRRA"` (10 characters)

#### Encoding Trace

| Step | Scan | Prefix $P$ | Lookahead $c$ | $P + c$ in Dict? | Action / Emitted Token | New Dict Entry |
| :---: | :---: | :---: | :---: | :---: | :--- | :--- |
| **0** | — | — | — | — | Initialize | `0: ""` |
| **1** | `'B'` | `""` | `'B'` | No (`"B"`) | Emit **`(0, 'B')`** | `1: "B"` |
| **2** | `'A'` | `""` | `'A'` | No (`"A"`) | Emit **`(0, 'A')`** | `2: "A"` |
| **3** | `'B'` | `""` | `'B'` | Yes (`"B"`) | $P \leftarrow \text{"B"}$ | — |
| **3 cont** | `'A'` | `"B"` | `'A'` | No (`"BA"`) | Emit **`(1, 'A')`** | `3: "BA"` |
| **4** | `'A'` | `""` | `'A'` | Yes (`"A"`) | $P \leftarrow \text{"A"}$ | — |
| **4 cont** | `'B'` | `"A"` | `'B'` | No (`"AB"`) | Emit **`(2, 'B')`** | `4: "AB"` |
| **5** | `'R'` | `""` | `'R'` | No (`"R"`) | Emit **`(0, 'R')`** | `5: "R"` |
| **6** | `'R'` | `""` | `'R'` | Yes (`"R"`) | $P \leftarrow \text{"R"}$ | — |
| **6 cont** | `'R'` | `"R"` | `'R'` | No (`"RR"`) | Emit **`(5, 'R')`** | `6: "RR"` |
| **7** | `'R'` | `""` | `'R'` | Yes (`"R"`) | $P \leftarrow \text{"R"}$ | — |
| **7 cont** | `'A'` | `"R"` | `'A'` | No (`"RA"`) | Emit **`(5, 'A')`** | `7: "RA"` |

### Emitted Stream of Tokens:
`[(0, 'B'), (0, 'A'), (1, 'A'), (2, 'B'), (0, 'R'), (5, 'R'), (5, 'A')]` (7 tokens)

### Decompression Trace:
- `(0, 'B')` $\implies "" + \text{'B'} = \text{"B"}$. Dict[1] = `"B"`.
- `(0, 'A')` $\implies "" + \text{'A'} = \text{"A"}$. Dict[2] = `"A"`.
- `(1, 'A')` $\implies \text{Dict}[1] + \text{'A'} = \text{"BA"}$. Dict[3] = `"BA"`.
- `(2, 'B')` $\implies \text{Dict}[2] + \text{'B'} = \text{"AB"}$. Dict[4] = `"AB"`.
- `(0, 'R')` $\implies "" + \text{'R'} = \text{"R"}$. Dict[5] = `"R"`.
- `(5, 'R')` $\implies \text{Dict}[5] + \text{'R'} = \text{"RR"}$. Dict[6] = `"RR"`.
- `(5, 'A')` $\implies \text{Dict}[5] + \text{'A'} = \text{"RA"}$. Dict[7] = `"RA"`.
- **Reconstructed**: `"B" + "A" + "BA" + "AB" + "R" + "RR" + "RA" = \text{"BABAABRRRA"}`. Perfect match!

---

## 4. LZ77 vs. LZ78 Comparison

| Dimension | LZ77 (Sliding Window) | LZ78 (Phrase Dictionary) |
| :--- | :--- | :--- |
| **Dictionary Structure** | Implicit in sliding memory window | Explicit tree/trie of numbered phrases |
| **Token Representation** | `(offset, length, next_char)` | `(dictionary_index, next_char)` |
| **Phrase Growth Rate** | Can match any length instantly | Phrases grow by only 1 character per step |
| **Memory Bound** | Fixed strictly by window size $W$ | Grows indefinitely unless pruned / reset |

---

## 5. Algorithmic Complexity

| Operation | Time Complexity | Space Complexity |
| :--- | :--- | :--- |
| **Encoding** | $O(N)$ (using Trie for prefix matching) | $O(\text{Dictionary Size})$ |
| **Decoding** | $O(N)$ (direct array lookup) | $O(\text{Dictionary Size})$ |

---

## 6. Implementation Checklist & Edge Cases

- [ ] **Trailing Unfinished Phrase**: If the file ends with a phrase $P$ that already exists in the dictionary without a following char.
- [ ] **Trie Structure for Encoder**: Using a prefix trie to achieve $O(1)$ child branch transitions during matching.
- [ ] **Dictionary Limit Strategy**: When dictionary reaches index $2^{16}$ (65,536), either freeze additions or clear the table and restart.
- [ ] **Empty Input Handling**: Correctly output 0 tokens for an empty stream.

