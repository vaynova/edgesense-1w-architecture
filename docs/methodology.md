# Design Methodology

This document explains the reasoning behind EdgeSense-1W's architectural choices. It is intended for engineers and reviewers who want to understand not just *what* the architecture does, but *why* it is shaped the way it is.

The short version: EdgeSense-1W is the result of working backwards from a single workload class — vibration-based industrial condition monitoring at the network edge — and asking what silicon would look like if it were designed only for that workload.

## Defining the target workload

Before any architectural decisions can be made, the workload has to be characterised. The target deployment for EdgeSense-1W has the following shape:

- **Input signal**: time-series data from one or more accelerometers mounted on rotating machinery. Typical sample rates are 1–25 kHz. Typical inference window lengths are 1024–4096 samples, corresponding to inference rates between 1 Hz and 25 Hz.
- **Inference task**: anomaly detection or fault classification. Outputs are either a low-dimensional anomaly score, a multiclass fault classification (bearing wear, imbalance, misalignment, looseness, etc.), or both.
- **Model class**: lightweight 1D convolutional networks, occasionally with a small dense head. The relevant temporal patterns are short-range (tens to hundreds of samples) and frequency-domain features matter as much as time-domain ones, so 1D convolution with appropriate stride and dilation is the natural primitive.
- **Power budget**: less than one watt continuous, ideally less than 500 mW under typical workload, with clock-gated idle power one to two orders of magnitude lower. This budget is set by the deployment context: battery, harvested, or low-current-loop powered, often with no thermal management.
- **Latency budget**: sub-millisecond per inference is comfortable. The sample rate, not inference latency, determines the real-time bound.
- **Integration constraint**: the device is integrated as a peripheral on an existing industrial gateway, PLC, or sensor node — not as a CPU replacement. The host interface must be simple, predictable, and compatible with existing industrial bus standards.

Once the workload is pinned down, most of the architectural decisions follow from it.

## Why depthwise-separable Conv1D as the core operator

A general-purpose neural accelerator is typically built around a 2D convolution primitive (for image workloads) or a matrix multiplication primitive (for transformer workloads). Neither is the right primitive for this workload.

For 1D temporal data, 2D convolution wastes compute and memory on a dimension that doesn't carry signal. Matrix multiplication is too general — it doesn't exploit the locality structure of convolution.

Depthwise-separable Conv1D (DS-Conv1D) decomposes a standard 1D convolution into two cheaper steps:

1. A **depthwise** filter that applies one 1D kernel per input channel independently
2. A **pointwise** (1×1) convolution that mixes the channels

For typical layer geometries used in condition-monitoring networks, this decomposition reduces the number of multiply-accumulate operations by approximately one order of magnitude relative to standard Conv1D. The accuracy cost is small to negligible on the relevant datasets, because the temporal-frequency structure of vibration signals is well-captured by per-channel filters followed by linear mixing.

Committing to DS-Conv1D as the primary MAC primitive — rather than supporting standard Conv1D natively — saves silicon area and power. Standard Conv1D is supported as a composable special case (depthwise filter with channel multiplier), at slightly worse efficiency, for the small number of layers in typical models that benefit from it.

## Why eliminate DRAM from the inference path

In edge inference workloads, off-chip memory access typically dominates total energy consumption. A single DRAM access can cost an order of magnitude more energy than the MAC operation itself. Architectures that page weights or activations from DRAM during inference pay this cost on every layer.

EdgeSense-1W eliminates DRAM from the inference path entirely. All weights and intermediate activations for supported model sizes fit on-chip in SRAM. This places a hard upper bound on supported model size, which is intentional and central to the design philosophy.

The trade-off is explicit: the architecture does not scale to arbitrary model sizes. For workloads requiring larger models, EdgeSense-1W is the wrong choice. For the target workload class, large models are not required, because the underlying inference task is narrow and well-defined.

## Why ping-pong activation buffering

A naive activation memory layout reads inputs to layer N+1 from the same buffer that layer N just wrote to. This works, but it forces the MAC array to wait while layer N completes its writes before layer N+1 can start.

Ping-pong buffering uses two activation banks. While layer N+1 reads from bank A, layer N+2 writes to bank B. At the layer boundary, the bank assignments swap. The MAC array never waits at a layer transition.

The cost is a doubling of activation memory area. The benefit is sustained MAC array utilisation across the inference pass, which translates directly into lower energy per inference (because the chip spends less wall-clock time at active power for each inference).

For the model sizes in scope, the area cost is small relative to weight memory, so the trade-off favours the ping-pong scheme.

## Why a minimal host interface

Sophisticated accelerator interfaces — shared cache coherence, DMA, command queues, multi-channel interrupts — are useful for high-throughput general-purpose accelerators integrated into modern CPU platforms. They are out of place on an industrial gateway running a real-time PLC operating system or a sensor node based on a low-power microcontroller.

The host interface is restricted to dual-mode SPI/I²C with a 10-register control surface. This is the lowest common denominator of industrial peripheral integration. Any host capable of running an industrial monitoring application can drive EdgeSense-1W without bespoke driver work, kernel changes, or platform-level integration.

The cost is some latency that a richer interface would have hidden. For this workload, the latency is acceptable, and the integration simplicity is worth more than the latency saved.

## Why fixed quantised arithmetic

Floating-point silicon is large and power-hungry relative to fixed-point arithmetic. For the model class in scope, quantisation-aware training reliably produces models that meet the accuracy requirements of fault detection and classification at fixed-point precision.

EdgeSense-1W therefore implements only fixed quantised arithmetic. The Vaynova toolchain handles quantisation-aware training and model compilation; users ship floating-point training models to the toolchain and receive quantised, EdgeSense-compatible binaries in return.

This removes a significant area and power burden from the silicon, in exchange for a constraint on the toolchain.

## The general principle

Each of the design choices above follows the same logic: a workload-specific architecture pays for itself by removing silicon area, power, and complexity that a general-purpose architecture would carry but that the target workload does not benefit from. The cost is loss of generality. The benefit is a power and cost envelope that a general-purpose architecture cannot reach.

EdgeSense-1W is built on the assumption that for industrial condition monitoring at scale, the right answer is a specialised device sitting next to general-purpose compute on the host, not a more powerful general-purpose accelerator inside it.

## What this approach does not solve

Workload specialisation comes with limitations that should be stated openly:

- **Workload drift**: if the dominant inference task in the field shifts (for example, toward attention-based models for time-series, or toward 2D spectrogram-based classification), the architecture's specialisation becomes a liability.
- **Model size growth**: if useful models for this workload grow significantly beyond the on-chip SRAM envelope, the SRAM-first design becomes a constraint.
- **Multi-modal sensing**: if condition monitoring evolves to fuse vibration with thermal imaging, acoustic sensing, or current signature analysis at the inference layer, a vibration-specialised device serves only part of the pipeline.

These risks are real and are part of the long-term technical roadmap. For the current product generation, the workload specialisation is correct.

## Further reading

- [Architecture overview](architecture-overview.md) — block diagram and subsystem descriptions
- [Memory hierarchy](memory-hierarchy.md) — detailed SRAM organisation
- [Interface specification](interface-spec.md) — host-side integration
