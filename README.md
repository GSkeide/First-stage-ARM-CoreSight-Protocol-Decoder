# ARM CoreSight Frame Deformatter on FPGA (Zynq-7000)

A bufferless, VHDL hardware pipeline that deformats ARM CoreSight TPIU trace frames in real time on a Zynq-7000's programmable logic. It keeps pace with the TPIU's maximum output of 32 bits per clock cycle.

This is the first stage of a hardware Control Flow Integrity (CFI) pipeline. The goal is to detect control-flow attacks such as ROP and buffer overflows from processor trace, and to halt the CPU the moment one is detected.

Bachelor thesis in Electronics, Western Norway University of Applied Sciences (HVL), 2026. [Full report (PDF)](./ELN_06_Bachelor_Report.pdf)

<img width="1324" height="496" alt="Project overview" src="https://github.com/user-attachments/assets/4ef66516-948f-4256-9c0b-d99d66cfb1f7" />

## Key results

- **125 MHz** on a Zynq-7000 (xc7z020), all timing constraints met (WNS +0.490 ns)
- **4 bytes/cycle (500 MB/s)**, matching the TPIU's full 32-bit port width with no buffering
- **Byte-for-byte agreement with OpenCSD** across ~25 KB of real multi-source trace data
- **Small footprint:** 4.07% LUTs and 3.11% registers, leaving room for later pipeline stages

**Tech:** VHDL, Vivado, Vitis (bare-metal C), AXI4-Stream / AXI DMA, ILA, CSAL, OpenCSD, TCL, C#, Yocto


## Background

ARM CoreSight is a large on-chip debug and trace infrastructure. Its trace sources (here, the Cortex-A9's Program Trace Macrocell, PTM) record every control-flow-changing instruction with zero overhead on the running software.

<img width="1530" height="1294" alt="CoreSight debug and trace architecture" src="https://github.com/user-attachments/assets/52da1be4-7a85-40d1-9bbb-bb11bc16dae4" />

_Source: ARM CoreSight documentation_

The full research project is a three-stage pipeline:

1. **Frame deformatter** (this project)
2. **Instruction trace decoder**
3. **Trace analyzer** that compares execution against a precomputed Control Flow Graph. On a violation, it signals the Cross Trigger Interface (CTI) through the Fabric Trigger Macrocell (FTM) to halt the processor.

This project is a collaboration between HVL and the SUSHI team at CentraleSupélec.

## System overview

We used the CoreSight Access Library (CSAL) to configure the trace sources and route the trace stream through the TPIU into the FPGA fabric.

<img width="1500" height="1028" alt="System overview: PTM trace routed through the TPIU into the PL pipeline and out to DDR via AXI DMA" src="https://github.com/user-attachments/assets/382b7ceb-1901-4add-89e3-6eb3479e4ffa" />

## The design constraint

The TPIU interleaves trace from multiple sources into **16-byte frames**. The last byte holds auxiliary bits that are needed to interpret every byte before it, so a frame can't be deformatted until all of it has arrived.

<img width="1552" height="756" alt="CoreSight 16-byte frame format" src="https://github.com/user-attachments/assets/9c9d3050-54c0-447f-a37e-8c5732deddb0" />

The TPIU delivers at most 32 bits per cycle and can't be paused. At full throughput, a new frame is complete every 4 clock cycles, so the deformatter must also finish each frame in 4 cycles.

Many existing hardware decoders process 1 byte per cycle and accept occasional buffer overflows. In a security context that isn't acceptable, because a dropped packet can hide an attack or cause a false alarm. **High throughput is a correctness requirement here, not just a performance goal.**

## Architecture

**Frame generator.** This block filters out synchronization and idle words, then assembles four valid 32-bit words into a 128-bit frame. It also gates the pipeline until the first sync word arrives. We added this after finding on hardware that the TPIU outputs filler data before it is enabled.

**Frame deformatter.** A 4-state pipelined FSM (IDLE, Cycle_A, Cycle_B, Cycle_C). Each state deformats 3–4 bytes, which spreads the combinational logic evenly across the four cycles to maximize clock frequency. It handles delayed source-ID changes across cycle boundaries and outputs 4 channels per cycle, each containing a data byte tagged with its source ID.

<img width="1396" height="1096" alt="Deformatter 4-state FSM" src="https://github.com/user-attachments/assets/b5347688-2288-4aa5-9b91-c255eb234bda" />

**AXI master and DMA.** An 8 KiB buffer with its own FSM (FILLING, STREAMING, DONE) streams the deformatted data to DDR through AXI DMA when the PS triggers it over GPIO.

<img width="1652" height="918" alt="AXI master FSM and AXI DMA connections" src="https://github.com/user-attachments/assets/4a96b543-a47f-4d53-be2d-9e9d641b56ac" />

## Verification

1. Unit testbenches for each VHDL block
2. A BRAM-fed integration pipeline: real trace captured from the Embedded Trace Buffer (ETB) is replayed through the hardware, and a C# script compares the output against OpenCSD. **Result: byte-for-byte match on ~25 KB of multi-source trace.**
3. On-hardware validation with the Integrated Logic Analyzer (ILA) on the live TPIU stream. This confirmed correct framing, idle filtering and AXI FSM behaviour.

<img width="1440" height="1326" alt="Verification flow against OpenCSD" src="https://github.com/user-attachments/assets/0ddfbc4b-bec4-4b4e-be2c-df01837aaf12" />

**Known limitation:** the live TPIU → DMA → OpenCSD comparison wasn't completed. A late Vitis configuration corruption broke the software side of the DMA readout. ILA inspection confirmed that the VHDL pipeline itself was still producing correct data.

## Implementation results

**Timing at 125 MHz**

| Design | Setup (WNS) | Hold (WHS) | Pulse width (WPWS) |
|---|---|---|---|
| TPIU system | 0.490 ns | 0.009 ns | 2.750 ns |
| BRAM test system | 3.257 ns | 0.084 ns | 3.020 ns |

**Resource utilization (xc7z020)**

| Slice LUTs | Slice registers | Slices | BRAM tiles |
|---|---|---|---|
| 2164 (4.07%) | 3313 (3.11%) | 956 (7.19%) | 6.5 (4.64%) |

**Comparison with prior hardware decoders (worst-case throughput)**

| Work | Device | Target | Bytes/cycle | Freq | Throughput | Buffered |
|---|---|---|---|---|---|---|
| Weingarten et al. (DATE 2024) | Zynq UltraScale+ | ETMv4 | 4 | 250 MHz | 1 GB/s | No |
| Zeinabolin | Virtex-6 | ETMv4 | 1 | 125 MHz | 125 MB/s | Yes |
| Schmid | Zynq-7000 | PTM | 1 | 125 MHz | 125 MB/s | Yes |
| **This work** | **Zynq-7000** | **Deformatter** | **4** | **125 MHz** | **500 MB/s** | **No** |

## Future work

- **Stage 2:** an instruction trace decoder that consumes the deformatter's 64-bit output stream
- **Faster fabric:** routing dominates the critical path on the Zynq-7000 (5.06 ns of 7.38 ns). Later stages will add more logic, so we recommend moving to a Zynq UltraScale+, which has severely improved routing architecture, potentially improving the timing by 4 times.
- **Buffered alternative:** in normal operation there are hundreds of idle words between trace words, so buffering could work. However, an attacker could deliberately generate bursts of trace to overflow the buffer, which makes it risky for security use.


## Authors and acknowledgements

Georg Ansgar Skeide, Johan Romarheim, Saleh Barakat.
Supervisor: Endre Håland. Client contacts: Volker Stolz and Eivind Vågslid Skjæveland (HVL).
