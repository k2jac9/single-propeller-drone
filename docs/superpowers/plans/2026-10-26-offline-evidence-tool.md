# Offline MCOUT Evidence Tool Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an offline command-stream decoder and evidence report that can expose source/configuration and timing issues without controlling hardware.

**Architecture:** A bounded streaming wire parser feeds sequence statistics and NDJSON records. A validated session manifest supplies source identity and theoretical link constraints; a CLI writes a new report directory while retaining raw data provenance.

**Tech Stack:** Rust edition 2021, standard file I/O, serde/serde_json for strict schemas, SHA-256 hashing; exact toolchain and dependency versions pinned during the implementation change.

**Spec:** [Offline evidence tool design](../specs/2026-10-26-offline-evidence-tool-design.md). This is proposed work, not an implementation completed by the research PR.

## Global Constraints

- Implement an offline CLI named `mcout-evidence` using Rust, edition 2021. Pin an exact supported toolchain and dependency lockfile in its implementation change.
- Accept regular files only. No serial, CAN, network, firmware upload, live command generation or actuator-control interface is included.
- Use explicit little-endian decoding without unsafe casts or host-layout assumptions.
- Retain at most 84 unconsumed candidate bytes in the decoder; stream records and accumulate bounded counters. Do not load the capture or decoded records into memory in full.
- Limit the manifest to 65,536 bytes before parsing. Reject an existing output path and never overwrite an input or previous report.
- Preserve raw bytes, unknown flag bits, raw spool-state values and IEEE-754 bits. A non-finite float is an observed anomaly, not permission to replace the measurement silently.
- Separate observed data, derived estimates and unavailable metrics. An analysis report never asserts aircraft readiness or verified physical compatibility.

## Review Focus

- Overlapping magic and truncated/corrupt candidates: recover the next complete valid frame without unbounded buffering. Task 1.
- Capture resets, sequence wrap and duplicates: do not turn a reboot into billions of apparently lost packets. Task 2.
- NaN/infinity and unknown enum bits: retain evidence and emit valid JSON rather than silently coercing data. Task 3.
- Unknown firmware SHA or scheduler assumptions: report limited knowledge rather than fabricated compatibility/latency. Task 4.
- Existing paths, giant input, malformed manifest or interrupted output: preserve input and distinguish incomplete reports. Task 5.

---

## File structure and shared types

Create one crate under `tools/mcout-evidence/`. Keep its lockfile and toolchain local to that crate so this CAD repository does not acquire a misleading flight-firmware build command.

| Path relative to crate | Responsibility |
|---|---|
| `Cargo.toml`, `Cargo.lock`, `rust-toolchain.toml` | Reproducible tooling and dependency selection |
| `src/lib.rs` | Export the types/modules below |
| `src/wire.rs` | Fixed wire constants, CRC, decoder and raw frame representation |
| `src/stats.rs` | Epoch-aware source/sequence statistics |
| `src/record.rs` | JSON-safe raw/decoded frame representation |
| `src/manifest.rs` | Strict bounded input schema and static compatibility table |
| `src/report.rs` | Summary schema, theoretical budgets and streamed report writing |
| `src/main.rs` | Argument validation, regular-file checks and exit behavior |
| `tests/wire.rs`, `tests/stats.rs`, `tests/record.rs`, `tests/manifest.rs`, `tests/cli.rs` | Independent behavioral verification |
| `tests/fixtures/golden.hex` | Exact synthetic reference from the spec |
| `README.md` | Usage, schema, evidence limits and validation commands |

Use `Frame { offset: u64, raw: [u8; 84], sequence: u32, timestamp_usec: u64, flags: u16, spool_state: u8, desired_spool_state: u8, float_bits: [u32; 15] }`. Preserve the float field order from the spec. `ProtocolStats` has u64 counters named exactly as the spec's parser counters. Task 1 defines shared `AnalysisError` variants `CounterOverflow` and `ByteOffsetOverflow`; analysis stops on either and the CLI reports exit 2 with incomplete-output cleanup. Each implementation task owns the public type definitions it produces; later tasks consume them.

### Task 1: Decode the fixed wire contract

**Files:** Create crate metadata, `src/lib.rs`, `src/wire.rs`, `tests/wire.rs`, fixture and pinned toolchain/lockfile.

**Interfaces:** Produce `crc16_xmodem(bytes: &[u8]) -> u16`, `Decoder::new() -> Decoder`, `Decoder::feed(&mut self, byte: u8) -> Result<Option<Frame>, AnalysisError>`, `Decoder::finish(&mut self) -> Result<(), AnalysisError>`, and `Decoder::stats(&self) -> &ProtocolStats`. Decoder tracks the absolute byte position internally. `finish` finalizes trailing-byte accounting without emitting a partial frame. Define `AnalysisError` in `src/lib.rs`; use module-local tests to place counters/offsets near their limits and assert explicit errors rather than wraparound.

- [ ] Write `golden_decodes_independently`: read the spec's hex literal, assert 84 bytes, CRC `0x04E2`, sequence 1, timestamp 1000, flags 71, spool 3/2 and expected f32 bit representations. Assert CRC of `123456789` is `0x31C3`.
- [ ] Write `recovers_overlapping_candidates_and_truncation`: prefix noise and a false valid-looking header, corrupt a frame, append a valid frame, and test every split position; only the valid reference is emitted. EOF incomplete candidates appear only in diagnostics. Add unsupported-version/length/message-ID cases.
- [ ] Run `cargo test --test wire`; confirm these tests fail because the implementation is absent.
- [ ] Implement the public interface using an 84-byte sliding candidate buffer with one-byte rescan after invalid candidates. Keep diagnostic semantics distinct from the embedded parser.
- [ ] Run `cargo test --locked --test wire`; require all cases pass and a repeated synthetic stream does not grow decoder storage. Commit the decoder and its evidence.

### Task 2: Classify source sequence and time

**Files:** Create `src/stats.rs`, `tests/stats.rs`; export types in `src/lib.rs`.

**Interfaces:** Consume `Frame` and `AnalysisError`. Produce `StreamStats::new() -> StreamStats`, `StreamStats::observe(&mut self, frame: &Frame) -> Result<SequenceEvent, AnalysisError>`, and `StreamStats::summary(&self) -> StreamSummary`. `SequenceEvent` variants are `EpochStart`, `Contiguous`, `Duplicate`, `ForwardGap { estimate: u32 }`, `ResetOrReorder`, `TimestampReset`. Keep current-epoch state plus bounded aggregate statistics; do not retain an epoch vector.

- [ ] Write tests asserting `0xFFFFFFFF → 0` is contiguous; `7 → 7` is duplicate; `7 → 10` estimates two missing positions; `1000 → 1` is reset/reorder; decreasing timestamp starts a new epoch before sequence interpretation.
- [ ] Add equal-timestamp, timestamp-reset-with-forward-sequence and near-half-range delta boundaries. Check duplicate intervals are excluded, zero intervals retained and no transport latency metric is produced. Exercise many epochs and near-overflow aggregate counters; require bounded storage and explicit `AnalysisError`.
- [ ] Run `cargo test --test stats`; confirm expected failures before implementation.
- [ ] Implement the spec's wrapping-delta classification and bounded running source-interval aggregates with checked counters.
- [ ] Run `cargo test --locked --test stats`; require all classifications/aggregates pass, then commit this analysis unit.

### Task 3: Preserve raw evidence in valid JSON

**Files:** Create `src/record.rs`, `tests/record.rs`; export public record type.

**Interfaces:** Consume `Frame` and `SequenceEvent`. Produce `FrameRecord::from_frame(frame: &Frame, event: SequenceEvent) -> FrameRecord` with serde serialization. Record fields include raw hex, raw masks/states, named values, float bit patterns and an anomaly list.

- [ ] Write tests injecting NaN bits `0x7FC00001`, positive infinity `0x7F800000` and negative infinity `0xFF800000` into CRC-valid synthetic frames. Assert numeric JSON values are null, raw bits/hex are exact and named anomalies exist.
- [ ] Test actual spool 2 is named spooling-up while desired spool 2 is throttle-unlimited. Preserve unknown state 255, flags `0xC000`, and sender pilot-passthrough bit `0x2000` without inventing actuator state.
- [ ] Run `cargo test --test record`; confirm failures, then implement record conversion and named masks without mutating the source frame.
- [ ] Run `cargo test --locked --test record`; parse every result with a strict JSON parser and require successful round trips of raw evidence, then commit.

### Task 4: Validate session identity and report timing limits

**Files:** Create `src/manifest.rs`, initial `src/report.rs`, `tests/manifest.rs`.

**Interfaces:** Produce `SessionManifest::parse(bytes: &[u8]) -> Result<SessionManifest, ManifestError>`, `assess_compatibility(session: &SessionManifest) -> CompatibilityAssessment`, and `theoretical_budget(session: &SessionManifest) -> BudgetReport`. Use the exact fields, bounds, enum strings and known commits in the spec. No online source lookup occurs at runtime.

- [ ] Write tests for the known current SHA with serial IDs 51/50, the known older SHA with ID50, and unknown SHAs; assert only serial-ID knowledge, never pair qualification. Test unknown fields, 65,537-byte input, nonhex/short commits, fractional/zero/out-of-range requested rates and unsupported schema versions.
- [ ] Write the 190/400/115200 budget test: 159600 requested bit/s, approximately 137.142857 maximum packets/s, 133.333333 ideal scheduler Hz and 112000 ideal bit/s. Add requested 101 Hz / scheduler 10000 Hz: period 9900 µs, 99 steps and approximately 101.0101 Hz. Assert the explanation does not label these rates measured or assert sustained overload. An unknown sender has no scheduler-policy estimate.
- [ ] Run `cargo test --test manifest`; confirm failures. Implement strict schema parsing, cited static compatibility and current sender's integer-microsecond scheduling approximation.
- [ ] Run `cargo test --locked --test manifest`; require all cases pass, then commit.

### Task 5: Deliver the streaming CLI and reproducible report

**Files:** Create `src/main.rs`, `tests/cli.rs`, crate README; complete `src/report.rs`.

**Interfaces:** Consume the preceding interfaces. Produce `inspect(session: &SessionManifest, input: &Path, output: &Path) -> Result<AnalysisOutcome, ReportError>`; `AnalysisOutcome` exposes `has_anomalies: bool`. Wrap `AnalysisError` in `ReportError` and map it to exit 2. `summary.json` includes schema version, input hash/byte count, manifest identity, protocol and stream summaries, compatibility, budgets, fixed-category anomaly counters, at most 16 bounded exemplars and explicit null/reasons for unrecorded arrival-clock metrics.

- [ ] Write CLI tests in temporary directories for the spec's invocation, golden capture, empty/no-valid-frame capture, corruption and incompatible manifest. Assert exit 0/3/2 as specified, newline-delimited valid records, SHA-256 of the unchanged input and exact required output names.
- [ ] Test an already-existing output directory, output equal to input, nonregular input, oversized manifest and a writer that fails after partial output. Assert input preservation and an incomplete-report error. Test large streams containing many timestamp resets and non-finite frames as well as clean frames; generate them incrementally, assert no growing epoch/anomaly collection and at most 16 summary exemplars. Inject checked-arithmetic failure and require exit 2 with incomplete-output cleanup.
- [ ] Run `cargo test --test cli`; confirm failures. Implement regular-file/canonical-path checks, create-new output handling, buffered streaming, input hashing, summary finalization and cleanup/error behavior.
- [ ] Run `cargo test --locked`, `cargo fmt --check` and `cargo clippy --locked --all-targets -- -D warnings`. Record the exact toolchain and outcomes. A passing report is evidence about the offline tool, not the aircraft.
- [ ] Document a synthetic example and the distinct clocks, parser diagnostics and compatibility limits. Commit the complete tool and its usage evidence; request review of the implementation change before any live-capture extension.

## Plan self-review and boundary

The five tasks cover the spec's offline scope, wire format, raw-data preservation, strict input bounds, bounded parsing, sequence/time ambiguity, compatibility, timing arithmetic and output failure semantics. Types above are shared consistently. Arrival timestamps, MCAP conversion, plots, live capture, firmware patches and hardware experiments are intentionally later slices with separate specifications. The implementer should read both this plan and the linked design before beginning; this research contribution leaves all implementation checkboxes unchecked.
