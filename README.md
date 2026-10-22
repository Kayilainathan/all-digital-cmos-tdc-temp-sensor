# 45nm CMOS All-Digital Temperature Transducer & Time-to-Digital Converter (TDC)

[![EDA Tool](https://img.shields.io/badge/Simulator-LTspice%2026.1-blue.svg)](https://www.analog.com/en/design-center/design-tools-and-calculators/ltspice-simulator.html)
[![Technology Node](https://img.shields.io/badge/Node-45nm%20PTM%20BSIM4-orange.svg)](https://ptm.asu.edu/)
[![Supply Voltage](https://img.shields.io/badge/V_DD-1.0V-brightgreen.svg)](#)
[![Architecture](https://img.shields.io/badge/Architecture-Tapped%20Delay%20Line%20TDC-purple.svg)](#)
[![Affiliation](https://img.shields.io/badge/Anna%20University-%20MIT-red.svg)](https://www.annauniv.edu/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

An on-chip, all-digital CMOS temperature sensor and Time-to-Digital Converter (TDC) core modeled and verified at the transistor level using the **Predictive Technology Model (PTM) 45nm BSIM4** device models in LTspice. 

This design translates die junction temperature variations ($-20^\circ\text{C}$ to $+100^\circ\text{C}$) into picosecond-scale propagation delay shifts by exploiting intrinsic carrier mobility degradation ($\mu \propto T^{-1.5}$) driven by acoustic phonon scattering. An 8-bit synchronous D-Flip-Flop register bank strobed at $t = 0.54\text{ ns}$ freezes the propagating wavefront, quantizing delay into a monotonic, glitch-free thermometer code.

---

## Table of Contents

- [Architectural Overview](#architectural-overview)
- [VLSI Principles & Physical Transduction Mechanism](#vlsi-principles--physical-transduction-mechanism)
  - [1. Carrier Mobility Degradation](#1-carrier-mobility-degradation)
  - [2. Propagation Delay & Equivalent Resistance Formulation](#2-propagation-delay--equivalent-resistance-formulation)
  - [3. Wavefront Propagation Dynamics across Inverter Taps](#3-wavefront-propagation-dynamics-across-inverter-taps)
- [Circuit Implementation & Transistor Sizing](#circuit-implementation--transistor-sizing)
- [Simulation Results & Quantitative Characterization](#simulation-results--quantitative-characterization)
  - [Extracted Delay vs. Temperature Summary](#extracted-delay-vs-temperature-summary)
  - [Thermal Delay Sensitivity Extraction](#thermal-delay-sensitivity-extraction)
  - [Thermometer Code Digital Readout](#thermometer-code-digital-readout)
- [Waveform & Plot Catalog](#waveform--plot-catalog)
- [Backend Digital Integration: Thermometer-to-Binary Decoder](#backend-digital-integration-thermometer-to-binary-decoder)
- [Repository Structure](#repository-structure)
- [Simulation & Reproduction Guide](#simulation--reproduction-guide)
- [License](#license)

---

## Architectural Overview

As silicon fabrication scales into deep sub-micron regimes (45nm and below), nominal supply voltage drops ($V_{DD} \le 1.0\text{ V}$), severely degrading analog headroom. Conventional analog bandgap reference (BGR) thermal sensors suffer from low operational amplifier open-loop gain, intricate current-mirror biasing, high quiescent power dissipation, and substantial silicon area penalties.

This architecture processes physical temperature variations entirely in the **time domain**, utilizing standard static CMOS logic gates: an 8-stage cascaded inverter delay chain and an 8-bit parallel Transmission-Gate D-Flip-Flop (TG-DFF) register bank.

```
       START Trigger Edge (0.2 ns)
               │
               ▼
        ┌─────────────┐     tap0     ┌─────────────┐     tap1            tap7     ┌─────────────┐
        │ Inverter 0  ├──────┬──────►│ Inverter 1  ├──────┬──────► ... ────┬─────►│ Inverter 7  │
        │ (Wp=2x, Wn) │      │       │ (Wp=2x, Wn) │      │                │      │ (Wp=2x, Wn) │
        └─────────────┘      │       └─────────────┘      │                │      └─────────────┘
                            ┌┴┐                          ┌┴┐              ┌┴┐
                       C1   │ │ 2fF                 C2   │ │ 2fF     C8   │ │ 2fF
                            └┬┘                          └┬┘              └┬┘
                             │                            │                │
                             ▼                            ▼                ▼
                          ┌──────┐                     ┌──────┐         ┌──────┐
                          │ D  Q0│                     │ D  Q1│         │ D  Q7│
                          │  A1  │                     │  A2  │  ...    │  A8  │
                          │CLK   │                     │CLK   │         │CLK   │
                          └──▲───┘                     └──▲───┘         └──▲───┘
                             │                            │                │
SAMPLE Reference (0.54 ns) ──┴────────────────────────────┴────────────────┘
                                  (Synchronous Register Sampling Strobe)
```

---

## VLSI Principles & Physical Transduction Mechanism

### 1. Carrier Mobility Degradation
Within silicon crystal lattices, thermal lattice vibrations increase proportionally with die temperature $T$. This exacerbates **acoustic phonon scattering**, reducing the effective drift velocity and mobility ($\mu$) of electrons and holes:

$$\mu(T) = \mu(T_0) \left( \frac{T}{T_0} \right)^{-1.5}$$

### 2. Propagation Delay & Equivalent Resistance Formulation
From the course lecture notes and standard CMOS switching theory (Weste & Eshraghian; Rabaey), the propagation delay of an inverter stage discharging or charging a capacitive load $C_L$ is:

$$t_p = \int_{V_1}^{V_2} \frac{C_L(v)}{i(v)} \, dv \approx \frac{t_{pHL} + t_{pLH}}{2} = 0.69 \cdot C_L \left( \frac{R_{eqn} + R_{eqp}}{2} \right)$$

where the effective channel on-resistance $R_{eq}$ of the MOSFET in the triode/saturation switching region is inversely proportional to mobility:

$$R_{eq} \approx \frac{1}{\mu C_{ox} \left(\frac{W}{L}\right) (V_{DD} - V_{TH})}$$

Because $R_{eq}$ increases monotonically with temperature while the lumped interconnect and gate dielectric capacitance ($C_L = 2.0\text{ fF}$) remains essentially invariant, the stage delay stretches quasi-linearly as temperature climbs from $-20^\circ\text{C}$ to $+100^\circ\text{C}$.

### 3. Wavefront Propagation Dynamics across Inverter Taps
When `START` transitions from $0 \to 1.0\text{ V}$:
- **Odd Inverter Taps (`tap1, tap3, tap5, tap7`):** Undergo a **rising transition ($0 \to 1$)**.
- **Even Inverter Taps (`tap0, tap2, tap4, tap6`):** Undergo a **falling transition ($1 \to 0$)**.

The active rising temperature wavefront travels exclusively through the **odd taps**. When sampled at $t = 0.54\text{ ns}$ by the `SAMPLE` clock, odd-tap register outputs ($Q_1, Q_3, Q_5, Q_7$) reflect the spatial position of the traveling edge.

---

## Circuit Implementation & Transistor Sizing

The core transducer and TDC array are designed in accordance with static CMOS ratio rules and standard cell layout principles:

| Parameter | Symbol | Sizing / Value | Technical Design Motivation |
| :--- | :---: | :---: | :--- |
| **Technology Node** | — | 45nm PTM (Level 54 BSIM4) | Sub-micron predictive modeling |
| **Supply Voltage** | $V_{DD}$ | $1.0\text{ V}$ | Deep sub-micron standard rail |
| **NMOS Dimensions** | $(W/L)_n$ | $90\text{ nm} / 45\text{ nm}$ | Baseline channel unit ($1\times$) |
| **PMOS Dimensions** | $(W/L)_p$ | $180\text{ nm} / 45\text{ nm}$ | Sized $2\times$ to balance $\mu_n / \mu_p$ and symmetric switching |
| **Stage Load Capacitance** | $C_1 - C_8$ | $2.0\text{ fF}$ | Lumped parasitic wiring and gate load per tap |
| **Quantizer Storage Element** | $A_1 - A_8$ | Transmission-Gate DFF (`dflop`) | Edge-triggered, race-free latching |
| **Asynchronous Control Lines**| `PRE`, `CLR` | Connected to $0\text{ V}$ (GND) | Prevents floating inputs and unforced logic overrides |

---

## Simulation Results & Quantitative Characterization

Transient simulations were executed using a multi-corner temperature sweep (`.step temp -20 100 40`) and high-precision integration (`method=trap`, maximum timestep $= 0.01\text{ ns}$).

### Extracted Delay vs. Temperature Summary

Values measured at the 50% voltage switching threshold ($V = 0.5\text{ V}$) via `.measure tran t_delay_stage7`:

| Step | Die Temp ($T$) | Trigger Instant (`START` 50%) | Target Arrival (`tap7` 50%) | Measured Total Delay ($t_{pd}$) | Delay Shift ($\Delta t_{pd}$) |
| :---: | :---: | :---: | :---: | :---: | :---: |
| **1** | **$-20^\circ\text{C}$** | $0.225\text{ ns}$ | $0.4974\text{ ns}$ | **$272.41\text{ ps}$** | Baseline ($0.00\text{ ps}$) |
| **2** | **$+20^\circ\text{C}$** | $0.225\text{ ns}$ | $0.5241\text{ ns}$ | **$299.13\text{ ps}$** | $+26.72\text{ ps}$ |
| **3** | **$+60^\circ\text{C}$** | $0.225\text{ ns}$ | $0.5523\text{ ns}$ | **$327.33\text{ ps}$** | $+54.92\text{ ps}$ |
| **4** | **$+100^\circ\text{C}$** | $0.225\text{ ns}$ | $0.5821\text{ ns}$ | **$357.06\text{ ps}$** | $+84.65\text{ ps}$ |

### Thermal Delay Sensitivity Extraction

The thermal sensitivity $S$ is extracted directly from the slope of the linear transfer characteristic:

$$S = \frac{\Delta t_{pd}}{\Delta T} = \frac{357.06\text{ ps} - 272.41\text{ ps}}{100^\circ\text{C} - (-20^\circ\text{C})} = \frac{84.65\text{ ps}}{120^\circ\text{C}} = \mathbf{0.705\text{ ps}/^\circ\text{C}}$$

### Thermometer Code Digital Readout

At $t = 1.50\text{ ns}$ steady state, sampling registers exhibit thermometer code contraction as temperature scales:

| Step | Temperature ($T$) | $Q_1$ | $Q_3$ | $Q_5$ | $Q_7$ | Thermometer Word ($Q[1,3,5,7]$) | Active Code Count |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **1** | **$-20^\circ\text{C}$** | $1.0\text{ V}$ | $1.0\text{ V}$ | $1.0\text{ V}$ | $1.0\text{ V}$ | `1 1 1 1` | **4 Bits** (Fastest) |
| **2** | **$+20^\circ\text{C}$** | $1.0\text{ V}$ | $1.0\text{ V}$ | $1.0\text{ V}$ | $1.0\text{ V}$ | `1 1 1 1` | **4 Bits** (Nominal) |
| **3** | **$+60^\circ\text{C}$** | $1.0\text{ V}$ | $1.0\text{ V}$ | $1.0\text{ V}$ | $0.0\text{ V}$ | `1 1 1 0` | **3 Bits** (Contracting) |
| **4** | **$+100^\circ\text{C}$** | $1.0\text{ V}$ | $1.0\text{ V}$ | $1.0\text{ V}$ | $0.0\text{ V}$ | `1 1 1 0` | **3 Bits** (Thermal Throttle) |

*Note on Even Tap Outputs:* Even taps carry falling edges ($1 \to 0$). At $-20^\circ\text{C}, +20^\circ\text{C},$ and $+60^\circ\text{C}$, the falling edge clears $tap6$ to $0\text{ V}$ prior to the sample strobe ($Q_6 = 0$). At $+100^\circ\text{C}$, increased thermal channel resistance prevents the falling edge from reaching Stage 6 before the sample strobe, leaving $tap6$ at its pre-charged high state and latching $Q_6 = 1$.

---

## Simulation Verification & Characteristic Plots

All vector plots, waveforms, and schematics are available in the repository under the [doc/figures/](doc/figures/) directory:

### 1. Circuit Schematic
* [ **schematic.pdf**](doc/figures/schematic.pdf) — Complete transistor-level schematic of the 8-stage cascaded inverter delay chain, 2 fF load capacitors, and synchronous 8-bit D-flip-flop register bank.

### 2. Transfer Characteristic: Delay vs. Temperature
* [ **time_vs_temp.pdf**](doc/figures/time_vs_temp.pdf) — 2D calibration response generated via `.step'ed .meas` data plotting $t_{pd}$ across $-20^\circ\text{C} \le T \le +100^\circ\text{C}$. The linear response confirms a constant sensitivity of **$0.705\text{ ps}/^\circ\text{C}$**.

### 3. Low-Temperature Transient Waveforms
* [ **low_propagation.pdf**](doc/figures/low_propagation.pdf) — Transient waveforms at $-20^\circ\text{C}$ and $+20^\circ\text{C}$. The wavefront reaches Stage 7 before the $0.54\text{ ns}$ sample strobe, causing all odd outputs ($V(Q_1), V(Q_3), V(Q_5), V(Q_7)$) to latch HIGH ($1.0\text{ V}$).

### 4. High-Temperature Transient Waveforms
* [ **propagation_delay.pdf**](doc/figures/propagation_delay.pdf) — Transient waveforms at elevated temperatures ($+60^\circ\text{C}$ and $+100^\circ\text{C}$). Increased propagation delay pushes Stage 7 arrival beyond the strobe instant, holding $V(Q_7)$ at $0.0\text{ V}$ while earlier stages latch successfully.

### 5. Strobe vs. Tap 7 Wavefront Margin
* [ **t7_withSample.pdf**](doc/figures/t7_withSample.pdf) — Detailed overlay of $V(\text{tap7})$ transitions against the rising edge of $V(\text{sample})$ at $0.54\text{ ns}$, confirming setup and hold timing margins across temperature corners.

## Backend Digital Integration: Thermometer-to-Binary Decoder

In physical mixed-signal VLSI implementations, thermometer code is selected at the sensor core to guarantee monotonicity and prevent output code glitching. Conversion into a compact 2-bit binary index ($B_1 B_0$) for on-chip Dynamic Voltage and Frequency Scaling (DVFS) cores is synthesized via combinational priority logic:

$$\text{Thermometer Input } (Q_7 Q_5 Q_3 Q_1) \longrightarrow \left[ \text{Priority Encoder} \right] \longrightarrow \text{Binary Output } (B_1 B_0)$$

### Truth Table

| Operating Thermal Regime | $Q_7$ | $Q_5$ | $Q_3$ | $Q_1$ | Binary Output ($B_1 B_0$) | Decimal | System Action |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **Sub-zero / Maximum Frequency ($-20^\circ\text{C}$)** | `1` | `1` | `1` | `1` | **`1 1`** | 3 | Turbo Clock Enabled |
| **Nominal Operating Regime ($+20^\circ\text{C}$)** | `1` | `1` | `1` | `1` | **`1 1`** | 3 | Nominal Performance |
| **Elevated Thermal Load ($+60^\circ\text{C}$)** | `0` | `1` | `1` | `1` | **`1 0`** | 2 | Fan Speed Boost / Intermediate Throttle |
| **Critical Thermal Limit ($+100^\circ\text{C}$)** | `0` | `0` | `1` | `1` | **`0 1`** | 1 | Hardware DVFS Throttle Triggered |

### Synthesized Boolean Equations
$$B_1 = Q_5$$
$$B_0 = Q_7 + \left(Q_3 \cdot \overline{Q_5}\right)$$

---

## Repository Structure

```text
all-digital-cmos-tdc-temp-sensor/
├── README.md                           # Master technical project documentation
├── LICENSE                             # MIT Open-Source License
├── sim/
│   ├── TDC_Temp_Sensor.asc             # Master LTspice circuit schematic
│   ├── TDC_Temp_Sensor.net             # Exported SPICE netlist
│   ├── TDC_Temp_Sensor.raw             # Binary transient waveform simulation data
│   └── TDC_Temp_Sensor.log             # Simulation execution log containing .measure data
├── docs/
   └── figures/
        ├── schematic.pdf               # Complete schematic drawing
        ├── time_vs_temp.pdf            # 2D delay vs. temperature calibration plot
        ├── low_propagation.pdf         # Sampled waveforms at low/nominal temperature
        ├── propagation_delay.pdf       # Sampled waveforms under high thermal delay
        ├── t7_withSample.pdf           # Wavefront vs. sampling strobe alignment
 
 ```

---

## Simulation & Reproduction Guide

### Prerequisites
- **LTspice 17.x or 26.1+** installed on Windows, macOS, or Linux.
- 45nm BSIM4 Predictive Technology Model cards (embedded directly in the schematic netlist).

### Step-by-Step Execution
1. **Clone the Repository:**
   ```bash
   git clone https://github.com/<your-username>/all-digital-cmos-tdc-temp-sensor.git
   cd all-digital-cmos-tdc-temp-sensor
   ```
2. **Open Design:**
   Launch LTspice and open `sim/TDC_Temp_Sensor.asc`.
3. **Run Simulation:**
   Click **Simulate $\to$ Run** (or click the Running Man icon). LTspice will execute the 4-corner transient sweep across $-20^\circ\text{C}$, $+20^\circ\text{C}$, $+60^\circ\text{C}$, and $+100^\circ\text{C}$.
4. **Display Calibration Curve:**
   - Press `Ctrl + L` to open the **SPICE Error Log**.
   - Right-click inside the log window $\to$ select **"Plot .step'ed .meas data"**.
   - Press `Ctrl + A` (Add Trace) $\to$ choose `t_delay_stage7`.
5. **Inspect Sampled Registers:**
   Add a second plot pane and probe `V(q1)`, `V(q3)`, `V(q5)`, and `V(q7)`. Use `Ctrl + S` (Select Steps) to toggle between Step 1 ($-20^\circ\text{C}$) and Step 4 ($+100^\circ\text{C}$) to inspect thermometer bit contraction.

---


### Core References
1. N. H. E. Weste, K. Eshraghian, *Principles of CMOS VLSI Design: A Systems Perspective*, 2nd ed., Addison-Wesley, 1988/2004.
2. J. M. Rabaey, A. Chandrakasan, B. Nikolic, *Digital Integrated Circuits: A Design Perspective*, 2nd ed., Prentice Hall, 2003.
3. D. A. Pucknell, K. Eshraghian, *Basic VLSI Design*, 3rd ed., Prentice Hall of India, 2007.

---

## License

This project is open-source and distributed under the [MIT License](LICENSE).
