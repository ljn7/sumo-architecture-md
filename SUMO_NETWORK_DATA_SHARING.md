# How network data is stored - and shared between sim and GUI

## 1. Where it lives - the ownership tree

There is **one** copy of the network in memory. Both the simulation and the GUI read/write that same copy through pointers; the GUI does not keep a parallel cache of live state.

```
GUINet  (singleton, accessed via MSNet::getInstance() - derives from MSNet)
 │
 ├── MSVehicleControl* myVehicleControl       ← MSNet.h:897
 │     └── std::map<id, SUMOVehicle*> myVehicleDict      ← MSVehicleControl.h:658
 │           (each entry is a heap-allocated GUIVehicle)
 │
 ├── MSEdgeControl* myEdges                   ← MSNet.h:903
 │     └── MSEdgeVector myEdges               ← MSEdgeControl.h:266 (vector<MSEdge*>)
 │           plus myActiveLanes (std::list<MSLane*>) for the step loop
 │
 ├── MSJunctionControl* myJunctions           ← MSNet.h:905
 │
 ├── MSDetectorControl, MSTLLogicControl,
 │   MSInsertionControl, 3× MSEventControl    ← MSNet.h:907..917
 │
 ├── std::vector<GUIEdge*>             myEdgeWrapper       ← GUINet (GUI-only)
 ├── std::vector<GUIJunctionWrapper*>  myJunctionWrapper   ← GUINet (GUI-only)
 └── SUMORTree                          myGrid             ← spatial index for picking/draw
```

Each `MSEdge` owns its lanes via:

```cpp
std::shared_ptr<const std::vector<MSLane*>> myLanes;       // src/microsim/MSEdge.h:924
```

Each `MSLane` stores its geometry inline (`PositionVector myShape`) and references to the cars currently on it:

```cpp
typedef std::vector<MSVehicle*> VehCont;                   // MSLane.h:119
VehCont myVehicles, myPartialVehicles, myTmpVehicles;      // MSLane.h:1486/1498/1502
```

So the lane only **references** vehicles - the actual `MSVehicle`/`GUIVehicle` objects are owned by `MSVehicleControl::myVehicleDict`, deleted only when `MSVehicleControl::deleteVehicle()` runs.

## 2. How the GUI sees the same data

The GUI variant uses **inheritance**, not composition. Confirmed in headers:

| Class declaration | File |
|---|---|
| `class GUINet  : public MSNet, public GUIGlObject`     | `src/guisim/GUINet.h:82` |
| `class GUIEdge : public MSEdge, public GUIGlObject`    | `src/guisim/GUIEdge.h:51` |
| `class GUILane : public MSLane, public GUIGlObject`    | `src/guisim/GUILane.h:60` |
| `class GUIVehicle : public MSVehicle, public GUIBaseVehicle` | `src/guisim/GUIVehicle.h:52` |
| `class GUIVehicleControl : public MSVehicleControl`    | `src/guisim/GUIVehicleControl.h:44` |

The build-time switch happens in the loader: `GUIEdgeControlBuilder::buildEdge` returns `new GUIEdge(...)`, `addLane` returns `new GUILane(...)`, and `GUIVehicleControl::buildVehicle` returns `new GUIVehicle(...)`. So **every "MSEdge\*" pointer in `MSEdgeControl` actually points at a `GUIEdge`**, and the GUI just down-casts (or relies on the virtual `drawGL` method).

The only true wrapper is `GUIJunctionWrapper`, which holds a reference to an `MSJunction` because junctions have several concrete subclasses (`MSRightOfWayJunction`, `MSInternalJunction`, …) and subclassing them all would be impractical.

**Consequence:** `GUIVehicle::getVisualPosition()` (called from `drawGL`) reads the *live* `MSVehicle::myState` directly. There is no copy step, no snapshot, no per-frame "render data" struct.

## 3. The mutexes - five real ones, plus the queue lock

All locks below are `FXMutex` (FOX-Toolkit's mutex). They form a **layered, mostly disjoint** scheme rather than one big global lock.

### 3.1 `GUIRunThread::mySimulationLock` - *step barrier*

```cpp
FXMutex mySimulationLock;                                  // GUIRunThread.h:184
```

Acquired around the whole step in `GUIRunThread::makeStep` (`GUIRunThread.cpp:193–196`):

```cpp
mySimulationLock.lock();
myNet->simulationStep();
myNet->guiSimulationStep();
mySimulationLock.unlock();
```

Also held during `init()` (when routes are first loaded) and `deleteSim()` (cleanup).
**Who else takes it?** Only the run thread, plus a spin-check in shutdown. The GUI does **not** acquire `mySimulationLock` during drawing - that is a deliberate choice (see §4).

### 3.2 `GUINet::myLock` - *net-internal step guard*

```cpp
mutable FXMutex myLock;                                    // GUINet.h:462
void GUINet::simulationStep() {
    FXMutexLock locker(myLock);                            // GUINet.cpp:243
    MSNet::simulationStep();
}
```

This is partly redundant with `mySimulationLock` (both held during a step), but `GUINet::myLock` also covers code paths reached *not* through `GUIRunThread` - e.g. libsumo calls into `MSNet::simulationStep()` indirectly. Think of it as belt-and-braces.

### 3.3 `GUILane::myLock` - *per-lane fine-grained lock* (the important one for drawing)

```cpp
mutable FXMutex myLock;                                    // GUILane.h:400
```

`GUILane` re-implements two MSLane hooks specifically so the GUI can borrow the vehicle list safely:

```cpp
const MSLane::VehCont& GUILane::getVehiclesSecure() const {  // GUILane.cpp:168
    myLock.lock();                                           //  ← lock acquired here
    return myVehicles;
}
void GUILane::releaseVehicles() const {                      // GUILane.cpp:175
    myLock.unlock();
}
```

Then the lock is acquired around **every** mutation that the run thread performs on the lane and around the draw-side iteration:

* All the methods that change the lane vehicle list lock it: `GUILane.cpp:161, 182, 188, 195, 202, 209, 216, 223, 230, 237, 244` - these are `planMovements`, `executeMovements`, `integrateNewVehicles`, `swapAfterLaneChange`, etc., all inherited from `MSLane` and overridden specifically to insert the lock.
* The drawing code uses the secure pair:

```cpp
const MSLane::VehCont& vehicles = getVehiclesSecure();      // GUILane.cpp:804
... draw each vehicle ...
releaseVehicles();                                          // GUILane.cpp:822
```

This is the **real** synchronisation point between sim and GUI: at the lane level. The simulation can be running freely; whichever thread reaches the lane first holds it, the other waits.

### 3.4 `GUIEdge::myLock` - *per-edge lock for transportables and meso vehicles*

```cpp
mutable FXMutex myLock;                                    // GUIEdge.h:259
```

Used in `GUIEdge::drawGL` to iterate persons, containers, and (in meso mode) vehicles segment-by-segment:

```cpp
FXMutexLock locker(myLock);    // GUIEdge.cpp:376  - drawing persons
FXMutexLock locker(myLock);    // GUIEdge.cpp:384  - drawing containers
FXMutexLock locker(myLock);    // GUIEdge.cpp:401  - meso vehicle segments
```

Inline helpers in `GUIEdge.h:145, 150` lock around add/remove of transportables.

### 3.5 `GUIVehicleControl::myLock` - *per-fleet lock*

```cpp
mutable FXMutex myLock;                                    // GUIVehicleControl.h:123
```

Used so the GUI can iterate the vehicle dictionary (e.g. for the locator or stats) while the run thread inserts/deletes vehicles. The GUI calls `secureVehicles()` / `releaseVehicles()`; you can see the release at `GUIEdge.cpp:443`.

### 3.6 `GUIRunThread::myBreakpointLock` - *unrelated to network data*

```cpp
FXMutex myBreakpointLock;                                  // GUIRunThread.h:190
```

Protects the `std::vector<SUMOTime> myBreakpoints` list, which is mutated by the GUI (when the user adds a breakpoint) and read by the run thread (in `tryStep`, `GUIRunThread.cpp:150–156`).

### 3.7 `MFXSynchQue::myMutex` - *the event queue*

```cpp
mutable FXMutex myMutex;                                   // MFXSynchQue.h:196
```

Protects `myEvents` (`GUIEvent*` queue) on every `push_back / top / pop / empty`. Events carry no live network data, just metadata (step number, message string, end reason).

## 4. The locking strategy in one paragraph

The simulation thread holds `mySimulationLock` + `GUINet::myLock` for the whole step (coarse), but **inside** the step it also acquires the affected `GUILane::myLock` / `GUIEdge::myLock` whenever it touches a lane's vehicle list. The GUI thread does **not** take the coarse locks during drawing - it acquires only the fine-grained per-lane / per-edge locks, via `getVehiclesSecure()` / `releaseVehicles()` and `FXMutexLock locker(myLock)`. This means:

* Within a single lane or single edge, the snapshot rendered is consistent (no half-updated vehicle list).
* Across two lanes, you can momentarily see one lane in its post-move state and another in its pre-move state - that is the deliberate trade-off the design accepts in exchange for never blocking the simulation on the GUI redraw.
* The run thread never blocks waiting for a frame to render: after releasing the locks, it pushes a `GUIEvent_SimulationStep` and immediately starts the next iteration.

## 5. Quick "who-holds-what" matrix

| Resource | Owner | Mutex protecting it | Held by sim during | Held by GUI during |
|---|---|---|---|---|
| `MSNet::myEdges`, `myJunctions`, `myVehicleControl` (the pointers themselves) | `GUINet` (heap) | none - set once at load, never mutated | - | - |
| `MSVehicleControl::myVehicleDict` | `MSVehicleControl` | `GUIVehicleControl::myLock` | `addVehicle`, `deleteVehicle`, `scheduleVehicleRemoval` | locator, stats iteration |
| `MSLane::myVehicles / myPartialVehicles / myTmpVehicles` | `MSLane` (== `GUILane`) | `GUILane::myLock` | `planMovements`, `executeMovements`, `integrateNewVehicles`, lane-change | `GUILane::drawGL` (via `getVehiclesSecure/releaseVehicles`) |
| `MSEdge::myPersons / myContainers` | `MSEdge` (== `GUIEdge`) | `GUIEdge::myLock` | transportable add/remove | `GUIEdge::drawGL` |
| Meso segments on edge | `GUIEdge` | `GUIEdge::myLock` | meso step | `GUIEdge::drawGL` |
| Whole net invariants | `GUINet` | `GUIRunThread::mySimulationLock` + `GUINet::myLock` | entire `simulationStep` | not held by GUI (deliberate) |
| Breakpoint list | `GUIRunThread` | `myBreakpointLock` | `tryStep` check | `onCmdEditBreakpoints`, accept/clear |
| Event queue | `GUIApplicationWindow::myEvents` | `MFXSynchQue::myMutex` | every `push_back` from worker | every `top/pop` in `eventOccurred` |
| `GUIGlObjectStorage::myObjects` (id↔ptr map) | singleton | `GUIGlObjectStorage::myLock` (+ access count for deferred delete) | register/unregister on vehicle create/destroy | `getObjectBlocking(id)` during picking |

The single most important thing to remember: **there is no GUI-side copy of the network**. Both threads work on the same heap objects, and synchronisation is per-lane/per-edge fine-grained, not one global "render lock".
