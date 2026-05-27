# VehPhysics — Coordinate System

## Internal Standard

All physics calculations inside VehPhysics use a single fixed coordinate convention:

| Axis | Direction |
|---|---|
| **+X** | Vehicle forward |
| **+Y** | Vehicle right |
| **+Z** | Up (world up) |

- **Right-handed** coordinate system
- Angles follow the **right-hand rule** around each axis
- This matches the SAE vehicle dynamics convention (J670) used in automotive engineering literature, which is the same convention used by Pacejka Magic Formula

The engine integration code never needs to know this standard exists. Conversion is applied automatically at the API boundary.

---

## Why a Fixed Internal Standard

Every vector quantity that crosses the API (positions, velocities, forces, torques, raycast origins/directions) must be in consistent axes for the physics math to be correct.

Without a standard, every subsystem (tire slip, suspension axis, steering angle direction) would need to parameterize the convention, making the formulas harder to read, harder to verify, and more prone to sign errors.

By enforcing one standard internally and converting only at the boundary, all physics code reads exactly as the textbooks write it.

---

## Supported Engine Presets

| Preset | Forward | Up | Right | Handedness | Engine(s) |
|---|---|---|---|---|---|
| `VPH_COORD_Y_UP_Z_FORWARD` | +Z | +Y | +X | Left | Unity |
| `VPH_COORD_Y_UP_Z_FORWARD` | +Z | +Y | -X | Right | Godot |
| `VPH_COORD_Z_UP_X_FORWARD` | +X | +Z | +Y | Left | Unreal Engine |
| `VPH_COORD_Z_UP_X_FORWARD` | +X | +Z | -Y | Right | Blender |
| `VPH_COORD_Y_UP_Z_BACKWARD` | -Z | +Y | +X | Right | OpenGL convention |
| `VPH_COORD_CUSTOM` | any | any | derived | either | Custom engine |

For custom engines, provide the three basis vectors directly:

```c
float forward[3] = {0, 0, 1};
float up[3]      = {0, 1, 0};
vphf_set_coord_custom(forward, up, VPH_HANDEDNESS_LEFT);
// right is computed as cross(forward, up) for right-handed,
// or cross(up, forward) for left-handed
```

---

## Conversion Matrix Derivation

Given the engine's basis vectors expressed in engine space:

- $\hat{f}$ = forward unit vector
- $\hat{u}$ = up unit vector
- $\hat{r}$ = right unit vector

The internal coordinate standard has basis:

$$\hat{x}_{int} = \text{vehicle forward}, \quad \hat{z}_{int} = \text{up}, \quad \hat{y}_{int} = \text{right}$$

The rotation matrix **from engine space to internal space** $R_{e \to i}$ is built by expressing each internal basis vector in terms of engine basis vectors:

$$R_{e \to i} = \begin{bmatrix} \hat{f} \\ \hat{r} \\ \hat{u} \end{bmatrix}$$

(rows are the internal axes expressed in engine space)

The inverse (internal → engine) is simply the transpose since $R$ is orthonormal:

$$R_{i \to e} = R_{e \to i}^{\top}$$

### Left-Handed Engines

Unity uses a left-handed coordinate system where cross products have the opposite sign. Before applying $R_{e \to i}$, mirror one axis (the right axis is negated) to convert from left-handed to right-handed space.

---

## What Gets Converted

Every vector and quaternion that crosses the API boundary goes through `CoordSystem`. Nothing else does.

### Inputs (engine → internal, in `vphf_get_raycast_requests` and `vphf_update`)

| Field | Conversion |
|---|---|
| `VphBodyState.position` | $R_{e \to i} \cdot p$ |
| `VphBodyState.linearVelocity` | $R_{e \to i} \cdot v$ |
| `VphBodyState.angularVelocity` | $R_{e \to i} \cdot \omega$ |
| `VphBodyState.orientation` (quaternion) | Re-encode quaternion: extract axis-angle, rotate axis by $R_{e \to i}$ |
| `VphWheelRaycastResult.hitPoint` | $R_{e \to i} \cdot p$ |
| `VphWheelRaycastResult.hitNormal` | $R_{e \to i} \cdot n$ |
| `VphDriverInputs.steering` | Sign flip for left-handed engines only |

### Outputs (internal → engine, before writing to caller buffer)

| Field | Conversion |
|---|---|
| `VphWheelRaycastRequest.origin` | $R_{i \to e} \cdot p$ |
| `VphWheelRaycastRequest.direction` | $R_{i \to e} \cdot d$ |
| `VphForceOutput.force` | $R_{i \to e} \cdot F$ |
| `VphForceOutput.torque` | $R_{i \to e} \cdot \tau$ |

---

## Where in the Code Conversion Happens

`CoordSystem` is called from exactly two places:

1. **Entry point of `BusInstance::getRaycastRequests`** — converts incoming `VphBodyState` to internal before any math, and converts each `VphWheelRaycastRequest` to engine space before writing to the caller's buffer.

2. **Entry/exit of `BusInstance::update`** — converts all input structs to internal on entry, converts `VphForceOutput` array to engine space on exit.

No other code in the library touches coordinate conventions.

---

## Verification Approach

The coordinate conversion can be independently verified by:

1. Creating a bus instance at a known position with a known orientation.
2. Calling `vphf_get_raycast_requests` and checking that the returned `direction` vectors point downward in the engine's own convention (negative-Y for Y-up engines).
3. Applying a constant downward gravity force in `VphWheelRaycastResult.hitNormal` (straight up) and zero friction, then verifying the `VphForceOutput.force` from suspension is vertical and upward in engine convention.

These checks can be automated as unit tests in `tests/` and do not require a real engine.

---

## Summary

```
Engine space                    Internal space (SAE Z-up X-forward)
    │                                    │
    │  R_e→i (built from coord preset)   │
    ├───────────────────────────────────►│  All physics math happens here
    │                                    │
    │  R_i→e (= transpose of R_e→i)      │
    │◄───────────────────────────────────┤  Forces come back out here
    │                                    │
```

The engine never sees the internal convention. The physics never sees the engine convention.
