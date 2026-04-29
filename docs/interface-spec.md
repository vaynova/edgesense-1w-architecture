# Interface Specification

This document specifies the host-side integration of EdgeSense-1W. It defines the register map, command set, and bus timing requirements for both supported interface modes.

This is a public interface contract. Vaynova reserves the right to extend the register map in future revisions in a backward-compatible manner, but will not change the meaning of any register defined here.

## Bus modes

EdgeSense-1W implements a dual-mode peripheral interface. The mode is selected at power-up by sampling the `MODE_SEL` strap pin:

| MODE_SEL | Mode |
|---|---|
| 0 | I²C slave |
| 1 | SPI slave (mode 0) |

Mode selection is latched at reset and cannot be changed during operation. Both modes expose an identical register-based control surface; only the underlying bus transport differs.

### I²C mode

- Standard 7-bit slave addressing
- Default device address: `0x42` (configurable via `ADDR_SEL[1:0]` strap pins, see hardware datasheet)
- Supported speeds: 100 kHz (Standard-mode), 400 kHz (Fast-mode), 1 MHz (Fast-mode Plus)
- Register access uses standard repeated-start sequences

### SPI mode

- SPI mode 0 (CPOL = 0, CPHA = 0)
- MSB-first bit order
- Maximum clock rate: 20 MHz
- Chip-select active low
- Register access uses a 1-byte command + register-address protocol (see Command Set)

## Register map

The register map exposes 10 registers. Reserved registers are unused in the current revision and read as zero; writes to reserved registers are ignored.

| Address | Name | Access | Width | Description |
|---|---|---|---|---|
| `0x00` | `DEVICE_ID` | R | 16 | Fixed identifier `0xE51W`. Used by the host to confirm device presence. |
| `0x01` | `STATUS` | R | 16 | Device status flags. See [Status Flags](#status-flags). |
| `0x02` | `CONTROL` | R/W | 16 | Operational control. See [Control Bits](#control-bits). |
| `0x03` | `MODEL_CFG` | R/W | 16 | Model configuration: input length, channel count, output mode. See [Model Configuration](#model-configuration). |
| `0x04` | `INPUT_PTR` | R/W | 16 | Write pointer for streaming input samples to activation memory. |
| `0x05` | `INPUT_DATA` | W | 16 | Write port for input samples. Auto-increments `INPUT_PTR`. |
| `0x06` | `TRIGGER` | W | 16 | Inference trigger. See [Trigger Commands](#trigger-commands). |
| `0x07` | `RESULT` | R | 16 | Inference result. Format depends on `MODEL_CFG.output_mode`. |
| `0x08` | `WEIGHT_PORT` | R/W | 16 | Weight memory access port (for compiled-model loading). |
| `0x09` | `WEIGHT_ADDR` | R/W | 16 | Weight memory address pointer. Auto-increments on `WEIGHT_PORT` access. |

### Status flags

`STATUS` register bits (read-only, cleared automatically on completion or on relevant `CONTROL` write):

| Bit | Name | Description |
|---|---|---|
| 0 | `READY` | Device is ready to accept inference triggers. |
| 1 | `INFERENCE_BUSY` | Inference is currently in progress. |
| 2 | `RESULT_VALID` | Result register holds a valid result from the most recent inference. |
| 3 | `MODEL_LOADED` | A valid compiled model is present in weight memory. |
| 4 | `INPUT_READY` | Input buffer holds enough samples for inference per `MODEL_CFG`. |
| 5 | `ERROR` | An error condition occurred. See `CONTROL.error_code`. |
| 6 | `OVERRUN` | Input write attempted past end of input buffer. |
| 7 | `MODEL_CRC_OK` | Loaded model passed CRC validation at load time. |
| 8–15 | Reserved | Read as zero. |

### Control bits

`CONTROL` register bits (read/write):

| Bit | Name | Description |
|---|---|---|
| 0 | `ENABLE` | Master enable. Clearing this bit places the device in a low-power idle state. |
| 1 | `RESET` | Soft reset. Self-clearing. Returns device to power-on state without losing loaded model. |
| 2 | `MODEL_RESET` | Clears loaded model and returns to bare-firmware state. |
| 3 | `INT_ENABLE` | Enables completion interrupt on the `IRQ` pin. |
| 4–7 | `error_code` | Error detail (read-only when `STATUS.ERROR` is set). |
| 8–15 | Reserved | Write as zero. |

### Model configuration

`MODEL_CFG` register fields:

| Bits | Name | Description |
|---|---|---|
| 0–3 | `input_channels` | Number of input channels (1–8). |
| 4–15 | `input_length` | Number of samples per inference window (must match compiled model). |

The output mode is determined by the compiled model and is not host-configurable.

### Trigger commands

Write the following values to `TRIGGER` to initiate device actions:

| Value | Action |
|---|---|
| `0x0001` | Start inference using current input buffer. |
| `0x0002` | Reset input pointer to start of buffer. |
| `0x0003` | Begin compiled-model load sequence. Subsequent writes to `WEIGHT_PORT` populate weight memory. |
| `0x0004` | Finalise model load. Triggers CRC validation. |

All other values are reserved.

## Typical inference sequence

A complete inference cycle from the host's perspective:

```
1. Read DEVICE_ID, confirm value is 0xE51W.
2. (One-time per power-up) Load compiled model:
   a. Write 0x0003 to TRIGGER.
   b. Stream model bytes to WEIGHT_PORT (auto-increments WEIGHT_ADDR).
   c. Write 0x0004 to TRIGGER.
   d. Wait for STATUS.MODEL_LOADED and STATUS.MODEL_CRC_OK.
3. Configure MODEL_CFG to match the compiled model's input geometry.
4. (Per inference)
   a. Write 0x0002 to TRIGGER (reset input pointer).
   b. Stream input samples to INPUT_DATA (auto-increments INPUT_PTR).
   c. Wait for STATUS.INPUT_READY.
   d. Write 0x0001 to TRIGGER (start inference).
   e. Wait for STATUS.RESULT_VALID, or for IRQ if INT_ENABLE is set.
   f. Read RESULT.
```

End-to-end inference, exclusive of input streaming time, is targeted at sub-millisecond for typical model sizes.

## Timing diagrams

Timing diagrams for SPI mode 0 transactions and I²C standard register-read/register-write sequences are provided in the [diagrams/](../diagrams/) directory.

## Error handling

When `STATUS.ERROR` is asserted, the host should:

1. Read `CONTROL.error_code` to determine the cause
2. Address the underlying condition (typically by clearing `MODEL_RESET` or asserting `RESET` and reloading the model)
3. Confirm `STATUS.ERROR` clears

Error codes are documented in the full hardware datasheet (available under NDA from Vaynova).

## Backward compatibility commitment

Future revisions of EdgeSense will preserve the meaning of all registers and bits defined in this specification. New functionality will be exposed via currently-reserved register space or via newly-allocated registers above `0x09`. The `DEVICE_ID` register will change to reflect the new revision; hosts should check the device ID at startup and adjust their behaviour accordingly.
