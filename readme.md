# 8-Bit Binary-Weighted Current Steering DAC
### Analog Peripheral for a RISC-V SoC | 0.18 µm CMOS | 1.8 V Supply

---

## Table of Contents

1. [What is a DAC?](#1-what-is-a-dac)
2. [DAC Architectures — A Comparative Study](#2-dac-architectures--a-comparative-study)
3. [Why Current Steering?](#3-why-current-steering)
4. [Binary-Weighted vs Thermometer-Coded vs Partially Segmented](#4-binary-weighted-vs-thermometer-coded-vs-partially-segmented)
5. [Why Binary Weighted?](#5-why-binary-weighted)
6. [The Current Cell — Razavi's Cascode Switch Concept](#6-the-current-cell--razavis-cascode-switch-concept)
7. [Biasing — Current Mirror Architectures](#7-biasing--current-mirror-architectures)
8. [Why Wide-Swing Cascode Current Mirror?](#8-why-wide-swing-cascode-current-mirror)
9. [Choosing the Reference Current — Why 4 µA?](#9-choosing-the-reference-current--why-4-µa)
10. [Transistor Sizing via the gm/ID Methodology](#10-transistor-sizing-via-the-gmid-methodology)
11. [Complete Transistor Sizing Table](#11-complete-transistor-sizing-table)
12. [Bit-Weighting and Current Mapping](#12-bit-weighting-and-current-mapping)
13. [Input Switching — Digital Control](#13-input-switching--digital-control)
14. [Circuit Architecture and Operation](#14-circuit-architecture-and-operation)
15. [References](#15-references)

---

## 1. What is a DAC?

A **Digital-to-Analog Converter (DAC)** is a circuit that translates a discrete digital code into a continuous analog quantity — typically a voltage or current.

In a digital system, information is represented as binary words. In the real world, signals are analog — voltages, currents, temperatures, pressures. A DAC bridges this gap: it takes a number written by a processor and produces a proportional physical signal that can drive speakers, actuators, displays, sensors, or any analog load.

### The Transfer Function

For an N-bit DAC with reference quantity X_ref:

```
X_out = (Digital Code / 2^N) × X_ref
```

For our 8-bit design:
- 256 possible output levels (0 through 255)
- Each step corresponds to 1 LSB = X_ref / 256
- At code 0x00: output = 0
- At code 0xFF (255): output = X_ref × (255/256) ≈ X_ref

### Key Performance Metrics

| Metric | What it Measures |
|--------|-----------------|
| **Resolution (N bits)** | Number of discrete steps: 2^N levels |
| **LSB Size** | Smallest output step = Full Scale / 2^N |
| **INL (Integral Non-Linearity)** | Max deviation of actual transfer curve from ideal straight line |
| **DNL (Differential Non-Linearity)** | Max deviation of any step from ideal 1-LSB step |
| **Monotonicity** | Output always increases as code increases (DNL > -1 LSB guaranteed) |
| **SFDR (Spurious-Free Dynamic Range)** | Ratio of signal to largest spurious tone in the output spectrum |
| **Settling Time** | Time for output to settle to within ½ LSB of final value |

### Where DACs Are Used

- Audio playback (audio codec DACs)
- Display drivers (LCD/OLED bias generation)
- Motor control (PWM signal generation)
- RF signal synthesis (high-speed DACs in radios)
- Sensor calibration and trimming
- **SoC analog peripherals** ← *our application*

---

## 2. DAC Architectures — A Comparative Study

Before committing to any topology, five standard DAC architectures were evaluated. The project targets are:

- **Resolution:** 8 bits
- **Speed:** Low frequency (SoC peripheral, not RF)
- **Power:** Low (microamp-range)
- **Supply:** 1.8 V, 0.18 µm CMOS
- **Integration:** On-chip, no external components, no op-amps

---

### 2.1 Binary-Weighted Resistor DAC

The simplest conceptual DAC. Each bit drives a resistor scaled as R, 2R, 4R, 8R… The currents sum at a virtual-ground node (typically an op-amp input), producing a voltage output.

```
b7 ──[R]──┐
b6 ──[2R]─┤
b5 ──[4R]─┤
   ...     ├──── (summing node) ──[R_f]──> V_out
b1 ──[64R]─┤
b0 ──[128R]┘
```

| Pros | Cons |
|------|------|
| Conceptually simple | Requires 128:1 resistor ratio spread for 8 bits |
| Fast settling | On-chip resistor matching is extremely difficult beyond 4–6 bits |
| Linear in principle | Mandatory op-amp adds power, area, and offset error |
| | INL/DNL degrade rapidly with resistor mismatch |

**Decision: Eliminated.** The 128:1 resistor spread and precision matching requirement are impractical in standard CMOS. On-chip resistors have poor absolute accuracy (~20%) and matching is typically limited to ~0.1%, which translates to significant linearity errors at 8-bit resolution.

---

### 2.2 R-2R Ladder DAC

Uses only two resistor values (R and 2R) in a ladder network. Each rung of the ladder contributes a bit-weighted current. The clever topology means regardless of how many bits are added, the Thevenin resistance looking into any node is always R — making it fully scalable.

```
          R       R       R       R
Vref ──[R]──┬──[R]──┬──[R]──┬──[R]──> (to op-amp)
            |       |       |
           [2R]    [2R]    [2R]
            |       |       |
           GND     GND     GND
           (controlled by bit switches)
```

| Pros | Cons |
|------|------|
| Only two resistor values — much more matchable | Output impedance varies with digital code |
| No exponential resistor spread | Requires buffer op-amp at output to prevent loading |
| Scales easily to higher bits | RC ladder creates signal-dependent delay |
| | Still needs two precision resistor values |

**Decision: Eliminated.** The code-dependent output impedance means the output voltage changes with load, and the mandatory buffer op-amp conflicts with the low-power, no-external-components requirement of our SoC peripheral.

---

### 2.3 Capacitive (Charge Redistribution) DAC

Uses a binary-weighted capacitor array. A reference voltage charges/discharges capacitors proportionally to the digital code. The output voltage is sampled on a capacitor. This is the heart of SAR ADC architectures.

```
         2^(N-1)C   2^(N-2)C    C     C
           |           |        |     |
b(N-1) ──[=]───┐  b(N-2)──[=]──┤ ... b0──[=]──┤
                └──────────────────────────────> V_out
                                                 (sampled on capacitor)
```

| Pros | Cons |
|------|------|
| Zero static power (capacitors don't draw DC) | Large capacitor array for 8 bits (area-intensive) |
| Natural fit for SAR ADC architectures | Speed limited by RC redistribution time constants |
| Good linearity if caps are well-matched | Capacitor mismatch limits linearity |
| | Not suited as a standalone continuous-time output DAC |

**Decision: Eliminated.** The charge redistribution architecture is fundamentally a sampled-data system — it produces a held voltage on a capacitor, not a continuously available output suitable for driving analog circuits. Area overhead is also disproportionate.

---

### 2.4 Sigma-Delta (ΣΔ) DAC

Uses oversampling and noise shaping. A 1-bit (or few-bit) DAC operates at a frequency many times higher than the signal bandwidth. A digital modulator shapes quantization noise to out-of-band frequencies. A reconstruction filter removes this noise, leaving a high-resolution analog output.

```
Digital input ──> [ΣΔ Modulator] ──> [1-bit DAC] ──> [Reconstruction Filter] ──> V_out
                  (digital, runs    (simple, low    (analog low-pass,
                   at N×f_signal)    mismatch)        removes HF noise)
```

| Pros | Cons |
|------|------|
| Very high resolution achievable | High latency due to oversampling and filter group delay |
| Relaxed analog matching (only 1-bit DAC) | Narrow bandwidth — signal frequency limited by oversampling ratio |
| Excellent for audio applications | Complex digital modulator required |
| | Incompatible with low-latency APB register writes |

**Decision: Eliminated.** A ΣΔ DAC is fundamentally incompatible with the SoC model where the CPU writes an 8-bit value to a register and expects an immediate, settled analog output. The oversampling and filtering introduce latency that cannot be reconciled with direct register-mapped peripheral operation. Additionally, the narrow bandwidth is unsuitable if any moderate-frequency output is needed.

---

### 2.5 Current Steering DAC

Each bit controls a dedicated current source. The current is steered between two output nodes (I_out+ and I_out−) via a differential switch pair. The output currents of active branches sum at the output node, producing a current proportional to the digital code.

```
                VDD
                 |
    ┌──────┬──────┬──────┬── ... ──┐
   [I_7] [I_6] [I_5]            [I_0]    ← current sources
    128u   64u   32u              1u
    |      |     |                |
   [SW7] [SW6] [SW5]            [SW0]    ← differential switches
    |      |     |                |
    └──────┴──────┴──────┬── ... ─┘
                         |
                      I_out ──> R_load ──> V_out
```

| Pros | Cons |
|------|------|
| MOSFET is a natural current source — no resistors needed | Output voltage swing limited by current source headroom |
| No op-amp or buffer required | High output impedance must be maintained for good linearity |
| Total supply current nearly constant — low noise injection | Requires careful layout for current source matching |
| Fast switching — no large RC nodes to charge | |
| Natural differential output — good SFDR | |
| Scales cleanly with CMOS processes | |
| Low power with microamp-range currents | |

---

## 3. Why Current Steering?

Three fundamental properties made current steering the only viable candidate for our target specifications.

### 3.1 Native CMOS Implementation — No Passive Components

A MOSFET biased in saturation **is** a current source. The drain current is:

```
I_D = (1/2) × µ_n × C_ox × (W/L) × V_ov²
```

This is a current determined by transistor dimensions and bias — not by resistors or capacitors. The entire DAC is built from transistors only, making it compact and fully compatible with any standard CMOS process without special options (no poly resistors, no MIM capacitors required).

### 3.2 No Buffer Amplifier Required

The high-impedance current output directly drives the load resistor to produce a voltage. This eliminates the op-amp that all resistive DAC topologies require — removing its power consumption, area, offset, and bandwidth limitations. For a low-power SoC peripheral, eliminating the op-amp is critical.

### 3.3 Constant Total Supply Current → Minimal Digital Noise Injection

In a current steering DAC, each current source is **always active** — it never turns off. The current is steered between the positive and negative output nodes, not switched on and off. Therefore, the total current drawn from VDD is nearly constant regardless of the digital input code.

This is essential in a mixed-signal SoC: digital logic switching causes supply current transients that appear as noise on the supply rails and substrate. If the DAC itself draws a code-dependent current, every code change injects a switching glitch into the supply, corrupting the analog output. Constant total current eliminates this mechanism entirely.

---

## 4. Binary-Weighted vs Thermometer-Coded vs Partially Segmented

Within current steering, three encoding schemes can be used to map the N-bit digital input to the current source array.

---

### 4.1 Thermometer-Coded Current Steering DAC

The N-bit binary input is first decoded into a (2^N − 1)-bit thermometer code. Each bit of the thermometer code controls one identical unit current source (I_unit).

```
8-bit binary input
       ↓
[8-to-255 thermometer decoder]
       ↓
255 identical unit current sources, each = I_unit
       ↓
I_out = (code) × I_unit
```

For 8 bits: 255 unit current sources, 255 switch pairs, and an 8-to-255 decoder.

| Metric | Thermometer Coded |
|--------|------------------|
| Number of current sources | 2^N − 1 = **255 for 8 bits** |
| Decoder required? | Yes — 8-to-255 |
| Monotonicity | Guaranteed by construction |
| DNL | Excellent — each step adds exactly one unit source |
| Glitch at major carry | None — only one source changes per step |
| SFDR | Best |
| Area | Very large (255 matched cells + complex decoder) |
| Routing complexity | Extremely high |

**Key advantage:** Inherent monotonicity and zero major-carry glitch.  
**Key disadvantage:** 255 current sources, matching 255 cells, routing 255 switches, and decoding with a 255-output decoder for 8-bit resolution.

---

### 4.2 Binary-Weighted Current Steering DAC

Each of the N bits directly controls a current source scaled to 2^k × I_unit. No decoder is needed — bit k drives its corresponding switch directly.

```
b7 ──[SW]── I_7 = 128 × I_unit
b6 ──[SW]── I_6 =  64 × I_unit
b5 ──[SW]── I_5 =  32 × I_unit
  ...
b0 ──[SW]── I_0 =   1 × I_unit
                ↓
           I_out = Σ (b_k × 2^k × I_unit)
```

For 8 bits: only 8 current sources, directly driven by the 8 CPU output bits.

| Metric | Binary Weighted |
|--------|----------------|
| Number of current sources | N = **8 for 8 bits** |
| Decoder required? | **No** — bits drive switches directly |
| Monotonicity | Not guaranteed (depends on matching) |
| DNL | Can degrade at MSB transitions |
| Glitch at major carry (0111...1 → 1000...0) | Present — MSB switches, all others turn off |
| SFDR | Good at low frequencies |
| Area | Minimal |
| Routing complexity | Minimal |

**Key advantage:** 8 sources instead of 255; CPU binary output connects directly.  
**Key disadvantage:** MSB glitch at major code transition; MSB current source is 128× larger than LSB.

---

### 4.3 Partially Segmented (Hybrid) Architecture

A compromise: the upper M bits use thermometer coding (for linearity and glitch suppression at the most significant transitions), while the lower (N−M) bits use binary weighting (for area efficiency). Commonly used in high-speed, high-resolution designs.

Example for 8-bit with M=4 upper bits thermometer + 4 lower bits binary:
- Upper 4 bits: 15 unit current sources (thermometer, 4-to-15 decoder)
- Lower 4 bits: 4 binary-weighted sources

| Metric | Partially Segmented |
|--------|-------------------|
| Sources | (2^M − 1) + (N−M) = 15 + 4 = **19 for 8-bit (M=4)** |
| Decoder required? | Yes — for upper M bits only |
| Glitch | Suppressed at major transitions |
| Area | Moderate — better than full thermometer |

**Used in:** High-speed, high-SFDR DACs where glitch energy must be minimized.

---

## 5. Why Binary Weighted?

Despite the thermometer and hybrid options, binary weighting was chosen for this design. The decision rests on four interlocking reasons.

### Reason 1 — Direct SoC Compatibility

The RISC-V CPU writes a natural 8-bit binary value to the DAC register via the APB bus. In a binary-weighted DAC, each bit of that register **directly drives** its corresponding switch — no additional hardware sits between the CPU and the DAC cells.

In a thermometer DAC, an 8-to-255 decoder must sit between the CPU and the switches. This decoder adds silicon area, power dissipation, propagation delay, and timing skew across 255 outputs. For a simple analog peripheral in an SoC, this complexity is completely unjustified.

### Reason 2 — 255 Branches is Impractical at 8-Bit Resolution

An 8-bit thermometer DAC requires:
- 255 identical unit current sources, each requiring common-centroid placement for matching
- 255 differential switch pairs
- A 255-output decoder with matched propagation delays to all 255 cells
- Extensive routing of 255 control signals

The area and routing overhead is disproportionate for 8-bit resolution. Binary weighting achieves identical resolution with exactly 8 current sources.

### Reason 3 — Glitch Energy is Irrelevant at Low Frequency

The principal disadvantage of binary weighting is glitch energy at the major carry transition (code 0111 1111 → 1000 0000). At this transition, the MSB switches on while all 7 lower bits switch off simultaneously. The timing mismatch between these switching events creates a current glitch at the output.

This matters in **high-speed RF DACs** where the glitch repeats at the output sample rate and folds into the signal band, degrading SFDR. Our DAC operates at **low frequency** — the output fully settles between each code update and the glitch decays before the next transition. The glitch is irrelevant to our application.

### Reason 4 — Binary Weighting is Proven at 8-Bit Resolution

Deveugele and Steyaert (2006) demonstrated that a fully binary-weighted current-steering DAC achieves **>60 dB SFDR at 250 MS/s** — a demanding high-speed specification. If binary weighting delivers adequate SFDR at 10-bit resolution and 250 MS/s, it is more than sufficient at 8-bit resolution and low frequency.

**Summary of the decision:**

| Criterion | Thermometer | Binary Weighted | Winner |
|-----------|-------------|-----------------|--------|
| CPU interface | Needs 8→255 decoder | Direct bit drive | **Binary** |
| Number of cells | 255 | 8 | **Binary** |
| Area | Very high | Minimal | **Binary** |
| Glitch at our frequency | Irrelevant | Irrelevant | Tie |
| Proven at 8-bit | Yes | Yes | Tie |
| SoC integration | Complex | Simple | **Binary** |

---

## 6. The Current Cell — Razavi's Cascode Switch Concept

The design of the individual current cell follows Razavi's analysis (Razavi, 2018).

### 6.1 The Problem with Simple Current Switching

A naive implementation would connect a current source to a switch MOSFET and turn it on/off. When the switch opens, the current source node collapses toward ground. When the switch closes again, the parasitic capacitance at that node must charge from 0 V, pulling a large transient current from the output node — this is **glitch energy**.

```
(Naive — WRONG approach)
         VDD
          |
        [M_cs]   ← current source
          |
     node X        ← collapses to 0V when switch opens!
          |
        [SW]     ← switch
          |
       I_out
```

The voltage excursion at node X creates large glitch transients at every switching event.

### 6.2 Razavi's Solution — Differential Pair Steering

Instead of switching the current source off, the current is **always flowing** and is **steered** between two complementary output nodes via a differential switch pair:

```
              I_out+         I_out−
                 |               |
           [  M1  ]         [  M2  ]    ← differential switch pair
           (bit=1:ON)       (bit=1:OFF)
                 └─────┬─────┘
                       │
                  node X  ← stays near constant voltage!
                     [M_cas]            ← cascode transistor
                       │
                     [M_cs]             ← current source (ALWAYS ON)
                       │
                      GND / Vbias
```

- `bit = 1, bit_bar = 0`: M1 ON, M2 OFF → current flows to I_out+
- `bit = 0, bit_bar = 1`: M1 OFF, M2 ON → current flows to I_out−

**The key insight (Razavi):** The current source never turns off. Node X never collapses. The voltage at X changes by only a small amount during switching, so parasitic capacitance at X causes negligible output disturbance. This is why current **steering** fundamentally outperforms current **switching** for dynamic accuracy.

### 6.3 The Cascode Above the Current Source

A cascode transistor (M_cas) is inserted between the current source and the switch pair:

**Function 1 — Increase output impedance:**  
Without a cascode, the output impedance of a single MOSFET current source is r_ds. With a cascode, the output impedance becomes approximately g_m × r_ds², orders of magnitude higher. Higher output impedance means the current varies less as the output voltage swings — lower INL from finite impedance effects.

**Function 2 — Shield node X from output voltage swings:**  
During switching, the output voltage changes rapidly. Without the cascode, this swing couples directly back to node X, disturbing the current source. The cascode absorbs most of the output swing, keeping node X nearly constant.

### 6.4 Complete Unit Cell Structure

```
      VDD
       │
    [M_cs]    ← PMOS current source: wide/long for matching and high r_ds
       │
    [M_cas]   ← PMOS cascode: boosts output impedance, shields node X
       │
  [M1]   [M2] ← NMOS differential switch pair: min-W for low capacitance
   │         │
I_out+     I_out−

Control: bit → M1 gate, bit_bar → M2 gate
```

This exact structure — **current source + cascode + differential switch pair** — is replicated for every bit cell in the 8-bit DAC.

---

## 7. Biasing — Current Mirror Architectures

The current source transistors in all 8 bit cells must produce accurate, ratioed currents. This is achieved using a **current mirror**: one reference current (set by a precision resistor or bias circuit) is mirrored to all output branches with precise ratios.

Four standard current mirror architectures were considered.

---

### 7.1 Simple (Basic) Current Mirror

The most basic form. A reference transistor (diode-connected) forces V_GS_ref. All output transistors share the same V_GS and ideally carry the same current.

```
         VDD
          │
    [M_ref (diode)]──[M_out1]──[M_out2]── ...
          │              │          │
         I_ref        I_out1    I_out2
          │              │          │
         GND            GND        GND
```

**Problem: Channel-length modulation (λ effect)**  
In the simple mirror, V_DS_ref = V_GS (fixed by the diode connection), but V_DS_out varies with the output voltage. Since I_D depends on V_DS through channel-length modulation:

```
I_D = I_D0 × (1 + λ × V_DS)
```

If the output voltage changes by 1 V and λ = 0.1 V⁻¹, the output current changes by 10%. This is unacceptable for a DAC current source where accuracy is paramount.

**Output impedance:** r_ds only (~10–100 kΩ) — insufficient.

---

### 7.2 Cascode Current Mirror

Adds a cascode transistor above each mirror output transistor. The cascode fixes V_DS of the current source transistor, suppressing the λ effect.

```
         VDD
          │
    [M_ref1]──────────────────────────
    [M_ref2 (cascode, diode)]──[M_cas_out]── ...
          │              [M_cs_out]
         I_ref                │
                           I_out
```

**Output impedance:** g_m × r_ds² (~1–10 MΩ) — much better.  
**Problem: Voltage headroom.**  
The simple cascode requires the output node to be above `V_DS_sat (M_cs) + V_DS_sat (M_cas)`. For a 1.8 V supply with µA-range currents, this typically needs ~0.8–1.0 V headroom just for the current source stack. With output swings and switch overhead on top, headroom becomes critically tight.

---

### 7.3 Wide-Swing Cascode Current Mirror

The wide-swing cascode (also called the regulated cascode or improved cascode) sets the bias voltage of the cascode gate precisely such that the bottom current source transistor operates just at the edge of saturation — using the minimum possible V_DS. This recovers the headroom problem of the simple cascode while retaining its high output impedance.

```
         VDD
          │
      [M_p1]────[M_p3]   ← PMOS: reference + bias generation
      [M_p2]────[M_p4]   ← PMOS cascode pair (diode-connected for bias)
          │         │
         I_ref    I_out (mirrored)
          │         │
         GND       GND
```

The bias voltage for the cascode gate is generated from within the mirror itself, not from an external bias. The key property: V_DS of M_cs is set to exactly V_DS_sat = V_GS − V_th = V_ov. This is the **minimum** V_DS for saturation — maximizing output voltage swing.

**Output impedance:** g_m × r_ds² — same as simple cascode.  
**Minimum output voltage:** V_ov + V_DS_sat ≈ 2 × V_ov (much lower than simple cascode).

---

## 8. Why Wide-Swing Cascode Current Mirror?

The wide-swing cascode was selected over all alternatives for three reasons that align precisely with our design constraints.

### Reason 1 — Maximum Output Impedance is Required for DAC Linearity

INL in a current steering DAC is directly related to the output impedance of the current sources. If the output impedance is finite, the current from each source changes slightly as the output voltage varies across the DAC range. This variation is code-dependent and appears as INL.

The output impedance of a wide-swing cascode is g_m × r_ds² — orders of magnitude higher than a simple mirror's r_ds. This suppresses the V_DS sensitivity of each current source, keeping all currents accurate across the full output voltage range.

### Reason 2 — 1.8 V Supply Demands Minimum Headroom Usage

With a 1.8 V supply, every millivolt counts. The simple cascode current mirror wastes significant headroom in the bias network — roughly V_GS + V_DS_sat ≈ 1.0–1.2 V just for the bias voltages. The wide-swing cascode uses self-generated bias voltages to reduce this to ≈ 2 × V_ov ≈ 0.5 V, leaving more headroom for the output swing and switch stack.

### Reason 3 — Self-Contained Biasing — No External Bias Generator Needed

The wide-swing cascode generates its own cascode gate bias internally from the reference branch. No external bias voltage generator (no additional op-amp bias loop, no separate PTAT circuit) is required. This is critical for a clean SoC peripheral — the entire biasing is self-contained within the current mirror.

**Summary:**

| Mirror Type | Output Impedance | Min V_out | Bias Needed | Verdict |
|-------------|-----------------|-----------|-------------|---------|
| Simple | r_ds (~50 kΩ) | V_DS_sat | None | Insufficient Z_out |
| Cascode | g_m r_ds² (~5 MΩ) | 2V_GS − V_th (~1.0V) | External V_b | Headroom too high |
| Wide-Swing Cascode | g_m r_ds² (~5 MΩ) | **2V_ov (~0.5V)** | **Self-generated** | ✅ Selected |

---

## 9. Choosing the Reference Current — Why 4 µA?

The reference current of **4 µA per unit cell** was not an arbitrary choice. It emerged from three converging constraints that simultaneously satisfied all design requirements.

### Constraint 1 — Low Power Target

The DAC must consume minimal power. With I_LSB = 4 µA, the maximum total output current (all 8 bits ON, code = 0xFF) is:

```
I_total_max = 4 µA × (1 + 2 + 4 + 8 + 16 + 32 + 64 + 128)
            = 4 µA × 255
            = 1.02 mA
```

This is a sensible full-scale current for a low-power peripheral — usable output voltage across a 10 kΩ load, with total dissipation well under 2 mW.

### Constraint 2 — The gm/ID Methodology Naturally Yields ~4 µA

The transistor sizing methodology (Section 10) uses gm/ID = 8 and selects layout-friendly dimensions. With W = 3 µm and L = 2.2 µm:

```
I_D = (1/2) × µ_n × C_ox × (W/L) × V_ov²
    = (1/2) × 180 µA/V² × (3/2.2) × (0.25)²
    ≈ 3.78 µA ≈ 4 µA
```

The sizing methodology and chosen transistor dimensions **naturally arrive at 4 µA**. Rather than distorting the transistor sizes to hit an arbitrary current target, we accepted what the methodology gave — a clean, process-justified result.

### Constraint 3 — The Output Voltage Swing is Well-Suited to a 1.8 V Supply

With I_LSB = 4 µA and a 10 kΩ load resistor:

```
V_out_max = 255 × 4 µA × 10 kΩ = 1.02 V
```

This is well within the 1.8 V supply with comfortable headroom remaining for the current source stack and output swing.

### Why Not Other Values?

| I_LSB | W/L Required | Problem |
|-------|-------------|---------|
| 1 µA | ~0.18 | Extremely narrow — dominated by edge effects, very poor matching |
| 2 µA | ~0.35 | Small — matching concerns, sensitive to process variation |
| **4 µA** | **~0.71 → W=3 µm, L=2.2 µm** | **Layout-friendly, excellent matching, process-justified** |
| 10 µA | ~1.78 | Manageable but total I_max ≈ 2.5 mA — unnecessarily high power |

4 µA simultaneously satisfies: the gm/ID methodology result, the low-power requirement, and layout-friendly transistor dimensions. This convergence is the hallmark of a well-constrained design.

### Why Is 4 µA Given to the Current Mirror Reference?

The current mirror reference is set to **4 µA** because this is the unit cell current from the gm/ID sizing. The mirror then generates integer multiples of this reference for each bit:

- Bit 0 (LSB): mirror ratio 1× → 1 µA *(Note: actual LSB current from one unit cell)*
- Bit 1: mirror ratio 2× → 2 µA
- Bit 2: mirror ratio 4× → 4 µA (= 1 reference unit)
- ...
- Bit 7 (MSB): mirror ratio 128× → 128 µA

All ratios are exact multiples of the same reference. A single reference current defines all 8 output currents — providing excellent ratiometric accuracy. If the reference drifts (e.g. with temperature), all outputs drift proportionally, preserving relative accuracy and linearity.

The **4 µA reference sits at Bit 2** in the weight table, which is a natural midpoint. It was chosen because the gm/ID sizing gives this current for the smallest physically reasonable transistor — and the LSB current (1 µA for bit 0) is derived by using a ¼-scaled mirror ratio from this reference.

---

## 10. Transistor Sizing via the gm/ID Methodology

### 10.1 What is the gm/ID Methodology?

The gm/ID methodology (Silveira, Flandre, Jespers, 1996; Murmann & Jespers, 2017) provides a systematic, region-independent approach to transistor sizing. Instead of using square-law equations that break down in weak/moderate inversion, it uses the transconductance efficiency gm/ID as a design variable that continuously spans all operating regions:

```
gm/ID (V⁻¹)   Operating Region         Typical Use
  25–35        Weak inversion           Ultra-low-power, log-domain
   8–20        Moderate inversion       Speed-power trade-off
   2–8         Strong inversion         High speed, best matching
```

The core equation relating gm/ID to gate overdrive (in strong inversion):

```
gm/ID = 2 / V_ov    where V_ov = V_GS − V_th
```

### 10.2 Why gm/ID = 8 for DAC Current Sources?

DAC current source transistors must be **well-matched** — their drain currents must be accurate relative to each other across all process corners and temperatures. The dominant matching error is threshold voltage mismatch, described by Pelgrom's rule:

```
σ(V_th) = A_VTH / √(W × L)
```

The fractional current error from V_th mismatch is:

```
σ(I_D) / I_D = (gm / I_D) × σ(V_th) = (gm/ID) × (A_VTH / √(WL))
```

**Lower gm/ID = lower sensitivity to V_th mismatch = better current matching.**

Operating at gm/ID = 8 (strong inversion boundary) gives:
- Reduced sensitivity to V_th mismatch vs. moderate/weak inversion
- Gate overdrive V_ov = 0.25 V — robustly in saturation, insensitive to threshold variation
- Good balance of power and matching performance

### 10.3 Step-by-Step Sizing Calculation

**Given process parameters (0.18 µm CMOS):**
- µ_n × C_ox = 180 µA/V²
- V_th = 0.51 V (NMOS)
- Target I_D = 4 µA (reference current)

```
Step 1 — Gate overdrive from gm/ID:
  V_ov = 2 / (gm/ID) = 2 / 8 = 0.25 V

Step 2 — Gate-source voltage:
  V_GS = V_th + V_ov = 0.51 + 0.25 = 0.76 V

Step 3 — Required W/L from drain current equation:
  I_D = (1/2) × µ_n × C_ox × (W/L) × V_ov²
  W/L = (2 × I_D) / (µ_n × C_ox × V_ov²)
      = (2 × 4×10⁻⁶) / (180×10⁻⁶ × 0.25²)
      = 8×10⁻⁶ / (180×10⁻⁶ × 0.0625)
      = 8×10⁻⁶ / 11.25×10⁻⁶
      = 0.711

Step 4 — Physical dimensions:
  L = 2.2 µm  (long channel — see reasons below)
  W = 0.711 × 2.2 = 1.56 µm → rounded to W = 3 µm (layout-friendly)

Step 5 — Verify current with rounded W:
  I_D = (1/2) × 180×10⁻⁶ × (3/2.2) × (0.25)²
      = (1/2) × 180×10⁻⁶ × 1.364 × 0.0625
      ≈ 3.78 µA ≈ 4 µA  ✓ CONFIRMED
```

### 10.4 Why L = 2.2 µm and Not Minimum Length (0.18 µm)?

Choosing minimum channel length would maximize speed but is entirely wrong for a current source. Three reasons dictate long channel:

**1. Higher Early Voltage:**  
Early voltage V_A ∝ L. A longer channel gives higher V_A, which gives higher r_ds:

```
r_ds = V_A / I_D
```

Higher r_ds = higher output impedance of each current source = lower INL from finite impedance effects. With L = 2.2 µm vs 0.18 µm, r_ds improves by approximately 12×.

**2. Better Matching (Pelgrom's Rule):**  
```
σ(V_th) = A_VTH / √(W × L)
```
With L = 2.2 µm and W = 3 µm: WL = 6.6 µm².  
With L = 0.18 µm and W = 3 µm: WL = 0.54 µm².  
The long-channel transistor has √(6.6/0.54) ≈ 3.5× lower V_th mismatch → 3.5× better current matching between bit cells.

**3. Reduced Channel-Length Modulation:**  
The λ coefficient is inversely proportional to L. Longer transistors are less sensitive to V_DS variation — current stays more constant as the output voltage swings across the DAC range.

---

## 11. Complete Transistor Sizing Table

The base unit cell is **W = 3 µm, L = 2.2 µm**. All current source transistors use this same unit cell dimension. Binary-weighted currents are achieved by connecting M unit cells in parallel — keeping all physical transistors identical for best matching.

### Current Mirror and Current Source Transistors

| Transistor | Role | L (µm) | W (µm) | Multiplier M |
|------------|------|---------|---------|-------------|
| pcm1 | Wide-swing cascode mirror — reference leg 1 | 2.2 | 3 | 3 |
| pcm2 | Wide-swing cascode mirror — cascode leg 2 | 2.2 | 3 | 3 |
| pcm3 | Wide-swing cascode mirror — output leg 3 | 2.2 | 3 | 3 |
| n1 | Bit 0 (LSB) current source | 2.2 | 3 | 1 |
| n2 | Bit 1 current source | 2.2 | 3 | 2 |
| n3 | Bit 2 current source | 2.2 | 3 | 4 |
| n4 | Bit 3 current source | 2.2 | 3 | 8 |
| n5 | Bit 4 current source | 2.2 | 3 | 16 |
| n6 | Bit 5 current source | 2.2 | 3 | 32 |
| n7 | Bit 6 current source | 2.2 | 3 | 64 |
| n8 | Bit 7 (MSB) current source | 2.2 | 3 | 128 |
| n9, n10, n11 | Mirror bias cells | 2.2 | 3 | 32 |
| n12, n13, n14 | Mirror bias cells | 2.2 | 3 | 16 |

### Switch Transistors

| Transistor | Role | L (µm) | W (µm) |
|------------|------|---------|---------|
| S1 (per bit) | Switch — bit side (M1) | 0.18 | 0.2 |
| S2 (per bit) | Switch — bit_bar side (M2) | 0.18 | 1.2 |

Switch transistors use **minimum or near-minimum length** to minimize parasitic capacitance at node X. Small switches mean fast switching and low glitch injection. The slight size asymmetry between S1 and S2 is used to equalize the on-resistance for both switching paths, balancing rise and fall times.

---

## 12. Bit-Weighting and Current Mapping

Each bit controls a current source that is a binary-weighted multiple of the LSB current:

| Bit | Weight | Ideal Current Contribution |
|-----|--------|---------------------------|
| b0 (LSB) | 2⁰ = 1× | ~1 µA |
| b1 | 2¹ = 2× | ~2 µA |
| b2 | 2² = 4× | ~4 µA |
| b3 | 2³ = 8× | ~8 µA |
| b4 | 2⁴ = 16× | ~16 µA |
| b5 | 2⁵ = 32× | ~32 µA |
| b6 | 2⁶ = 64× | ~64 µA |
| b7 (MSB) | 2⁷ = 128× | ~128 µA |
| All ON (code 0xFF) | 255× | ~255 µA |

The output current for any 8-bit code is:

```
I_out = Σ(k=0 to 7) b_k × 2^k × I_LSB
      = I_LSB × (digital code)
```

With a 10 kΩ output load resistor:
- Full-scale output voltage: 255 × 1 µA × 10 kΩ = **2.55 V** *(or with 4 µA LSB: 255 × 4 µA × 10 kΩ = 1.02 V)*
- 1 LSB voltage: **~10 mV** per step

---

## 13. Input Switching — Digital Control

### How the Digital Inputs Are Applied

Each bit drives a differential switch pair. The bit signal and its complement (bit_bar) are both required:

```
bit = 1, bit_bar = 0  →  M1 ON, M2 OFF  →  I steers to I_out+
bit = 0, bit_bar = 1  →  M1 OFF, M2 ON  →  I steers to I_out−
```

In the final SoC, the RISC-V CPU writes to the DAC register via APB. Each clock cycle can update all 8 bits simultaneously.

### Example — Input Word 1011 0011

```
Bit:      b7  b6  b5  b4  b3  b2  b1  b0
Value:     1   0   1   1   0   0   1   1
             ↑               ↑       ↑  ↑  ↑
          Active           Active       Active

Active bits: b7 (128µA) + b5 (32µA) + b4 (16µA) + b1 (2µA) + b0 (1µA)
Total I_out = 128 + 32 + 16 + 2 + 1 = 179 × I_LSB = ~179 µA
```

### Simulation — Pulse Inputs

In simulation, the CPU register writes are mimicked with rectangular pulse waveforms applied to each bit input. The pulses step through different input combinations at defined time intervals. This allows:

- Observation of output current/voltage steps for each code transition
- Verification of settling time (time for output to reach within ½ LSB of final value)
- Linearity checking across the full code range (INL/DNL extraction)
- Transient glitch measurement at the major carry transition (0x7F → 0x80)

---

## 14. Circuit Architecture and Operation

### Overall Block Diagram

```
RISC-V CPU  ──APB Bus──>  DAC Register  ──>  8-Bit Current Steering DAC  ──>  Analog Output
             (digital)                         (this design)                    I_out / V_out
```

### Internal DAC Structure

```
              VDD
               │
  ┌────────────┴──────────────────────────────────────────┐
  │         Wide-Swing Cascode Current Mirror              │
  │         (pcm1, pcm2, pcm3 — sets 4µA reference)       │
  └──┬────┬────┬────┬────┬────┬────┬────┬────────────────┘
     │    │    │    │    │    │    │    │
    n8   n7   n6   n5   n4   n3   n2   n1      ← current sources
   128µ  64µ  32µ  16µ   8µ   4µ   2µ   1µ     (each = M unit cells)
     │    │    │    │    │    │    │    │
   [cascode transistors per bit cell]
     │    │    │    │    │    │    │    │
  [S1,S2 diff pair] ×8                          ← differential switch pairs
     │         │         │         │
    b7  b6    b5  b4    b3  b2    b1  b0        ← digital inputs
     │
 I_out+ / I_out−  ──>  R_load (10kΩ)  ──>  V_out
```

### Validation: 2-Bit Sub-DAC Testbench

Before building the full 8-bit DAC, a **2-bit sub-DAC using bits b6 and b7** was implemented and simulated first. This approach:

- Validates the wide-swing cascode mirror biasing at the highest current levels (64 µA and 128 µA)
- Confirms differential switch pair operation (4 output codes: 0, 64µA, 128µA, 192µA)
- Verifies settling time and glitch behavior before scaling to all 8 branches

The 8-bit DAC is structurally identical — 6 additional bit branches (b0 through b5) added in parallel with identical unit cell structure.

### Output Code Table

| Input Code | Active Bits | I_out | V_out (10 kΩ load) |
|------------|------------|-------|-------------------|
| 0x00 | None | 0 µA | 0 V |
| 0x01 | b0 | ~1 µA | ~10 mV |
| 0x80 | b7 only | ~128 µA | ~1.28 V |
| 0xB3 (1011 0011) | b7+b5+b4+b1+b0 | ~179 µA | ~1.79 V |
| 0xFF | All bits | ~255 µA | ~2.55 V |

*(Exact values depend on LSB = 1 µA; if mirror is set to 4 µA reference with ¼× for LSB, scale accordingly.)*

---

## 15. References

| # | Reference | Used For |
|---|-----------|----------|
| [1] | F. Silveira, D. Flandre, P.G.A. Jespers, *"A gm/ID Based Methodology for the Design of CMOS Analog Circuits,"* IEEE JSSC, vol. 31, no. 9, Sep. 1996 | gm/ID theory; strong inversion requirement for current mirrors; matching vs. region |
| [2] | B. Razavi, *"The Current-Steering DAC,"* IEEE Solid-State Circuits Magazine, Winter 2018 | Differential pair switch cell; cascode switch concept; glitch energy; output impedance analysis |
| [3] | A. Narayanan et al., *"A 0.35 µm CMOS 6-bit Current Steering DAC,"* IEEE, 2012 | Hybrid thermometer/binary architecture reference; segmentation boundary selection |
| [4] | J. Deveugele, M.S.J. Steyaert, *"A 10-bit 250-MS/s Binary-Weighted Current-Steering DAC,"* IEEE JSSC, vol. 41, no. 2, Feb. 2006 | Proof that binary weighting achieves >60 dB SFDR; glitch analysis; pseudo-segmentation performance |
| [5] | D.A. Mercer, *"Low-Power Approaches to High-Speed Current-Steering DACs in 0.18-µm CMOS,"* IEEE JSSC, vol. 42, no. 8, Aug. 2007 | Low-power biasing techniques; cascode output impedance vs. power; matching vs. V_ov trade-off |
| [6] | B. Murmann, P.G.A. Jespers, *Systematic Design of Analog CMOS Circuits,* Cambridge University Press, 2017 | gm/ID design methodology; sizing equations; design flow |
| [7] | B. Razavi, *Design of Analog CMOS Integrated Circuits,* 2nd ed., McGraw-Hill, 2017 | DAC architecture fundamentals; current mirror topologies; cascode analysis |

---

*8-Bit Binary-Weighted Current Steering DAC — Analog Peripheral of RISC-V SoC*  
*Technology: 0.18 µm CMOS | Supply: 1.8 V | Reference Current: 4 µA | Resolution: 8 bits / 256 levels*  
*Simulation results and layout details to be added upon completion.*

