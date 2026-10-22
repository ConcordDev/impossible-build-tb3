# Impossible Build — failure analysis (DRAFT, for the submission repo)

Status: draft prepared 2026-10-15 from the verified trial records in
`validation-receipt.json` and `night-watch-state.md`. Official trials
(Claude Code ×3 + cheat, Codex ×3 + cheat, on physics v3) are still to run;
their results will be added before submission. The bonus trials below were
run by the task author against the two strongest models reachable on his
Ollama Cloud subscription — they are a calibration floor, not the official
record.

## Trial record

| Round | Model | Result | Runtime | Submitted build | Failed checks |
|---|---|---|---|---|---|
| v1 | glm-5.3:cloud | PASS 1.0 (16/16) | 3m50s | CH-F/BT-5/MT-4/GB-6/IN-3/CL-2/TR-2/SU-4 | — |
| v1 | gpt-oss:120b-cloud | PASS 1.0 (16/16) | 2m34s | CH-F/BT-5/MT-4/GB-3/IN-3/CL-2/TR-5/SU-4 | — |
| v2 | glm-5.3:cloud | PASS 1.0 (18/18) | 8m13s | CH-C/BT-6/MT-4/GB-3/IN-4/CL-4/TR-5/SU-1 | — |
| v2 | gpt-oss:120b-cloud | FAIL 0.0 (14/18) | 1m23s | CH-B/BT-2/MT-2/GB-4/IN-2/CL-2/TR-3/SU-2 | top_speed_rpm, cooling, cooling_climb, range |
| v3 | gpt-oss:120b-cloud | FAIL 0.0 (15/18) | 2m27s | CH-F/BT-5/MT-5/GB-2/IN-4/CL-4/TR-6/SU-2 | grade_torque, soc_floor, range |
| v3 | glm-5.3:cloud | FAIL 0.0 (14/18) | 15m12s | CH-C/BT-5/MT-4/GB-2/IN-4/CL-2/TR-2/SU-4 | tire_load, grade_torque, soc_floor, range |

One further v3 attempt (glm-5.3) was cancelled after 28 minutes when a
single cloud API request hung with no agent activity; per TB3 rules an
infrastructure-interrupted run is not a trial and is not counted. The
retry above is the genuine attempt.

## Why v1 fell

SPEC.md publishes every formula in full (by design — the task tests
evaluation, not guessing). Against v1's stateless, per-operating-point
physics, faithful transcription of the spec into a solver plus exhaustive
search over 486k voltage-compatible builds was sufficient: both models
found members of the 7-build valid set in under 4 minutes. The
datasheet-naive trap (453 builds) only catches hand-reasoning; it does
not catch an agent that implements the spec exactly. The valid set also
sat in a single component family, so partial correctness could still
land on a winner.

## Why v2 split the field

v2 kept the physics stateless but interlocked it: motor torque envelopes
evaluated at the operating rpm, a per-point efficiency map, duty-cycle
range with worst-segment C-rate derating, a cooling loop whose own power
draw feeds the energy budget, axle-load tire checks, load-dependent
rolling resistance, and an implicit top-speed solve. That stopped
gpt-oss cold (it missed exactly the interlocking mechanisms). It did not
stop GLM-5.3: the model wrote a faithful validator, caught and fixed its
own interpolation edge bug mid-run, and searched all 2.54M combinations.
Lesson: stateless physics, however interlocked, is still a transcription
task; a strong enough transcriber with enough search budget wins.

## Why v3 holds (against the bonus pair)

v3 changes the kind of difficulty in three ways:

1. **Statefulness.** The range requirement is evaluated by a simulation
   over 210 one-km substeps in which coolant temperature and battery
   state of charge evolve and feed back into the physics of every later
   substep (thermal derating of the motor's continuous envelope; voltage
   sag shrinking the inverter's power cap). Errors compound along the
   route and cannot be spot-checked at isolated operating points.
2. **Search economics.** A full evaluation costs ~1.44 ms, so naive
   exhaustive search over 4,741,632 combinations takes ~1.9 hours — the
   whole agent window. Only 4,474 builds (0.38%) survive the cheap checks
   to reach the route simulation, so the agent must invent staged
   pruning to search at all.
3. **Two vehicles at once.** A towing load case (+450 kg, +0.35 m² drag,
   shifted center of gravity) re-evaluates grade torque/power, launch,
   tire load and both top-speed checks. A build must close in both load
   cases simultaneously.

Evidence this is what stopped the agents: both v3 submissions fail the
route simulation's energy pair (`soc_floor`, `range`) and both fail
towing `grade_torque`. GLM-5.3 passed the new thermal checks
(`thermal_peak`, `route_power`) — its thermal model was right — and
*still* lost on energy accounting, after spending more tokens (591,648
input) than its entire winning v2 run. Both submissions were well-formed
(14–15 of 18 verifier tests passed), so the verifier is measuring the
intended physics skill, not formatting compliance. Neither submission
resembles any of the 8 valid builds.

## Calibration notes

- Exactly 8 valid builds of 4,741,632 combinations (1,185,408
  voltage-compatible), spanning both voltage families and three motor
  families; tightest winning margin 0.20% (budget), so the valid set is
  findable but not guessable.
- 308,393 builds satisfy a datasheet-naive evaluation (nameplate values,
  constant efficiency, no state) yet fail the real verifier — the trap
  the task is built around. (Methodology, including why a literal
  v2-physics trap count is structurally 0 on this catalog, is documented
  in `calibration-receipt.json` and `physics-v3-design.md`.)
- The verifier is an independent reimplementation of the spec (no code
  shared with the oracle solver); the two implementations agree on
  4,000/4,000 random builds, 376/376 single-part swaps, and 220/220
  stateless survivors.
