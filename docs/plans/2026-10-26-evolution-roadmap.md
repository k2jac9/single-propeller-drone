# Evolution roadmap: reproducibility, measurement and staged autonomy

**Status: proposed development program.** Session date: 26 October 2026, America/Toronto; source review continued 27 October UTC. This document proposes work and acceptance evidence. It does not describe new firmware/CAD implementation, tested hardware, an approved operating envelope or a promised flight date.

## Objective and starting constraints

Evolve the existing single-motor hinged-propeller design through attributable experiments. Preserve ArduPilot as the initial outer flight controller and moteus as the local rotor-synchronous waveform executor. Establish a reproducible reference, measure the electromechanical plant, and admit controller or autonomy changes only when they improve declared requirements under comparable conditions. The [architecture decision](../architecture/ADR-0001-evolution-strategy.md), [baseline assessment](../research/01-baseline-assessment.md), [firmware audit](../research/03-firmware-audit.md) and [hardware research](../research/04-control-and-hardware-research.md) supply the evidence and limitations behind this sequence.

Three baseline facts shape commissioning. Current inspected ArduPilot assigns MCOUT protocol **51**, while the historical parameter export and moteus guide use **50**. The saved 190 Hz command-rate request is not an achieved measurement: with its idealized 400 Hz scheduler, the inspected sender can quantize to approximately **133.3 Hz**. The one-way command path does not report actual motor saturation back through MCOUT. Resolve these as versioning and measurement tasks, rather than silently copying settings. [Configuration and timing evidence](../research/03-firmware-audit.md).

The physical target is mean lift and controlled thrust direction while the body can rotate. Independent fixed-yaw control is not assumed. Rotor azimuth, body heading, sensor acquisition time and capture arrival time remain separate quantities. [Mechanism and frame distinction](../research/01-baseline-assessment.md).

## Gate policy

Every milestone has an evidence bundle and a reviewable pass/fail decision. Before collecting acceptance data, declare the required behavior, metric, operating conditions and acceptance bound. Choose bounds from mission requirements, component limits and a measured baseline; this roadmap does not invent already-validated numerical tolerances. If the requirement is unknown, record the characterization result and keep the gate open.

An experiment can pass within a limited domain without authorizing a larger one. Replacing a rotor, print process, motor, controller, guard, payload or important filter invalidates the affected calibration until reviewed. Failed and inconclusive experiments remain deliverables when their raw evidence and limitations are preserved. No later milestone can substitute for missing evidence at an earlier prerequisite.

## Milestones and acceptance evidence

### M0: Pin a compatible, identifiable reference

**Dependencies:** none. This is the first hardware prerequisite and can largely advance without an aircraft.

Create a release manifest that independently records CAD revision, both firmware commits, board identities, toolchains/build flags, binary hashes, parameter/configuration exports, protocol version, baud/framing and assembly identifier. Preserve the original exports unchanged as historical evidence; provide a separately reviewed configuration migration for the chosen current source. Include a versioned BOM, wiring/encoder/rotor-zero conventions and provenance/permissions record. Unknown part identities stay explicitly unresolved. Public availability is not a substitute for recorded reuse permissions. [Repository and provenance assessment](../research/01-baseline-assessment.md).

**Gate:** a second person can identify the exact inputs and distinguish historical examples from the selected compatible configuration. Builds and protocol/configuration consistency are evidenced for the selected boards before hardware commissioning. Review any redistribution restrictions component by component. Source availability alone does not close the hardware gate.

### M1: Build the offline evidence tool

**Dependencies:** M0's packet/schema identity; physical hardware is unnecessary.

Implement the bounded offline MCOUT decoder/report workflow described in the [tool design](../superpowers/specs/2026-10-26-offline-evidence-tool-design.md) and [implementation plan](../superpowers/plans/2026-10-26-offline-evidence-tool.md). Rust is the proposed language, not existing software. Pin the toolchain and dependencies. Preserve raw input and expose corrupt/truncated frames, resynchronization, sequence behavior, flags and capture metadata. Separate sender timestamps from capture-clock timestamps and disclose when clocks cannot be aligned.

Use synthetic fixtures and documented source-derived records before claiming real-capture support. Test streaming boundaries, invalid frames and resource limits with independent expected outcomes. Produce machine-readable decoded data plus a concise human report; reports must expose missing data and unsupported fields.

**Gate:** reproducible tests demonstrate correct parsing, bounded behavior and truthful provenance reporting for the stated input format. Capture-derived rate is labeled observed; ideal scheduler calculations remain derived. The tool does not calculate physical latency from unrelated clocks or turn UART arrival into rotor execution time.

### M2: Identify the restrained electromechanical plant

**Dependencies:** M0 compatible hardware/configuration; M1 or an equivalently traceable capture path. Software schemas and analysis fixtures can precede apparatus availability.

Use an appropriately contained, instrumented restrained apparatus to measure mean thrust, horizontal moments, motor angle/speed, requested and actual current, supply voltage and controller timing. Record fixture compliance, sensor calibration, bandwidth and frame transformations. Separate drive response, hinge/blade response, body/fixture motion and IMU filtering. Identify gain, phase, transient response, deadband, saturation and uncertainty across a declared operating grid. Do not infer moment authority from encoder data alone. [Identification research](../research/04-control-and-hardware-research.md).

Attach material batch/conditioning, print orientation/settings, geometry, balance, hinge friction/backlash, fastener retention and accumulated cycles to each specimen. Track response drift, temperature and inspection findings across repeated runs and assemblies. Start by defining the apparatus and abort policy; increasing RPM is a separate reviewed experiment, not an automatic next step.

Separate equal-lift aerodynamic/power references from control-cost comparisons. A rigid single rotor is useful as an aerodynamic reference but cannot supply matched cyclic moments. Compare suitable hinged variants at matched lift and usable moment authority, and complete controllable aircraft at matched tasks. Record electrical energy and thermal effects; do not transfer PULSAR's published saving to this design. Service life and a qualified construction process remain unknown until measured.

**Gate:** repeatable datasets support a fitted plant with a stated domain, independent validation, uncertainty and documented saturation. The record includes a tested construction/inspection envelope rather than an unsupported fatigue-life claim. Any missing instrument, unverified frame or unexplained response drift leaves the relevant acceptance item open.

### M3: Validate an identified multirate simulation and phase candidate

**Dependencies:** M1 and an M2 identified domain for physically meaningful claims. Synthetic software integration can proceed sooner but remains unvalidated.

Construct a reduced-order model with measured actuator dynamics, gain/phase, saturation, body rotation, clock relationships and sensor filtering. Exercise the actual custom MCOUT sender/parser boundary through an explicit bridge or test harness; ordinary ArduPilot SITL alone does not reproduce the moteus/hinged-rotor plant. Record bridge versions and any emulated scheduling or transport behavior. Validate against held-out captures, not only the data used to fit the model.

Treat the [community phase-scheduling PR](https://github.com/jyjblrd/moteus/pull/1) as an opt-in hypothesis. Run destination CI on its exact clean head and intended build inputs, including configuration round trips and source/test compatibility. Existing third-party CI is useful evidence but not a substitute for that exact-tree check. First preserve fixed-phase behavior; then compare a physically calibrated schedule against it using the same specimen, conditions, metrics and raw-data path. Scheduling based on speed must not be described as automatic phase identification or transient compensation. [PR audit](../research/03-firmware-audit.md).

**Gate:** held-out measurements establish model adequacy for the declared experiments; mismatches are quantified. The phase candidate remains disabled unless its measured A/B result improves the declared objective without unacceptable saturation, thermal or stability effects. A successful build does not close the physical phase gate.

### M4: Validate restrained control and failure transitions

**Dependencies:** M2 characterized apparatus/plant and M3 comparison environment.

Establish a transparent reduced-attitude reference controller: regulate mean lift and thrust-axis direction through available moments, without independent yaw assumptions. Compare successive changes with identical initial conditions, estimator configuration, actuator limits and disturbance cases. Consider INDI only after filtering, acceleration estimates and input/output timing are credible; consider constrained S² MPC only after the model and computation budget support a meaningful comparison. [Controller research](../research/04-control-and-hardware-research.md).

Exercise startup, disarmed/interlock-off behavior, stale data, UART interruption/restoration, CPU update interruption, emergency stop, beacon loss/first acquisition and saturation. Specifically test recovery while incoming flags still indicate armed: inspected moteus fault clearing checks eligibility after disengagement, not a newly observed pilot arm edge. Record actual torque-enable/stop behavior and resolve the intended operator transition policy before further use. Test servo watchdogs separately from packet freshness. [Failure-transition evidence](../research/03-firmware-audit.md).

**Gate:** the reference and each accepted candidate meet predeclared tracking/disturbance limits within the tested domain, respect actuator/thermal constraints, and match the documented stop/recovery policy. Computation deadlines and estimator validity are measured. Restrained validation is not unrestricted-flight approval; subsequent flight work requires its own bounded protocol and evidence review.

### M5: Admit external navigation and perception conditionally

**Dependencies:** M4's demonstrated control/transition domain and a payload/power decision supported by measurements.

Begin with independent external position/heading reference data where available. Validate estimation under body rotation, translation and changing beacon bearing before relying on it for navigation feedback. Compare lightweight vision/inertial and LiDAR/inertial candidates with recordings or rotating rigs, accounting for blur, exposure/acquisition timing, sensor extrinsics, vibration, compute and measurement age. Published autonomy on another aircraft is evidence of an approach, not a compatible payload package. [Autonomy and tooling research](../research/05-autonomy-simulation-tooling.md).

Assess payload mass, center of mass, inertia, electrical draw and heat together with available lift/current/modulation headroom. Repeat affected plant identification after installation. Define estimator-loss behavior and test it before enabling autonomous trajectories. Avoid treating a local IR heading reference as a full position system or a constant-bearing beacon during arbitrary translation.

**Gate:** independent reference trajectories demonstrate the required estimator accuracy/timing, and the integrated aircraft retains declared control, power and thermal margins across the intended envelope. If no candidate fits the budget, retain external instrumentation and improve the baseline; onboard autonomy is conditional rather than mandatory.

## Prioritized backlog

| ID | Priority / milestone | Concrete deliverable | Dependency |
|---|---|---|---|
| B01 | P0 / M0 | Immutable source/build/configuration manifest and protocol-50-to-51 reconciliation | None |
| B02 | P0 / M0 | BOM, annotated wiring, coordinate conventions and permissions ledger | B01; hardware inspection where needed |
| B03 | P0 / M1 | Offline parser, corruption fixtures and provenance-preserving report | B01 |
| B04 | P0 / M1 | Capture format and clock/frame metadata contract | B01, B03 |
| B05 | P0 / M2 | Apparatus design, calibration evidence and abort/containment record | B02, B04 |
| B06 | P0 / M2 | Specimen traveler, material/hinge/balance and cycle records | B02, B05 |
| B07 | P1 / M2 | Force/moment/phase/current dataset and held-out fit report | B05, B06 |
| B08 | P1 / M2 | Matched-lift and suitable-variant control-energy comparison | B07 |
| B09 | P1 / M3 | Identified multirate simulator with custom MCOUT bridge | B03, B07 |
| B10 | P1 / M3 | Exact-clean-head phase-PR CI and measured fixed/scheduled A/B report | B07, B09 |
| B11 | P1 / M4 | Failure-transition matrix, including still-armed reconnect/fault clearing | B05, B09 |
| B12 | P1 / M4 | Reduced-attitude reference results and optional INDI/MPC comparison | B07, B09, B11 |
| B13 | P2 / M5 | Rotation/translation estimator recordings and calibration report | B04; M4 for feedback integration |
| B14 | P2 / M5 | Payload/power/headroom decision and affected re-identification | B08, B12, B13 |

P0 establishes trustworthy inputs and apparatus. P1 establishes measured improvement. P2 is conditional integration. Work can close a software deliverable while its associated hardware gate remains open.

## Required experiment record

Every experiment record must contain:

- **Hypothesis and comparison:** expected mechanism, baseline/candidate, controlled variables and requirements being tested.
- **Identity:** source commits, binary hashes, build flags, configurations, apparatus/calibration IDs and specimen/assembly history.
- **Raw evidence and provenance:** recoverable captures, clock mappings/uncertainty, frame/units/sign conventions, acquisition timing, dropped samples and saturation.
- **Declared assessment:** metric, acceptance bound chosen before acceptance collection, operating domain, repetitions, held-out validation and uncertainty treatment.
- **Abort and rollback:** observable stop conditions, operator responsibility, restoration of the known reference, and treatment of interrupted runs.
- **Result:** pass/fail/inconclusive, deviations, limitations, raw-data links and which calibration or milestone decisions changed.

## Proposed first 30 / next 30 / next 30 days

**Days 1–30:** prioritize B01–B04, source/configuration review, offline tooling and apparatus planning. Produce useful synthetic-data reports without hardware. If assemblies and calibrated instruments exist, begin M2 only after the apparatus/identity prerequisites are recorded; otherwise publish the unresolved BOM/instrument requirements.

**Days 31–60:** target B05–B09 when hardware is available: repeatable specimen records, contained characterization, held-out model validation and bridge integration. Without hardware, finish software robustness, documentation and synthetic fault cases; label all plant parameters provisional. Do not report simulated gains as measured improvements.

**Days 61–90:** conditionally evaluate B10–B12 and collect B13 recordings. Begin B14 only after the preceding budgets and gates support it. Prefer a narrow, reproducible controller or estimator comparison over simultaneous mechanical, firmware and payload changes. These are planning windows, not promised completion or flight dates; instrument access, build compatibility and failed hypotheses can change their order or duration.

The reviewable outcome is a traceable chain from source and construction to raw measurements and accepted behavior. Advancing that chain is worthwhile even when a candidate controller, material or autonomy payload is rejected.
