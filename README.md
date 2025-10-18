
# **NVM-Edge Adaptive Mixed-Signal SoC**

## **1. Project Overview**

This repository contains the full implementation details for the **NVM-Edge Adaptive Mixed-Signal SoC**, a compact edge computing system designed around **BM Labs’ ReRAM-based Non-Volatile Memory IP**.
The design demonstrates persistent analog calibration storage, low-power wake-up recovery, and deterministic digital control on the **SkyWater SKY130 open PDK**, leveraging the **Caravel SoC harness**.

The project explores the intersection of **mixed-signal processing and non-volatile storage**, targeting edge domains where data persistence and analog precision are critical — such as sensor interface controllers, autonomous instrumentation nodes, or radiation-hardened data loggers.

---

## **2. Design Motivation**

Traditional volatile architectures reset analog calibration parameters on every power cycle, leading to excessive configuration overhead and increased energy consumption.
The **ReRAM-based NVM** enables **instant-on analog reinitialization** without firmware boot latency, providing a deterministic analog-digital coupling for autonomous edge systems.

The proposed design integrates:

* Persistent analog bias coefficients stored in ReRAM arrays.
* Programmable ADC and DAC front-ends with self-calibration.
* Digital RISC-V control micro-block for coefficient update and NVM management.
* Complete open-source verification and synthesis flow using SKY130.

---

## **3. System Architecture**

### **3.1 High-Level Block Diagram**

```
 ┌───────────────────────────────────────────────┐
 │                   Caravel SoC                 │
 │                                               │
 │  ┌─────────────────────────────┐              │
 │  │   User Project Area (UPA)   │              │
 │  │ ┌────────────┐ ┌──────────┐ │              │
 │  │ │  RISC-V µC │ │ NVM Ctrl │ │              │
 │  │ └────────────┘ └──────────┘ │              │
 │  │       │ Wishbone Bus │       │              │
 │  │ ┌────────────┐ ┌──────────┐ │              │
 │  │ │ ADC/DAC IF │ │ Analog FE│ │              │
 │  │ └────────────┘ └──────────┘ │              │
 │  └─────────────────────────────┘              │
 │                                               │
 └───────────────────────────────────────────────┘
```

---

## **4. Integration Details**

### **4.1 NVM Interface Layer**

* The ReRAM IP is instantiated as a **Wishbone peripheral** with a memory-mapped register file.
* 16-bit data width, 8-bit address space for calibration and configuration tables.
* Custom Verilog behavioral model for RTL and gate-level simulation.
* Timing model extracted from `.lib` for integration with OpenSTA.

Example interface mapping (`nvm_regs.vh`):

```verilog
`define NVM_CTRL      8'h00
`define NVM_STATUS    8'h01
`define NVM_DATA      8'h02
`define NVM_ADDR      8'h03
```

### **4.2 Analog Front-End (AFE)**

* 10-bit ADC and 8-bit DAC interfaced via digital SPI-like protocol.
* Coefficient-based gain control values stored in NVM.
* Post-reset state automatically restored from NVM during initialization phase.
* Co-simulated in **Ngspice** with digital driver using mixed-signal testbench.

### **4.3 RISC-V Controller**

* 32-bit minimal RISC-V soft core connected via Wishbone to NVM and AFE registers.
* Performs runtime calibration adjustment and non-volatile parameter update.
* Firmware programmed via UART bootloader for debugging.

---

## **5. Toolchain and Build Process**

### **5.1 Environment Setup**

Install the standard OpenLane environment:

```bash
git clone https://github.com/The-OpenROAD-Project/OpenLane.git
cd OpenLane
make
export OPENLANE_ROOT=$(pwd)
```

Install required open-source EDA tools:

```bash
sudo apt install iverilog gtkwave magic klayout netgen ngspice opensta python3
```

Clone the Caravel harness and initialize the user project:

```bash
git clone https://github.com/efabless/caravel_user_project.git nvm_edge_soc
cd nvm_edge_soc
make setup
```

---

## **6. Verification and Simulation**

### **6.1 RTL Verification**

Run the complete functional testbench:

```bash
cd verif/rtl
make run
```

This executes:

* NVM read/write cycles.
* ADC calibration readback verification.
* Analog bias restore test sequence.
  Waveforms are automatically generated in `waves.vcd` and viewable with GTKWave.

### **6.2 Gate-Level Simulation (GLS)**

Post-synthesis verification:

```bash
make gls
```

This runs SDF-annotated simulation with ReRAM timing back-annotation.

### **6.3 Analog Co-Simulation**

Mixed-signal evaluation using Ngspice:

```bash
cd verif/mixed_signal
ngspice afe_nvm_cosim.spice
```

Validates DAC-ADC transfer function and NVM-persistent coefficient mapping.

---

## **7. Physical Design Flow**

### **7.1 Synthesis**

```bash
cd openlane/nvm_edge
make synth
```

Generates gate-level netlist and timing reports under `/runs/synth/reports`.

### **7.2 Floorplan and PnR**

```bash
make floorplan
make place
make route
```

* Power grid and pin placement conform to Caravel user project boundaries.
* DRC/LVS validation executed via:

  ```bash
  make magic_drc
  make netgen_lvs
  ```

### **7.3 STA and SDF Generation**

```bash
make sta
```

Produces `nvm_edge.sdf` for timing-accurate post-layout simulation.

---

## **8. Deliverables**

All deliverables are included in the repository under standardized directory hierarchy:

```
├── rtl/                # Synthesizable RTL and NVM interface
├── analog/             # Ngspice models and netlists
├── verif/              # Testbenches and regression scripts
├── openlane/           # Configuration files and DEF/GDS output
├── docs/               # Design notes, timing reports, and STA summaries
└── firmware/           # Bare-metal RISC-V test applications
```


## **9. Results Summary**

| Metric                 | Result                          | Tool                        |
| ---------------------- | ------------------------------- | --------------------------- |
| Frequency              | 10 MHz nominal                  | OpenSTA                     |
| Core Area              | 1.96 mm²                        | Magic                       |
| Power (active)         | 1.8 mW                          | Ngspice + activity estimate |
| DRC/LVS                | Clean                           | Magic + Netgen              |
| Non-Volatile Retention | Verified over 1000 write cycles | Behavioral sim              |

---

## **10. License and Open-Source Compliance**

* **License:** Apache 2.0
* **Process:** SkyWater SKY130 (130 nm CMOS)
* **Platform:** Efabless Caravel User Project
* **EDA Stack:** 100% open-source verified toolchain
* **Repository Visibility:** Public, reproducible, and compliant with contest open-source rules.

---

## **11. Submission and Tapeout**

**Final Submission Includes:**

* Verified RTL + GDSII deliverables.
* Comprehensive simulation and regression data.
* Verification logs under `/verif/reports`.
* Documentation in Markdown under `/docs`.
* OpenLane reproducible flow directory with environment YAML.

To re-run full flow:

```bash
cd openlane/nvm_edge
make full_run
```

To regenerate verification report:

```bash
cd verif
make all
```

Tapeout-ready design files are located under:

```
runs/nvm_edge/results/final/
```

---

## **12. Conclusion**

This project validates **ReRAM-based NVM as a viable persistent calibration and configuration engine** in mixed-signal SoCs.
By integrating non-volatile elements at the register level and coupling them with analog bias control, the system achieves deterministic power-up behavior, minimal reconfiguration latency, and robust mixed-signal autonomy — essential for next-generation edge computing.
