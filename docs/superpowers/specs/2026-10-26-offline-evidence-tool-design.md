# Offline MCOUT evidence tool: proposed design

**Status:** Proposed first implementation slice; this documentation change does not implement it.  
**Purpose:** Make a recorded MCOUT stream inspectable and relate it to exact firmware/configuration identities before introducing live instrumentation or controller changes.  
**Parent decision:** [ADR-0001](../../architecture/ADR-0001-evolution-strategy.md).

## 1. Scope and constraints

- Implement an offline CLI named `mcout-evidence` using Rust, edition 2021. Pin an exact supported toolchain and dependency lockfile in its implementation change.
- Accept regular files only. No serial, CAN, network, firmware upload, live command generation or actuator-control interface is included.
- Use explicit little-endian decoding without unsafe casts or host-layout assumptions.
- Retain at most 84 unconsumed candidate bytes in the decoder; stream records and accumulate bounded counters. Do not load the capture or decoded records into memory in full.
- Limit the manifest to 65,536 bytes before parsing. Reject an existing output path and never overwrite an input or previous report.
- Preserve raw bytes, unknown flag bits, raw spool-state values and IEEE-754 bits. A non-finite float is an observed anomaly, not permission to replace the measurement silently.
- Separate observed data, derived estimates and unavailable metrics. An analysis report never asserts aircraft readiness or verified physical compatibility.

Rust is a proposed tooling choice, not a requirement to migrate either firmware. The wire contract comes from the [pinned sender header](https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/libraries/AP_MotorControlOutput/AP_MotorControlOutput.h) and [pinned receiver](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/fw/mcout_protocol.h). This is a custom protocol; a MAVLink decoder does not parse it.

## 2. CLI and input contract

```text
mcout-evidence inspect --manifest session.json --input capture.bin --output report-directory
```

Create a new directory containing `summary.json` and `frames.ndjson`. Stream raw frame hex and decoded values into NDJSON; include the input capture's SHA-256 in the summary. Keep the original capture separately, unchanged. On an I/O error after creation, remove incomplete outputs where possible and issue a clear incomplete-report error. Do not call a partial report complete.

The proposed session manifest has these required fields:

```json
{
  "schema_version": 1,
  "capture_id": "example-synthetic",
  "autopilot_commit": "455b266cc2a73505eb98c56a8049fa6a4d50f9bf",
  "moteus_commit": "704a0dd9f30a3704e8915a82c1631301fdbff9bd",
  "mcout_wire_version": 1,
  "serial_protocol_id": 51,
  "baud": 115200,
  "requested_rate_hz": 190,
  "scheduler_rate_hz": 400
}
```

This is an illustrative software input, not a recommended flight setup. Commit fields are exactly 40 hexadecimal characters; `capture_id` is 1–128 UTF-8 bytes; baud is an integer from 1 to 4,000,000; requested rate is an integer from 1 through 400, matching the sender's effective integer parameter range; scheduler rate is either null or a finite number in `(0, 10000]`. Calculator bounds do not assert that a physical board supports every accepted value. Unknown manifest fields are rejected to expose misspellings. `schema_version` and wire version must be 1. The checked-in research baseline manifest has a different purpose and is not this CLI's session schema.

No arrival-time sidecar is included in this first slice. Therefore transport latency, receiver update rate and arrival jitter are always `null` with reason `arrival_clock_not_recorded`. Sender timestamps permit source-interval summaries within monotonic epochs, not an end-to-end latency claim.

Exit codes: `0` means analysis completed without report-level anomalies; `3` means analysis completed with data/configuration anomalies; `2` means invalid invocation, invalid manifest, I/O failure or checked-arithmetic overflow. Overflow is a fatal analysis error with a specific diagnostic; discard incomplete output rather than saturating counters. No-valid-frame input is an anomaly, including an empty input. Neither 0 nor 3 is an actuator enable decision.

## 3. Wire contract

The fixed packet is 84 bytes; CRC-16/XMODEM uses polynomial `0x1021`, initial value 0, no reflection and no final XOR, over bytes 0–81. The stored CRC is little-endian at 82–83. The standard byte string `123456789` has CRC `0x31C3`.

| Offset | Type | Field |
|---|---|---|
| 0 | u16 | Magic `0xC04D`, wire bytes `4D C0` |
| 2 | u8 | Version 1 |
| 3 | u8 | Total length 84 |
| 4 | u16 | Message ID 1 |
| 6 | u32 | Sequence |
| 10 | u64 | Sender timestamp, microseconds |
| 18 | u16 | Flags, raw mask preserved |
| 20 | u8 | Actual spool state |
| 21 | u8 | Desired spool state, a different enum |
| 22–81 | 15 × f32 | Fields listed below, in order |
| 82 | u16 | CRC |

Float order: `roll`, `pitch`, `yaw`, `throttle`, `roll_feedback`, `pitch_feedback`, `yaw_feedback`, `roll_feedforward`, `pitch_feedforward`, `yaw_feedforward`, `throttle_filtered`, `throttle_out`, `forward`, `lateral`, `dt`.

Actual spool states 0–4 mean shutdown, ground idle, spooling up, throttle unlimited, spooling down. Desired states 0–2 mean shutdown, ground idle, throttle unlimited. In particular, actual state 2 and desired state 2 have different meanings. Preserve other values and report them as unknown. Bits 0–13 of the sender flags have names in the pinned sender header, including bit 13 for pilot passthrough; preserve bits 14–15 as unknown. A report may describe flag combinations, but must not infer actual torque or freshness from flags alone.

Scan for the header. Reject unsupported version/length/message ID and CRC mismatch while preserving overlapping candidates by advancing one byte and rescanning. Only a fully valid header and CRC produces a decoded frame. At EOF, report an incomplete candidate as `truncated_tail_bytes`; count unrelated bytes separately. This offline forensic resynchronization is deliberately more exhaustive than the embedded parser and must not be represented as an exact emulation of moteus parser statistics.

## 4. Records and statistics

`frames.ndjson` contains byte offset, 168-character `raw_hex`, raw sequence/timestamp/flags/states, named known flags, unknown flag mask, float values and their 32-bit raw bit patterns. For a NaN or infinity, emit JSON `null` for the numeric value and an anomaly with field name and raw bits. Do not reject an otherwise CRC-valid frame or emit nonstandard JSON NaN tokens.

Count valid frames, CRC-rejected candidates, header-rejected candidates, discarded bytes, truncated tail bytes and non-finite fields. Candidate counts are parser diagnostics; they are not measurements of UART bit-error rates. Keep counters and absolute offsets as u64 and check arithmetic overflow. The summary stores counters by a fixed finite anomaly category set and at most 16 total exemplars, each containing only bounded field names/codes, raw numeric values and an offset. Stream per-frame anomaly details to NDJSON. Keep only the current epoch and aggregate statistics, never an unbounded list of epochs or anomalies.

For sequence analysis within an epoch, compute `delta = current.wrapping_sub(previous)`:

- First frame starts an epoch. A backwards sender timestamp starts a new epoch before interpreting sequence.
- Delta 0 is a duplicate; delta 1 is contiguous, including `0xFFFFFFFF → 0`.
- Delta from 2 through `0x7FFFFFFF` produces a `forward_gap_estimate` of delta minus one.
- Delta at least `0x80000000` is `reset_or_reorder`; start a new epoch and do not add a loss estimate.

These classifications cannot distinguish every reboot, capture omission and reorder. Report the ambiguity and never rename a forward-gap estimate “confirmed link packet loss.” Duplicate frames do not contribute source-interval samples. Timestamp equality is allowed; record zero intervals explicitly. Use bounded running count/min/max/mean statistics within monotonic epochs. Do not calculate percentiles by silently retaining all samples.

## 5. Compatibility and theoretical timing report

For the exact current ArduPilot SHA in the example, known MCOUT serial ID is 51. For exact `new-features` SHA `7cbdd13b67fc3d9707ad3839fb4bf031875d60f9`, it is 50. The static table must cite the corresponding enum source. A mismatched ID is `mismatch`; a match is only `known_serial_id_only`; any other SHA is `unknown`. No matching case means the firmware pair has been physically validated.

Report `requested_wire_bit_rate = 84 × 10 × requested_rate_hz` and `theoretical_packet_ceiling_hz = baud / 840`. For the current sender's known scheduler policy only, optionally report an idealized rate: if requested rate is at least scheduler rate, use scheduler rate; otherwise calculate `period_us = floor(1000000 / requested_rate_hz)`, `scheduler_steps = ceil(period_us * scheduler_rate_hz / 1000000)`, then `ideal_rate_hz = scheduler_rate_hz / scheduler_steps`. This includes the source's integer period but idealizes scheduler timing; exclude jitter, UART blocking and frame skips explicitly. Unknown sender SHA means no scheduler-policy estimate. A numerical boundary test at requested 101 Hz and scheduler 10000 Hz must give a 9900 µs period, 99 steps and approximately 101.0101 Hz; a ratio-only ceiling would incorrectly give 100 Hz.

The 190 Hz / 400 Hz / 115200 example must report requested demand 159600 bit/s, theoretical ceiling approximately 137.142857 Hz, ideal scheduler result approximately 133.333333 Hz and ideal demand approximately 112000 bit/s. The finding is `requested_rate_exceeds_wire_capacity`; its explanation must acknowledge that scheduler quantization may allow the actual stream to fit. It must not assert observed overload.

## 6. Golden fixture and completion criteria

The following 84-byte reference was independently constructed using Python's explicit little-endian `struct.pack_into` and `binascii.crc_hqx(..., 0)` during this research:

```text
4dc00154010001000000e803000000000000470003020000803e000000bf000000009a99193f000000000000000000000000000000000000000000000000000000000000000000000000000000000ad7233be204
```

Expected: sequence 1, timestamp 1000 µs, flags 71, actual spool 3, desired spool 2, roll 0.25, pitch -0.5, yaw 0, throttle approximately 0.6, dt approximately 0.0025, other floats zero; CRC `0x04E2`. Floating comparisons use the represented f32 values or explicit tolerance, not exact decimal equality for 0.6 or 0.0025. This fixture is synthetic, not a flight recording.

Completion requires independent golden-vector decoding, recovery after noise/corruption at all chunk boundaries, correct wrap/reset classification, standards-compliant output for non-finite values, bounded capture processing, exact compatibility/timing qualifications and refusal to overwrite existing artifacts. Document `cargo test --locked`, formatting and lint results for the pinned toolchain. A real capture can be added later with provenance; do not fabricate one to satisfy a demonstration.
