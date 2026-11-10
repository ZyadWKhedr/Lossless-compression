# Burrows-Wheeler Transform (BWT) & Move-to-Front (MTF)

Invented by Michael Burrows and David Wheeler in 1994, the Burrows-Wheeler Transform (BWT) is one of the most remarkable algorithmic discoveries in computer science. Paired with Move-to-Front (MTF) coding, it forms the high-compression engine of **bzip2**.

---

## 1. Core Intuition & The "bzip2" Pipeline

The BWT is **not a compression algorithm by itself**; it is a **reversible permutation (sorting transform)**:
- It rearranges characters in a block such that characters preceding similar contexts are clustered together.
- In English text, the letter `'h'` is almost always preceded by `'t'` (in words like *the, that, this, there*). BWT groups all those `'t'`s together into contiguous runs.
- **Move-to-Front (MTF)** converts these clustered runs into sequences dominated by the integer `0`.
- Finally, standard RLE followed by Huffman or Arithmetic coding achieves compression ratios that rival or beat LZ77.

```
Raw Text  ==>  [ BWT ]  ==>  [ MTF ]  ==>  [ RLE ]  ==>  [ Huffman / Arithmetic ]  ==>  Compressed
```

---

## 2. Forward BWT: The Cyclic Rotations Matrix

Given input string $S$ of length $N$:
1. Form all $N$ cyclic rotations of $S$.
2. Sort the $N$ rotations lexicographically into an $N \times N$ matrix.
3. The **BWT output** is:
   - The **Last Column ($L$)** of the sorted matrix.
   - The **Primary Index ($I$)**: The 0-based row index containing the original string $S$.

---

## 3. Step-by-Step Worked Example (Hand Trace)

### Input String: `"banana$"` ($N = 7$, `$` is EOF sentinel)

#### 1. All 7 Cyclic Rotations
```
0: banana$
1: anana$b
2: nana$ba
3: ana$ban
4: na$bana
5: a$banan
6: $banana
```

#### 2. Lexicographically Sorted Matrix

| Row | Sorted Rotations | First Column ($F$) | Last Column ($L$) |
| :---: | :--- | :---: | :---: |
| **0** | `$banana` | `$` | **`a`** |
| **1** | `a$banan` | `a` | **`n`** |
| **2** | `ana$ban` | `a` | **`n`** |
| **3** | `anana$b` | `a` | **`b`** |
| **4** | `banana$` | `b` | **`$`**  $\leftarrow$ **Original string (Index = 4)** |
| **5** | `na$bana` | `n` | **`a`** |
| **6** | `nana$ba` | `n` | **`a`** |

- **BWT Output String ($L$)**: `"annb$aa"`
- **Original Row Index ($I$)**: 4
- *Notice*: The letters `'n'` and `'a'` are grouped into adjacent runs!

---

## 4. The Inverse BWT: Linear Time via LF-Mapping

How can the decoder reconstruct the entire original string from only the last column `"annb$aa"` and index 4, without storing the $N \times N$ matrix?

### The LF-Mapping Property
The $k$-th occurrence of character $C$ in the **Last column ($L$)** is the exact same character in the input string as the $k$-th occurrence of $C$ in the **First column ($F$)**.

1. The first column $F$ is simply column $L$ sorted alphabetically:
   $$F = \text{sorted}(L) = \text{['\$', 'a', 'a', 'a', 'b', 'n', 'n']}$$
2. Pair occurrences:
   - $L = [a_0, n_0, n_1, b_0, \$_0, a_1, a_2]$
   - $F = [\$_0, a_0, a_1, a_2, b_0, n_0, n_1]$
3. Build LF mapping table $LF[i]$ which gives the row in $F$ corresponding to $L[i]$:
   - $L[0] = a_0 \implies$ in $F$, $a_0$ is at row 1.
   - $L[1] = n_0 \implies$ in $F$, $n_0$ is at row 5.
   - $L[2] = n_1 \implies$ in $F$, $n_1$ is at row 6.
   - $L[3] = b_0 \implies$ in $F$, $b_0$ is at row 4.
   - $L[4] = \$_0 \implies$ in $F$, $\$_0$ is at row 0.
   - $L[5] = a_1 \implies$ in $F$, $a_1$ is at row 2.
   - $L[6] = a_2 \implies$ in $F$, $a_2$ is at row 3.

4. Traverse backwards starting at row $I = 4$:
   - Row 4: $L[4] = \text{'\$'}$. Follow $LF$: previous character is at row 0.
   - Row 0: $L[0] = \text{'a'}$. Follow $LF \implies$ row 1.
   - Row 1: $L[1] = \text{'n'}$. Follow $LF \implies$ row 5.
   - Row 5: $L[5] = \text{'a'}$. Follow $LF \implies$ row 2.
   - Row 2: $L[2] = \text{'n'}$. Follow $LF \implies$ row 6.
   - Row 6: $L[6] = \text{'a'}$. Follow $LF \implies$ row 3.
   - Row 3: $L[3] = \text{'b'}$. Follow $LF \implies$ row 4 (reached sentinel).
- Reading backwards yields: `b` $\to$ `a` $\to$ `n` $\to$ `a` $\to$ `n` $\to$ `a` $\to$ `$` $\implies$ **`"banana$"`**!

---

## 5. Move-to-Front (MTF) Transform

MTF maintains a dynamic list of alphabet symbols (initially $0\dots255$):
1. For each character $C$ in the BWT output:
   - Emit the integer position of $C$ in the list.
   - Move $C$ to position 0 (the front of the list).
2. Clustered runs of identical characters in BWT output are converted to **long runs of zeros**!

---

## 6. Algorithmic Complexity

| Phase | Naive Complexity | Optimized Complexity |
| :--- | :--- | :--- |
| **Forward BWT** | $O(N^2 \log N)$ (string sorting) | $O(N \log N)$ or $O(N)$ (via Suffix Array / SA-IS) |
| **Inverse BWT** | $O(N \log N)$ | $O(N)$ linear time via LF-Mapping |
| **MTF Transform** | $O(N \cdot K)$ | $O(N \log K)$ (Fenwick tree or balanced BST) |

---

## 7. Implementation Checklist & Edge Cases

- [ ] **Block-Based Processing**: BWT is a block transform; real systems chunk data into 100KB–900KB blocks.
- [ ] **Sentinel vs. Index**: In pure byte streams, avoid using special sentinel characters that conflict with byte values `0x00..0xFF`. Instead, transmit the 32-bit primary row index $I$ in the header.
- [ ] **Strings with Uniform Characters**: Handling `"AAAAAAA"` without sorting ambiguity.
- [ ] **Round-Trip Test**: `inverse_bwt(bwt(data)) == data`.

