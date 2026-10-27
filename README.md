# Single Propeller Drone

A drone that flies on a single propeller, mounted on top of a finned body.

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="media/drone-dark.png">
    <img src="media/drone-light.png" alt="The drone" width="420">
  </picture>
</p>

## Test stand

<p align="center">
  <img src="media/test-stand.gif" alt="Rotor spinning on the test stand" width="720">
</p>

## Firmware

- [jyjblrd/ardupilot](https://github.com/jyjblrd/ardupilot): ArduPilot fork (flight controller)
- [jyjblrd/moteus](https://github.com/jyjblrd/moteus): moteus fork (motor controller)

## Files

| Part | Fusion 360 | STEP |
| --- | --- | --- |
| Body | `body.f3z` | `body.step` |
| Prop plate | `prop plate.f3d` | `prop plate.step` |
| Propeller v1 | `prop_v1.f3d` | `prop_v1.step` |
| Propeller v2 | `prop_v2.f3d` | `prop_v2.step` |

## Research and development in this fork

The [research guide](docs/research/README.md) documents the original branches, 91 verified CAD forks, linked firmware and contributed changes, and primary research on control, hardware, simulation and autonomy.

- [Evolution roadmap](docs/plans/2026-10-26-evolution-roadmap.md): staged experiments, development priorities and completion criteria.
- [Proposed architecture](docs/architecture/ADR-0001-evolution-strategy.md): measured rotor dynamics, compatible firmware and staged navigation.
- [First software implementation plan](docs/superpowers/plans/2026-10-26-offline-evidence-tool.md): an offline MCOUT evidence tool.

The research distinguishes inspected source and published results from proposed work. It does not establish a validated firmware pair or a tested operating envelope for a new assembly.
