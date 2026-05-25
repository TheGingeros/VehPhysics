# VehPhysics
Dynamic link library for vehicle physics simulation

Currently work in progress.

## Plan Notes:
- expose the library using a pure C interface so that the DLL is independant to engines
- passing vectors stricly as `float[3]` or `double[3]`
- compile both to .dll and .so

### Two pass data oriented update loop
- the game engine drives the tick
- first pass:
- game engine calls DLL function like `GetWheelRaycastRequests()`
- DLL calculates where the bus's wheels currently are in the WS based on the suspension state and returns an array of origins and directions
- engine takes these coords and does raycasts against the terrain and packages the result
- secind pass:
- engine calls the main update function `UpdateBusPhysics(DeltaTime, Inputs, RaycastResults)`
- DLL takes inputs and the terrain hit data
- DLL calculates engine RPM, air suspension etc.
- DLL outputs the resulting force and torque vectors
- engine takes these forces and applies them on its side

### Instance managment - multiple busses in the scene
- simulating traffic or multiple players
- init: engine calls `CreateBusInstance()`, DLL allocates memory for the new bus and returns a unique ID to the engine
- Exec: every frame, engine passes ID to DLL so the DLL knows exactly which bus's memory to update
- CleanUp: engine callsd `destroyBusInstance(ID)` so the DLL can free memory

### Coords system conversion
- different engines use different coords systems
- dll establishes strict internal coord standard
- on start, dll configures which engine its talking to and applies the necesarry conversion on the vectors before calculating math and invert them back before sending the forces out