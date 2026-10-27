# Research handoff: single-propeller drone

**Session label:** 26 October 2026; source/API observations completed on 27 October UTC.  
**Development repository:** [k2jac9/single-propeller-drone](https://github.com/k2jac9/single-propeller-drone).  
**Entry point:** [Research guide](../research/README.md).

## Objective and completed research

The requested work was to inspect the fork, original branches and other forks with actual changes, then research and document a development direction. The documentation covers the full observed CAD fork count, original firmware branch inventories, verified direct firmware descendants, relevant contributed code, primary control/hardware literature, simulation/perception tooling, and staged implementation/experimental work.

The pre-documentation baseline is CAD `efc9176bfc83777b870b967c1f62a686c099be2c`, ArduPilot `455b266cc2a73505eb98c56a8049fa6a4d50f9bf`, and moteus `704a0dd9f30a3704e8915a82c1631301fdbff9bd`. These identities describe reviewed source, not a flight-tested combination. The [baseline manifest](../../research-data/baseline-manifest-2026-10-26.json) retains provenance and explicit validation limits.

All 91 verified CAD forks initially exposed the same sole main commit. The subsequent documentation branch in this development repository is outside that recorded network snapshot. The firmware review found eight ArduPilot descendants and ten moteus descendants, with unique returned work only in two `metalogiclabs/moteus` feature branches. Exact discovery limitations and comparison evidence are in the linked research ledgers.

## Decisions proposed for review

The [architecture decision](../architecture/ADR-0001-evolution-strategy.md) recommends preserving the ArduPilot/moteus execution boundary, implementing offline evidence tooling, identifying the plant, building a causal multirate simulator and then evaluating control/navigation improvements. Rust is proposed for offline tools; Python remains suitable for scientific analysis. Neither firmware is being rewritten by this contribution.

Treat the phase-scheduling PR as an experimental candidate. Its three contribution blobs match those built by external CI, but the exact clean PR tree still needs destination CI and physical calibration. A legacy yaw-beacon branch is an alternative to the current IR integration, not a set of additive improvements.

## Facts that the next implementation must preserve

- Current ArduPilot MCOUT serial ID is 51; historical exports and the older feature branch use 50. Resolve source/configuration identity explicitly.
- MCOUT is an 84-byte custom little-endian packet with CRC-16/XMODEM. It is separate from MAVLink. Desired and actual spool-state enums differ.
- The saved 190 Hz request exceeds 115200-baud capacity at that actual rate, but an ideal 400 Hz sender scheduler quantizes it to approximately 133.3 Hz. Delivered rate and latency remain unmeasured.
- A stale-link recovery with armed flags still present can interact with fault clearing and re-engagement. Specify and test the intended transition policy before changing it.
- Rotor azimuth relative to the body and body heading relative to the world are separate quantities. The current mapping does not supply independent yaw control.
- Encoder/RPM traces alone do not identify useful cyclic moment. Force/moment, current, phase, timing and mechanical repeatability are required.

Sources and detailed qualifications: [baseline](../research/01-baseline-assessment.md), [firmware audit](../research/03-firmware-audit.md), [control research](../research/04-control-and-hardware-research.md).

## Next bounded software task

Implement the [offline tool specification](../superpowers/specs/2026-10-26-offline-evidence-tool-design.md) using its [five-task implementation plan](../superpowers/plans/2026-10-26-offline-evidence-tool.md). It reads files, preserves raw evidence, reports protocol/configuration/timing limitations, and has no hardware-control interface. The synthetic golden fixture is explicitly labeled; real logs should be added only with acquisition provenance.

Use a separate implementation change. Pin the exact Rust toolchain and dependencies, run the specified behavioral checks, and retain the evidence distinctions in CLI output. No implementation checkbox in this research contribution indicates completed software.

## Inputs needed for physical development

The current report does not establish access to an assembled article, a complete BOM or a measured reference capture. Before physical milestones, identify the actual board/motor/encoder/power system, print process, assembly revision, mass properties and available instrumentation. Select the intended experiment or mission so acceptable tracking error, latency, load and operating limits can be declared from requirements and baseline data.

The [roadmap](../plans/2026-10-26-evolution-roadmap.md) separates work possible now from work dependent on those inputs. Its calendar windows are contingent planning windows, not promised flight readiness.

## Verification boundary and refresh

The research inspected source, repository metadata, primary publications and third-party CI metadata. It did not compile or execute the firmware locally, operate hardware or independently reproduce published flight results. The PULSAR II advertised code link returned 404; access to its full text and the September 2026 cyclic-speed paper was limited as documented.

Before adopting a contribution, refresh its state, pin its exact commit, inspect the complete changed files and dependencies, and rerun relevant tests on the destination. Before repeating the fork scan, account for this documentation branch so it is not mistaken for a new CAD or flight-control improvement.
