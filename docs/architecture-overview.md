# Architecture Overview

This document describes the top-level architecture of EdgeSense-1W. It is intended for engineers evaluating the device for integration, researchers comparing edge-AI accelerator approaches, and reviewers assessing the technical soundness of the design.

## Top-level block diagram

EdgeSense-1W is organised as five primary subsystems connected by a unified on-chip interconnect:

```
                    ┌────────────────────────────┐
                    │     Host Interface     │
                    │      SPI / I²C         │
                    └─────────────┘
                                │
                    ┌────────────┘
                    │    Control Unit        │
                    │  Sequencer · Registers │
                    └──┘────────────└
                        │               │
         ┌────────────────┐         ┌─────────────────┐
         │  Weight Memory   │         │  Activation Memory │
         │   On-chip SRAM   │         │  Ping-Pong SRAM    │
         └─────────────┐         └──────────────┐
                        │               │
                    ┌──────────────────────┐
                    │      MAC Array         │
                    │     DS-Conv1D core     │
                    └──────────────┘
                                │
                    ┌────────────────────────────┐
                    │   Post-processing      │
                    │  Activation · Pooling  │
                    └──────────────────────┘
                                │
                    ┌─────────────────────────────────┐
                    │   Result Register      │
                    │      Read by host      │
                    └──────────────┘
```

## Subsystem descriptions

### Host Interface

The host interface is a dual-mode SPI/I²C peripheral block. The mode is selected at power-up via a strap pin. Both modes expose the same register-based control surface; the choice between them is left to the system integrator based on existing host hardware and bus availability.

The interface implements no DMA, no shared memory, and no interrupt-driven inference triggering beyond a single completion-status interrupt line. Inference is host-initiated by a register write. This design choice keeps the host integration simple and predictable, at the cost of leaving some latency optimisation on the table.

### Control Unit

The control unit contains the inference sequencer, the configuration register file, and the layer-by-layer scheduling logic. It reads a compiled model description from weight memory at the start of an inference pass and walks the layers, issuing MAC operations and orchestrating activation buffering as it goes.

Models are compiled offline by the Vaynova toolchain into a fixed-format binary blob loaded into weight memory. Runtime model swapping is supported via reload, but the chip does not support multiple resident models.

### Weight Memory

A single bank of on-chip SRAM holds the compiled model. Weights are stored in a quantised, packed format optimised for the MAC array's access pattern. The capacity is sized to hold the largest model in the architecture's target envelope; models above that envelope are out of scope and require recompilation against a reduced architecture.

### Activation Memory

A two-bank ping-pong SRAM holds intermediate activations. While the MAC array reads inputs to layer N from bank A, it writes outputs from layer N to bank B. At the layer boundary, the banks swap roles. This design eliminates wait states at layer transitions and allows the MAC array to operate at near-peak utilisation across the inference pass.

### MAC Array

The core compute engine. The array is dimensioned and laid out specifically for depthwise-separable 1D convolution as the primary operation. Standard Conv1D, pointwise convolution (1×1), and dense layers are supported as composable subsets of the same primitive. Activation functions and pooling are handled by the post-processing block downstream.

The array operates on a fixed quantised numerical format. Floating-point inference is not supported. Quantisation-aware training is part of the Vaynova toolchain.

### Post-processing

A small fixed-function block applies the per-layer activation function (ReLU and a configurable LUT-based nonlinearity are supported), pooling where required, and final classification logic. Output is written to a result register accessible to the host.

## Datapath flow

A single inference pass proceeds as follows:

1. Host writes input data (a buffer of accelerometer samples, typically 1024–4096 samples per inference) to the input region of activation memory via the SPI/I²C interface.
2. Host writes to the inference-trigger register.
3. The control unit walks the compiled model layer-by-layer:
   - Reads weights for the current layer from weight memory
   - Reads activations from the current "read" activation bank
   - Issues MAC operations to the array
   - Receives outputs through post-processing
   - Writes activations to the current "write" activation bank
   - Swaps bank roles at the layer boundary
4. The final layer writes its output (a classification vector or anomaly score, depending on the model) to the result register.
5. The control unit asserts the completion interrupt line.
6. Host reads the result register.

End-to-end inference latency is dominated by MAC operations and activation memory access. The architecture targets sub-millisecond inference for typical condition-monitoring model sizes, leaving substantial headroom for sample-rate-bound real-time monitoring.

## Power and area envelope

Target operating power is below one watt under sustained inference workload at the target model size. Idle power, with the MAC array clock-gated and only the host interface active, is targeted at one to two orders of magnitude lower.

The architecture is designed for a mature CMOS process node appropriate for industrial applications, with emphasis on cost, yield, and thermal robustness rather than absolute performance density.

## What's deliberately out of scope

EdgeSense-1W is not a general-purpose neural accelerator. The architecture explicitly does not support:

- Floating-point arithmetic
- 2D convolution (image-domain workloads)
- Attention mechanisms or transformer architectures
- Models above the target envelope
- Runtime model graph modification
- Multi-tenant inference (one resident model at a time)

These omissions are design decisions, not gaps. Each is a deliberate choice to remove silicon area, power, and complexity that the target workload does not benefit from. A general-purpose accelerator that does all of these things at sub-1W is not currently achievable; specialisation is what makes the power envelope possible.

## Further reading

- [Memory hierarchy](memory-hierarchy.md) — detailed SRAM organisation and ping-pong scheme
- [Interface specification](interface-spec.md) — register map and host-side integration
- [Methodology](methodology.md) — reasoning behind the architectural choices
