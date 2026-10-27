# Research and evolution guide

This documentation turns the original single-propeller drone release into a traceable development program. The review covers the CAD, both linked firmware repositories, their exposed branches and relevant fork networks, and primary literature on control, mechanics, simulation and navigation.

**Snapshot:** 27 October 2026 UTC, during the evening of 26 October in America/Toronto. Dated filenames retain the research session's 26 October label. Repository collections and open PRs can change after this snapshot. The fork inventory predates this contribution's documentation branch.

## Recommended direction

Develop this fork as a **reproducible single-actuator research platform**. First establish an identified electromechanical plant and reliable evidence collection. Preserve the existing ArduPilot/moteus division while comparing targeted improvements. Add independent navigation only after the actuator, timing, heading and payload budgets are understood.

The source audit supports that ordering:

- The original CAD repository and all **91 verified CAD forks** expose the same sole `main` commit. No distinct committed CAD implementation was found to adopt. [Full network audit](02-fork-network.md)
- The firmware audit covers **73 ArduPilot branches, 34 moteus branches, eight verified ArduPilot child forks and ten verified moteus child forks**. Most branches are inherited history. The only distinct child-fork work found is in two branches of `metalogiclabs/moteus`, including the optional phase-scheduling PR. [Firmware audit](03-firmware-audit.md)
- Published work supports torque-modulated hinged rotors and reduced-attitude control, but does not establish this printed assembly's efficiency, endurance, fatigue life or navigation performance. [Control and hardware research](04-control-and-hardware-research.md)
- The first practical development slice is an **offline MCOUT evidence tool**: decode captures, check source/configuration compatibility and report timing limitations without operating a motor. [Implementation specification](../superpowers/specs/2026-10-26-offline-evidence-tool-design.md)

## Read in this order

| Document | Purpose |
|---|---|
| [Baseline assessment](01-baseline-assessment.md) | What the fork contains, how the aircraft works, and which claims remain unvalidated |
| [Original branches and CAD forks](02-fork-network.md) | Full 91-fork inventory, exact source equality and discovery limitations |
| [Firmware audit](03-firmware-audit.md) | Branches, actual contributed changes, transport/timing issues and phase PR review |
| [Control and hardware research](04-control-and-hardware-research.md) | Mechanism lineage, system identification, controller choices, energy and structural experiments |
| [Autonomy, simulation and tooling](05-autonomy-simulation-tooling.md) | What to reuse, what needs adaptation, and how to evaluate perception without premature payload commitments |
| [Architecture decision](../architecture/ADR-0001-evolution-strategy.md) | Proposed technical direction and alternatives considered |
| [Evolution roadmap](../plans/2026-10-26-evolution-roadmap.md) | Ordered milestones, decision gates, experiment records and actionable backlog |
| [First implementation plan](../superpowers/plans/2026-10-26-offline-evidence-tool.md) | Bounded software work ready for a separate implementation change |
| [Research handoff](../memory/2026-10-26-research-handoff.md) | Current decisions, outstanding questions and the next task |

## Evidence rules

**Observed source** means a file, branch identity or API result was inspected. **Published result** means an author or research group reports a result; it has not been reproduced here. **Derived** means arithmetic or an inference from inspected code, with its assumptions stated. **Proposed** means a design or experiment for this fork. **Validated on this aircraft** requires a reproducible run, identified hardware, exact software/configuration, raw evidence and a stated acceptance criterion.

This review contains the first four classes. It does not establish the fifth. No firmware was compiled locally, uploaded, or flight-tested as part of the documentation work. The third-party phase PR's remote CI evidence is described with its exact commit limitations in the firmware audit.

## Machine-readable evidence

The ledgers are normalized review records, not substitutes for raw logs or complete Git mirrors:

- [Baseline manifest](../../research-data/baseline-manifest-2026-10-26.json)
- [CAD fork network](../../research-data/fork-network-2026-10-26.json)
- [Firmware inventory and PR evidence](../../research-data/firmware-inventory-2026-10-26.json)
- [Control and hardware sources](../../research-data/control-research-sources-2026-10-26.json)
- [Autonomy and tooling sources](../../research-data/autonomy-sources-2026-10-26.json)

The baseline manifest records what was reviewed. It is explicitly **not a flight configuration or an assertion that the listed firmware commits form a tested pair**.

## Scope of this contribution

This contribution adds research, architecture and planning documents plus normalized evidence. Firmware changes, parameter changes, new CAD, physical experiments and upstream contribution proposals belong in subsequent focused changes, each tied to the relevant roadmap gate.
