# The Impossible Build

Configure a compact electric utility vehicle from a 55-part catalog so the
configuration satisfies a coupled set of engineering requirements: gross
mass, range over a duty cycle simulated substep-by-substep with evolving
coolant temperature and battery state of charge, sustained top speed,
gradeability at gross weight, a towing load case, launch acceleration,
thermal limits, and budget. The deliverable is one JSON file naming a
part for each of eight slots. Of the 4,741,632 possible configurations,
exactly 8 satisfy every requirement.

## Task Metadata

- **Author:** Ramaj Duncan (dutch@concord-os.org)
- **Category:** `Science`
- **Tags:** <code>constraint-satisfaction</code> <code>vehicle-engineering</code> <code>physics</code> <code>search</code> <code>configuration</code>
- **Expert time:** 3 hours
- **Agent timeout:** 2 hours
- **CPUs:** 2
- **Memory:** 4 GB

## Difficulty explanation

The difficulty is not any single formula — every formula is published in
`environment/SPEC.md`. It is that the model is **stateful and sequential**,
so nothing can be spot-checked in isolation. The duty-cycle route is
integrated in 210 one-kilometre substeps: coolant temperature rises with
drivetrain losses and is rejected through the radiator, and above 90 °C
the motor's continuous torque envelope derates, weakening every later
substep; battery state of charge falls with every substep's energy, and
the falling pack voltage shrinks the inverter's power budget exactly when
the route's climbing segments demand the most from it. An implementation
that is right at 209 substeps and wrong at one produces a build that
fails — invisibly, because there is no intermediate artifact to inspect.

Around that simulation, the stateless couplings interlock as before: a
motor's torque is an envelope over rpm, so the gearbox ratio that
gradeability demands is checked against falling torque at top speed;
drivetrain efficiency is a bilinear map over speed and torque fraction,
re-evaluated at every operating point; the cooling unit consumes power
in proportion to its own capacity, so a bigger radiator taxes every
substep's energy while it buys thermal headroom; tire load is set by the
heavier axle, rolling resistance grows with that same load, and the tire
rating derates at the top speed the vehicle can actually achieve. On top
of this, a **towing load case** (450 kg trailer, more drag, shifted
center of gravity) re-evaluates the torque, power, launch, tire and
top-speed checks under different thresholds, so a build must close twice,
in two different vehicles. Finally, the search itself must be structured:
a full stateful evaluation of every configuration would take hours, so
candidates must be filtered by cheap necessary conditions before the
simulation is ever run. A solver that implements most of the model
produces a confident, plausible, wrong build: 308,393 configurations pass
a naive datasheet evaluation of this catalog (nameplate torques, one
constant efficiency, cruise-only range, no deratings) and fail the real
specification. Both of the builds that frontier-class models actually
submitted against that earlier specification fail here too, one of them
on five checks. Solving the task means implementing the complete
published model, exactly, in the published integration order, and
letting the couplings close.

## Solution explanation

The reference solution (`solution/solve.py`) loads the catalog,
partitions the search by voltage class (battery, motor, and inverter must
match), and evaluates configurations with the reference physics
(`solution/physics.py`) in stages ordered by cost: mass, cost and
compatibility bookkeeping; the launch and gearbox checks; the grade event
in both load cases; the pointwise rpm checks; the two top-speed scans;
the tire check; and only for the few thousand survivors of those necessary
conditions, the full 210-substep route simulation with its thermal and
state-of-charge integration. It writes the first fully valid
configuration to `/app/build.json` and asserts that the build also passes
a complete unstaged evaluation before exiting. The answer is derived from
the catalog at run time; nothing is hardcoded. The staged search finds a
valid build in about 9 seconds; even a full staged pass over every
voltage-compatible configuration takes under a minute, where a naive
full evaluation of the whole catalog would take hours.

## Verification explanation

The verifier (`tests/`) is an independent reimplementation of the
specification's mathematics that shares no code with the reference
solution. It reads only `/app/build.json`, validates the schema strictly
(exactly the eight slots, every ID present in the ground-truth catalog it
carries), and recomputes all sixteen requirements from first principles —
including its own substep-by-substep route simulation. Reward is binary:
every requirement must hold. The two implementations were cross-checked
against each other on all 8 valid builds, on every single-part mutation
of them (376 cases), on 4,000 random configurations, and on 220
configurations that pass every stateless check (87 of them decided only
by the route simulation), with complete agreement.

## Relevant experience

The author builds verified-compute systems for a living: Concord, a
cognitive engine whose engineering core (ConKay) configures real vehicles
from sourced parts libraries under constraint solvers and acceptance
checks that recompute every claim independently and refuse designs that
do not close. This task distills that acceptance discipline — an answer
is only as good as an independent recomputation of it — into a benchmark
task.
