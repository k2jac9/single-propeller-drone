# ADR-0001: Evolve through measurement, compatible firmware and staged autonomy

**Status:** Proposed engineering direction for review.  
**Date:** 26 October 2026 session; verified 27 October UTC.  
**Scope:** Development strategy. This decision does not approve an operating envelope or select flight hardware.

## Context

The CAD release links a customized ArduPilot flight controller and moteus motor controller. Local rotor-angle synchronization is already implemented, but source/configuration versions are not packaged as one tested release. No divergent CAD fork was found, and the meaningful community firmware addition is an optional phase schedule awaiting physical calibration. Primary research shows that hinge friction, phase delay, amplitude response and rotating-frame estimation matter to this class of aircraft. [Baseline](../research/01-baseline-assessment.md), [firmware audit](../research/03-firmware-audit.md), [literature](../research/04-control-and-hardware-research.md).

The architectural problem is therefore how to make improvements attributable and repeatable across mechanics, motor firmware, flight control and estimation. A rewrite would introduce additional uncertainty before the existing plant has been measured.

## Decision

1. **Keep motor-angle cyclic execution in moteus and outer flight control in ArduPilot for the initial development stages.** Record their boundary explicitly: MCOUT wire version, frame definition, clock provenance, state flags and configuration compatibility. Preserve a fixed-phase reference for comparisons.
2. **Build offline evidence tooling before adding live control software.** Rust is the proposed implementation language for a bounded streaming decoder/report tool, with a pinned toolchain and reproducible tests. The scientific identification scripts may use Python. No language migration of flight-critical firmware is part of this decision.
3. **Use an identified, multirate reduced-order simulator first.** It must include actuator delay, gain/phase, saturation, body rotation, sensor timing and the actual MCOUT path. Adopt richer Gazebo or MuJoCo environments only when they resolve a specific sensor, contact or geometry question.
4. **Design around thrust direction and position while measuring body heading.** Independent yaw authority is not assumed. Evaluate a transparent reduced-attitude baseline before INDI or constrained S² MPC. Controller sophistication must earn its cost against held-out measurements and identical comparison scenarios.
5. **Treat autonomy as a conditional payload and estimation decision.** External reference tracking is the first benchmark candidate. Compare lightweight visual-inertial and LiDAR-inertial options using recordings and rotating rigs before committing onboard mass, power or compute.

These are proposed choices for this fork, derived from the evidence rather than claims of completed implementation. The [tool specification](../superpowers/specs/2026-10-26-offline-evidence-tool-design.md) and [roadmap](../plans/2026-10-26-evolution-roadmap.md) define the first bounded steps.

## Alternatives considered

| Approach | Benefit | Cost or unresolved assumption | Decision |
|---|---|---|---|
| Extend existing ArduPilot/moteus with pinned compatibility and measurements | Small changes preserve the demonstrated architecture; failures can be attributed | Requires careful maintenance across two upstreams | Preferred baseline |
| Port immediately to a PULSAR-derived PX4 stack | Closer to published single-actuator autonomy architecture | Different hardware and plant; PULSAR II's advertised code was unavailable during review; introduces a port before measurement | Revisit only with a concrete missing capability and accessible implementation |
| Rewrite the motor or flight firmware in Rust | Potentially attractive tooling and type discipline | Timing, drivers, numerical behavior and hardware validation would all need re-establishment | Defer; use Rust offline first |
| Begin with a detailed digital twin or CFD campaign | Can investigate geometry and flow interactions | Unknown hinge/friction/drive parameters can make precise-looking results misleading; high modeling cost | Use measured reduced-order model first; add fidelity where residuals demand it |
| Add LiDAR, SLAM and trajectory planning immediately | Demonstrates an ambitious end goal | Payload/power limits, rotating sensor calibration and estimator timing are not established | Defer hardware integration until the relevant gates pass |

## Interfaces that must remain explicit

- **Source identity:** CAD revision, both firmware commits, toolchain and binary hashes, parameters, hardware revision and specimen ID are independent fields. A branch name is not a release identity.
- **Commands:** MCOUT v1 is a custom 84-byte little-endian record. MAVLink tooling belongs to separate flight-controller telemetry. Any future wire change needs versioning and compatibility tests on both ends.
- **Clocks:** Sender time, motor time, capture arrival time and sensor acquisition time remain separate until a measured mapping exists. Reports must not label their raw difference as latency.
- **Frames:** World frame, body frame, rotor azimuth convention and sensor extrinsics have named units, direction and sign tests. A roll/pitch sign or phase error is observable on a restrained apparatus before navigation.
- **State:** Arming, spool state, watchdog timeout, stale commands, sensor loss, fault clearing and re-engagement have a documented transition policy. Existing source behavior is recorded before proposing a change.
- **Data:** Raw records remain recoverable. Derived models carry training/validation dataset IDs, operating domain and uncertainty; logs include saturation and dropped-data evidence.

## Consequences and reconsideration triggers

The first releases will improve reproducibility more than visible flight features. That is deliberate: measurement infrastructure supports every later hypothesis and makes unsuccessful experiments useful to other builders. It also allows meaningful software progress before a test article is available.

The strategy does not lock the project permanently to ArduPilot, a specific estimator or a particular controller. Reconsider a component when an identified requirement cannot be met, its measured timing or accuracy fails a declared budget, or a maintained alternative demonstrates the required capability on comparable inputs. Record the requirement, comparison and migration cost in a subsequent ADR.

A source update, altered print process, new rotor geometry, changed motor/controller, added guard or payload changes some part of the plant. Determine which measurements and regression cases it invalidates before carrying forward an earlier calibration.
