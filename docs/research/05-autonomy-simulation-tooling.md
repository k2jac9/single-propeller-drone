# Autonomy, simulation, and tooling research

**Research date: 26 October 2026, America/Toronto (27 October UTC).** This chapter recommends an experimental development path. It distinguishes inspected source/documentation, authors' published results, and proposed engineering work. No simulator, estimator, or aircraft was built or flown for this research.

## 1. Recommended architecture and order

Start with a deterministic, measured, reduced-order simulator coupled to the selected ArduPilot and moteus control path. Establish the aircraft's control authority and timing before adding autonomous navigation. A rendered world can test visibility, terrain, and collisions later; it cannot identify an unknown rotor, hinge, motor, or phase response.

The proposed simulation state should distinguish world position, body attitude and spin, rotor azimuth relative to the body, rotor speed, actuator dynamics, and relevant hinge/lag states. Fit thrust, torque, cyclic response, delay, and saturation to bench measurements. Keep uncertain coefficients explicit and sweep them. A simple model validated over a stated operating envelope is more useful than detailed geometry with assumed aerodynamics.

Use a multirate scheduler: motor/azimuth dynamics and phase-triggered modulation can require finer integration than the flight-controller update, while navigation and map updates can run more slowly. Choose rates from measured bandwidth and numerical convergence; do not select a convenient single timestep and assume it resolves rotor events. Compare results after halving the integration step, and log the applied actuator history. These are proposed requirements, not features already present in the CAD repository.

## 2. Connect the real control path to SITL

ArduPilot's [JSON backend overview][A01] and the inspected fork's [protocol description][A02] provide a practical interface for a custom physics backend. The backend receives a binary servo packet and returns JSON containing physics time, IMU quantities, position, velocity, and attitude. Verify conventions with elementary stationary and rotated-body cases before closed-loop tests.

**The stock servo packet does not contain the custom MCOUT stream.** The inspected [MCOUT implementation][A03] publishes a separate UART packet containing normalized controller commands, state flags, sequence, timestamp, and CRC. Its receiver performs vehicle-specific mixing. Feeding normal PWM channels into an idealized thrust vector would bypass this aircraft's defining external mixer and phase-dependent actuator path.

The proposed causal bridge must route the selected firmware's actual MCOUT output into a decoder and motor/rotor model, then return only the resulting physical sensor state to SITL. Preserve command issue, receipt, acceptance, and application times. Model packet loss, duplicates, CRC rejection, stale-command handling, arming/interlock transitions, and saturation. Compare an ideal transport against measured transport delays as an ablation. An implementation must verify the precise serial routing supported by its pinned SITL build; this report has not established a working bridge.

Use a single simulation clock and explicit event ordering. A command produced from state at time t must not affect the plant before its scheduled application. Source timestamps should not be silently replaced with host wall time. Deterministic seeds and stored configuration should reproduce a run; real-time speed is a separate performance measurement.

## 3. Gazebo and MuJoCo: useful roles, defined limits

The [official ArduPilot Gazebo plugin][A04] offers JSON integration, lockstep, sensors, and rendering. Its direct SITL workflow does not require ROS. It is a reasonable later choice for synthetic camera/LiDAR scenes and interaction with surroundings. A single-propeller model would still need the measured actuator and aerodynamic model, plus the MCOUT bridge; an existing quadcopter world does not supply either.

[MuJoCo's fluid documentation][A05] explicitly describes phenomenological, stateless models. These can support fitted body drag and rigid-body experimentation. They are not full rotor CFD and do not automatically reproduce blade lag, induced flow, hinge coupling, or the measured cyclic phase response. Use custom forces or fitted dynamics where needed and document the calibration envelope.

Keep scene simulation replaceable. First compare measured and predicted force/torque response, spin evolution, and attitude response without perception. Then hold that plant model fixed while evaluating navigation. Otherwise an apparent estimator improvement may simply compensate for a changed simulator.

## 4. Estimation candidates for a spinning body

The following is a shortlist for replay evaluation, not an onboard payload selection. Repository descriptions and demonstrations establish research relevance, not performance on this airframe.

| Candidate | Why evaluate it | Main burden and license observation |
| --- | --- | --- |
| [Point-LIO][A06] | Processes LiDAR observations at point sampling times; relevant aggressive-motion examples | Requires synchronized IMU/LiDAR and retained per-point timestamps. Actual LICENSE has BSD-style three-clause text; GitHub detector reports NOASSERTION. |
| [FAST_LIO][A07] | Useful LiDAR-inertial comparison baseline | Point timing, deskew, calibration, payload and compute remain necessary. GPLv2 stated in README and metadata. |
| [FAST-LIVO2][A08] | Adds camera information to LiDAR-inertial estimation | Extra sensor mass/power, exposure, synchronization and camera/LiDAR/IMU calibration. GPLv2. |
| [OpenVINS][A09] | Camera/IMU baseline that can avoid carrying LiDAR | Blur, feature visibility, scale/initialization, extrinsics and time offset must be tested. GPLv3 LICENSE. |

Point-LIO's current default branch is `point-lio-with-grid-map`, pinned in the ledger. Its README reports 4–8 kHz output and aggressive-motion testing at 75 rad/s. These are authors' research-specific claims, not guaranteed usable feedback rates, sensor limits, or aircraft operating targets. Its tested baseline is Ubuntu 20.04 with ROS Noetic; isolate that environment for reproducibility rather than assuming a contemporary ROS release works unchanged. Preserve timestamp units and actual IMU saturation settings.

For all candidates, evaluate the delivered estimate's age and error, including worst intervals and resets. Publishing frequently can still provide delayed or highly correlated estimates. Measure orientation and position error against an independent reference, processing latency, queue buildup, data loss, compute utilization, and failure recovery. Avoid ranking algorithms solely by map appearance or advertised update frequency.

Treat licenses as component-specific integration constraints. Review dependency and file notices before distributing combined work; the Point-LIO detector result should not replace reading its license text. No licensing compatibility of a future combined product has been established here.

## 5. Calibration and spinning-sensor research

[OpenVINS calibration guidance][A10] emphasizes accurate timestamps and varied motion. Its observability discussion requires translation plus multiple orientation changes; pure spin about one axis is insufficient for a general camera/IMU extrinsic-and-time calibration. Calibrate with deliberate, varied motion off the aircraft, inspect residuals, and test whether estimates remain consistent after remounting and vibration.

The January 2026 [LM-Calibr/EVA-LIO paper][A12] studies actuated spinning LiDAR systems; its arXiv entry reports RA-L acceptance. LM-Calibr addresses LiDAR-to-motor calibration, while EVA-LIO adapts processing to scene scale. These are published results on the authors' apparatus. At the inspected [repository commit][A13], the EVA-LIO guide and hardware-design sections remain TODO. The paper's availability statement should therefore not be treated as a complete reproduction package.

Transfer the calibration questions: encoder timing, axis geometry, rigid transforms, changing field of view, and feature-poor scan sectors. A motor rotates the sensor relative to its supporting platform in that research; a sensor rigidly attached to a spinning aircraft has a different kinematic chain, body dynamics, vibration, and reference frame. Body-spin angle is not interchangeable with rotor azimuth. A useful experiment would compare externally referenced body heading with inertial propagation and scan matching, rather than assume a rotor encoder supplies world heading.

## 6. Feed external navigation to ArduPilot carefully

The [ArduPilot external-navigation guide][A11] prefers MAVLink ODOMETRY and specifies at least 4 Hz. That is an interface minimum, not a suitable control-design target for this vehicle. Select a rate and latency budget from measured dynamics and stability requirements.

Define coordinate transforms, sensor lever arms, quaternion convention, units, timestamp interpretation, estimator origin, uncertainty, and reset behavior before integration. Set the EKF origin when required without GPS. Populate the supported covariance fields from a defensible error model and increment `reset_counter` when the external estimate resets. Verify source-set configuration and quality rejection on the pinned firmware.

First replay known trajectories through the adapter with deliberate frame rotations, time shifts, resets, and dropouts. Then test the aircraft response to degraded or missing estimates. Supplying pose messages is an estimation interface; it does not establish obstacle avoidance, mission execution, recovery behavior, or safe autonomous flight.

## 7. Logging and optional Rust tools

Use immutable raw capture alongside decoded data. The [MCAP specification][A15] provides channels, schemas, message sequence, recording time, and publication time. Preserve the original device/sensor timestamp and its clock domain explicitly in payload or metadata; publication time is not automatically sensor acquisition time. Record clock mappings, offset/drift estimates, and arrival time so replay can separate physical latency from logging delay.

Suggested channels include raw MCOUT bytes, accepted decoded commands, motor telemetry, rotor/body angle, IMU, navigation estimates, reference pose, validity flags, and applied simulator forces. Attach firmware SHAs, parameter files, calibration, units, schema versions, and model coefficients. Index the result for examining a single revolution or reset, while retaining enough raw data to fix a decoder later.

Rust is optional for offline parsers, deterministic replay, parameter sweeps, and report generation. [rust-mavlink][A14] supplies typed MAVLink messages and transports; MCOUT is a separate custom protocol requiring its own validated decoder. [Foxglove's SDK][A16] supports MCAP logging and visualization from Rust, Python, and C++. A Rust rewrite of working flight firmware is not required to obtain these benefits.

## 8. Experiments and promotion gates

Use existing sensors and datasets first; the evidence here does not justify purchasing hardware. Establish payload feasibility with measured mass, center-of-mass shift, inertia, power, battery endurance, vibration, field of view, and thermal/compute limits before proposing an onboard stack.

| Stage | Required evidence before progressing |
| --- | --- |
| Offline replay | Repeatable results with retained timing; frame/reset tests; independent error and latency measurements |
| Rotating rig | Sweep body spin, exposure, vibration, time offsets and scene features; test calibration away from pure spin |
| Restrained integrated tests | Validate control timing and estimator faults together; account for rig/tether forces and altered dynamics |
| Limited flight experiments | Predeclared operating envelope, independent tracking, recovery criteria and repeatable comparison runs |

The author's [Low-Cost-Mocap][A17] can be an optional outside-in laboratory reference if an appropriate installation already exists. Its README calls `main` tested and explicitly calls `no-cv-sfm` completely untested. Measure the reference's own timing, scale, occlusion sensitivity and uncertainty; neither the repository title nor another drone demonstration establishes ground-truth accuracy for this spinning vehicle.

Run targeted ablations: spin angle/rate, acquisition-time versus arrival-time processing, imposed time offset/drift, per-point deskew, camera exposure/blur, sensor lever arm, feature-poor sectors, reference-heading availability, added payload, power draw, and compute throttling. Report error versus spin angle as well as averages, because one problematic sector can repeatedly disrupt feedback. Retain failed runs and resets.

Success means a measured improvement over a named baseline within the tested envelope. The proposed tooling and estimator shortlist are a route to gathering that evidence; they do not guarantee autonomy.

## Source ledger

Pinned repository links identify inspected snapshots; live documentation can change. Companion JSON: [autonomy-sources-2026-10-26.json](../../research-data/autonomy-sources-2026-10-26.json).

- **A01**: [ArduPilot JSON backend overview](https://ardupilot.org/dev/docs/sitl-with-JSON.html).
- **A02**: [JSON simulation protocol in inspected ArduPilot fork](https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/libraries/SITL/examples/JSON/readme.md).
- **A03**: [MCOUT implementation and packet documentation](https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/libraries/AP_MotorControlOutput/AP_MotorControlOutput.cpp); [supporting file](https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/libraries/AP_MotorControlOutput/README.md).
- **A04**: [Official ArduPilot Gazebo plugin](https://github.com/ArduPilot/ardupilot_gazebo/blob/80dc9664dda7efd6c5d1a71a79198d1d6b708c0e/README.md).
- **A05**: [MuJoCo fluid-force models](https://mujoco.readthedocs.io/en/stable/computation/fluid.html).
- **A06**: [Point-LIO](https://github.com/hku-mars/Point-LIO/blob/4b86a469eb5572e70ed575af25b5f15dd06e8e3c/README.md); [supporting file](https://github.com/hku-mars/Point-LIO/blob/4b86a469eb5572e70ed575af25b5f15dd06e8e3c/LICENSE).
- **A07**: [FAST_LIO](https://github.com/hku-mars/FAST_LIO/blob/7cc4175de6f8ba2edf34bab02a42195b141027e9/README.md).
- **A08**: [FAST-LIVO2](https://github.com/hku-mars/FAST-LIVO2/blob/0d2c0346107b75b59934975adec9a6eeeb913c64/README.md).
- **A09**: [OpenVINS](https://github.com/rpng/open_vins/blob/69488123ed9362dd44b6f28e7f4680abbff1442b/ReadMe.md); [supporting file](https://github.com/rpng/open_vins/blob/69488123ed9362dd44b6f28e7f4680abbff1442b/LICENSE).
- **A10**: [OpenVINS calibration guidance](https://docs.openvins.com/gs-calibration.html).
- **A11**: [ArduPilot external navigation interface](https://ardupilot.org/dev/docs/mavlink-nongps-position-estimation.html).
- **A12**: [Spinning actuated LiDAR paper](https://arxiv.org/abs/2601.15946v2).
- **A13**: [LM-Calibr/EVA-LIO repository](https://github.com/zijiechenrobotics/lm_calibr/blob/fe96c7f2face3e0daee366d316371f93ef4affee/README.md).
- **A14**: [rust-mavlink](https://github.com/mavlink/rust-mavlink/blob/dd8c2722832215e93d6678bc29a447cb15d3a471/README.md).
- **A15**: [MCAP specification](https://mcap.dev/spec).
- **A16**: [Foxglove SDK](https://docs.foxglove.dev/docs/sdk).
- **A17**: [Low-Cost-Mocap](https://github.com/jyjblrd/Low-Cost-Mocap/blob/8ac4d4107fda026d787d5e197f0dcbe6257b2720/README.md).

[A01]: https://ardupilot.org/dev/docs/sitl-with-JSON.html
[A02]: https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/libraries/SITL/examples/JSON/readme.md
[A03]: https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/libraries/AP_MotorControlOutput/AP_MotorControlOutput.cpp
[A04]: https://github.com/ArduPilot/ardupilot_gazebo/blob/80dc9664dda7efd6c5d1a71a79198d1d6b708c0e/README.md
[A05]: https://mujoco.readthedocs.io/en/stable/computation/fluid.html
[A06]: https://github.com/hku-mars/Point-LIO/blob/4b86a469eb5572e70ed575af25b5f15dd06e8e3c/README.md
[A07]: https://github.com/hku-mars/FAST_LIO/blob/7cc4175de6f8ba2edf34bab02a42195b141027e9/README.md
[A08]: https://github.com/hku-mars/FAST-LIVO2/blob/0d2c0346107b75b59934975adec9a6eeeb913c64/README.md
[A09]: https://github.com/rpng/open_vins/blob/69488123ed9362dd44b6f28e7f4680abbff1442b/ReadMe.md
[A10]: https://docs.openvins.com/gs-calibration.html
[A11]: https://ardupilot.org/dev/docs/mavlink-nongps-position-estimation.html
[A12]: https://arxiv.org/abs/2601.15946v2
[A13]: https://github.com/zijiechenrobotics/lm_calibr/blob/fe96c7f2face3e0daee366d316371f93ef4affee/README.md
[A14]: https://github.com/mavlink/rust-mavlink/blob/dd8c2722832215e93d6678bc29a447cb15d3a471/README.md
[A15]: https://mcap.dev/spec
[A16]: https://docs.foxglove.dev/docs/sdk
[A17]: https://github.com/jyjblrd/Low-Cost-Mocap/blob/8ac4d4107fda026d787d5e197f0dcbe6257b2720/README.md
