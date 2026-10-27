# Control and hardware research for the single-propeller drone

**Evidence cutoff: 26 October 2026, America/Toronto.** Research accessed during that local evening (27 October UTC). Scope: evolving Joshua Bird's high-RPM, single-motor, hinged-propeller aircraft. This is a research synthesis and proposed experimental program; no hardware experiments were performed. Source IDs resolve to ordinary URLs below and to the accompanying JSON ledger.

## 1. Establish the actual plant before selecting a controller

The author reports a PPA-CF hinged hub, almost 20,000 RPM, approximately 300 torque modulations per second, and complementary blade-pitch changes produced by blade lag during motor acceleration. His custom motor-controller firmware generates the pulses onboard. The body slowly rotates and obtains a heading reference from an external infrared beacon once per body revolution. These are first-person statements, not independently measured results. The original explanatory video was linked but could not be retrieved; this report does not claim to have watched it. [C01]

The main repository supplies body, propeller-plate, and two propeller-version CAD files, media, and links to ArduPilot and moteus forks. It does not itself supply a validated identification dataset, fatigue qualification, complete manufacturing procedure, or efficiency comparison. [C02] Those omissions define the most valuable research contribution a development fork can make.

There are two distinct angles: fast **rotor azimuth relative to the body**, and slower **body heading relative to the world**. Motor-angle synchronization solves the first problem; it does not solve the second. A heading error rotates the intended horizontal control direction. Measuring heading while accepting body spin also does not establish independent yaw authority. Design the controller around thrust direction and position unless experiments demonstrate more.

## 2. Lineage and transferability

Paulos and Yim's IROS 2013 work already demonstrated passive, inclined-hinge cyclic control through modulated drive torque; its flight vehicle used two coaxial motors. Their university project explains that waveform amplitude and phase set response magnitude and direction. [C03, C04] This is the relevant ancestry, rather than treating the aircraft as a conventional fixed-pitch propeller with an ordinary speed command.

The 2015 continuation flew a 227 g, two-motor coaxial vehicle with a 30 cm upper rotor, encoder-based drive electronics, and motion-capture trajectory tracking. [C05] It establishes feasibility of integrated cyclic control, not a one-motor endurance or autonomy result. A separate 2018 two-actuator vehicle demonstrated position/orientation separation using two independently vectored rotor forces; its additional actuator authority cannot be imported into Bird's single-rotor model. [C08]

| Platform / publication | Demonstrated or reported evidence | What transfers to this fork | Boundary |
|---|---|---|---|
| Bird, October 2026 [C01–C02] | Author reports near-20,000-RPM cyclic operation, slowly rotating body and external IR heading | Exact development target; retain motor-local synchronization | No published endurance, fatigue, payload or tracking benchmark in the main repository |
| Paulos–Yim, IROS 2013 / ICRA 2015 [C04–C05] | Hinged torque-modulated rotor; two-motor coaxial free flight; 2015 vehicle 227 g | Physical mechanism and controller integration precedents | Counter-rotating second rotor changes yaw and trim dynamics |
| Paulos–Yim, JGCD 2018 [C06] | Bench measurements of torque, hub speed, lag/pitch and flap responses versus a dynamic model | Identification methodology and nonlinear friction warning | Different hinge/rotor design and operating regime |
| Paulos–Yim, SciTech 2018 [C07] | Dynamically similar 10 cm and 1 m test rotors; reported envelope 300–9,000 RPM | Nondimensional modeling, inertia and drive identification | Does not validate 20,000-RPM printed assemblies |
| PULSAR, Science Robotics, 15 March 2023 [C09] | One-motor self-rotating vehicle; onboard LiDAR navigation; 26.7% lower hover power than its matched-disk-area/avionics quadrotor benchmark | Rotating-frame control and benchmark discipline | Different complete aircraft; saving is not a prediction for Bird |
| PULSAR II, Nature Communications, 21 May 2026 [C10] | Identified nonlinear propulsion; S² MPC; flatness planning; protected frame and rotation-decoupled disturbance observer | Strongest directly related architecture for later development | Accessible university abstract supports methods; full paper retrieval blocked; linked code returned 404 |
| Zhangliu et al., Aerospace Science and Technology, September 2026 [C13] | Publisher describes FOC cyclic-speed control and high-speed tracking limitations | Timely comparison for motor-loop experiments | Abstract/highlights inspected; no reproduced result or verified 20,000-RPM equivalence |

PULSAR's main conceptual advance for this project is obtaining control through **rotor-frequency** modulation while allowing body rotation; ordinary spinning monocopters often modulate at the slower body-rotation rate. Its paper explicitly distinguishes those architectures. [C09] Consequently, the motor loop, airframe control, and heading corrections need separate bandwidth budgets.

## 3. Identify response magnitude and phase, not just RPM

The JGCD rotor study is unusually actionable: it fits first harmonics to measured responses, observes a low-amplitude hinge-stiction threshold, and finds weaker phase agreement near that threshold. Its flap predictions can overestimate measured amplitude by about a factor of two. It also contrasts useful articulated flapping with undesirable blade bending and higher harmonics. [C06] An ideal hinge equation or encoder trace alone is therefore insufficient evidence of usable cyclic moment.

**Proposed identification model:** write the local command as

`tau_cmd(psi) = tau_mean + a_cos*cos(psi) + a_sin*sin(psi)`.

This is a waveform parameterization, not a claim that three independently controllable instantaneous actuators exist. Estimate the mapping from these coefficients to mean thrust and two horizontal moments, including the transient response when coefficients change. Store operating RPM, bus voltage, temperature, assembly identity, and uncertainty with every fitted map.

Use a restrained instrumented stand to measure force/moment while recording motor angle, requested/actual current, speed and controller timestamps. Synchronized strobe images can independently estimate blade lag and tip-path motion. Fit the fundamental and residual harmonics; report confidence and repeatability across assembled specimens. Proposed angular bins and sample rates must be chosen from measured bandwidth, not copied from a slower paper.

Separate **electrical delay**, **torque-production delay**, **hinge/blade dynamics**, and **flight-IMU filtering delay**. At 20,000 RPM, a revolution lasts 3 ms: a 0.2 ms delay is 24°, and 10 microseconds is 1.2°. These are arithmetic examples at exactly that speed; the author's approximate 20° account is not contradicted by them. Identify compensation experimentally rather than assuming a universal 90° phase correction.

Paulos–Yim's scaling paper accounts for hub/blade inertia, mass distribution, hinge position and drive dynamics, and proposes motor identification using known added inertias. It concludes that, at fixed aircraft weight, larger/slower rotors reduce the relative modulation burden. [C07] Use that result to motivate an RPM-versus-diameter study, not an unsupported claim that higher RPM always improves agility or efficiency.

## 4. Controller choices after identification

**First retain a transparent reduced-attitude baseline.** Control mean lift and thrust-axis direction on S², using the two achievable horizontal moments to regulate that direction. Do not assume force and moment can be independently commanded. Use measured body orientation to transform commands consistently. Keep constraints explicit: current/voltage headroom, modulation amplitude, minimum usable hinge response and estimator validity. A simple baseline is necessary to show whether a more complex controller actually helps.

**INDI is a promising intermediate experiment.** Smeur, Chu and de Croon's 2016 work estimates angular acceleration and uses a control-effectiveness model; it explicitly addresses sensor/actuator delays, online effectiveness changes and propeller momentum. [C11] For this rotor, the effectiveness matrix should operate on the identified waveform coefficients and appropriate filtered/averaged moments. Directly treating every oscillatory IMU sample as commanded aircraft acceleration could amplify rotor vibration. Test filtering and input/output alignment before permitting adaptation.

**MPC becomes attractive once the actuator model is credible.** PULSAR II specifically uses an S² model-predictive controller, nonlinear propulsion compensation and differential-flatness planning. Its disturbance observer uses a fictional frame that decouples self-rotation, with robustness parameters optimized through μ synthesis. [C10] The transferable idea is to model the available thrust direction, actuator limitations and rotation consistently. Its controller should be a comparison target, not a drop-in implementation presumed to match this rotor.

**Disturbance estimation needs an identifiable residual.** Begin with a limited-bandwidth residual between predicted and measured rigid-body response. Separate steady bias, rotor-synchronous loads and external disturbances. Otherwise an observer may “cancel” a phase-model error or structural vibration. Validate bounded disturbances on a restrained apparatus before free-flight recovery claims. A guard or fin change alters the aerodynamic plant and requires renewed identification.

Related rotor-failure research demonstrates position control of a spinning, three-remaining-rotor quadrotor using onboard vision, and explains the challenges of rapid yaw rotation. [C12] It supports reduced-attitude reasoning and estimator testing. It does not demonstrate single-actuator controllability, identical disturbance margins, or tolerance of the sole motor failing.

## 5. Measure the energy trade, including cyclic control

PULSAR's 26.7% comparison is a measured complete-system result against a specified quadrotor with similar total rotor area and the same avionics payload. The paper distinguishes total power from other efficiency comparisons. [C09] Fewer motors alone cannot establish a saving in this fork.

The proposed bench campaign should first compare a rigid reference rotor and hinged rotors at zero cyclic command under equal net lift. Then measure the incremental power of cyclic control within the hinged variants at matched lift and usable moment response. A rigid single rotor does not supply equivalent cyclic authority, so it is only an aerodynamic and power reference. Record mean thrust, torque, RPM, electrical input, current RMS, winding/controller temperatures and usable moment bandwidth. Repeat with and without the finned body to quantify aerodynamic interaction and counter-torque effects. Reserve complete task and control-authority comparisons for controllable aircraft or variants.

Compute electrical energy from time-aligned `integral(V*I dt)`, and shaft power from `mean(Q*omega)` when trustworthy torque sensing is available. With modulation, `mean(Q)*mean(omega)` can differ from the mean instantaneous product. Distinguish recovered energy, dissipated braking, current clipping and battery-voltage sag in the drive logs; do not assume which behavior the hardware implements.

UIUC's primary propeller database provides measured static RPM sweeps, thrust/torque coefficients and geometry, including printed propellers; its power coefficients derive from measured torque and speed. [C14] Borrow its reporting discipline and use comparable rigid-propeller data as a sanity check. Its specimens are not performance predictions for a hinged rotor operating in a finned wake. In hover, forward-flight propulsive efficiency `T*V/P` is uninformative at zero forward speed; report thrust per electrical watt, disk loading and uncertainty, with aerodynamic and electrical metrics clearly separated.

## 6. Manufacturing and fatigue are unresolved research variables

The author's PPA-CF statement identifies material family, not a qualified process. [C01] Bambu's manufacturer page warns that print orientation, walls, infill, geometry and conditioning affect performance, including materially different XY and Z behavior. [C15] Neither a tensile-strength headline nor a short successful flight supplies rotor fatigue life.

At an assumed 300 cyclic events per second, one million events accumulate in approximately 56 minutes. That calculation illustrates why flight minutes alone conceal large cycle counts; it is not a life prediction. Centrifugal load scales with mass, radius and angular-speed squared, while hinge oscillation adds cyclic and contact loads. Do not extrapolate the lower-speed literature's successful tests into an allowable RPM or service interval.

Create specimen records for filament batch/conditioning, printer and orientation, layer settings, postprocessing, measured mass balance, pivot geometry, bearing fit, fastener retention and accumulated cycles. Measure runout, hinge friction, backlash and resonance before and after endurance runs. Replicate assemblies and examine failed specimens; distinguish printing defects, creep, hinge wear, blade bending and hub damage. A process comparison must hold geometry and conditioning constant before assigning an improvement to “carbon fiber.”

The required outputs are measured response drift versus cycles, documented failure modes, and a conservative tested operating envelope. A protective enclosure and remote instrumented stand are necessary experimental apparatus for high-speed structural qualification; a photographed test stand alone is not evidence of containment capability.

## 7. Ranked next experiments and decision gates

These are proposed priorities, not completed validations:

1. **Establish synchronized phase and moment measurements.** Highest value: a repeatable thrust/moment map with delay decomposition, current headroom and harmonic residuals. Gate: reproduce the same commanded direction across RPM and voltage conditions before changing flight-control architecture.
2. **Characterize repeatability and hinge deadband.** Test multiple nominally identical rotor assemblies across temperature and wear. Gate: identify whether observed controller variation comes primarily from mechanics or software; publish raw data and uncertainty.
3. **Benchmark power at matched lift and control authority.** Compare modulation amplitude, rotor variants and fin/body configurations. Gate: demonstrate an energy–bandwidth trade rather than reporting an unloaded RPM or an unmatched hover-current reduction.
4. **Qualify construction and cycle life in containment.** Inspect hub, pivot and blade condition and response drift over repeated runs. Gate: define a demonstrated operating envelope and inspection/replacement policy before increasing RPM or exposure.
5. **Compare reduced-attitude baseline with INDI.** Use identical disturbances, initial conditions, estimator and actuator limits. Gate: show tracking improvement without increased saturation or vibration amplification.
6. **Evaluate constrained S² MPC and disturbance estimation.** Begin in simulation using the measured plant and uncertainty, then restrained tests. Gate: compare tracking, robustness, compute deadlines and energy against the simpler controller; retain the simpler option if gains are not repeatable.

The highest-confidence advance is a measured, reproducible electromechanical plant. Better models, lower uncertainty and qualified construction make later control sophistication assessable.

## Primary-source references

- **C01: Author explanation (October 2026):** https://www.reddit.com/r/robotics/comments/1x8zyvp/i_built_a_drone_with_only_one_motor_and_a_poor/
- **C02: Bird project repository:** https://github.com/jyjblrd/single-propeller-drone
- **C03: ModLab mechanism/project bibliography:** https://www.modlabupenn.org/underactuated-rotor/
- **C04: Paulos & Yim, IROS 2013:** https://www.modlabupenn.org/wp-content/uploads/paulos_an_underactuated_propeller_IROS_2013.pdf
- **C05: Paulos & Yim, ICRA 2015:** https://www.modlabupenn.org/wp-content/uploads/paulos_flight_performance_ICRA_2015.pdf
- **C06: Paulos & Yim, JGCD 2018, 41(3), 689–700:** https://www.modlabupenn.org/wp-content/uploads/2018/05/paulos_cyclic_blade_pitch_JGCD_2018.pdf
- **C07: Paulos & Yim, AIAA 2018-0532:** https://www.modlabupenn.org/wp-content/uploads/2018/03/paulos_scalability_of_cyclic_AIAA_2018.pdf
- **C08: Paulos, Caraher & Yim, ICRA 2018:** https://www.modlabupenn.org/wp-content/uploads/2018/05/paulos_emulating_ICRA_2018.pdf
- **C09: Chen et al., Science Robotics 2023, eade4538:** https://lawrence-cn.github.io/files/SR.pdf ; https://doi.org/10.1126/scirobotics.ade4538
- **C10: Chen et al., Nature Communications 2026, 17:6730:** https://doi.org/10.1038/s41467-026-73283-x ; accessible university explanation: https://mech.hku.hk/an-autonomous-single-actuator-uav-with-omnidirectional-field-of-view-high-agility-and-collision-resistance/
- **C11: Smeur, Chu & de Croon, JGCD 2016, 39(3), 450–461:** https://research.tudelft.nl/en/publications/adaptive-incremental-nonlinear-dynamic-inversion-for-attitude-con-2/
- **C12: Sun et al., RA-L 2021, 6(2), 580–587:** https://arxiv.org/abs/2102.13406 ; https://rpg.ifi.uzh.ch/docs/RAL21_Sun.pdf
- **C13: Zhangliu et al., Aerospace Science and Technology 2026, 176(A):112010:** https://doi.org/10.1016/j.ast.2026.112010 ; https://www.sciencedirect.com/science/article/pii/S1270963826003901
- **C14: UIUC Propeller Database, Volume 2:** https://m-selig.ae.illinois.edu/props/volume-2/propDB-volume-2.html
- **C15: Bambu Lab PPA-CF manufacturer information:** https://uk.store.bambulab.com/products/ppa-cf

**Evidence limits:** Search used both available web systems. The PULSAR II paper was verified through publisher indexing and HKU's publication announcement; direct Nature/PMC full-text access failed. Its published GitHub link, https://github.com/hku-mars/PULSAR2, returned 404 at access time. C13 and C15 rely on indexed primary publisher/manufacturer text where direct pages blocked retrieval. No claims of new implementation, flight testing, purchases, or independent replication are made.
