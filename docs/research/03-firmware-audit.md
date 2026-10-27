# Firmware audit: control architecture, branches, forks, and reuse

**Assessment:** the linked firmware contains a coherent experimental control path, useful host tests, and a credible third-party phase-compensation contribution. It does not establish a reproducible, flight-qualified aircraft. First preserve a matched firmware/configuration baseline; then resolve the documented protocol mismatch and validate measured control response before expanding the operating envelope.

**Scope and evidence.** Retrieved on 2026-10-27 UTC; the associated inventory retains the requested project snapshot label, 2026-10-26. This is a read-only source/API audit, with no local build, firmware execution, or hardware testing. “Observed” below means source or API evidence; “interpretation” identifies engineering implications; “proposed” identifies future work. All 73 original ArduPilot branches, 34 original moteus branches, and branches/PRs of 18 verified direct children are inventoried in [the machine-readable inventory](../../research-data/firmware-inventory-2026-10-26.json). They were enumerated, not all individually code-reviewed.

## 1. Lineage and branch evolution

GitHub identifies `jyjblrd/ardupilot` as a child of `ArduPilot/ardupilot`, and `jyjblrd/moteus` as a child of `mjbots/moteus`. Their reported child-fork counters are eight and ten. The connector rejected both direct `/forks` endpoints. Exact-description [repository search](https://api.github.com/search/repositories?q=%22Single%20Motor%20Drone%20Fork%22%20fork%3Aonly&per_page=100) recovered 20 repositories including the originals; individual metadata verified all 18 parent relationships. This matches the counters, but search indexing and changed descriptions prevent treating it as proof of an exhaustive upstream-network census. [ArduPilot metadata](https://api.github.com/repos/jyjblrd/ardupilot); [moteus metadata](https://api.github.com/repos/jyjblrd/moteus).

The comparison direction below is **base → head**. Ahead counts belong to head, behind counts to commits present only on base. Counts describe ancestry, not aircraft-specific improvements.

| Repository and comparison | Observed result | Meaning |
| --- | --- | --- |
| ArduPilot `new-features` → `master` | Head ahead 7,398; behind 0 | Feature branch is incorporated; master also carries a much newer upstream baseline |
| ArduPilot `jbird/ir-beacon-yaw` → `master` | Ahead 7,405; behind 0 | Current IR implementation is incorporated |
| ArduPilot `claude/vigorous-einstein-8a53eb` → `master` | Ahead 7,407; behind 5 | Five alternate yaw-beacon commits remain unmerged |
| ArduPilot `copilot/fix-autotest-sitltest-sub` → `master` | Ahead 956; behind 2 | Separate Sub autotest repair/plan; not drone control |
| moteus `mcout-uart` → `main` | Ahead 17; behind 0 | MCOUT branch incorporated, plus other upstream changes |
| moteus `zero-angle-trigger-aux2-rt` → `main` | Ahead 27; behind 0 | Trigger/AUX work incorporated |

Sources: [ArduPilot branch API](https://api.github.com/repos/jyjblrd/ardupilot/branches?per_page=100), [moteus branch API](https://api.github.com/repos/jyjblrd/moteus/branches?per_page=100), and exact comparison endpoints recorded in the inventory. Large ArduPilot comparisons returned the API’s 300-file ceiling; those lists are not complete diffs.

The original defaults are ArduPilot `455b266cc2a73505eb98c56a8049fa6a4d50f9bf` and moteus `704a0dd9f30a3704e8915a82c1631301fdbff9bd`. Comparing branch names and SHAs with retrieved upstream inventories shows **66/73 ArduPilot and 30/34 moteus branches exactly identical to current upstream branches**. Thus even many `claude/*` moteus branches are inherited, not Joshua Bird drone developments. ArduPilot’s two remaining release branches are simply behind their upstream counterparts by one and 288 commits. Other unmatched moteus branches are `main`, `mcout-uart`, `zero-angle-trigger-aux2-rt`, and `gh-pages`; documentation publication is not flight-control evidence. [Upstream ArduPilot branches](https://api.github.com/repos/ArduPilot/ardupilot/branches?per_page=100); [upstream moteus branches](https://api.github.com/repos/mjbots/moteus/branches?per_page=100).

The five alternate yaw commits introduce `AP_YawBeacon`/`YBCN_*`, source ID 9, and a shared external-yaw buffer path. Current master instead uses `AP_IRBeaconYaw`/`IRYAW_*` with dedicated fusion buffering. These are alternative implementations, not additive upgrades; source ID reuse makes blind cherry-picking inappropriate. [Alternate branch head](https://github.com/jyjblrd/ardupilot/commit/30e0e7d9bd22e82fda2541ff262c70e80a0c3bbf); [current integration merge](https://github.com/jyjblrd/ardupilot/commit/7f89c44c663e259107e8091424833f4ec29fdd38).

## 2. Where descendants actually added work

| Verified child group | Branch evidence | Reusable contribution |
| --- | --- | --- |
| ArduPilot: fiamor, mcleber, Darknight92228, charanakreddy, gyillikci, sadhu-sanjay, kalmancito, tsai1278 | Defaults identical to original master; kalmancito also preserves all 73 original branches at identical SHAs | No unique returned branch commits |
| moteus: tsai1278, gyillikci, charanakreddy, Aquahoodjd, mcleber, Darknight92228, kalmancito, sadhu-sanjay, fiamor | Defaults identical; kalmancito and sadhu-sanjay preserve all 34 original branches unchanged | No unique returned branch commits |
| metalogiclabs/moteus | Main identical; two additional feature branches | Optional speed-dependent cyclic phase advance |

Actual default comparisons for the 17 non-contributing children return `identical`, ahead 0, behind 0; retained non-default branch identity is also proved by exact SHA equality. This conclusion does not rely on `pushed_at`. Every child-local PR collection returned empty. The originals returned zero ArduPilot PRs and one open moteus PR. Metadata, branch lists, PR endpoints, and comparisons are preserved in the inventory.

[metalogiclabs/moteus PR #1](https://github.com/jyjblrd/moteus/pull/1) is therefore the only discovered unique downstream control contribution. Its clean head `7bd39cf10a045b10c26a32241117e2dad60d08c9` is three commits ahead of original main and changes three files: mapping/configuration, tests, and guide. The earlier feature head `2055adfcc542650dd1bfeffe7385e8bf6096194e` has different history and an extra validation workflow. [PR files](https://api.github.com/repos/jyjblrd/moteus/pulls/1/files); [fork branches](https://api.github.com/repos/metalogiclabs/moteus/branches?per_page=100).

## 3. Control algorithm and timing

**Observed architecture:** ArduPilot retains estimation, attitude control, arming, interlock, and spool-state bookkeeping. MCOUT publishes normalized commands over a one-way UART to moteus. Stabilized roll/pitch combine feedback, feedforward, and configured body-frame trim; throttle is the filtered motor command. An 84-byte little-endian packet carries state, sequence, timestamp, and CRC-16. [Publisher](https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/libraries/AP_MotorControlOutput/AP_MotorControlOutput.cpp#L153-L204).

moteus maps normalized throttle to a rotor-speed target by default, with optional base torque feedforward. The signed roll/pitch vector determines cyclic amplitude and phase:

```text
velocity = max(idle_velocity, throttle * throttle_max_velocity)
amplitude = min(1, sqrt(roll² + pitch²)) * scheduled_cyclic_max
phase = atan2(roll, pitch)/(2π) + configured_phase_offset
cyclic_torque = amplitude * sin(2π * (harmonic*(angle-zero_angle) + phase))
```

The vector saturates in magnitude. Optional amplitude scheduling interpolates with filtered absolute output speed. The sine is evaluated locally against measured output angle, then added to feedforward torque inside the servo control path. Consequently, UART updates specify a waveform rather than individually streaming its samples. The guide attributes thrust-vector tilt to accelerating/decelerating the underactuated rotor within each revolution; this audit does not independently establish its aerodynamic response. [Mapping](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/fw/flight_control_structs.h#L284-L334); [ISR torque](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/fw/bldc_servo_control.h#L1187-L1241).

**PWM and control rates differ above 30 kHz.** `RateConfig` uses interrupt divisor 1 through 30,000 Hz PWM and 2 above it; control rate equals PWM/divisor. The export’s 30 kHz PWM therefore implies a nominal 30 kHz ISR rate, while 60 kHz PWM gives 30 kHz control. These are source-derived nominal rates, not measured execution deadlines or encoder bandwidth. [RateConfig](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/fw/bldc_servo_control.h#L133-L183).

Yaw demands are ignored by the motor mapping: there is no independent yaw authority. The IR beacon supplies an intermittent absolute **body heading measurement**, with gyros propagating between pulses; it does not create yaw actuation. Before the first pulse there is no beacon reference. Receiver delay, field of view, pulse blanking, and changing beacon bearing during translation require calibration. The rotor zero-angle AUX trigger is a separate signal. [Motor guide](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/docs/guides/ardupilot-mcout.md#L3-L18); [IR guide](https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/libraries/AP_IRBeaconYaw/README.md#L18-L30).

Mode 29, `MCOUT_PASS`, sends calibrated sticks directly and bypasses attitude control. It zeros inputs when its RC-valid condition fails, but remains dependent on the surrounding configured receiver/failsafe behavior. It is a useful commissioning mode, not evidence of stabilization. [Implementation](https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/ArduCopter/mode_mcout_pass.cpp#L14-L48).

## 4. Matched configuration and failsafe implications

The exported ArduPilot configuration identifies version 4.6.3, revision `836c4ec5`; it belongs to an older feature baseline. Its `SERIAL3_PROTOCOL=50` matches `new-features`, whereas current master assigns MCOUT **51** after an upstream ID collision. The current moteus guide still says 50. PR #1 corrects that guide to 51, but does not migrate the parameter export. [Export](https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/configs/mcout-working.params#L1-L8); [master enum](https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/libraries/AP_SerialManager/AP_SerialManager.h#L90); [older enum](https://github.com/jyjblrd/ardupilot/blob/7cbdd13b67fc3d9707ad3839fb4bf031875d60f9/libraries/AP_SerialManager/AP_SerialManager.h#L89).

Both exports select 115200 baud, while prose examples use 921600. At 84 bytes and conventional 8N1 framing, 190 actual packets/s would require 159600 bit/s. However, **do not infer that the export necessarily overloads the link**: its 400 Hz scheduler and requested MCOUT rate 190 quantize, under ideal timing, to one packet per three loops, approximately 133.3 Hz and 112000 bit/s. The publisher updates its send timestamp rather than accumulating fractional periods, and skips packets if UART space is insufficient. Measure receiver statistics rather than equating requested and achieved rate. [Scheduling and writes](https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/libraries/AP_MotorControlOutput/AP_MotorControlOutput.cpp#L66-L79); [UART guard](https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/libraries/AP_MotorControlOutput/AP_MotorControlOutput.cpp#L217-L229); [moteus UART export](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/configs/moteus-mcout-working.cfg#L78-L111).

moteus requires fresh valid packets, arm/soft-arm/interlock/output-enabled flags, and emergency-stop clear. Exported packet and servo watchdogs are 100 ms; losing the gate commands stop/coast. Defaults disable MCOUT and leave torque limit zero, but the working export enables it and supplies actuator-specific calibration, gains, phase/inversion, current limits, and a 300 rev/s maximum speed target. That speed is a setting, not measured performance; the 25 Nm command ceiling is not evidence of available motor torque. [Gating](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/fw/flight_control_structs.h#L351-L431); [working settings](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/configs/moteus-mcout-working.cfg#L2104-L2146).

Two integration implications need explicit acceptance. First, fault clearing checks `!engaged && servo_faulted`, not an observed arm-switch edge. Freshness loss sets disengaged; subsequent fresh still-armed packets can therefore clear a latched fault and resume output without a new pilot arm transition. Second, the UART is one-way: actual motor torque/current saturation is not returned through MCOUT to ArduPilot. Whether inherited limit flags and anti-windup adequately represent the physical cyclic actuator requires experiment. These are source-derived behaviors/risks, not observed flight failures. [State transitions](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/fw/flight_control_structs.h#L365-L406); [disengagement](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/fw/flight_control_structs.h#L468-L479).

## 5. Phase-scheduling PR: useful experiment, conditional integration

PR #1 adds three zero-default parameters. With finite values and `0 <= low_speed < high_speed`, it adds a signed, clamped linear phase advance based on absolute measured speed. Invalid scheduling inputs retain the fixed offset. Double-precision interpolation guards extreme finite values; the sine’s existing phase wrapping remains downstream. No MCOUT packet, gating, watchdog, or torque-map change is introduced. Tests cover endpoints, interpolation, negative speed/advance, disabled/malformed inputs, non-finite values, and overflow. [Exact implementation](https://github.com/metalogiclabs/moteus/blob/7bd39cf10a045b10c26a32241117e2dad60d08c9/fw/flight_control_structs.h#L456-L487); [tests](https://github.com/metalogiclabs/moteus/blob/7bd39cf10a045b10c26a32241117e2dad60d08c9/fw/test/flight_control_logic_test.cc#L234-L305).

The linked successful CI is real, but runs in `metalogiclabs/mathgraph`. Its workflow explicitly checks out feature commit `2055adf`, and job metadata confirms successful host tests and target build. The three PR file blobs are **identical** at validated feature head and clean PR head; the inventory records each blob SHA. This supports source/test equivalence for the contribution, not equivalence of every build input. No rotor-response measurement or flight validation is claimed. [Successful run](https://github.com/metalogiclabs/mathgraph/actions/runs/39724720017); [pinned workflow](https://github.com/metalogiclabs/mathgraph/blob/c54cac3cb08134a343a6db37f3476d457f5680db/.github/workflows/moteus-phase-native-validation.yml); [job API](https://api.github.com/repos/metalogiclabs/mathgraph/actions/runs/39724720017/jobs?per_page=100).

**Proposed disposition:** retain it as an opt-in experimental candidate; do not enable compensation or call it a flight improvement without measured phase-versus-speed data. Before integration, fix the guide’s escaped closing code fence, run the exact clean PR tree in the destination’s CI, verify persistent-configuration round trips, and review direction assumptions: an absolute-speed schedule cannot model direction-dependent lag automatically. Apache-2.0 headers remain on the changed source/test; preserve attribution. ArduPilot and moteus retain separate GPL-3.0 and Apache-2.0 licensing contexts. [PR diff](https://api.github.com/repos/jyjblrd/moteus/pulls/1/files).

## 6. Verification boundary and next evidence

Existing custom moteus tests cover protocol framing/CRC/sequence tracking and control mapping, gates, scheduling, and fault clearing. ArduPilot contains a custom IR-beacon SITL test including delay compensation and telemetry. These sit within much larger inherited upstream suites; README badges pointing upstream are not proof that the fork defaults passed. The queried Actions collections returned zero runs for both current default SHAs. [moteus tests](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/fw/test/flight_control_logic_test.cc); [parser tests](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/fw/test/mcout_protocol_test.cc); [IR SITL test](https://github.com/jyjblrd/ardupilot/blob/455b266cc2a73505eb98c56a8049fa6a4d50f9bf/Tools/autotest/arducopter.py#L13830-L13903).

The most valuable reusable pieces are MCOUT’s bounded packet interface, encoder-synchronous torque generation, pure host-testable mapping/gating, and dedicated beacon-yaw integration. The next concrete evidence should be a manifest pinning both firmware SHAs, board/build flags, calibrated encoder/rotor geometry, compatible protocol and UART settings, measured link/ISR timing, failure-transition behavior, and restrained-fixture thrust/phase response. That would convert the present working-example exports into a reproducible experimental baseline.

A focused acceptance record should distinguish these outcomes:

| Proposed check | Evidence to record |
| --- | --- |
| Startup, disarmed, interlock-off, emergency stop | Decoded flags and actual torque-enable state agree |
| UART interruption and restoration while still armed | Stop latency, coast behavior, fault-clear/reengagement policy, and operator expectation agree |
| CPU-side update interruption | Servo watchdog behavior and configured timeout mode are measured separately from link freshness |
| Increasing rotor speed at fixed cyclic command | Measured amplitude, phase, clipping, and current establish the usable envelope |
| Beacon loss, first acquisition, and translation | Heading errors and estimator/mode behavior are documented |

The example’s amplitude schedule has equal low/high torque endpoints, both 0.15 Nm, so its full-command amplitude remains constant despite having a nonzero speed interval. Phase remains a manually tuned 0.27 cycles with pitch inversion. Those values encode a particular assembly and coordinate convention. The documentation mentions r4 AUX2 SER R/T pads under a build flag and the normal onboard magnetic encoder arrangement; neither proves the exact flight-controller or motor-controller model mounted on the pictured aircraft. Hardware identification remains an explicit manifest requirement. [Configuration endpoints](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/configs/moteus-mcout-working.cfg#L2128-L2134); [wiring and encoder prerequisites](https://github.com/jyjblrd/moteus/blob/704a0dd9f30a3704e8915a82c1631301fdbff9bd/docs/guides/ardupilot-mcout.md#L39-L70).

