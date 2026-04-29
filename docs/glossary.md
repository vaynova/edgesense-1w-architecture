# Glossary

Terms used across the EdgeSense-1W documentation. This glossary is non-exhaustive; it focuses on terms where meaning is precise within this project and might differ slightly from general usage in adjacent fields.

## Architecture and silicon

**Activation** — the output tensor of a single layer in a neural network, used as the input to the next layer.

**Activation memory** — the on-chip SRAM in EdgeSense-1W that holds activations during the inference pass. Implemented as two ping-pong banks.

**Clock-gating** — a circuit technique that disables the clock signal to a subsystem when it is not in use, dramatically reducing dynamic power without losing state.

**DS-Conv1D** — Depthwise-Separable 1D Convolution. A factorisation of standard 1D convolution into a depthwise step (one filter per channel) followed by a pointwise mixing step (1×1 convolution). Reduces multiply-accumulate operations by approximately one order of magnitude for typical layer geometries used in this project.

**MAC** — Multiply-Accumulate. The fundamental compute operation in convolutional and dense neural network layers. MAC operations per inference is the standard measure of compute load.

**Ping-pong buffering** — a double-buffered memory scheme where two banks alternate roles between layers, allowing read-write access without pipeline stalls.

**Quantisation** — the process of representing neural network weights and activations with reduced numerical precision (typically integer rather than floating-point). EdgeSense-1W operates exclusively on quantised arithmetic.

**Scratchpad** — small, fast, non-host-visible on-chip memory used for intermediate values within fused-layer operations.

**SRAM-first** — the design philosophy of holding all weights and activations on-chip in SRAM during inference, eliminating off-chip memory access from the inference path.

**Weight memory** — the on-chip SRAM that holds compiled model weights. Loaded once per power-up and retained through idle periods.

## Workload and application

**Anomaly detection** — the inference task of identifying that an input signal departs from a learned distribution of normal operation, without necessarily classifying the cause.

**Bearing fault** — a category of mechanical failure in rotating machinery characterised by distinctive vibration signatures at frequencies related to bearing geometry. Common faults include outer-race defects, inner-race defects, ball defects, and cage defects.

**Condition monitoring** — the continuous or periodic assessment of the operational state of a machine, typically using vibration, temperature, acoustic, or current-signature data, to detect emerging faults before failure.

**Fault classification** — the inference task of identifying the specific type of fault present in a machine, usually as a multiclass output (e.g. bearing wear, imbalance, misalignment, looseness).

**TRL** — Technology Readiness Level. A scale from 1 to 9 used by funders such as Innovate UK to characterise the maturity of a technology. EdgeSense-1W is currently at TRL 3 with a target of TRL 5 within the active development programme.

**Vibration monitoring** — the use of accelerometer data to assess the operational state of vibrating machinery. The dominant non-destructive condition-monitoring technique for rotating equipment.

## Interface and integration

**I²C** — Inter-Integrated Circuit. A two-wire serial bus standard widely used in industrial and embedded systems.

**PLC** — Programmable Logic Controller. The standard real-time controller class used in industrial automation. EdgeSense-1W is designed to integrate as a peripheral on PLC and gateway hardware.

**Register map** — the set of host-accessible memory-mapped registers exposed by a peripheral device. EdgeSense-1W exposes 10 registers through its dual-mode SPI/I²C interface.

**SCADA** — Supervisory Control and Data Acquisition. The umbrella term for industrial monitoring and control systems that aggregate data from PLCs, sensors, and edge devices. EdgeSense-1W output is intended to flow upward into SCADA systems via the host gateway.

**SPI** — Serial Peripheral Interface. A four-wire synchronous serial bus standard. EdgeSense-1W supports SPI mode 0.
