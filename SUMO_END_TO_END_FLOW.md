# SUMO‑GUI: from launch to a running simulation

A compact end‑to‑end walkthrough - what happens when you start sumo‑gui, open a network, click Play - with a deep look at how the simulation thread and the GUI thread share network data.

---

## 1. The setup: three threads, one network

SUMO‑GUI is a multi‑threaded program with **three threads** of interest:

| Thread | Created where | Job |
|---|---|---|
| **GUI thread** (FOX main) | `application.run()` in `src/guisim_main.cpp:98` | Draws the window, handles mouse/keyboard, drains events |
| **Load thread** (`GUILoadThread`) | `GUIApplicationWindow::dependentBuild`, `src/gui/GUIApplicationWindow.cpp:342` | Parses XML files and builds the network in memory |
| **Run thread** (`GUIRunThread`) | same place, `src/gui/GUIApplicationWindow.cpp:343` | Advances the simulation one step at a time |

They all talk via **one** shared object: a thread‑safe queue of `GUIEvent*` (`MFXSynchQue<GUIEvent*> myEvents` on `GUIApplicationWindow.h:465`). The queue moves *control messages*, not network data. The actual network sits on the heap and is read by all three threads through pointers.

The core fact you need to remember:

> **There is exactly one copy of the network in memory.** The GUI does not keep a parallel cache of vehicle positions or lane state. When the renderer draws a vehicle, it reads the *live* `MSVehicle` object that the simulation thread is updating, with per‑lane locks providing safety.

The rest of this document explains how that works.

---

## 2. Opening the app

`src/guisim_main.cpp:53` is the `main()` function. It:

1. Initialises subsystems (`XMLSubSys::init`, `MSFrame::fillOptions`, `OptionsIO::getOptions`).
2. Constructs `FXApp application(...)` on the stack (line 76) - the FOX‑Toolkit application object.
3. Checks OpenGL is available (`FXGLVisual::supported`, line 80).
4. Creates the main window: `GUIApplicationWindow* window = new GUIApplicationWindow(&application)` (line 85).
5. Calls `window->dependentBuild(false)` (line 88) - this is where everything is wired up.
6. Calls `application.create()` (line 91) and finally `application.run()` (line 98), which enters FOX's event loop. From here on, every action is driven by an event.

Inside `dependentBuild` (`GUIApplicationWindow.cpp:284‑352`):

* The menu bar, eight toolbars, status bar, splitter, MDI client (the area that holds view windows) and message log are constructed.
* Two `FXEX::MFXThreadEvent` objects are wired to message IDs `ID_LOADTHREAD_EVENT` and `ID_RUNTHREAD_EVENT` (lines 298‑301). These are the "doorbells" the worker threads use to wake the GUI.
* The two worker threads are created (`new GUILoadThread(...)`, `new GUIRunThread(...)` - lines 342‑343) and the run thread is started immediately (`myRunThread->start()`, line 349).

The run thread is now *alive but idle*. Its main loop (`GUIRunThread::run`, `GUIRunThread.cpp:126`) calls `tryStep()` in a loop, but `tryStep` checks a flag `myHalting` and, since no network is loaded yet, it just sleeps 50 ms and tries again. This is the situation when you first see the empty sumo‑gui window.

---

## 3. Opening a network or .sumocfg

You click **File ▸ Open Simulation…** (Ctrl+O). What happens:

```
onCmdOpenConfiguration                  GUIApplicationWindow.cpp:1065
  → FXFileDialog                        (FOX shows a file picker)
  → loadConfigOrNet(file)               GUIApplicationWindow.cpp:2206
       myAmLoading = true
       closeAllWindows()
       myLoadThread->loadConfigOrNet(file)
            → start()                   ← spawns the load thread
                 → GUILoadThread::run()  GUILoadThread.cpp:84
```

`GUILoadThread::run` does five things, in order:

1. **Parse the .sumocfg** (lines 94‑149). `OptionsCont::setByRootElement` reads the XML, `OptionsIO::getOptions` fills the global option container.
2. **Allocate the vehicle controller** (line 155):
   ```cpp
   MSVehicleControl* vehControl = MSGlobals::gUseMesoSim
       ? (MSVehicleControl*) new GUIMEVehicleControl()
       : (MSVehicleControl*) new GUIVehicleControl();
   ```
   The crucial detail is that the GUI build picks `GUIVehicleControl`, which overrides `buildVehicle` to produce `GUIVehicle` objects instead of `MSVehicle` objects.
3. **Allocate the network**:
   ```cpp
   GUINet* net = new GUINet(vehControl,
                            new GUIEventControl(),
                            new GUIEventControl(),
                            new GUIEventControl());
   ```
4. **Build the road graph** by parsing the `.net.xml` through `NLBuilder` and a chain of GUI builders:
   * `GUIEdgeControlBuilder::buildEdge` returns `new GUIEdge(...)` (`GUIEdgeControlBuilder.cpp:65‑70`)
   * `GUIEdgeControlBuilder::addLane` returns `new GUILane(...)` (`.cpp:47‑61`)
   * `GUIDetectorBuilder` and `GUITriggerBuilder` similarly produce `GUIInductLoop`, `GUIE2Collector`, `GUIBusStop`, etc.
   * `NLBuilder::build()` (`src/netload/NLBuilder.cpp:125`) drives `XMLSubSys::runParser(NLHandler, …)` - a SAX parser that calls back into the builders.
5. **Build the GUI spatial index** (`GUINet::initGUIStructures`, `GUINet.cpp:276‑360`): wraps each junction in a `GUIJunctionWrapper`, registers every drawable object in `myGrid` (an R‑tree), computes the global bounding box.

When all this succeeds, the load thread posts a single message to the GUI:

```cpp
GUIEvent* e = new GUIEvent_SimulationLoaded(net, simStartTime, simEndTime, ...);
myEventQue.push_back(e);          // GUILoadThread.cpp:248 - mutex-protected push
myEventThrow.signal();            // .cpp:249 - wakes the GUI
```

On the GUI thread, FOX dispatches `onLoadThreadEvent` → `eventOccurred` → `handleEvent_SimulationLoaded` (`GUIApplicationWindow.cpp:1847`). That handler:

* Hands the network to the run thread: `myRunThread->init(net, begin, end)`. This sets `myRunThread->myNet = net`. Note: the run thread does **not** own the network - the load thread allocated it, and `GUIRunThread::deleteSim` will be the one to delete it later.
* Calls `openNewView(...)` which creates a `GUISUMOViewParent` MDI child containing a `GUIViewTraffic` (the OpenGL canvas).

At this point all three threads have a `GUINet*` pointing at the same heap object. Run thread is still idle (`myHalting == true`).

---

## 4. Clicking Play

```
onCmdStart                          GUIApplicationWindow.cpp:1277
  → myRunThread->begin()            (first time only - log "simulation started")
  → myRunThread->resume()           myHalting = false; mySingle = false
```

That one boolean flip is the entire start mechanism. There is no condition variable, no wakeup signal - the run thread is already spinning in `tryStep`, and on its next iteration it sees `myHalting == false` and falls into the step path:

```cpp
// GUIRunThread::makeStep, .cpp:186-253
mySimulationLock.lock();
myNet->simulationStep();          // the real microsim step
myNet->guiSimulationStep();       // updates per-step GUI counters
mySimulationLock.unlock();

myEventQue.push_back(new GUIEvent_SimulationStep());
myEventThrow.signal();             // ring the GUI doorbell
```

After `signal()`, the run thread sleeps for `mySimDelay × 1000 − step duration` ms (the visual delay slider) and then goes round again. **It does not wait for the GUI to finish drawing.**

Meanwhile the GUI thread wakes, drains the event queue, sees `SIMULATION_STEP`, calls `handleEvent_SimulationStep` (`.cpp:2021`), which updates the time LCD and counters, then calls `update()` on each view - which schedules a `SEL_PAINT` on the next FOX iteration. The actual painting happens in `GUISUMOAbstractView::onPaint` → `paintGL` → `GUIViewTraffic::doPaintGL` → `myGrid.Search(viewport, settings)`, which iterates visible objects and calls `drawGL` on each.

That's the whole control flow: **load thread builds the world → run thread mutates it step by step → GUI thread reads it during repaint, locking only the lanes it's currently drawing.**

---

## 5. Data sharing - the deep dive

This is the question that matters most: when `GUILane::drawGL` runs on the GUI thread and the run thread is busy moving vehicles around, *how do they not corrupt each other?*

### 5.1 Inheritance, not copying

The GUI build replaces every microsim class with a subclass that also inherits `GUIGlObject`:

| Microsim base | GUI subclass | Header |
|---|---|---|
| `MSNet` | `GUINet : public MSNet, public GUIGlObject` | `src/guisim/GUINet.h:82` |
| `MSEdge` | `GUIEdge : public MSEdge, public GUIGlObject` | `src/guisim/GUIEdge.h:51` |
| `MSLane` | `GUILane : public MSLane, public GUIGlObject` | `src/guisim/GUILane.h:60` |
| `MSVehicle` | `GUIVehicle : public MSVehicle, public GUIBaseVehicle` | `src/guisim/GUIVehicle.h:52` |
| `MSVehicleControl` | `GUIVehicleControl : public MSVehicleControl` | `src/guisim/GUIVehicleControl.h:44` |

Because of this, every `MSEdge*` pointer in `MSEdgeControl::myEdges` is *actually* a `GUIEdge*`, every `MSLane*` is a `GUILane*`, every vehicle in `myVehicleDict` is a `GUIVehicle*`. The run thread sees them as their microsim base class (`MSEdge`, `MSLane`, `MSVehicle`); the GUI thread sees them as their GUI subclass (`GUIEdge`, `GUILane`, `GUIVehicle`). **Same heap objects, two views.**

The only true wrapper is `GUIJunctionWrapper`, which holds a reference to an `MSJunction`. Junctions have several concrete subclasses (`MSRightOfWayJunction`, `MSInternalJunction`, …) so subclassing them all would be impractical - instead `GUINet::myJunctionWrapper` stores a parallel vector of wrappers.

When the renderer asks `GUIVehicle::getVisualPosition()` for a position, it walks straight into `MSVehicle::myState`. There is no copy, no per‑frame snapshot, no "render data" struct. The renderer reads the same field the simulation just wrote to.

### 5.2 The ownership tree (one copy, on the heap)

```
MSNet (==GUINet, singleton)
 ├── MSVehicleControl* myVehicleControl       ← MSNet.h:897
 │     └── std::map<id, SUMOVehicle*> myVehicleDict       ← MSVehicleControl.h:658
 │           (each entry is a heap-allocated GUIVehicle, owned here)
 ├── MSEdgeControl* myEdges                   ← MSNet.h:903
 │     └── MSEdgeVector myEdges               ← MSEdgeControl.h:266
 │           (vector<MSEdge*> - each is actually a GUIEdge, owned here)
 ├── MSJunctionControl* myJunctions           ← MSNet.h:905
 ├── MSDetectorControl, MSTLLogicControl, …
 ├── std::vector<GUIEdge*>            myEdgeWrapper       ← non-owning, parallel index
 ├── std::vector<GUIJunctionWrapper*> myJunctionWrapper   ← OWNED here
 └── SUMORTree                         myGrid             ← spatial index, non-owning
```

Each `MSEdge` owns its lanes via `std::shared_ptr<const std::vector<MSLane*>> myLanes` (`MSEdge.h:924`) - the shared_ptr is for sharing the lane vector, not the lanes themselves. Each `MSLane` references (does not own) the vehicles currently on it:

```cpp
typedef std::vector<MSVehicle*> VehCont;                 // MSLane.h:119
VehCont myVehicles, myPartialVehicles, myTmpVehicles;    // MSLane.h:1486/1498/1502
```

The vehicles themselves live in `MSVehicleControl::myVehicleDict` and are deleted only by `MSVehicleControl::deleteVehicle()`. The lane never deletes the vehicles it references.

### 5.3 Five real mutexes (and one for the queue)

Locking is **fine‑grained**, not one global "render lock". They form a layered scheme:

| Mutex | File:line | What it protects | Held by sim during | Held by GUI during |
|---|---|---|---|---|
| `GUIRunThread::mySimulationLock` | `GUIRunThread.h:184` | "a step is in progress" | the entire `simulationStep()` and `guiSimulationStep()` | not held during drawing |
| `GUINet::myLock` | `GUINet.h:462` | the simulationStep call | inside `GUINet::simulationStep` (`GUINet.cpp:243`, RAII via `FXMutexLock`) | not held during drawing |
| **`GUILane::myLock`** | `GUILane.h:400` | `myVehicles / myPartialVehicles / myTmpVehicles` on this lane | every mutation: `planMovements`, `executeMovements`, `integrateNewVehicles`, lane-change | iterating the vehicle list during `drawGL` |
| `GUIEdge::myLock` | `GUIEdge.h:259` | `myPersons`, `myContainers`, meso segments on this edge | add/remove of transportables, meso step | iterating persons/containers/segments during `drawGL` |
| `GUIVehicleControl::myLock` | `GUIVehicleControl.h:123` | the vehicle dictionary | `addVehicle`, `deleteVehicle`, `scheduleVehicleRemoval` | locator dialogs, statistics iteration |
| `MFXSynchQue::myMutex` | `MFXSynchQue.h:196` | the event queue (no network data) | every `push_back` | every `top` / `pop` in `eventOccurred` |

Plus one operational mutex unrelated to network state: `GUIRunThread::myBreakpointLock` (`.h:190`) for the breakpoint list.

### 5.4 The lane lock - where the action is

`GUILane` is the most important class for synchronisation. It overrides two `MSLane` hooks specifically so the GUI can borrow the live vehicle list safely:

```cpp
const MSLane::VehCont& GUILane::getVehiclesSecure() const {   // GUILane.cpp:168
    myLock.lock();
    return myVehicles;
}
void GUILane::releaseVehicles() const {                       // GUILane.cpp:175
    myLock.unlock();
}
```

The mutations the simulation thread performs on the lane all `lock` `myLock` first. Look at the long list at `GUILane.cpp:161, 182, 188, 195, 202, 209, 216, 223, 230, 237, 244` - those are `planMovements`, `executeMovements`, `integrateNewVehicles`, `swapAfterLaneChange`, `addLeaders`, etc., each starting with:

```cpp
FXMutexLock locker(myLock);
```

And the drawing path uses the `getVehiclesSecure / releaseVehicles` pair:

```cpp
const MSLane::VehCont& vehicles = getVehiclesSecure();   // GUILane.cpp:804
for (MSVehicle* veh : vehicles) {
    static_cast<GUIVehicle*>(veh)->drawGL(s);
}
releaseVehicles();                                       // GUILane.cpp:822
```

Whichever thread reaches the lane first holds the mutex; the other waits. So **at the granularity of a single lane, the snapshot is consistent**: you never see half a vehicle insertion or a vector being resized under your iterator.

### 5.5 What the GUI does *not* lock

This is the deliberate design choice: **the GUI thread does not acquire `mySimulationLock` or `GUINet::myLock` during drawing.** It only takes the per‑lane and per‑edge locks. The implication:

* Within one lane: consistent.
* Across two lanes drawn in sequence: you can momentarily see lane A in its post‑move state and lane B still in its pre‑move state. A vehicle that just hopped from A to B might briefly be drawn on neither, or on both.
* Across one full frame vs. one full step: there is no guarantee the frame represents a single point in simulated time.

The trade‑off is: the simulation never blocks on the GUI repaint. After a step, the run thread releases the locks, posts a `GUIEvent_SimulationStep`, and immediately starts the next step. If the GUI is too slow to keep up, multiple step events stack up in the queue (and the GUI just drains them all at the next wake‑up). On Windows, `handleEvent_SimulationStep` throttles redraws to ~50 fps so step events don't pile up; on Linux/macOS, the run thread itself injects a 100 ms sleep at least once per second so the GUI gets a chance to repaint even if `mySimDelay` is zero.

### 5.6 The event queue - control flow only, no network data

Step notifications pass through `MFXSynchQue<GUIEvent*> myEvents`. The events carry **metadata only**:

| Event | Payload |
|---|---|
| `GUIEvent_SimulationLoaded` | `GUINet*`, begin/end times, settings filenames |
| `GUIEvent_SimulationStep` | nothing - just "a step happened" |
| `GUIEvent_SimulationEnded` | end reason, last time step |
| `GUIEvent_Message` | message type and string |

When the GUI sees a `SIMULATION_STEP`, it doesn't extract any vehicle data from the event. It calls `update()` on the views, which causes a repaint, which calls `drawGL` on each visible object, which reads the live state via the per‑lane locks. So even though the queue is the *trigger*, the data path is direct memory access.

### 5.7 A worked example: drawing one vehicle

The GUI is drawing frame N while the simulation is computing step N+1.

1. Run thread enters `MSNet::simulationStep()` under `mySimulationLock`.
2. It iterates active lanes; for each lane it calls `MSLane::executeMovements`. The `GUILane` override locks `myLock`, updates `myVehicles` (positions, lane changes, removals), unlocks.
3. Run thread finishes the step, unlocks `mySimulationLock`, posts a `GUIEvent_SimulationStep`, signals the GUI.
4. GUI wakes, drains the queue, calls `update()`. FOX schedules `onPaint`.
5. `onPaint` → `paintGL` → `myGrid.Search(viewport, settings)` walks the R‑tree.
6. For each visible `GUILane`, it calls `drawGL(s)`.
7. `GUILane::drawGL` calls `getVehiclesSecure()` - locks `myLock`. If the run thread is just then re‑entering this lane for step N+2, it blocks on the same lock.
8. The GUI iterates the vehicle list. For each vehicle, it calls `GUIVehicle::drawGL` → `getVisualPosition()` → reads `MSVehicle::myState.myPos` directly. No copy.
9. `releaseVehicles()` unlocks. The run thread can now proceed onto this lane.

The vehicle's position field is a plain `double`. On x86/ARM, a `double` load is atomic, and the run thread only writes to it inside `executeMovements`, which holds `myLock`. So the GUI either reads the value as it was at the start of step N+1 (if the simulation hasn't reached this lane yet) or as it is at the end of step N+1 (if the simulation already passed by). Either way the read is well‑defined.

---

## 6. End‑to‑end summary

```
LAUNCH
  main()                                          guisim_main.cpp:53
    ├─ FXApp application                          (stack)
    ├─ new GUIApplicationWindow(&application)
    ├─ window->dependentBuild()                   builds menus, MDI, threads
    │     ├─ new GUILoadThread                    GUIApplicationWindow.cpp:342
    │     ├─ new GUIRunThread                     .cpp:343
    │     └─ myRunThread->start()                 .cpp:349  (idles on myHalting=true)
    ├─ application.create()
    └─ application.run()                          enter FOX event loop

OPEN A FILE
  User: File → Open Simulation… (Ctrl+O)
    onCmdOpenConfiguration                        GUIApplicationWindow.cpp:1065
      → loadConfigOrNet(file)
         → myLoadThread->loadConfigOrNet(file)
              → start() ⇒ GUILoadThread::run()    GUILoadThread.cpp:84
                   ├─ parse .sumocfg
                   ├─ new GUIVehicleControl
                   ├─ new GUINet
                   ├─ NLBuilder::build()           parses .net.xml via SAX
                   │     callbacks → new GUIEdge / GUILane / detectors / ...
                   ├─ net->initGUIStructures()    spatial R-tree, junction wrappers
                   └─ push GUIEvent_SimulationLoaded; signal GUI
  GUI thread (woken):
    onLoadThreadEvent → eventOccurred → handleEvent_SimulationLoaded
      ├─ myRunThread->init(net, begin, end)       run thread now has the net pointer
      └─ openNewView()                            creates GUIViewTraffic (FXGLCanvas)

CLICK PLAY
  User: Play (Space / Ctrl+A)
    onCmdStart → myRunThread->resume()            myHalting=false  ← single flag flip
  Run thread (already spinning in tryStep):
    next iteration sees myHalting=false → makeStep()
      ├─ mySimulationLock.lock()
      ├─ myNet->simulationStep()
      │     ├─ for each active lane:
      │     │     GUILane::myLock.lock(); update myVehicles; unlock
      │     └─ vehicle insertion / removal under GUIVehicleControl::myLock
      ├─ myNet->guiSimulationStep()
      ├─ mySimulationLock.unlock()
      ├─ push GUIEvent_SimulationStep; signal GUI
      └─ sleep(mySimDelay − step duration)
  GUI thread (woken):
    onRunThreadEvent → eventOccurred → handleEvent_SimulationStep
      └─ updateChildren(); update()                schedules onPaint
  GUI thread (next FOX iteration):
    onPaint → paintGL → GUIViewTraffic::doPaintGL
      └─ myGrid.Search(viewport)                   visits visible objects
            for each GUILane → drawGL
              ├─ getVehiclesSecure()              locks GUILane::myLock
              ├─ for each vehicle → GUIVehicle::drawGL(s)
              │     ↳ reads MSVehicle::myState directly (no copy)
              └─ releaseVehicles()                 unlocks
  Loop forever.
```

---

## 7. The takeaways

* **One copy of the network on the heap.** GUI classes inherit from microsim classes; same objects, two views.
* **Three threads:** GUI (FOX), load (one-shot per load), run (continuous).
* **Coarse locks** (`mySimulationLock`, `GUINet::myLock`) enclose a whole step on the run thread.
* **Fine‑grained locks** (`GUILane::myLock`, `GUIEdge::myLock`, `GUIVehicleControl::myLock`) are held by *both* threads but for very short critical sections - long enough to keep one lane's vehicle list internally consistent.
* **The GUI never holds a coarse lock during drawing.** This is intentional: the simulation is never blocked by repainting. The price is occasional cross‑lane inconsistency, which is invisible at typical step rates and frame rates.
* **The event queue carries no network data.** Step events are just notifications; the GUI re‑reads live state from the heap when it repaints.
* **TraCI runs on the simulation thread**, inside `MSNet::simulationStep`. There is no fourth thread for TraCI in sumo‑gui.

If you ever need to add a custom GUI overlay that reads simulation state safely, follow the existing pattern: (1) acquire the per‑lane or per‑edge lock with `FXMutexLock locker(lane->myLock)`, (2) read what you need, (3) release. Don't try to take `mySimulationLock` from the GUI - you'll deadlock against the run thread.
