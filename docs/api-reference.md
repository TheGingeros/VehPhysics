# VehPhysics — Public API Reference

This document describes every type, struct, enum, and function in the public C API.

All symbols are declared in `include/vehphysics.h` (umbrella), with sub-headers for types and precision-specific functions.

---

## Export Macro

```c
// On Windows (MSVC / MinGW)
#ifdef _WIN32
  #ifdef VEHPHYSICS_BUILD
    #define VP_API __declspec(dllexport)
  #else
    #define VP_API __declspec(dllimport)
  #endif
// On Linux / macOS (GCC / Clang)
#else
  #define VP_API __attribute__((visibility("default")))
#endif
```

All public functions are marked `VP_API`. Functions not marked `VP_API` are internal and not part of any stable contract.

---

## Versioning

```c
#define VPH_VERSION_MAJOR 0
#define VPH_VERSION_MINOR 1
#define VPH_VERSION_PATCH 0

VP_API const char* vphf_version_string(void);
// Returns e.g. "0.1.0"
```

---

## Enumerations

### `VphBusVariant`

Selects the bus body configuration. Passed in `VphBusConfig`.

```c
typedef enum {
    VPH_BUS_CITY        = 0,  // rigid body, 2-axle
    VPH_BUS_ARTICULATED = 1,  // 2-section, 3-axle, pivot joint
    VPH_BUS_DOUBLEDECKER= 2,  // rigid body, high CoM
    VPH_BUS_TROLLEYBUS  = 3   // electric drivetrain, no combustion
} VphBusVariant;
```

### `VphCoordPreset`

Named presets for common engine coordinate conventions. Passed to `vphf_set_coord_preset`.

```c
typedef enum {
    VPH_COORD_Y_UP_Z_FORWARD  = 0,  // Unity (left-handed), Godot (right-handed)
    VPH_COORD_Z_UP_X_FORWARD  = 1,  // Unreal Engine, Blender
    VPH_COORD_Y_UP_Z_BACKWARD = 2,  // OpenGL convention
    VPH_COORD_CUSTOM          = 3   // use vphf_set_coord_custom() instead
} VphCoordPreset;
```

### `VphHandedness`

Specifies handedness when using a named preset.

```c
typedef enum {
    VPH_HANDEDNESS_RIGHT = 0,
    VPH_HANDEDNESS_LEFT  = 1   // Unity uses left-handed
} VphHandedness;
```

### `VphResult`

Return code for functions that can fail.

```c
typedef enum {
    VPH_OK                  =  0,
    VPH_ERR_NOT_INITIALIZED = -1,
    VPH_ERR_INVALID_ID      = -2,
    VPH_ERR_INVALID_CONFIG  = -3,
    VPH_ERR_BAD_ARGUMENT    = -4,
    VPH_ERR_OUT_OF_MEMORY   = -5
} VphResult;
```

---

## Configuration Structs

These structs are passed once at instance creation time and are not modified afterwards.

### `VphSuspensionSpec`

Describes one suspension corner. Referenced inside `VphAxleSpec`.

```c
typedef struct {
    float position[3];                // attachment point relative to axle center (m)
    float springRate;                 // N/m
    float damperRate;                 // Ns/m
    float maxTravel;                  // total stroke distance (m)
    float restLength;                 // unloaded spring length (m)

    int   isAirSuspension;            // 0 = coil spring, 1 = air spring
    float airNominalPressure;         // bar — only used when isAirSuspension = 1
    float airSpringStiffnessFactor;   // how pressure affects spring rate
} VphSuspensionSpec;
```

### `VphTireSpec`

Describes tire geometry and Pacejka Magic Formula coefficients.

```c
typedef struct {
    float radius;                     // m
    float width;                      // m
    float mass;                       // kg (unsprung mass contribution)

    // Pacejka Magic Formula — simplified 6-parameter set
    // Index 0 = longitudinal (Fx), Index 1 = lateral (Fy)
    float pacejka_B[2];               // stiffness factor (BCD = cornering stiffness)
    float pacejka_C[2];               // shape factor (1 < C < 2 typically)
    float pacejka_D[2];               // peak value factor (≈ friction coefficient * Fz)
    float pacejka_E[2];               // curvature factor (-1 to 1)

    float rollingResistanceCoeff;     // dimensionless
} VphTireSpec;
```

Default Pacejka coefficients for a generic city bus tire are provided by:
```c
VP_API void vphf_get_default_tire_spec(VphTireSpec* out);
```

### `VphAxleSpec`

Describes one complete axle including its suspension, tires, and steering/drive properties.

```c
typedef struct {
    float position[3];                // axle center relative to body CoM (m)

    int   numTiresPerSide;            // 1 = single, 2 = dual rear
    int   isDriven;                   // 1 = receives drivetrain torque
    int   isSteered;                  // 1 = wheels can steer
    float maxSteerAngle;              // radians (outer wheel, Ackermann-corrected internally)

    VphSuspensionSpec suspension;
    VphTireSpec       tire;
} VphAxleSpec;
```

### `VphEngineSpec`

Describes the combustion engine. For `VPH_BUS_TROLLEYBUS` this struct is ignored; use `VphElectricMotorSpec` instead.

```c
typedef struct {
    // Torque curve as a piecewise-linear lookup table (RPM → Nm)
    float torqueCurveRPM[16];
    float torqueCurveNm[16];
    int   torqueCurveCount;           // number of valid entries (2–16)

    float idleRPM;
    float redlineRPM;
    float maxRPM;                     // fuel cutoff

    float engineBrakeTorque;          // Nm applied when throttle = 0, off clutch
    float engineInertia;              // kg·m² — affects RPM response speed

    int   hasTurbo;
    float turboLagTimeConst;          // seconds — 1st-order lag on boost buildup
    float turboMaxBoostBar;           // additional boost bar at max RPM
} VphEngineSpec;
```

### `VphElectricMotorSpec`

Used only for `VPH_BUS_TROLLEYBUS`.

```c
typedef struct {
    float peakTorqueNm;               // available from 0 RPM
    float peakPowerKW;
    float maxMotorRPM;
    float regenerativeBrakingFactor;  // 0–1, fraction of braking via motor
} VphElectricMotorSpec;
```

### `VphTransmissionSpec`

```c
typedef struct {
    int   numForwardGears;
    float gearRatios[10];             // index 0 = 1st gear
    float reverseRatio;
    float finalDriveRatio;

    int   isAutomatic;                // 1 = DLL handles shift logic
    float upshiftRPM;                 // RPM at which auto shifts up
    float downshiftRPM;               // RPM at which auto shifts down
    float torqueConverterMuliplier;   // 1.0 = fully locked (dry clutch)
    float clutchEngagementRPM;        // RPM at which clutch starts to engage
} VphTransmissionSpec;
```

### `VphBusConfig`

Top-level configuration struct. Passed to `vphf_create_bus`. Fully describes one bus instance.

```c
typedef struct {
    VphBusVariant variant;

    // Body
    float mass;                       // kg, vehicle kerb weight
    float inertiaTensor[9];           // 3×3 row-major, kg·m²
    float centerOfMassOffset[3];      // from geometric center (m)

    // Passenger model
    int   maxPassengers;
    float passengerMassKg;            // per person

    // Axles
    int         axleCount;            // 2–4
    VphAxleSpec axles[4];

    // Drivetrain
    VphEngineSpec         engine;
    VphElectricMotorSpec  electricMotor; // only read when variant == TROLLEYBUS
    VphTransmissionSpec   transmission;

    // Articulated bus only — ignored for other variants
    float pivotPositionFront[3];      // pivot attachment on front section (m from CoM)
    float pivotMaxAngle;              // radians, max yaw at joint
    float bellowsStiffness;           // Nm/rad — rubber bellows resistance
    float bellowsDamping;             // Nm·s/rad
} VphBusConfig;
```

---

## Per-Frame Input/Output Structs

These are allocated by the caller and passed by pointer every frame.

### `VphBodyState`

Snapshot of one rigid body section's current state, supplied by the engine.

```c
typedef struct {
    float position[3];                // world space (m)
    float orientation[4];             // quaternion, XYZW
    float linearVelocity[3];          // world space (m/s)
    float angularVelocity[3];         // world space (rad/s)
    float passengerLoad;              // 0.0–1.0 (fraction of maxPassengers)
} VphBodyState;
```

> For articulated buses: pass an array of 2. Index 0 = front section, index 1 = rear section.

### `VphDriverInputs`

```c
typedef struct {
    float throttle;                   // 0.0–1.0
    float brake;                      // 0.0–1.0
    float steering;                   // -1.0 (full left) to +1.0 (full right)
    int   gear;                       // -1=reverse, 0=neutral, 1–N=forward, 99=auto
    float handbrake;                  // 0.0–1.0
    int   kneeling;                   // 1 = activate air-suspension kneel for boarding
    float retarder;                   // 0.0–1.0 (exhaust/hydraulic engine brake)
} VphDriverInputs;
```

### `VphWheelRaycastRequest`

Returned by Pass 1. One entry per wheel.

```c
typedef struct {
    float origin[3];                  // world space cast start (wheel center)
    float direction[3];               // unit vector, downward in internal convention
    float maxDistance;                // max suspension travel + tire radius (m)
    float wheelRadius;                // m — for engine-side sphere-cast if desired
} VphWheelRaycastRequest;
```

### `VphWheelRaycastResult`

Passed back in Pass 2. One entry per wheel, matching the order of `VphWheelRaycastRequest`.

```c
typedef struct {
    int   hit;                        // 1 = terrain hit, 0 = wheel in the air
    float hitPoint[3];                // world space
    float hitNormal[3];               // unit vector
    float hitDistance;                // m from origin
    float frictionCoefficient;        // μ of the surface material (e.g. 0.8 for asphalt)
    float rollingResistanceMod;       // multiplier on top of tire's base value
} VphWheelRaycastResult;
```

### `VphForceOutput`

Returned by Pass 2. One entry per rigid body section.

```c
typedef struct {
    float force[3];                   // world space, Newtons — apply at CoM
    float torque[3];                  // world space, N·m
} VphForceOutput;
```

---

## Telemetry Structs

### `VphWheelTelemetry`

```c
typedef struct {
    float suspensionTravel;           // m — 0 = full droop, maxTravel = full compression
    float suspensionForce;            // N — current spring+damper force
    float wheelSpinRPM;               // positive = forward roll
    float steerAngle;                 // radians
    float longitudinalSlip;           // dimensionless (0–1 range nominally)
    float lateralSlipAngle;           // radians (slip angle α)
    float normalForce;                // N — ground reaction
    float brakeTorque;                // Nm currently applied
    float brakeTemperature;           // °C — for fade model and visual effects
    int   hasContact;                 // 1 = in contact with terrain
} VphWheelTelemetry;
```

### `VphTelemetry`

```c
typedef struct {
    // Drivetrain
    float engineRPM;
    float engineTorqueNm;
    int   currentGear;
    float clutchSlip;                 // 0 = fully engaged, 1 = fully slipping
    float turboBoostBar;
    float speedMs;                    // forward speed (m/s), can be negative

    // Air suspension
    float kneelingProgress;           // 0.0–1.0 for boarding animation
    float frontRideHeight;            // m, current air spring height
    float rearRideHeight;

    // Articulated bus joint
    float articulationAngle;          // radians — 0 for non-articulated

    // Per-wheel data
    int               wheelCount;
    VphWheelTelemetry wheels[8];      // indexed same order as axles[].numTiresPerSide
} VphTelemetry;
```

---

## Function Reference

### Library Lifecycle

```c
// Initialize the library. Must be called once before any other function.
// Returns VPH_OK or an error code.
VP_API VphResult vphf_init(void);

// Release all resources. Destroys all remaining instances.
VP_API void vphf_shutdown(void);

// Returns a null-terminated version string, e.g. "0.1.0".
VP_API const char* vphf_version_string(void);
```

### Coordinate System Setup

Call once after `vphf_init`, before creating any instances.  
See [coordinate-system.md](coordinate-system.md) for details.

```c
// Use a named preset.
VP_API VphResult vphf_set_coord_preset(
    VphCoordPreset  preset,
    VphHandedness   handedness
);

// Use a fully custom basis, all unit vectors in the engine's own space.
// forward: the direction the vehicle's nose points
// up:      the world up direction
// right:   computed internally as cross(forward, up) for right-handed
VP_API VphResult vphf_set_coord_custom(
    const float forward[3],
    const float up[3],
    VphHandedness handedness
);
```

### Instance Management

```c
// Allocate a new bus instance. Returns a non-negative ID on success,
// or VPH_INVALID_ID (-1) on failure.
VP_API VphInstanceID vphf_create_bus(const VphBusConfig* config);

// Release all memory for this instance. The ID becomes invalid.
VP_API void vphf_destroy_bus(VphInstanceID id);

// Reset internal state (suspension, RPM, gear, etc.) to initial conditions.
// Does not change the configuration.
VP_API VphResult vphf_reset_bus(
    VphInstanceID        id,
    const VphBodyState*  initialState  // one entry (or two for articulated)
);

// Provide default Pacejka coefficients for a generic city bus tire.
VP_API void vphf_get_default_tire_spec(VphTireSpec* out);
```

### Two-Pass Update Loop

#### Pass 1 — Raycast Requests

```c
// Compute wheel world positions and return raycast requests.
// Call this BEFORE doing raycasts on the engine side.
//
// id           — instance to query
// bodyStates   — array of 1 (rigid) or 2 (articulated) current body states
// outRequests  — caller-allocated array of at least (total wheel count) entries
// maxRequests  — capacity of outRequests
//
// Returns number of entries written (== total wheel count), or negative VphResult on error.
VP_API int vphf_get_raycast_requests(
    VphInstanceID                  id,
    const VphBodyState*            bodyStates,
    VphWheelRaycastRequest*        outRequests,
    int                            maxRequests
);
```

#### Pass 2 — Physics Update

```c
// Run the full physics update and output forces to apply.
// Call this AFTER receiving raycast results from the engine.
//
// id             — instance to update
// deltaTime      — frame time in seconds
// bodyStates     — same array passed to Pass 1, updated to current frame state
// inputs         — current driver inputs
// raycastResults — array matching the requests from Pass 1, same count and order
// raycastCount   — must equal the return value from the preceding Pass 1 call
// outForces      — caller-allocated array of 1 (rigid) or 2 (articulated) entries
//
// Returns VPH_OK or an error code.
VP_API VphResult vphf_update(
    VphInstanceID                   id,
    float                           deltaTime,
    const VphBodyState*             bodyStates,
    const VphDriverInputs*          inputs,
    const VphWheelRaycastResult*    raycastResults,
    int                             raycastCount,
    VphForceOutput*                 outForces
);
```

### Telemetry

```c
// Fill out with the latest telemetry from the last vphf_update call.
// Safe to call at any time between frames; data is from the last completed update.
VP_API VphResult vphf_get_telemetry(
    VphInstanceID  id,
    VphTelemetry*  out
);
```

---

## Double-Precision Variants

Every function above has an exact `vphd_*` counterpart. The struct types are separate (`VphBodyStateD`, `VphWheelRaycastRequestD`, etc.) with all `float` fields replaced by `double`. The enum types (`VphBusVariant`, `VphCoordPreset`, etc.) are shared.

```c
VP_API VphResult     vphd_init(void);
VP_API void          vphd_shutdown(void);
VP_API VphResult     vphd_set_coord_preset(VphCoordPreset preset, VphHandedness handedness);
VP_API VphInstanceID vphd_create_bus(const VphBusConfigD* config);
VP_API void          vphd_destroy_bus(VphInstanceID id);
VP_API int           vphd_get_raycast_requests(VphInstanceID id,
                         const VphBodyStateD* bodyStates,
                         VphWheelRaycastRequestD* outRequests, int maxRequests);
VP_API VphResult     vphd_update(VphInstanceID id, double deltaTime,
                         const VphBodyStateD* bodyStates,
                         const VphDriverInputs* inputs,
                         const VphWheelRaycastResultD* raycastResults, int raycastCount,
                         VphForceOutputD* outForces);
VP_API VphResult     vphd_get_telemetry(VphInstanceID id, VphTelemetryD* out);
```

> The float and double instance registries are independent. IDs from `vphf_create_bus` cannot be passed to `vphd_*` functions.

---

## Usage Contract

1. `vphf_init()` must be the first call. `vphf_shutdown()` must be the last.
2. `vphf_set_coord_preset()` must be called before `vphf_create_bus()`.
3. Pass 1 (`vphf_get_raycast_requests`) must be called before Pass 2 (`vphf_update`) every frame, in that order.
4. The `raycastCount` argument to `vphf_update` must equal the return value from the preceding `vphf_get_raycast_requests` call on the same instance.
5. Calling any function with an invalid or destroyed `VphInstanceID` returns `VPH_ERR_INVALID_ID` and is otherwise a no-op.
6. The library is **not thread-safe** across instances by default. External synchronization is required if multiple threads update different instances concurrently.
