# EdgeSense-1W

**A sub-watt inference co-processor for vibration-based industrial condition monitoring.**

EdgeSense-1W is a domain-specific silicon architecture designed to run always-on anomaly detection on vibration-instrumented industrial assets — bearings, motors, pumps, gearboxes, compressors — at a power envelope below one watt. This repository contains the public architecture overview, interface specification, design methodology, and reference diagrams for the chip.

The full implementation (RTL, synthesised netlists, model weights, and silicon-level documentation) is held privately as Vaynova IP. This repository is intended to make the architectural reasoning, interface contract, and design philosophy openly available to the engineering community.

---

## Why this exists

Industrial condition monitoring at scale has a hardware problem. The sensors are mature. The algorithms are mature. The deployment bottleneck is that running a competent anomaly-detection model on a vibrating piece of rotating machinery, in a remote location, on harvested or battery power, with thermal headroom, is hard. General-purpose microcontrollers running TinyML frameworks burn too much power for true always-on operation. General-purpose neural accelerators are over-provisioned for what's actually a narrow inference workload, dominated by 1D temporal convolutions on accelerometer data.

EdgeSense-1W is the result of asking what the silicon would look like if it were designed *only* for this workload class.

## Key design decisions

### Depthwise-separable 1D convolution as the core operator

Vibration-based condition monitoring is fundamentally a 1D temporal-frequency problem. Standard 2D convolutional architectures, ported from image-domain TinyML, waste compute and memory on dimensions that don't carry signal. Depthwise-separable Conv1D (DS-Conv1D) decomposes a standard convolution into a depthwise filter and a pointwise mixing step, reducing multiply-accumulate operations by approximately one order of magnitude for typical layer geometries — without measurable loss of detection accuracy on bearing-fault and imbalance datasets.

The architecture commits to DS-Conv1D as the primary MAC primitive. Standard Conv1D, dense, and pointwise operations are supported as composable subsets.

### SRAM-first memory hierarchy

DRAM access dominates energy consumption in edge inference workloads, often by an order of magnitude over compute. EdgeSense-1W eliminates DRAM from the inference path entirely. All weights and intermediate activations for supported model sizes fit within on-chip SRAM, organised as a tiered hierarchy of weight memory, activation buffers, and a small scratchpad for layer-fused operations.

This places a hard upper bound on supported model size, which is intentional. The architecture is designed for a target model envelope appropriate to its workload, not for arbitrary scalability.

### Ping-pong activation buffering

Activations between consecutive layers are double-buffered. While layer N+1 reads from buffer A, layer N+2 writes to buffer B, then they swap. This eliminates pipeline stalls between layers and allows the MAC array to run at near-peak utilisation across the inference pass, rather than idling during memory reorganisation.

### Minimal host interface

The host interface is deliberately minimal: a 10-register SPI/I\u00b2C control surface sufficient to load weights, trigger inference, read status, and retrieve output classifications. Hosts integrating EdgeSense-1W treat it as a peripheral, not a co-processor requiring shared memory or cache coherence. This makes integration with existing PLC, SCADA, and industrial gateway hardware straightforward.

## What's in this repository

- [`docs/architecture-overview.md`](docs/architecture-overview.md) — top-level block diagram and datapath description
- [`docs/memory-hierarchy.md`](docs/memory-hierarchy.md) — SRAM organisation, weight layout, activation buffering scheme
- [`docs/interface-spec.md`](docs/interface-spec.md) — register map, command set, SPI/I\u00b2C timing
- [`docs/methodology.md`](docs/methodology.md) — design reasoning, target workload definition, performance and power envelope
- [`docs/glossary.md`](docs/glossary.md) — terminology used across the documentation
- [`diagrams/`](diagrams/) — block diagrams, datapath diagrams, interface timing illustrations

## Status

EdgeSense-1W is at TRL 3 (analytical and experimental proof-of-concept). Active development is funded by founder time and pursuing external grant funding. The project is targeting TRL 5 (validation in relevant environment) within a 12-month development programme.

## Contributing and contact

This repository is documentation-only and is not currently accepting external contributions to the architecture itself. Issues raising questions about the public design, requests for clarification, or pointers to relevant prior art are welcome.

For technical or commercial enquiries, please contact Karan Patel at karan@vaynova.uk.

## Licence

The documentation in this repository is published under [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE). You are free to share and adapt the material with attribution.

The EdgeSense-1W silicon implementation, model weights, and proprietary tooling remain the intellectual property of Vaynova Ltd and are not covered by this licence.

---

*Vaynova Ltd · Company No. 16569408 · United Kingdom*
