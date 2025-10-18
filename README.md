# **Radiation-Tolerant Mixed-Signal SoC with ReRAM-Based Non-Volatile Memory Integration**

## **1. Technical Overview**

This repository documents the design and verification of a **radiation-tolerant mixed-signal telemetry acquisition System-on-Chip (SoC)** implemented using **BM Labs ReRAM-based Non-Volatile Memory (NVM) IP**, synthesized and physically verified under the **SkyWater SKY130 process** within the **Caravel open-source platform**.

The SoC implements **redundant digital control logic**, **radiation-hardened memory-mapped NVM configuration**, and **high-precision analog sampling and telemetry interfaces** designed for **long-duration mission operation** in environments subject to ionizing radiation or unpredictable power cycling.

The architecture targets **reliable configuration retention and analog parameter persistence**, enabling full state recovery without firmware intervention following single-event upsets or brownout events.

---

## **2. Design Motivation and Application Domain**

In radiation-exposed or intermittently powered systems — such as **low-orbit telemetry units, remote sensing payloads, or autonomous instrumentation nodes** — volatile memories are unsuitable for parameter retention between power cycles.
ReRAM NVM provides a **bitcell-level non-volatility** with **low write current** and **radiation resilience**, making it ideal for retaining calibration constants, acquisition parameters, and control states.

The system serves as a **complete telemetry controller**, consisting of:

* A multi-channel ADC/DAC interface for analog signal acquisition.
* Digital control logic for sampling scheduling and data formatting.
* ReRAM-based parameter storage for configuration persistence.
* Fault-tolerant reset and recovery logic for deterministic restart.

---

## **3. System Architecture**

### **3.1 High-Level Block Diagram**

```
                ┌───────────────────────────────────────┐
                │             SoC Top Module             │
                │---------------------------------------│
                │     • Clock & Reset Controller         │
                │     • Power Supervisor FSM             │
                │     • Wishbone Bus Interconnect        │
                │     • Interrupt Routing Logic          │
                ├───────────────────────────────────────┤
                │           Digital Subsystem            │
                │  ├───────────────────────────────┐     │
                │  │   NVM Control Unit            │     │
                │  │   ReRAM Access Sequencer      │     │
                │  │   ECC & Scrubbing Engine      │     │
                │  └───────────────────────────────┘     │
                │  ├───────────────────────────────┐     │
                │  │   Acquisition Controller      │     │
                │  │   DMA and FIFO Buffers        │     │
                │  └───────────────────────────────┘     │
                ├───────────────────────────────────────┤
                │            Analog Subsystem            │
                │  • 12-bit SAR ADC Interface            │
                │  • DAC Channel & Buffer Stage          │
                │  • Voltage Reference and Bias DAC       │
                │  • Analog Multiplexer Network          │
                ├───────────────────────────────────────┤
                │           Caravel Wrapper              │
                │  (Wishbone Bridge, IO Pads, PLL)       │
                └───────────────────────────────────────┘
```

---

## **4. Subsystem Descriptions**

### **4.1 NVM Control Subsystem**

Implements direct ReRAM IP interfacing through a **synchronous access sequencer** with built-in **error correction and scrubbing**.

Key features:

* **Address-mapped control registers** (`0x00`–`0x3F`) for configuration and diagnostics.
* **ECC layer (SEC-DED)** using Hamming(72,64) code on every 64-bit data word.
* **Redundant read verification** with programmable retry depth (default: 2).
* **Clock-domain crossing FIFOs** between the system and memory clock domains.
* **Power-aware state machine** managing safe NVM writes during voltage transients.

Timing of NVM transactions is enforced by `nvm_access_ctrl.v` and verified against provided `.lib` delay models.

---

### **4.2 Telemetry Control Logic**

* Implements a **sampling scheduler** driven by a 10-bit prescaler and hardware timer.
* Buffers sampled analog data into a **two-level FIFO**, each protected by parity bits.
* DMA logic transfers data to a shared SRAM region for serial output or debug readout.
* Control and status accessible through the **Wishbone bus**, with direct register-mapped access to the analog subsystem.

---

### **4.3 Analog Subsystem**

* **12-bit SAR ADC** with fully differential input and internal reference trimming.
* **Bias control coefficients stored in ReRAM** for deterministic power-up calibration.
* **Analog multiplexer (AMUX)** supporting 4 external and 2 internal channels.
* DAC channel for calibration loop closure and bias feedback path adjustment.
* Includes **Ngspice-based behavioral model** for mixed-signal simulation validation.

---

## **5. Power and Clock Domains**

| Domain  | Voltage | Description                           |
| ------- | ------- | ------------------------------------- |
| VDD_DIG | 1.8V    | Core digital logic and NVM controller |
| VDD_ANA | 3.3V    | ADC, DAC, and bias networks           |
| VDD_IO  | 3.3V    | Caravel IO interface                  |
| VDD_NVM | 1.8V    | ReRAM IP and NVM controller           |

* **Independent clock domains:**

  * `sys_clk` (25 MHz nominal, digital domain)
  * `nvm_clk` (5 MHz, controlled access domain)
  * `adc_clk` (2 MHz sampling domain)
* Cross-domain synchronization handled by asynchronous FIFOs and 2-stage metastability filters.

---

## **6. Verification Infrastructure**

### **6.1 RTL Simulation**

Executed using Icarus Verilog and automated through Makefile pipelines:

```bash
cd verif/rtl
make sim
```

Testbenches:

* `tb_nvm_ctrl.v` – ECC and scrubbing verification.
* `tb_telemetry.v` – ADC sampling and buffer handling.
* `tb_reset.v` – power-cycle persistence and fault recovery validation.

Functional coverage metrics generated using `verilator --coverage` and reported under `/verif/reports/`.

### **6.2 Gate-Level Simulation**

Post-synthesis timing back-annotation with OpenSTA-generated SDF:

```bash
make gls
```

Simulation ensures no hold-time or setup violations under worst-case process corners (SS, 125°C, 1.62V).

### **6.3 Mixed-Signal Co-Simulation**

Analog verification executed using Ngspice:

```bash
ngspice adc_nvm_coupling.sp
```

Validates the retention of analog reference coefficients stored in ReRAM and reloaded into bias DACs post-reset.

---

## **7. Physical Design Flow**

### **7.1 Floorplanning**

* Utilizes Caravel user area with placement grid alignment at 1.8V domain.
* ReRAM IP macro placed at (145μm, 310μm), analog isolation ring applied on its perimeter.
* Decoupling capacitors (16 × 1pF cells) distributed along VDD_NVM.

### **7.2 Routing and Verification**

Performed using OpenLane automated flow:

```bash
make synth
make place
make cts
make route
make magic_drc
make netgen_lvs
```

* Metal density within 42–48% across core area.
* All vias verified for current density limits below 0.8 mA/μm².
* DRC and LVS verified clean against SKY130A ruleset.

---

## **8. Memory Map**

| Address | Register     | Function                             |
| ------- | ------------ | ------------------------------------ |
| 0x00    | `NVM_CTRL`   | Enable/disable write operations      |
| 0x04    | `NVM_STATUS` | Status flags (BUSY, ERR, RDY)        |
| 0x08    | `NVM_ADDR`   | 16-bit word address                  |
| 0x0C    | `NVM_DATA`   | 64-bit data register (LSW/MSW split) |
| 0x10    | `ECC_LOG`    | Error log and syndrome vector        |
| 0x14    | `SCRUB_CFG`  | ECC scrubbing interval control       |
| 0x18    | `ADC_CFG`    | Sampling configuration word          |
| 0x1C    | `DAC_CFG`    | Bias DAC trim word                   |
| 0x20    | `SYS_INT_EN` | Interrupt enable flags               |

---

## **9. Deliverables and Repository Structure**

```
├── rtl/                 # Synthesizable RTL design files
├── analog/              # Ngspice models and SPICE decks
├── verif/               # Functional verification environment
├── openlane/            # Floorplan and route configuration
├── sta/                 # Static timing and corner reports
├── docs/                # Design specification and register documentation
├── gds/                 # Final DRC/LVS-clean GDSII layout
└── firmware/            # Bare-metal firmware for test stimulus
```

Deliverables include:

* Complete RTL source and verification testbenches.
* SDF, STA, and corner analysis results.
* Final routed GDSII layout.
* Timing closure report (TT, SS, FF corners).
* Documentation and schematic-level block diagrams.

---

## **10. Timing Summary**

| Parameter       | Typical  | Worst-Case |
| --------------- | -------- | ---------- |
| System Clock    | 25 MHz   | 22 MHz     |
| NVM Access      | 3.5 μs   | 4.2 μs     |
| ADC Throughput  | 50 kS/s  | 45 kS/s    |
| ECC Scrub Cycle | 6.8 μs   | 8.1 μs     |
| Setup Slack     | +0.18 ns | +0.04 ns   |
| Hold Slack      | +0.09 ns | +0.02 ns   |

---

## **11. Known Limitations and Future Work**

* Current analog co-simulation uses a simplified model for ADC reference drift; post-silicon measurements will be required for accurate calibration modeling.
* ReRAM endurance not stress-tested beyond 10⁶ cycles in behavioral model.
* Single voltage regulator domain assumed; multi-domain isolation planned for next revision.

---

## **12. Licensing and Reproducibility**

* **License:** Apache 2.0
* **PDK:** SkyWater SKY130A
* **EDA Flow:** Fully open-source (OpenLane, Magic, Klayout, Ngspice, Icarus Verilog).
* **Integration Platform:** Efabless Caravel.
* All design and verification steps are fully reproducible using provided Makefiles and scripts.

---


This project demonstrates a **radiation-tolerant telemetry acquisition SoC** integrating ReRAM-based NVM for persistent system configuration and calibration data retention.
It provides a **complete open-source flow from RTL to GDS**, including analog-mixed-signal interfacing, multi-domain clocking, ECC protection, and DRC/LVS-clean physical implementation suitable for fabrication under the SkyWater 130nm process.
