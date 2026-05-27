# VehPhysics — Project Overview

## What Is VehPhysics

VehPhysics is a dynamic link library that provides high-fidelity bus vehicle physics simulation. It is designed to be consumed by any game engine or simulation environment through a stable, pure C API.

The library does not integrate position or velocity. It acts as a **stateful force calculator**: the host engine owns and integrates the rigid body, and VehPhysics returns the forces and torques to apply each frame based on the current vehicle state, driver inputs, and terrain contact data.

---

## Core Design Principles

### Engine Agnostic

The public API is pure C (`extern "C"`) with no STL types, no exceptions, and no heap allocations visible to the caller. This means the library can be loaded by:

- Unreal Engine (C++ or Blueprint via plugin)
- Unity (C# P/Invoke)
- Godot (GDNative / GDExtension)
- Custom engines (direct `dlopen` / `LoadLibrary`)
- Any language with a C FFI

### The DLL Is a Force Provider, Not a Rigid Body Simulator

The engine is responsible for:
- Maintaining the bus position, orientation, linear velocity, and angular velocity
- Integrating the rigid body each frame using its own physics engine (PhysX, Jolt, Bullet, Havok, custom, etc.)
- Performing raycasts against the terrain

VehPhysics is responsible for:
- Tracking internal state (suspension travel, wheel spin RPM, engine RPM, gear, brake temperature, air spring pressure, etc.)
- Computing the correct forces and torques based on that state and the terrain contact data
- Exposing telemetry for audio, visual, and gameplay systems

This separation means the library works with any physics engine and never competes with it.

### Stable, Versioned C API

The public header declares a semantic version. Breaking changes increment the major version. The engine integration code compiled against `v1.x` will keep working with any `v1.y` release.

### Two Floating-Point Precisions

All public functions are available in two variants:
- `vphf_*` — 32-bit float (typical for game engines)
- `vphd_*` — 64-bit double (large open worlds, high-precision simulation)

The internal simulation always runs in double precision regardless of which API variant is called.

### Vectors as Plain Arrays

All vector and quaternion data in the public structs is passed as `float[3]`, `float[4]`, `float[9]` (3x3 matrix row-major). No engine-specific math types cross the API boundary.

---

## Supported Bus Variants

| Variant | Description | Sections | Axles |
|---|---|---|---|
| `VPH_BUS_CITY` | Standard rigid city bus | 1 | 2 |
| `VPH_BUS_ARTICULATED` | Accordion bus with pivot joint | 2 | 3 |
| `VPH_BUS_DOUBLEDECKER` | Rigid, high center of mass | 1 | 2 |
| `VPH_BUS_TROLLEYBUS` | Electric, no combustion gearbox | 1 | 2 |

Up to 4 axles are supported per configuration. Each axle can be independently configured as driven, steered, and with single or dual rear tires.

---

## Physics Subsystems

| Subsystem | Model Used |
|---|---|
| Tire lateral and longitudinal force | Pacejka Magic Formula |
| Suspension | Spring-damper; air spring with pressure model |
| Engine | Torque curve lookup with turbocharger lag |
| Transmission | Gear ratio table, auto-shift logic, torque converter |
| Brakes | Pneumatic brake model, ABS logic, brake fade by temperature |
| Steering | Ackermann geometry, steering rack ratio |
| Articulation joint | Pivot constraint force + rubber bellows resistance torque |

---

## Build Targets

| Platform | Output | Built By |
|---|---|---|
| Linux | `libvehphysics.so` | Local `cmake --build` or CI |
| Windows | `vehphysics.dll` + `vehphysics.lib` | GitHub Actions (Windows runner or MinGW cross-compile) |

Local development always targets the current host platform. The `.dll` is never cross-compiled by hand — that is the CI's job.

See the GitHub Actions workflows in `.github/workflows/` for the release pipeline.

---

## Lifecycle Summary

```
vphf_init()
  └─ vphf_set_coord_preset(...)
  └─ id = vphf_create_bus(&config)

  Per frame:
    vphf_get_raycast_requests(id, bodyState, requests[])
    ... engine does raycasts ...
    vphf_update(id, dt, bodyState, inputs, results[], forces[])
    ... engine applies forces[].force and forces[].torque ...
    vphf_get_telemetry(id, &telemetry)   // optional

  vphf_destroy_bus(id)
vphf_shutdown()
```

---

## Document Index

| Document | Contents |
|---|---|
| [overview.md](overview.md) | This file — goals, principles, build targets |
| [api-reference.md](api-reference.md) | All public types, structs, and function signatures |
| [architecture.md](architecture.md) | Internal module design and inter-module communication |
| [coordinate-system.md](coordinate-system.md) | Coordinate convention and conversion strategy |
