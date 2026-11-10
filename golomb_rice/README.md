# Golomb and Rice Coding

Golomb coding (invented by Solomon Golomb in 1966) and its power-of-two specialization, Rice coding (Robert Rice, 1971), are optimal prefix codes for integers that follow a **geometric (exponential) distribution**.

---

## 1. Why Golomb-Rice Coding?

General entropy coders like Huffman or Arithmetic coding require building a frequency table over an alphabet. However, in many scientific and multimedia applications (audio, video, time-series telemetry, image differentials), values are **non-negative integers** where small values are extremely common, and large values are exponentially rare:
$$P(X = x) = (1 - p) p^x \quad \text{for } x \in \{0, 1, 2, \dots\}$$

Golomb-Rice coding requires **no codebook or frequency table**. Given a single tuning parameter $M$ (or $k$ where $M = 2^k$), any integer can be encoded and decoded in $O(1)$ time with bitwise shifts.

### Real-World Applications
- **FLAC & ALAC (Apple Lossless)**: Encodes prediction residuals of PCM audio waveforms.
- **JPEG-LS**: The ISO standard for lossless continuous-tone still image compression.
- **MPEG-4 ALS**: Audio Lossless Coding standard.

---

## 2. Algorithm Mechanics

Given a non-negative integer $x \ge 0$ and parameter $M$:

1. **Split into Quotient and Remainder**:
   $$q = \lfloor x / M \rfloor, \quad r = x \bmod M$$
2. **Encode Quotient $q$ in Unary**:
   Output $q$ copies of bit `1` followed by a terminating `0` (or $q$ zeros followed by a `1`).
3. **Encode Remainder $r$ in Binary**:
   - **Rice Coding ($M = 2^k$)**: Remainder $r$ is always stored in exactly $k$ binary bits.
   - **General Golomb ($M$ is not power of 2)**: Remainder uses truncated binary coding with either $\lfloor \log_2 M \rfloor$ or $\lceil \log_2 M \rfloor$ bits.

### Rice Coding Bitwise Efficiency
When $M = 2^k$:
$$q = x \gg k$$
$$r = x \ \& \ ((1 \ll k) - 1)$$
Both calculations execute in a single CPU cycle.

---

## 3. Handling Signed Values (ZigZag Encoding)

In audio and image prediction, residuals can be negative ($\Delta \in \{\dots, -2, -1, 0, 1, 2, \dots\}$). Golomb-Rice requires non-negative inputs. We map signed numbers to unsigned integers via **ZigZag encoding**:

| Signed Integer ($z$) | ZigZag Unsigned ($x$) | Formula |
| :---: | :---: | :--- |
| **0** | **0** | $x = 2z$ if $z \ge 0$ |
| **-1** | **1** | $x = 2|z| - 1$ if $z < 0$ |
| **+1** | **2** | $x = 2(1) = 2$ |
| **-2** | **3** | $x = 2(2) - 1 = 3$ |
| **+2** | **4** | $x = 2(2) = 4$ |

Bitwise formula: `zigzag(z) = (z << 1) ^ (z >> 31)`

---

## 4. Step-by-Step Worked Example (Hand Trace)

### Configuration
Rice coding with $k = 2 \implies M = 2^2 = 4$.

### Stream of Integers: `[0, 2, 4, 11]`

| Value ($x$) | Quotient ($q = \lfloor x/4 \rfloor$) | Remainder ($r = x \bmod 4$) | Unary Code ($q$) | Binary Code ($r$, 2 bits) | Emitted Bitstream | Total Bits |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **0** | 0 | 0 | `0` | `00` | `000` | 3 |
| **2** | 0 | 2 | `0` | `10` | `010` | 3 |
| **4** | 1 | 0 | `10` | `00` | `1000` | 4 |
| **11** | 2 | 3 | `110` | `11` | `11011` | 5 |
| **Total** | — | — | — | — | `000 010 1000 11011` | **15 bits** |

- **Uncompressed (32-bit integers)**: $4 \times 32 = 128\text{ bits}$
- **Rice Encoded bitstream**: 15 bits
- **Space Savings**: $88.3\%$

---

## 5. Algorithmic Complexity

| Operation | Time Complexity | Auxiliary Space Complexity |
| :--- | :--- | :--- |
| **Integer Encoding** | $O(q + k) \approx O(1)$ | $O(1)$ |
| **Integer Decoding** | $O(q + k) \approx O(1)$ | $O(1)$ |
| **Full Stream ($N$ ints)** | $O(N)$ | $O(1)$ streaming memory |

---

## 6. Implementation Checklist & Edge Cases

- [ ] **Parameter Selection**: Optimal $k \approx \max(0, \lfloor \log_2(\text{mean} \times \ln 2) \rfloor)$.
- [ ] **Outlier Protection**: An abnormally large integer (e.g. $10^6$) will generate a million-bit unary string; implementations often support an escape code for large values.
- [ ] **Streaming Bit IO**: Efficient bitwise reader and writer classes for handling non-byte-aligned bit boundaries.
- [ ] **Round-Trip Test**: Ensure negative numbers preserved cleanly after ZigZag mapping.

