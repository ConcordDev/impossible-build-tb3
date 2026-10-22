# Vehicle Configuration Specification

You are configuring a compact electric utility vehicle from a fixed parts
catalog. The catalog is at `/app/data/catalog.json`. Your deliverable is a
single JSON file at `/app/build.json` naming exactly one part ID for each
of the eight slots: `chassis`, `battery`, `motor`, `gearbox`, `inverter`,
`cooling`, `tires`, `suspension`.

The catalog lists **nominal (datasheet) values**: peak torque, nameplate
energy, nominal tire load ratings. The requirements below are evaluated on
**derived values** instead: torque that survives the continuous envelope,
the efficiency map, the load-dependent rolling resistance and the thermal
state of the vehicle; energy that survives depth-of-discharge and C-rate
derating; tire capacity derated by speed and by the vehicle's actual axle
loads. Every formula is given in full below: nothing beyond this document
and the catalog is needed to decide whether a build passes.

Several requirements are evaluated in **two load cases** — the vehicle
solo, and the vehicle towing a trailer (section 4) — and the range
requirement is evaluated by a **stateful simulation** of the duty-cycle
route in 1 km substeps (sections 5.7–5.8): coolant temperature and battery
state of charge evolve along the route and feed back into the physics of
every later substep. There is no valid shortcut that evaluates each
operating point in isolation.

## 1. Constants and requirements

Physical constants (also in the catalog's `constants` block):

| Constant | Value |
|---|---|
| Gravity `g` | 9.81 m/s² |
| Air density `rho` | 1.225 kg/m³ |
| Drivetrain (gearbox input) torque cap | part-specific |

Requirements (also in the catalog's `requirements` block):

| Requirement | Limit |
|---|---|
| Gross mass (solo) | ≤ 1,420 kg |
| Total cost | ≤ $14,200 |
| Range on the duty cycle | ≥ 232 km |
| Top speed, solo | ≥ 125 km/h |
| Top speed, towing | ≥ 105 km/h |
| Gradeability | 24% grade at 25 km/h (both cases) |
| Launch acceleration, solo | ≥ 2.2 m/s² |
| Launch acceleration, towing | ≥ 1.6 m/s² |
| Coolant temperature at any point of the route | ≤ 120 °C |
| Battery state of charge at end of route | ≥ 0.10 |

## 2. Mass and cost accounting

**Gross mass** is the sum of the mass of all eight chosen parts:

```
gross_kg = sum of mass_kg over the 8 parts
```

**Total cost** is the sum of `cost_usd` over the 8 parts.

The chosen suspension's `capacity_kg` must be at least the gross mass.
Mass, cost and suspension capacity are properties of the delivered vehicle
and are checked on the solo configuration only; the trailer of section 4
is a load case, not part of the configuration.

## 3. Axle loads, rolling resistance and tire load

The chassis specifies `cg_rear_fraction`: the fraction of the gross mass
carried by the rear axle. Per-tire loads are:

```
rear_axle_kg  = gross_kg * cg_rear_fraction
front_axle_kg = gross_kg - rear_axle_kg
rear_tire_kg  = rear_axle_kg / 2
front_tire_kg = front_axle_kg / 2
governing_load_kg = max(rear_tire_kg, front_tire_kg)
```

The effective rolling-resistance coefficient grows with the load carried
by each tire relative to its nominal rating
(`load_degradation_per_kg = 0.004` per kg of overload, floored at 0):

```
crr_eff = crr * (1 + 0.004 * max(0, governing_load_kg - nominal_load_kg))
```

where `crr` and `nominal_load_kg` are the tire's catalog values. The tire
load check and `crr_eff` are evaluated per load case (section 4): each
case uses its own gross mass and center-of-gravity position.

**Tire load check.** At speed, a tire's load capacity derates. Let `v_top`
be the vehicle's achieved **solo** top speed (section 5.6). The load
factor is:

```
v_top <= 100        -> 1.00
100 < v_top <= 120  -> 0.95
120 < v_top <= 140  -> 0.90
140 < v_top         -> 0.85
```

The check passes in a load case if
`governing_load_kg <= max_load_kg * load_factor` for that case's governing
load. Because `v_top` is itself a derived quantity of the whole build,
this check cannot be evaluated from the tire's datasheet numbers alone.
Note the derating speed is the solo achieved top speed in **both** cases.

## 4. Load cases: solo and towing

Two load cases are evaluated:

| Parameter | Solo | Towing |
|---|---|---|
| Gross mass | `gross_kg` | `gross_kg + 450 kg` (trailer) |
| Drag area `CdA` | chassis `cda_m2` | chassis `cda_m2 + 0.35 m²` |
| `cg_rear_fraction` | chassis value | `min(chassis value + 0.04, 0.65)` |
| Top-speed requirement | ≥ 125 km/h | ≥ 105 km/h |
| Launch requirement | ≥ 2.2 m/s² | ≥ 1.6 m/s² |
| Gradeability requirement | 24% at 25 km/h | 24% at 25 km/h |

The following checks are **dual-case** — they must pass in the solo case
and in the towing case: `launch_accel`, `tire_load`, `top_speed_rpm`,
`top_speed_power`, `grade_torque`, `grade_power`. Each case uses its own
mass, drag area and center of gravity (and therefore its own axle loads
and `crr_eff`). All other checks — `gross_mass`, `suspension_capacity`,
`budget`, `voltage_match`, `gearbox_torque`, `grade_battery_c`,
`thermal_peak`, `route_power`, `soc_floor`, `range` — are evaluated on
the solo vehicle only; in particular the route simulation of sections
5.7–5.8 is a solo-case simulation.

## 5. Drivetrain model

### 5.1 Kinematics

For road speed `v` (m/s), wheel radius `r` (tires) and gear ratio `G`:

```
wheel_rpm = (v / r) * 60 / (2*pi)
motor_rpm = wheel_rpm * G
```

The tractive force required at the tire contact patch for a vehicle of
mass `m`, drag area `CdA`, effective rolling coefficient `crr_eff` and
road grade `grade_pct` is:

```
F = m * g * crr_eff * cos(theta)
  + 0.5 * rho * CdA * v^2
  + m * g * sin(theta)
theta = atan(grade_pct / 100)
```

On level ground (`grade_pct = 0`) this reduces to rolling + aerodynamic
force. The motor torque required to sustain that force is:

```
motor_torque = F * r / (G * gearbox_efficiency)
```

and the mechanical power at the wheels is `P_wheel = F * v / 1000` (kW).

### 5.2 Motor torque envelopes

A motor's catalog `peak_torque_nm` and `cont_torque_nm` are the plateaus
of its torque envelopes. The envelope torque available at motor speed `n`
(rpm) for plateau `T0` and knee speed `base_rpm` is:

```
T_env(n) = T0                          if n <= base_rpm
T_env(n) = T0 * base_rpm / n           if n > base_rpm
```

Two envelopes matter: the **peak** envelope (plateau `peak_torque_nm`)
bounds instantaneous torque; the **continuous** envelope (plateau
`cont_torque_nm`) bounds sustained torque and is the input to the
efficiency map. Inside the route simulation the continuous envelope is
additionally derated by coolant temperature (section 5.8); the stateless
checks of sections 5.6, 5.9 and 5.10 are evaluated cold, with no thermal
derating.

The gearbox has its own cap: motor torque must never exceed the gearbox's
`max_input_torque_nm` (checked against the peak envelope at the launch
operating point, section 5.9; the peak envelope does not depend on mass,
so this check is evaluated once, not per load case).

### 5.3 Auxiliary load and the cooling loop

The cooling unit is a pump/fan loop in **every** operating point of this
specification: the grade event, the top-speed scan, the launch point and
every route substep. The electrical draw of the loop and of the vehicle's
other auxiliary systems is:

```
aux_kw = 0.4 + 0.15 + 0.06 * cooling_capacity_kw
       = vehicle_aux_kw + cooling_draw_base_kw
         + cooling_draw_per_kw * cooling_capacity_kw
```

(`cooling_capacity_kw` is the cooling unit's catalog rating — its radiator
capacity, which also governs heat rejection in section 5.8.) A bigger
radiator therefore costs energy in **every** operating point, not only
when the vehicle is hot.

### 5.4 Efficiency map

Total drivetrain efficiency at an operating point is a product of three
terms: a motor efficiency from the map below, the inverter's
`eff_factor`, and the gearbox `efficiency`.

The motor map is defined on normalized axes: **speed fraction**
`sf = motor_rpm / max_rpm` and **torque fraction**
`tf = motor_torque / T_cont(rpm)`, where `T_cont` is the continuous
envelope of section 5.2 (thermally derated inside the route simulation,
section 5.8). The map is bilinearly interpolated on the grid
`sf, tf ∈ {0.25, 0.6, 1.0}`; fractions outside the grid are clamped to the
nearest grid edge:

| sf \ tf | 0.25 | 0.6 | 1.0 |
|---|---|---|---|
| **0.25** | 0.78 | 0.86 | 0.84 |
| **0.6** | 0.84 | 0.91 | 0.89 |
| **1.0** | 0.80 | 0.87 | 0.83 |

Electrical power drawn from the battery at an operating point, and the
battery C-rate, are:

```
eta_total = eta_motor * inverter_eff_factor * gearbox_efficiency
P_elec    = (P_wheel + aux_kw) / eta_total          [kW]
C_rate    = P_elec / battery_nameplate_kwh          [1/h]
```

### 5.5 Operating points

An **operating point** is the tuple (speed, grade, load case, thermal
state) evaluated through sections 5.1–5.4: required force, motor rpm,
motor torque, envelope limits, efficiency, `P_wheel`, `P_elec`, C-rate.
The stateless checks evaluate isolated cold operating points; the route
simulation chains 210 of them together with evolving state.

### 5.6 Top speed

Top speed is checked in two parts, in each load case:

1. **Pointwise rpm check (`top_speed_rpm`).** At the case's required top
   speed (125 km/h solo, 105 km/h towing), the motor rpm of section 5.1
   must not exceed the motor's `max_rpm`.
2. **Achieved top speed (`top_speed_power`).** The vehicle's achieved top
   speed is the highest integer speed `v` in 60..180 km/h at which the
   vehicle can **sustain** the operating point on level ground. A speed
   is sustainable if all of the following hold:
   - motor rpm at `v` ≤ `max_rpm`;
   - the required motor torque ≤ the continuous envelope `T_cont(rpm)`;
   - the electrical power `P_elec` ≤ the inverter's
     `max_cont_power_kw`.
   The achieved top speed must be at least the case's requirement
   (125 km/h solo, 105 km/h towing).

Note that the pointwise check at the required speed and the scan are not
redundant: a build can turn an acceptable rpm at 125 km/h yet be unable
to sustain 125 km/h, and it can sustain 125 km/h while already violating
the rpm cap at exactly that speed.

### 5.7 The duty-cycle route and range

The duty cycle is a fixed route of four segments (total 210 km):

| Segment | Distance | Speed | Grade |
|---|---|---|---|
| 1 | 90 km | 50 km/h | 0% |
| 2 | 25 km | 40 km/h | 6% |
| 3 | 85 km | 80 km/h | 0% |
| 4 | 10 km | 30 km/h | 12% |

The route is integrated in **1 km substeps**: each segment contributes
`distance` consecutive substeps at that segment's speed and grade — 210
substeps in all. Every substep is an operating point (solo case) whose
electrical power `P_elec` is computed with the **current** thermal and
state-of-charge deratings of section 5.8.

Energy bookkeeping:

```
E_route  = sum over substeps of  P_elec * (1 / speed_kmh)   [kWh]
           (each substep covers 1 km, so its duration is 1/speed hours)
C_worst  = max over substeps of C_rate
usable   = nameplate_kwh * depth_of_discharge * derate(C_worst)
range_km = 210 * usable / E_route
```

The C-rate derating (`c_rate_derate` table) accounts for the capacity a
real pack loses when discharged hard:

```
C <= 0.5   -> 1.00
0.5 < C <= 1.0  -> 0.97
1.0 < C <= 1.5  -> 0.93
1.5 < C <= 2.0  -> 0.88
2.0 < C       -> 0.82
```

Note the derating uses the worst **substep** C-rate along the whole
route — not a per-segment value. The range requirement is
`range_km ≥ 232`.

### 5.8 Stateful thermal and battery model

Two state variables evolve along the route, substep by substep, and feed
back into each substep's operating point:

- **Coolant temperature** `T`, starting at **35.0 °C**.
- **Battery state of charge** `SoC`, starting at **1.00**. SoC is measured
  against the usable capacity `nameplate_kwh * depth_of_discharge`
  (depth of discharge = 0.90), so an `SoC` of 0.50 means half of the
  usable energy remains.

For each substep, in this exact order:

1. **Thermal derating of the continuous envelope.** With the temperature
   `T` entering the substep:

   ```
   f_T = 1.0                                  if T <= 90
   f_T = max(0.70, 1 - 0.006 * (T - 90))      if T > 90
   ```

   Every continuous-envelope torque at this substep is multiplied by
   `f_T` — including the torque used for the efficiency map's torque
   fraction.

2. **Voltage sag.** With the `SoC` entering the substep, the pack voltage
   factor is:

   ```
   vf = 0.84 + 0.16 * SoC
   ```

3. **Operating point.** Compute the substep's operating point (sections
   5.1–5.4) with the derated envelope. The inverter's sustainable power
   at this substep is sag-adjusted:

   ```
   route_power check:  P_elec <= inverter_max_cont_power_kw * vf
   ```

   at **every** substep. (Outside the route — the grade event, the launch
   point, the top-speed scan — the inverter cap is the nameplate
   `max_cont_power_kw`, with no sag factor.)

4. **Thermal update.** The heat generated at the substep is the
   drivetrain loss, `Q = P_elec - P_wheel` (kW); the cooling draw is
   already inside `P_elec` via section 5.3, so it contributes to the
   heat like any other loss. The radiator rejects:

   ```
   rej = cooling_capacity_kw * clamp((T - 35) / 65, 0, 1)
   ```

   With `dt = 3600 / speed_kmh` seconds per 1 km substep and a thermal
   mass of 240 kJ/K:

   ```
   T <- T + (Q - rej) * dt / 240
   ```

5. **SoC update.** With the substep's energy
   `e = P_elec * (1 / speed_kmh)` kWh:

   ```
   SoC <- SoC - e / (nameplate_kwh * depth_of_discharge)
   ```

Constraints tying the simulation together:

- **`thermal_peak`:** the temperature entering every substep, and the
  temperature after the final substep, must be ≤ 120.0 °C.
- **`route_power`:** the sag-adjusted inverter check of step 3 must hold
  at every substep.
- **`soc_floor`:** the SoC after the final substep must be ≥ 0.10.

Because `f_T` degrades the continuous envelope as the vehicle heats up,
and the sag factor shrinks the inverter budget as the pack drains, later
substeps are computed on a weaker vehicle than earlier ones: the order of
integration and the entering-state convention above are part of the
specification, not implementation details.

### 5.9 Launch acceleration

Launch is evaluated at 10 km/h from the **peak** envelope:

```
a = (T_peak(rpm_at_10_kmh) * G * gearbox_efficiency / r) / gross_kg
```

with the load case's gross mass. The check passes if `a ≥ 2.2 m/s²`
(solo) and `a ≥ 1.6 m/s²` (towing). The gearbox input-torque cap is
checked against the same peak-envelope value (once, solo bookkeeping):
`T_peak(rpm_at_10_kmh) ≤ max_input_torque_nm`.

### 5.10 Gradeability event

The gradeability event is a **standalone cold test** — a single operating
point at 25 km/h on a 24% grade, evaluated with no thermal derating and
no voltage sag, in each load case (with that case's mass, drag area,
axle loads and `crr_eff`):

- `grade_torque`: the required motor torque ≤ the continuous envelope
  `T_cont(rpm)` at that point;
- `grade_power`: the electrical power `P_elec` ≤ the inverter's
  `max_cont_power_kw` (nameplate, no sag factor);
- `grade_battery_c`: the battery C-rate at that point ≤ the pack's
  `max_c_rate` (solo case only).

## 6. Checks summary

A build passes iff **all 16** checks pass:

| Check | Meaning | Cases |
|---|---|---|
| `gross_mass` | gross mass ≤ 1,420 kg | solo |
| `suspension_capacity` | suspension capacity ≥ gross mass | solo |
| `budget` | total cost ≤ $14,200 | solo |
| `voltage_match` | battery, motor, inverter share one voltage class | — |
| `gearbox_torque` | peak launch torque ≤ gearbox input cap | once |
| `launch_accel` | launch acceleration ≥ threshold | both |
| `tire_load` | governing per-tire load ≤ derated tire capacity | both |
| `top_speed_rpm` | motor rpm at required top speed ≤ `max_rpm` | both |
| `top_speed_power` | achieved top speed ≥ requirement | both |
| `grade_torque` | 24% @ 25 km/h torque within continuous envelope | both |
| `grade_power` | 24% @ 25 km/h power within inverter cap | both |
| `grade_battery_c` | 24% @ 25 km/h C-rate ≤ pack `max_c_rate` | solo |
| `thermal_peak` | coolant ≤ 120 °C throughout the route sim | solo (route) |
| `route_power` | `P_elec` ≤ sag-adjusted inverter cap at every substep | solo (route) |
| `soc_floor` | SoC ≥ 0.10 after the final substep | solo (route) |
| `range` | duty-cycle range ≥ 232 km | solo (route) |

## 7. Deliverable

Write `/app/build.json` as a JSON object with exactly the eight slot keys,
each naming a part ID from the catalog, e.g.:

```json
{
  "chassis": "CH-A",
  "battery": "BT-1",
  "motor": "MT-1",
  "gearbox": "GB-1",
  "inverter": "IN-1",
  "cooling": "CL-1",
  "tires": "TR-1",
  "suspension": "SU-1"
}
```

Any build satisfying all 16 checks is accepted. Finding one requires
evaluating the physics above — including the stateful route simulation —
across the catalog; datasheet values alone will mislead.
