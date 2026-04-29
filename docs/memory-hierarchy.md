# Memory Hierarchy

This document describes the on-chip memory organisation of EdgeSense-1W. The architecture is deliberately SRAM-first with no DRAM in the inference path, and this document explains how that constraint is achieved while sustaining MAC array utilisation across the inference pass.

## Memory subsystems

EdgeSense-1W contains three distinct on-chip memory subsystems, each tuned for the access pattern of its workload:

1. **Weight memory** — stores the compiled model
2. **Activation memory** — stores intermediate layer outputs during inference
3. **Scratchpad** — small fast memory for layer-fused operations

No DRAM, no off-chip memory, no caching tier between SRAM and the MAC array. The whole inference pass happens within the on-chip memory hierarchy.

## Weight memory

Weight memory is sized to hold the largest model in the architecture's target envelope. It is implemented as a single bank of on-chip SRAM, optimised for sequential read access during inference.

### Storage format

Weights are stored in a packed quantised format produced by the Vaynova compilation toolchain. The packing is optimised for the access pattern of the MAC array — specifically, the order in which weights are consumed during depthwise-separable Conv1D operations.

The layout includes:

- A fixed header containing model metadata (input geometry, output mode, layer count, CRC)
- A layer descriptor table indexing the weights of each layer
- The packed weight payload itself

Loading a model is a host-driven operation: the host streams the compiled binary blob into weight memory through the `WEIGHT_PORT` register, which auto-increments the internal address pointer. Once loaded, the model resides in SRAM until cleared.

### Capacity envelope

The weight memory capacity sets a hard upper bound on supported model size. Models exceeding this envelope cannot be deployed on EdgeSense-1W and must be either reduced (via pruning, quantisation refinement, or architectural simplification) or deployed on different hardware.

This is by design. The architecture trades scalability for efficiency. The capacity envelope is sized to accommodate the model class characterised in the [methodology document](methodology.md) — lightweight 1D CNN classifiers and anomaly detectors for vibration data.

## Activation memory

Activation memory holds intermediate layer outputs during the inference pass. It is implemented as two banks of SRAM in a ping-pong arrangement.

### Ping-pong scheme

At any point during inference, one bank is the "read bank" (the current layer reads its inputs from here) and the other is the "write bank" (the current layer writes its outputs here). At the layer boundary, the bank assignments swap. This eliminates the read-after-write hazard that would otherwise stall the MAC array between layers.

The control flow is:

```
At inference start:
  bank_read  ← bank A (loaded with input samples)
  bank_write ← bank B

For each layer:
  Read layer inputs from bank_read
  Compute MAC operations
  Write layer outputs to bank_write
  Swap: bank_read, bank_write ← bank_write, bank_read

At inference end:
  Final outputs are in bank_read.
  Post-processing reads from bank_read into the result register.
```

### Bank sizing

Each bank is sized to hold the largest single-layer activation tensor in any supported model. This is typically larger than would be needed for any individual layer — most layers' activations occupy only a fraction of the bank — but the worst-case sizing prevents architectural restrictions on which layer types can appear in compiled models.

### Why ping-pong rather than a single bank with read-modify-write

A single-bank design with read-modify-write semantics would halve the activation memory area at the cost of pipeline stalls between layers. For workloads where inference is bounded by compute throughput, those stalls translate directly into wasted active-power-time. The ping-pong scheme spends silicon area to recover that wasted time.

The trade-off favours ping-pong because activation memory is small relative to weight memory in the target envelope, so doubling activation area is a small total cost. If activations dominated the area budget, the trade-off would be different.

## Scratchpad

A small fast memory adjacent to the MAC array holds intermediate values for layer-fused operations — specifically, the intermediate result between the depthwise filter and the pointwise mixing step in a DS-Conv1D layer. Without the scratchpad, these intermediate values would need to round-trip through activation memory, doubling the bandwidth requirement on each DS-Conv1D layer.

The scratchpad is sized for the largest single intermediate tensor in any DS-Conv1D layer in the target envelope. It is not host-accessible.

## Power management

All three memory subsystems support clock-gating during idle. Weight memory remains powered (so the loaded model is preserved across idle periods) but is clock-gated. Activation memory and scratchpad can be both clock-gated and put into a low-leakage retention mode during long idle intervals.

Total idle power, with the MAC array clock-gated and only the host interface and weight retention active, is targeted at one to two orders of magnitude below the active inference power.

## What this design does not provide

For honesty and completeness:

- **No model paging**. Models that exceed the weight memory envelope cannot be deployed. There is no swap, no streaming, no compressed weights with on-the-fly decompression in the inference path.
- **No multi-model residency**. One model is resident at a time. Switching models requires a reload sequence.
- **No host-visible activation memory**. The host cannot inspect intermediate layer outputs. Only the final result register is exposed.
- **No persistent storage**. Models must be reloaded on every power-up. There is no on-chip flash.

These omissions are deliberate. Each one would add silicon area, leakage power, or complexity that the target workload does not benefit from.
