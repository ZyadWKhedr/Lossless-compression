# Delta & Predictive Coding

Delta coding and predictive modeling are **domain transformation techniques** rather than standalone entropy coders. By replacing absolute values with differences (residuals) between predicted and actual values, predictive coding collapses high-entropy, high-variance signals into tightly clustered, near-zero distributions ideal for downstream entropy coding.

---

## 1. Core Intuition & Motivation

Consider an uncompressed 16-bit audio waveform or a temperature sensor sampling at 10 Hz:
```
Raw Signal:  [ 1020, 1022, 1023, 1021, 1022, 1025, 1025, 1024 ]
```
Every number requires 16 bits to store because values hover around ~1,000. Storing raw values yields virtually zero compression.

However, consecutive samples change very little. If we compute the difference $\Delta_i = x_i - x_{i-1}$:
```
Residuals:   [ 1020,   +2,   +1,   -2,   +1,   +3,    0,   -1 ]
```
Now, all subsequent values are small integers clustered tightly around $0$. Passing these residuals through **Golomb-Rice** or **Huffman** coding shrinks data size dramatically.

### Real-World Applications
- **FLAC & ALAC Lossless Audio**: Linear Predictive Coding (LPC) models sound waves; residuals are stored via Rice coding.
- **PNG Images**: Every row (scanline) is filtered with Delta (Sub, Up, Average, or Paeth) prior to DEFLATE compression.
- **Git Packfiles**: Commits and file revisions are stored as deltas against previous tree objects.
- **Time-Series Databases (InfluxDB, Prometheus)**: Gorilla / double-delta compression for timestamps and metrics.

---

## 2. Predictive Schemes

### 1. First-Order Difference (Simple Delta)
Assumes the current value equals the previous value ($\hat{x}_i = x_{i-1}$):
$$\Delta_i = x_i - x_{i-1}$$
$$\text{Reconstruction: } x_i = x_{i-1} + \Delta_i$$

### 2. Second-Order Difference (Acceleration / Slope)
Assumes a constant linear velocity:
$$\hat{x}_i = 2x_{i-1} - x_{i-2}$$
$$\Delta_i = x_i - (2x_{i-1} - x_{i-2})$$

### 3. Modular Byte-Level Delta (Byte Streams)
For raw byte sequences ($0\dots255$), arithmetic is computed modulo 256:
$$\Delta_i = (x_i - x_{i-1}) \bmod 256$$
$$x_i = (x_{i-1} + \Delta_i) \bmod 256$$
This guarantees every delta fits in an unsigned 8-bit byte without sign bits.

---

## 3. Step-by-Step Worked Example (Hand Trace)

### Input Telemetry Signal (Byte Values)
`[ 140, 142, 143, 141, 142, 144, 144, 143 ]` (8 samples)

#### Step 1: Forward Delta Transformation (Modulo 256)
- $x_0 = 140$ (stored as raw anchor byte)
- $\Delta_1 = (142 - 140) \bmod 256 = 2$
- $\Delta_2 = (143 - 142) \bmod 256 = 1$
- $\Delta_3 = (141 - 143) \bmod 256 = -2 \equiv 254$ (or signed $-2$)
- $\Delta_4 = (142 - 141) \bmod 256 = 1$
- $\Delta_5 = (144 - 142) \bmod 256 = 2$
- $\Delta_6 = (144 - 144) \bmod 256 = 0$
- $\Delta_7 = (143 - 144) \bmod 256 = -1 \equiv 255$ (or signed $-1$)

**Signed Residual Array**: `[ 140 (anchor), +2, +1, -2, +1, +2, 0, -1 ]`

#### Step 2: Information Entropy Comparison
- **Original Array**: 8 distinct symbols spread across the alphabet. Shannon entropy $H(X) \approx 2.5\text{ bits/symbol}$.
- **Residual Array**: 6 out of 7 residuals fall in the set $\{-2, -1, 0, +1, +2\}$.
- Mapping to ZigZag unsigned values: `[2, 1, 3, 1, 2, 0, 1]`.
- Feeding into Rice coding ($k = 1$): Each residual is encoded in only $2\text{ to }3\text{ bits}$!

#### Step 3: Exact Reconstruction Trace
- $x_0 = 140$
- $x_1 = 140 + 2 = 142$
- $x_2 = 142 + 1 = 143$
- $x_3 = 143 + (-2) = 141$
- $x_4 = 141 + 1 = 142$
- $x_5 = 142 + 2 = 144$
- $x_6 = 144 + 0 = 144$
- $x_7 = 144 + (-1) = 143$
Reconstructed sequence is identical to input: **$100\%$ lossless fidelity**.

---

## 4. Algorithmic Complexity

| Operation | Time Complexity | Auxiliary Space Complexity |
| :--- | :--- | :--- |
| **Delta Transform (Encode)** | $O(N)$ single pass | $O(1)$ auxiliary |
| **Inverse Delta (Decode)** | $O(N)$ single pass | $O(1)$ auxiliary |

---

## 5. Implementation Roadmap & Edge Cases

- [ ] **First Sample Preservation**: The initial anchor sample $x_0$ must be stored verbatim to seed reconstruction.
- [ ] **Modular Wrapping**: In byte mode, ensure proper unsigned 8-bit wrap-around `(a - b) & 0xFF`.
- [ ] **Multi-Channel Streams**: Interleaved stereo audio (Left, Right) or RGB image pixels must compute deltas per-channel, not across adjacent channel boundaries.
- [ ] **Pipeline Chaining**: Delta encoding should integrate cleanly as a pre-filter before feeding into RLE, Huffman, or Rice coding.
- [ ] **Round-Trip Test**: `inverse_delta(delta(data)) == data`.

