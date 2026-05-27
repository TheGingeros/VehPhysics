# VehPhysics — Internal Architecture

This document describes how the library is structured internally, how each module communicates with others, and the data flow through a complete frame cycle.

---

## Module Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Public C API Layer                           │
│   vphf_init  vphf_create_bus  vphf_get_raycast_requests  vphf_update│
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │      Library        │  src/core/Library.cpp
                    │  (global init/      │  - calls CoordSystem::configure()
                    │   shutdown, coord   │  - owns the Registry singleton
                    │   system config)    │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │      Registry       │  src/core/Registry.cpp
                    │  (ID → BusInstance* │  - allocates BusInstance on create
                    │   map, ID pool)     │  - frees on destroy
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
 ┌────────▼────────┐  ┌────────▼────────┐  ┌────────▼────────┐
 │   BusInstance   │  │  ArticulatedBus  │  │  future types…  │
 │  (rigid body)   │  │  (2-section)    │  │                 │
 └────────┬────────┘  └────────┬────────┘  └─────────────────┘
          │                    │
          │      Both derive from BusInstance base
          │      and compose the same physics modules
          │
          └─────────────────────────────────────────────────────┐
                                                                │
   ┌────────────────────────────────────────────────────────────▼──────┐
   │                        Physics Subsystems                         │
   │                                                                    │
   │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐         │
   │  │  Suspension   │  │  TirePacejka  │  │  Drivetrain   │         │
   │  │  (per wheel)  │  │  (per wheel)  │  │  (per axle)   │         │
   │  └───────────────┘  └───────────────┘  └───────────────┘         │
   │                                                                    │
   │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐         │
   │  │  Transmission │  │    Brakes     │  │   Steering    │         │
   │  │  (1 per bus)  │  │  (per wheel)  │  │  (per axle)   │         │
   │  └───────────────┘  └───────────────┘  └───────────────┘         │
   │                                                                    │
   │  ┌──────────────────────┐   ┌──────────────────────────┐         │
   │  │  ArticulationJoint   │   │     AirSuspension        │         │
   │  │  (articulated only)  │   │  (pressure model,        │         │
   │  └──────────────────────┘   │   kneel logic)           │         │
   │                             └──────────────────────────┘         │
   └────────────────────────────────────────────────────────────────────┘

   ┌────────────────────────────────────────────────────────────────────┐
   │                          Utility Layer                             │
   │                                                                    │
   │  CoordSystem          Math (Vec3/Quat/Mat3)    Interpolation       │
   │  (input/output        (internal operations)   (torque curve,      │
   │   transform)                                   splines)           │
   └────────────────────────────────────────────────────────────────────┘
```

---

## Source Tree Layout

```
src/
  core/
    Library.cpp          ← vphf_init / vphf_shutdown / version string
    Registry.cpp         ← ID pool, ID → BusInstance* hash map
    CoordSystem.cpp      ← input/output vector transforms (see coordinate-system.md)

  vehicle/
    BusInstance.cpp      ← base class: owns all subsystem objects, drives two-pass loop
    ArticulatedBus.cpp   ← subclass: manages two body states + pivot joint

  physics/
    Suspension.cpp       ← spring-damper update, air spring pressure model
    TirePacejka.cpp      ← Magic Formula Fx, Fy, Mz; slip ratio and slip angle
    Drivetrain.cpp       ← engine torque curve, turbo lag, RPM integration
    Transmission.cpp     ← gear ratio lookup, auto-shift logic, torque converter
    Brakes.cpp           ← pneumatic model, ABS logic, temperature / fade
    Steering.cpp         ← Ackermann geometry, per-axle steer angle
    ArticulationJoint.cpp← pivot constraint force, bellows resistance torque

  util/
    Math.cpp             ← Vec3, Quat, Mat3 — strictly internal, never exposed
    Interpolation.cpp    ← piecewise-linear lookup (torque curve), cubic splines

include/
  vehphysics.h           ← umbrella include
  vph_types.h            ← enums, structs used by both float and double API
  vph_api_f.h            ← vphf_* float function declarations
  vph_api_d.h            ← vphd_* double function declarations
```

---

## Module Responsibilities

### `core/Library`

- Holds global state: whether the library is initialized, the `CoordSystem` instance, and the `Registry` instance.
- `vphf_init()` creates the Registry and marks the library ready.
- `vphf_shutdown()` calls `Registry::destroyAll()` then tears down global state.
- `vphf_set_coord_preset()` / `vphf_set_coord_custom()` delegate to `CoordSystem`.

**Does not** know about individual bus instances.

---

### `core/Registry`

- Owns a monotonically incrementing ID counter and a `std::unordered_map<VphInstanceID, BusInstance*>`.
- `allocate(config)` → calls `BusInstance::create(config)`, stores pointer, returns ID.
- `get(id)` → returns pointer or `nullptr` if ID is invalid.
- `free(id)` → destructs the `BusInstance`, removes from map.

**Owns** all `BusInstance` heap allocations.

---

### `vehicle/BusInstance`

Base class. Composes all physics modules. Drives the two-pass frame loop.

**Owns (as value members, not pointers):**
- One `Drivetrain` object
- One `Transmission` object
- One array of `Suspension` objects (one per wheel)
- One array of `TirePacejka` objects (one per wheel)
- One array of `Brakes` objects (one per wheel)
- One array of `Steering` objects (one per steered axle)
- One array of `WheelState` structs (current spin RPM, last contact, etc.)
- One `VphTelemetry` struct (updated in-place each frame)

**Pass 1 (`getRaycastRequests`):**
1. Receives `VphBodyState` (already in internal coords via `CoordSystem::toInternal`).
2. For each wheel: reads `Suspension::currentTravel` + axle config position → computes wheel center in world space.
3. Writes `VphWheelRaycastRequest` array (direction = world-down in internal coords).
4. Does not modify any state.

**Pass 2 (`update`):**
1. Receives body state, inputs, raycast results — all converted to internal coords.
2. For each wheel: calls `Suspension::update(dt, hitDistance, bodyVelocity)`.
3. For each wheel: calls `TirePacejka::computeForces(normalForce, slipRatio, slipAngle, μ)`.
4. Calls `Drivetrain::update(dt, throttle, currentRPM, driveshaftTorque)`.
5. Calls `Transmission::update(dt, inputs.gear, engineRPM)` → gear + output ratio.
6. For each driven wheel: distributes drivetrain output torque.
7. For each wheel: calls `Brakes::update(dt, brake, handbrake, normalForce)`.
8. Sums per-wheel forces and moments → single body-frame force + torque.
9. `CoordSystem::toExternal(outForces)` before writing to caller buffer.
10. Updates `VphTelemetry`.

---

### `vehicle/ArticulatedBus` *(extends BusInstance)*

Overrides the two-pass loop to handle two `VphBodyState` values and two `VphForceOutput` values.

**Additional state:**
- `ArticulationJoint` member.
- Wheel assignments are split between front and rear sections.

**Extra step in Pass 2:**
- Calls `ArticulationJoint::computeConstraintForce(frontState, rearState, dt)`.
- Adds constraint force to each section's output, with equal and opposite signs.
- Adds bellows resistance torque around the yaw axis of the joint.

---

### `physics/Suspension`

**State:** `currentTravel` (m), `prevTravel` (m), `airPressure` (bar, for air spring).

**Input:** `hitDistance` from raycast, current body vertical velocity at wheel attachment point.

**Output:** `springForce` (N, scalar along suspension axis), `damperForce` (N).

For air springs: pressure is updated each frame using a simplified adiabatic model. Kneeling adjusts the target ride height by changing nominal pressure.

The suspension force is what becomes `VphWheelTelemetry.normalForce` (the vertical load, which is also `Fz` input to the Pacejka tire model).

---

### `physics/TirePacejka`

Stateless per-call computation. Takes `Fz` (normal load), longitudinal slip ratio `κ`, slip angle `α`, and surface friction `μ`.

**Pacejka Magic Formula (simplified):**

$$F_x = D_x \cdot \sin\!\left(C_x \cdot \arctan\!\left(B_x \kappa - E_x(B_x \kappa - \arctan(B_x \kappa))\right)\right)$$

$$F_y = D_y \cdot \sin\!\left(C_y \cdot \arctan\!\left(B_y \alpha - E_y(B_y \alpha - \arctan(B_y \alpha))\right)\right)$$

`D` is scaled by `Fz` (normal load sensitivity) and `μ` (surface friction).

**Output:** `Fx` (longitudinal force, N), `Fy` (lateral force, N), `Mz` (aligning torque, N·m).

These forces are in the tire contact-patch frame (X = wheel forward, Y = wheel lateral). `BusInstance` transforms them to world space before summing.

---

### `physics/Drivetrain`

**State:** `currentRPM`, `turboBoost` (bar), `engineTorqueOutput` (Nm).

**Update:**
1. Look up torque from the torque curve at `currentRPM` using `Interpolation::linearLookup`.
2. Apply turbo boost modifier using a 1st-order lag on `turboBoost`.
3. Apply engine braking when throttle = 0.
4. Integrate RPM using the torque balance equation:

$$\dot{\omega} = \frac{T_{engine} - T_{load}}{I_{engine}}$$

**Output:** `driveshaftTorque` (Nm) — passed to `Transmission`.

---

### `physics/Transmission`

**State:** `currentGear`, `clutchEngagement` (0–1), `torqueConverterRatio`.

**Update:**
- For automatic mode: compare `engineRPM` against `upshiftRPM` / `downshiftRPM` thresholds.
- Compute output torque: `T_out = T_in × gearRatios[gear] × finalDrive × clutchEngagement`.
- Distribute `T_out` equally to driven axle wheels (simple open-differential model).

**Output:** Per-wheel drive torque (Nm).

---

### `physics/Brakes`

**State per wheel:** `temperature` (°C), `pneumaticPressure` (bar).

**Update:**
- Pneumatic pressure builds with a time constant (simulates air brake lag).
- Brake torque = `pneumaticPressure × brakeCylinderArea × padFriction × drumRadius`.
- Apply fade: `torque *= fadeFactor(temperature)`.
- ABS: if wheel slip ratio exceeds threshold, release pressure for one ABS cycle interval.
- Integrate temperature: brake energy = `T_brake × |wheelSpinRad/s|`, cool via convection.

**Output:** Per-wheel brake torque (Nm). Subtracted from wheel's net torque.

---

### `physics/Steering`

**State:** `currentSteerAngle` (rad, per steered axle), `steerRackPosition`.

**Update:**
- Map `inputs.steering` (-1..1) through steering rack ratio to rack displacement.
- Convert rack displacement to outer wheel angle.
- Ackermann correction: inner wheel gets a larger angle than outer wheel.

$$\delta_{inner} = \arctan\!\left(\frac{wheelbase}{R - \frac{trackWidth}{2}}\right), \quad
\delta_{outer} = \arctan\!\left(\frac{wheelbase}{R + \frac{trackWidth}{2}}\right)$$

**Output:** Per-wheel steer angle (rad). Used by `BusInstance` to orient the tire contact patch.

---

### `physics/ArticulationJoint`

**State:** `currentAngle` (rad), `angularVelocity` (rad/s).

**Update:**
- Compute angle between front and rear section heading vectors.
- Compute bellows resistance torque: `T = -stiffness × angle - damping × angularVelocity`.
- Compute constraint force to enforce pivot point co-location.

**Output:** Constraint force (N) and torque (N·m) to add to each section's `VphForceOutput`. Front and rear sections receive equal and opposite force.

---

### `util/CoordSystem`

Applied at the API boundary only — nowhere else inside the library.

- Stores a 3×3 rotation matrix from engine space to internal space, and its inverse.
- `toInternal(VphBodyState&)` → rotates `position`, `linearVelocity`, `angularVelocity`; re-encodes `orientation` quaternion.
- `toExternal(VphForceOutput&)` → rotates `force` and `torque` back to engine space.
- `toExternal(VphWheelRaycastRequest&)` → rotates `origin` and `direction` to engine space.

See [coordinate-system.md](coordinate-system.md) for the full derivation.

---

### `util/Math`

Internal-only Vec3, Quat, Mat3 types. Never appear in the public header.

Key operations:
- `Vec3::dot`, `Vec3::cross`, `Vec3::rotate(Quat)`
- `Quat::fromAxisAngle`, `Quat::toMat3`, `Quat::slerp`
- `Mat3::fromBasis`, `Mat3::transpose` (= inverse for orthonormal)

These are plain structs with free functions — no operator overloading that would slow compilation or cause surprises in a C-first codebase.

---

### `util/Interpolation`

- `linearLookup(float* xTable, float* yTable, int count, float x)` — piecewise linear lookup for the engine torque curve and other tables.
- `smoothstep(float t)` — for clutch engagement and kneeling animation curves.

---

## Data Ownership Summary

| Object | Created by | Owned by | Destroyed by |
|---|---|---|---|
| `BusInstance` heap object | `Registry::allocate` | `Registry` | `Registry::free` |
| `Suspension[n]` | `BusInstance` constructor | `BusInstance` | `BusInstance` destructor |
| `TirePacejka[n]` | `BusInstance` constructor | `BusInstance` | `BusInstance` destructor |
| `VphTelemetry` | `BusInstance` | `BusInstance` | `BusInstance` destructor |
| `VphBusConfig` copy | `BusInstance` constructor | `BusInstance` | `BusInstance` destructor |
| All per-frame structs | **Caller (engine)** | Caller | Caller |

The DLL never holds a pointer to caller-allocated memory between frames.

---

## Frame Timing Diagram

```
Engine                              VehPhysics DLL
  │                                       │
  │── vphf_get_raycast_requests() ───────►│
  │                                       │  CoordSystem::toInternal(bodyState)
  │                                       │  for each wheel:
  │                                       │    Suspension::currentTravel → wheelCenter
  │                                       │    CoordSystem::toExternal(request)
  │◄── requests[N] ───────────────────────│
  │                                       │
  │  (engine performs N raycasts)         │
  │                                       │
  │── vphf_update(dt, state, inputs, ────►│
  │              results[N])              │  CoordSystem::toInternal(all inputs)
  │                                       │  Suspension::update(dt)
  │                                       │  TirePacejka::computeForces()   ─┐ per wheel
  │                                       │  Brakes::update(dt)             ─┘
  │                                       │  Drivetrain::update(dt)
  │                                       │  Transmission::update(dt)
  │                                       │  Steering::update(inputs)
  │                                       │  sum forces → body force + torque
  │                                       │  [ArticulatedBus: + joint forces]
  │                                       │  CoordSystem::toExternal(outForces)
  │                                       │  update VphTelemetry
  │◄── forces[M] ─────────────────────────│  (M = 1 rigid, 2 articulated)
  │                                       │
  │  (engine applies forces to body)      │
  │  (engine steps its physics)           │
  │                                       │
  │── vphf_get_telemetry() ─────────────►│  (read-only copy of last telemetry)
  │◄── VphTelemetry ──────────────────────│
```

---

## Error Handling Strategy

- Functions that can meaningfully fail return `VphResult`.
- `vphf_create_bus` returns `VPH_INVALID_ID` instead of a result code since it returns an ID.
- Passing `NULL` pointers or a destroyed ID is a recoverable error — the function returns an error code and is otherwise a no-op (no crash, no UB).
- Invalid config values (e.g. `axleCount = 0`) are caught in `BusInstance` constructor and cause `vphf_create_bus` to return `VPH_INVALID_ID`.
- No exceptions cross the API boundary. All internal exceptions are caught and converted to error codes.

---

## Threading Model

The library is **not thread-safe** across calls to the same instance. Two threads calling `vphf_update` on the **same** `VphInstanceID` simultaneously is undefined behaviour.

Two threads calling `vphf_update` on **different** `VphInstanceID` values is safe, because each `BusInstance` is fully self-contained and shares no mutable state with other instances. The `Registry::get(id)` lookup is protected by a `std::shared_mutex` (reader-writer lock) to allow concurrent reads.

If the engine requires stricter guarantees, wrap each instance update in an external mutex.
