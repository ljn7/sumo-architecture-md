# SUMO‑GUI Architecture - Deep Static Analysis

> Repository: `eclipse-sumo` (Eclipse SUMO source tree)
> Scope: `src/gui/`, `src/guisim/`, `src/guinetload/`, `src/microsim/`, `src/netload/`, `src/utils/gui/`, `src/utils/foxtools/`, `src/foreign/rtree/`
> Goal: build a complete mental model of GUI initialization, file loading, simulation threading, memory ownership, inter‑thread communication, user interaction, and the rendering pipeline - with concrete file:line references throughout.

---

## Table of Contents

1. [GUI Initialization](#1-gui-initialization)
2. [Opening Network / SUMO Configuration](#2-opening-network--sumo-configuration)
3. [Simulation Startup Flow](#3-simulation-startup-flow)
4. [User Interaction Flow](#4-user-interaction-flow)
5. [Memory Architecture](#5-memory-architecture)
6. [Data Sharing Between Simulation and GUI](#6-data-sharing-between-simulation-and-gui)
7. [Communication Mechanism](#7-communication-mechanism)
8. [Rendering / Drawing Architecture](#8-rendering--drawing-architecture)
9. [End‑to‑End Flow Diagrams](#9-endtoend-flow-diagrams)
10. [Handling Uncertainty](#10-handling-uncertainty)

---

## 1. GUI Initialization

The sumo‑gui binary is built around the **FOX‑Toolkit** (FXApp/FXMainWindow/FXGLCanvas/FXMDI…). The entry point is a thin `main()` that constructs the FOX application object, builds a `GUIApplicationWindow`, and enters the FOX event loop.

### 1.1 Entry point - `src/guisim_main.cpp`

| Line | Action |
|------|--------|
| 53 | `int main(int argc, char** argv)` |
| 55 | `MsgHandler::setFactory(&MsgHandlerSynchronized::create)` - thread‑safe message handlers |
| 61‑63 | Read FOX registry to determine UI language |
| 67‑70 | `XMLSubSys::init()`, `MSFrame::fillOptions()`, `OptionsIO::setArgs/getOptions` |
| 76 | `FXApp application("SUMO GUI", "sumo-gui")` - *the* FOX application object (stack‑allocated) |
| 78 | `application.init(argc, argv)` - opens display |
| 80‑82 | `FXGLVisual::supported()` - abort if no OpenGL |
| 85 | `GUIApplicationWindow* window = new GUIApplicationWindow(&application)` |
| 87 | `gSchemeStorage.init(&application)` - built‑in colour schemes |
| 88 | `window->dependentBuild(false)` - build menus, toolbars, MDI client, message window, threads |
| 90 | `application.addSignal(SIGINT, window, MID_HOTKEY_CTRL_Q_CLOSE)` |
| 91 | `application.create()` - realise all FOX widgets |
| 93‑95 | `window->loadOnStartup()` if a config was passed on the command line |
| 98 | `ret = application.run()` - **enter FOX event loop** (blocks until quit) |

### 1.2 `GUIApplicationWindow` - the main window

`src/gui/GUIApplicationWindow.h` declares the class (line 60); `src/gui/GUIApplicationWindow.cpp` implements it.

* **Constructor** (`.cpp` 255‑271) initialises menu panes (`myFileMenuRecentNetworks`, `myFileMenuRecentConfigs`), recent‑file lists, and icon/texture/cursor subsystems. **It does not** build the visible UI yet.
* **`dependentBuild(bool)`** (`.cpp` 284‑352) is the real builder, in this order:
    1. `myMenuBar` and call `buildToolBars()` (293‑296).
    2. Wire thread‑event objects to message IDs:
       ```cpp
       myLoadThreadEvent.setTarget(this);
       myLoadThreadEvent.setSelector(ID_LOADTHREAD_EVENT);
       myRunThreadEvent.setTarget(this);
       myRunThreadEvent.setSelector(ID_RUNTHREAD_EVENT);            // 298‑301
       ```
    3. Status bar with TraCI/coordinate frames and statistics buttons (303‑324).
    4. `myMainSplitter` (vertical FXSplitter) → `myMDIClient` (FXMDIClient) → `myMDIMenu` and the four MDI window buttons (326‑332).
    5. `myMessageWindow = new GUIMessageWindow(myMainSplitter, this)` (334).
    6. `fillMenuBar()` (336) - File / Edit / Settings / Locate / Simulation / Window / Help.
    7. **Threads created** (342‑343):
       ```cpp
       myLoadThread = new GUILoadThread(getApp(), this, myEvents, myLoadThreadEvent, isLibsumo);
       myRunThread  = new GUIRunThread(getApp(), this, mySimDelay, myEvents, myRunThreadEvent);
       ```
    8. `myRunThread->start()` (349) - run thread starts immediately and idles in its main loop until a network is loaded.

### 1.3 Member fields driving the architecture (`GUIApplicationWindow.h`)

| Field | Type | Purpose | Line |
|-------|------|---------|------|
| `myLoadThread` | `GUILoadThread*` | Background loader | 375 |
| `myRunThread` | `GUIRunThread*` | Background simulator | 378 |
| `myEvents` | `MFXSynchQue<GUIEvent*>` | Cross‑thread event queue | 465 |
| `myLoadThreadEvent` | `FXEX::MFXThreadEvent` | Load‑thread → GUI signal | 485 |
| `myRunThreadEvent`  | `FXEX::MFXThreadEvent` | Run‑thread  → GUI signal | 488 |
| `myMessageWindow` | `GUIMessageWindow*` | Message log | 429 |
| `myMainSplitter` | `FXSplitter*` | MDI ⇕ messages splitter | 432 |
| `myMDIClient` | `FXMDIClient*` (in base) | Container of view windows | inherited |
| `mySimDelay` | `double` | ms / simulated second (slider value) | 444 |
| `mySimDelaySpinner/Slider` | `FXRealSpinner* / FXSlider*` | UI bound to `mySimDelay` | 450/453 |

### 1.4 Base class - `GUIMainWindow`

`src/utils/gui/windows/GUIMainWindow.{h,cpp}` (constructor lines 54‑89). Owns:

* `myGLVisual = new FXGLVisual(app, VISUAL_DOUBLEBUFFER)` (line 58) - **shared GL visual** used by every view canvas in the application.
* `myMDIClient` (FXMDIClient) - set by `GUIApplicationWindow::dependentBuild`.
* Four dock sites (top, bottom, left, right) for floating toolbars (79‑82).
* Fonts, default tooltip pane, the `myGLWindows` and `myTrackerWindows` lists.

### 1.5 The Main Window → View → GL canvas chain

```
FXApp                               (sumogui_main.cpp:76, stack)
└── GUIApplicationWindow (FXMainWindow)
    ├── myMenuBar (FXMenuBar)
    ├── myToolBar1‑8 (FXToolBar)            ← buildToolBars()
    ├── myStatusbar (FXStatusBar)
    ├── myMainSplitter (FXSplitter)
    │   ├── myMDIClient  (FXMDIClient)      ← container of views
    │   │   └── GUISUMOViewParent (FXMDIChild)        ← src/gui/GUISUMOViewParent.{h,cpp}
    │   │       └── myChildWindowContentFrame (FXVerticalFrame)
    │   │           ├── nav / coloring / screenshot / speed toolbars
    │   │           └── myView : GUIViewTraffic | GUIOSGView   (FXGLCanvas subclass)
    │   │                       ↑
    │   │                       FXGLVisual (shared)            ← src/utils/gui/windows/GUIMainWindow.cpp:58
    │   └── myMessageWindow (GUIMessageWindow)
    ├── myLoadThread (GUILoadThread)        ← MFXSingleEventThread
    ├── myRunThread  (GUIRunThread)         ← MFXSingleEventThread (started immediately)
    └── myEvents (MFXSynchQue<GUIEvent*>)   ← shared with both threads
```

* `GUISUMOViewParent::init()` (`src/gui/GUISUMOViewParent.cpp` 90‑107) instantiates either `GUIViewTraffic` (2D, OpenGL) or `GUIOSGView` (3D, OpenSceneGraph) into `myChildWindowContentFrame`, sharing the parent's GL context (`myGUIMainWindowParent->getGLVisual()`).
* `GUISUMOAbstractView` (`src/utils/gui/windows/GUISUMOAbstractView.{h,cpp}` - class at 83, ctor 136‑153) inherits from `FXGLCanvas`. Its FOX message map (107‑128) wires SEL_PAINT / SEL_CONFIGURE / mouse / keyboard / wheel events to handler methods.

### 1.6 Initialisation order summary

```
main()
  ├─ XMLSubSys::init / MSFrame::fillOptions / OptionsIO::getOptions
  ├─ FXApp application(...)
  ├─ application.init()
  ├─ new GUIApplicationWindow(&application)
  │     ├─ GUIMainWindow ctor: FXMainWindow + FXGLVisual + dock sites + fonts
  │     └─ icon / texture / cursor subsystems
  ├─ window->dependentBuild(false)
  │     ├─ menu bar + toolbars
  │     ├─ status bar
  │     ├─ MDI client + message window
  │     ├─ fillMenuBar()
  │     ├─ new GUILoadThread / new GUIRunThread
  │     └─ myRunThread->start()                ← idles on myHalting=true
  ├─ application.create()                      ← realise FOX widgets
  ├─ window->loadOnStartup()                   ← if argc > 1
  └─ application.run()                         ← FOX event loop
```

---

## 2. Opening Network / SUMO Configuration

Loading is asynchronous: the GUI thread spawns a `GUILoadThread`, which builds the entire `GUINet` and posts a `GUIEvent_SimulationLoaded` back via the shared event queue. The GUI then opens the first view.

### 2.1 User‑click path

`src/gui/GUIApplicationWindow.cpp`:

| Action | Handler | Line |
|--------|---------|------|
| File ▸ Open Simulation… (`Ctrl+O`) | `onCmdOpenConfiguration` | 1065‑1081 |
| File ▸ Open Network… (`Ctrl+N`) | `onCmdOpenNetwork`        | 1085‑1101 |
| File ▸ Reload (`Ctrl+R`)         | `onCmdReload`             | (search for `MID_HOTKEY_CTRL_R_RELOAD`) |

Both handlers create an `FXFileDialog`, store the chosen path in the recent files list, then call:

```cpp
loadConfigOrNet(file);                            // GUIApplicationWindow.cpp:2206‑2218
```

`loadConfigOrNet` flips `myAmLoading=true`, closes existing windows, saves the current viewport, and forwards to the worker:

```cpp
myLoadThread->loadConfigOrNet(file);              // line 2214
```

### 2.2 `GUILoadThread` - `src/gui/GUILoadThread.{h,cpp}`

* Inherits `MFXSingleEventThread` (which inherits `FXObject` + `FXThread`).
* Constructor (`.cpp` 66‑74): captures `MFXSynchQue<GUIEvent*>& myEventQue` and `FXEX::MFXThreadEvent& myEventThrow` references.
* Entry point on the GUI thread:
    ```cpp
    void GUILoadThread::loadConfigOrNet(const std::string& file) {
        myFile = file;
        if (myFile != "") OptionsIO::setArgs(0, nullptr);
        start();                                  // FXThread::start spawns run()
    }                                             // GUILoadThread.cpp:257‑263
    ```

* `GUILoadThread::run()` (`.cpp` 84‑235) - executed on the loader thread:
    1. **Options parsing** (94‑149): `OptionsCont::clear()` → `MSFrame::fillOptions()` → `OptionsCont::setByRootElement(OptionsIO::getRoot(myFile), myFile)` (parses the `.sumocfg` XML root) → `OptionsIO::getOptions()`.
    2. **Vehicle control** (lines around 155‑160):
       ```cpp
       MSVehicleControl* vehControl = MSGlobals::gUseMesoSim
           ? (MSVehicleControl*) new GUIMEVehicleControl()
           : (MSVehicleControl*) new GUIVehicleControl();
       ```
    3. **Network construction** (163‑213):
       ```cpp
       GUINet* net = new GUINet(vehControl,
                                new GUIEventControl(),  // begin‑of‑step
                                new GUIEventControl(),  // end‑of‑step
                                new GUIEventControl()); // insertion
       GUIEdgeControlBuilder* eb  = new GUIEdgeControlBuilder();
       GUIDetectorBuilder      db(*net);
       NLJunctionControlBuilder jb(*net, db);
       GUITriggerBuilder        tb;
       NLHandler handler("", *net, db, tb, *eb, jb);
       NLBuilder  builder(oc, *net, *eb, jb, db, handler);
       if (!builder.build()) throw ProcessError();
       net->initGUIStructures();
       ```
    4. **Submit completion** (238‑253):
       ```cpp
       GUIEvent* e = new GUIEvent_SimulationLoaded(net, simStartTime, simEndTime,
                                                   myTitle, guiSettingsFiles, osgView, viewportFromRegistry);
       myEventQue.push_back(e);                  // mutex‑protected push
       myEventThrow.signal();                    // wake GUI thread
       ```

### 2.3 `NLBuilder` - XML → in‑memory network

`src/netload/NLBuilder.cpp`:

* `NLBuilder::build()` (125‑233): drives `load("net-file", true)` then `buildNet()`, then loads `additional-files` and routes.
* `NLBuilder::load()` (451‑466): wraps `XMLSubSys::runParser(myXMLHandler, file, isNet)` - a SAX parser that calls back into `NLHandler` for each XML element.
* `NLBuilder::buildNet()` (400‑447): finalises edges/junctions, traffic lights, route loaders, and calls `myNet.closeBuilding(...)`.

### 2.4 GUI builders - `src/guinetload/`

These are the polymorphic seams that turn microsim objects into their GUI variants. They subclass the netload builders and override the factory methods.

| File | Class | Replaces `new MSXxx` with `new GUIXxx` |
|------|-------|----------------------------------------|
| `GUIEdgeControlBuilder.{h,cpp}` | `GUIEdgeControlBuilder : NLEdgeControlBuilder` | `buildEdge` → `new GUIEdge` (line 65‑70 of .cpp); `addLane` → `new GUILane` (47‑61) |
| `GUIDetectorBuilder.{h,cpp}` | `GUIDetectorBuilder : NLDetectorBuilder` | induction loops (`GUIInductLoop`), E2/E3 (`GUIE2Collector`, `GUIE3Collector`), instant induction loops |
| `GUITriggerBuilder.{h,cpp}` | `GUITriggerBuilder : NLTriggerBuilder` | `GUILaneSpeedTrigger`, `GUITriggeredRerouter`, `GUIBusStop`, `GUIChargingStation`, `GUIParkingArea`, `GUIOverheadWire`, `GUICalibrator`, … |
| `GUILoader` (compiled into the same target via the cmakelist) | helpers around the above | - |

`GUIEdgeControlBuilder::buildEdge`:

```cpp
MSEdge* GUIEdgeControlBuilder::buildEdge(const std::string& id, const SumoXMLEdgeFunc function,
                                         const std::string& streetName, const std::string& edgeType,
                                         const std::string& routingType, int priority, double distance) {
    return new GUIEdge(id, myCurrentNumericalEdgeID++, function, streetName,
                       edgeType, routingType, priority, distance);
}                                                 // GUIEdgeControlBuilder.cpp:65‑70
```

`GUIEdgeControlBuilder::addLane` similarly returns `new GUILane(...)`.

### 2.5 `GUINet::initGUIStructures()`

`src/guisim/GUINet.cpp` 276‑360 - runs *after* the basic XML build. It:

1. Builds wrappers for each detector / calibrator (`GUIDetectorWrapper`, `GUICalibrator`) and inserts them into `myGrid` (the spatial R‑tree).
2. Calls `initTLMap()` to wire up `GUITrafficLightLogicWrapper` instances.
3. Re‑casts every `MSEdge*` to `GUIEdge*` into `myEdgeWrapper` and inserts each edge's bounding box into `myGrid`.
4. For every junction, allocates a `GUIJunctionWrapper` and inserts it into `myGrid` and `myJunctionWrapper`.
5. Computes the global `myBoundary` used by views.

### 2.6 GUI side of completion

The `GUIEvent_SimulationLoaded` event has signature (`src/gui/GUIEvent_SimulationLoaded.h` 47‑92):

```cpp
GUIEvent_SimulationLoaded(GUINet* net, SUMOTime startTime, SUMOTime endTime,
                          const std::string& file,
                          const std::vector<std::string>& settingsFiles,
                          bool osgView, bool viewportFromRegistry);
GUINet* myNet;            SUMOTime myBegin, myEnd;
std::string myFile;       std::vector<std::string> mySettingsFiles;
bool myOsgView;           bool myViewportFromRegistry;
```

When the event arrives, FOX dispatches `onLoadThreadEvent` → `eventOccurred()` → `handleEvent_SimulationLoaded(GUIEvent*)` (`GUIApplicationWindow.cpp` 1847‑1968):

* `myAmLoading = false` (re‑enable UI).
* `myRunThread->init(net, begin, end)` - see §3.4.
* For each entry in `ec->mySettingsFiles` create a view via `openNewView(...)` and apply colour scheme, viewport, decals, breakpoints, delay.
* If no settings files exist, call `openNewView(defaultType)` once.

### 2.7 `openNewView` - create a view window

`GUIApplicationWindow.cpp` 2222‑2254. The interesting bits:

```cpp
GUISUMOViewParent* w = new GUISUMOViewParent(
    myMDIClient, myMDIMenu, FXString(caption.c_str()),
    this, GUIIconSubSys::getIcon(GUIIcon::SUMO_MINI), MDI_TRACKING, 10,10,200,100);
GUISUMOAbstractView* v = w->init(getBuildGLCanvas(), myRunThread->getNet(), vt);
if (oldView != nullptr) oldView->copyViewportTo(v);
w->create();
if (myMDIClient->numChildren() == 1) w->maximize();
else                                  myMDIClient->vertical(true);
myMDIClient->setActiveChild(w);
```

`getBuildGLCanvas()` (2258‑2265) returns the first existing `FXGLCanvas` so subsequent canvases can **share GL resources** (textures, display lists).

### 2.8 Call chain (open network/config)

```
File ▸ Open Configuration / Network
   └─ onCmdOpenConfiguration / onCmdOpenNetwork              [GUIApplicationWindow.cpp:1065 / 1085]
        └─ FXFileDialog → loadConfigOrNet(file)              [.cpp:2206]
             └─ myLoadThread->loadConfigOrNet(file)
                  └─ FXThread::start() → GUILoadThread::run()  [LOADER THREAD]
                       ├─ OptionsIO / OptionsCont parses .sumocfg
                       ├─ new GUIVehicleControl, new GUINet
                       ├─ GUIEdgeControlBuilder / GUIDetectorBuilder / GUITriggerBuilder
                       ├─ NLBuilder::build()
                       │    ├─ XMLSubSys::runParser(NLHandler, .net.xml)
                       │    │     └─ callbacks → builders → new GUIEdge / GUILane / …
                       │    └─ buildNet() → MSNet::closeBuilding()
                       ├─ net->initGUIStructures()           [populate spatial index]
                       └─ submitEndAndCleanup()
                            ├─ myEventQue.push_back(new GUIEvent_SimulationLoaded(...))
                            └─ myEventThrow.signal()         [wake GUI]
                                  ↓
                                  FOX event loop fires SEL_THREAD_EVENT
                                       └─ onLoadThreadEvent → eventOccurred()
                                            └─ handleEvent_SimulationLoaded()
                                                 ├─ myRunThread->init(net, begin, end)
                                                 └─ openNewView() → GUISUMOViewParent → GUIViewTraffic
```

---

## 3. Simulation Startup Flow

The simulation core is `MSNet::simulationStep()`. In sumo‑gui this is invoked from a dedicated worker thread, `GUIRunThread`, that is started during `dependentBuild` and idles until a network is loaded and the user clicks Play.

### 3.1 `GUIRunThread` class - `src/gui/GUIRunThread.{h,cpp}`

```cpp
class GUIRunThread : public MFXSingleEventThread {            // .h:53
    GUINet*    myNet;                                          // .h:145  - assigned in init()
    SUMOTime   mySimStartTime, mySimEndTime;                   // .h:148
    bool       myHalting;                                      // .h:151  - start/stop gate
    bool       myQuit;                                         // .h:155  - terminate run loop
    bool       mySimulationInProgress;                         // .h:159
    bool       myOk;                                           // .h:162
    bool       mySingle;                                       // .h:165  - single‑step mode
    bool       myHaveSignaledEnd;                              // .h:168
    double&    mySimDelay;                                     // .h:175  - bound to GUIApplicationWindow::mySimDelay
    MFXSynchQue<GUIEvent*>& myEventQue;                        // .h:178
    FXEX::MFXThreadEvent&   myEventThrow;                      // .h:181
    FXMutex    mySimulationLock;                               // .h:184  - guards myNet during a step
    std::vector<SUMOTime> myBreakpoints; FXMutex myBreakpointLock; // .h:187‑190
    long       myLastEndMillis, myLastBreakMillis;             // .h:193‑196
};
```

### 3.2 The run loop

```cpp
FXint GUIRunThread::run() {                                   // .cpp:126
    while (!myQuit) {
        if (myAmLibsumo) myApp->run();
        else             tryStep();                           // .cpp:133
    }
    deleteSim();
    return 0;
}
```

`tryStep()` (`.cpp` 142‑183) gates execution on `!myHalting`:

* Checks breakpoints under `myBreakpointLock` (150‑156).
* If `mySingle` is set, schedules a halt after this step (157‑160).
* Calls `makeStep()` (162).
* Sleeps `mySimDelay × TS − step duration` ms to honour the requested visual delay (164‑178).
* If halted, sleeps 50 ms (181) and loops.

### 3.3 `makeStep` - execute one step

```cpp
void GUIRunThread::makeStep() {                               // .cpp:186‑253
    mySimulationInProgress = true;
    try {
        mySimulationLock.lock();
        myNet->simulationStep();          // microsim core
        myNet->guiSimulationStep();       // GUI value‑pass connectors
        mySimulationLock.unlock();

        myEventQue.push_back(new GUIEvent_SimulationStep());
        myEventThrow.signal();            // wake GUI

        // Check for end conditions; possibly emit GUIEvent_SimulationEnded
        // and set myHalting = true.
        mySimulationInProgress = false;
    } catch (ProcessError&) {
        mySimulationLock.unlock();
        myEventQue.push_back(new GUIEvent_SimulationEnded(SIMSTATE_ERROR_IN_SIM, ...));
        myEventThrow.signal();
        myHalting = true; myOk = false;
    }
}
```

### 3.4 Network‑binding: `GUIRunThread::init`

```cpp
bool GUIRunThread::init(GUINet* net, SUMOTime start, SUMOTime end) {  // .cpp:84‑122
    myOk = true;
    myNet = net;
    mySimStartTime = start;
    mySimEndTime   = end;
    myHaveSignaledEnd = false;
    // register error/warning/message retrievers
    mySimulationLock.lock();
    try {
        net->setCurrentTimeStep(start);
        net->loadRoutes();
    } catch (...) { myHalting = true; myOk = false; }
    mySimulationLock.unlock();
    return myOk;
}
```

After `init`, the thread is alive but `myHalting == true`. It will spin in the 50 ms sleep until the user starts the simulation.

### 3.5 GUI control handlers

`src/gui/GUIApplicationWindow.cpp`:

| Handler | Line | Effect |
|---------|------|--------|
| `onCmdStart` | 1277‑1291 | `if (!myWasStarted) myRunThread->begin(); myRunThread->resume();` |
| `onCmdStop`  | 1294‑1298 | `myRunThread->stop();` |
| `onCmdStep`  | 1302‑1316 | `myRunThread->singleStep();` |
| `onUpdStart` | 1459‑1469 | enables/disables Play; rebinds `Space` to Start when paused |
| `onUpdStop`  | 1473‑1483 | enables/disables Stop; rebinds `Space` to Stop when running |
| `onUpdStep`  | 1487‑1493 | enables/disables Step |

The corresponding `GUIRunThread` mutators (`.cpp` 257‑282) do nothing more than flip three booleans - there is no condition variable, no queue. The simulation loop notices the change at the top of the next iteration of `tryStep()`:

```cpp
void GUIRunThread::resume()    { mySingle = false; myHalting = false; }   // 257
void GUIRunThread::singleStep(){ mySingle = true;  myHalting = false; }   // 264
void GUIRunThread::stop()      { mySingle = false; myHalting = true;  }   // 279
void GUIRunThread::begin()     { WRITE_MESSAGEF(...); myOk = true; }      // 271
```

`simulationIsStartable / Stopable / Stepable` (`.cpp` 338‑353) are the predicates used by `onUpdXxx`.

### 3.6 What `MSNet::simulationStep()` does

`src/microsim/MSNet.cpp` 794‑930. One step (`DELTA_T` = 1000 ms by default) runs:

1. TraCI command processing (805‑822).
2. State dumps (831‑847).
3. `myBeginOfTimestepEvents->execute(myStep)` (848).
4. Rail‑signal updates (849‑851).
5. Vehicle movement: lane activation, safe‑velocity, junction approach, right‑of‑way, movement, lane changing, collision detection (864‑885).
6. Pending vehicle removal (888) + route loading (889).
7. Person / container waiting checks (892‑898).
8. New vehicle insertion from demand (903‑914).
9. End‑of‑timestep events (917).
10. `myStep += DELTA_T` and post‑move output (929).

`GUINet::simulationStep` overrides this with an RAII lock for thread safety:

```cpp
void GUINet::simulationStep() {                               // GUINet.cpp:242‑245
    FXMutexLock locker(myLock);
    MSNet::simulationStep();
}
```

`GUINet::guiSimulationStep` (`.cpp` 235‑238) updates the value‑pass connectors so any open parameter tables get fresh values:

```cpp
void GUINet::guiSimulationStep() {
    GLObjectValuePassConnector<double>::updateAll();
    GLObjectValuePassConnector<std::pair<SUMOTime, MSPhaseDefinition>>::updateAll();
}
```

### 3.7 Start/Stop/Step sequence diagrams

```
USER clicks Play (Ctrl+A or Space)
─────────────────────────────────────
[GUI thread]  onCmdStart()
                 ├─ if (!myWasStarted) myRunThread->begin()
                 └─ myRunThread->resume()       myHalting = false   ← key flag flip
                                                mySingle  = false

[Run thread]  in next tryStep() iteration:
                 ├─ if (!myHalting && myNet && myOk) → run step
                 │     ├─ mySimulationLock.lock()
                 │     ├─ MSNet::simulationStep()
                 │     ├─ GUINet::guiSimulationStep()
                 │     ├─ mySimulationLock.unlock()
                 │     └─ push GUIEvent_SimulationStep, signal()
                 ├─ sleep( mySimDelay·TS − duration )
                 └─ loop

[GUI thread]  FOX event loop fires SEL_THREAD_EVENT
                 onRunThreadEvent → eventOccurred()
                     └─ handleEvent_SimulationStep
                          ├─ updateTimeLCD
                          ├─ updateChildren()
                          └─ update()    ← schedules onPaint on each view
```

```
USER clicks Stop
[GUI]  onCmdStop → myRunThread->stop()       myHalting = true
[Run]  next tryStep(): condition false → sleep(50)        (loops)
```

```
USER clicks Step (Ctrl+D)
[GUI]  onCmdStep → myRunThread->singleStep() mySingle = true; myHalting = false
[Run]  next tryStep(): runs makeStep, then because mySingle is true, sets myHalting = true
       → next iteration sleeps(50)
```

---

## 4. User Interaction Flow

User input is routed via the FOX‑Toolkit **target/selector** message map. There is no central command dispatcher - every widget knows its target and a numeric `MID_*` selector, and FOX matches these against `FXDEFMAP` tables.

### 4.1 Message map basics

```cpp
FXDEFMAP(GUIApplicationWindow) GUIApplicationWindowMap[] = {
    FXMAPFUNC(SEL_COMMAND, MID_HOTKEY_CTRL_O_OPENSIMULATION_OPENNETWORK,
              GUIApplicationWindow::onCmdOpenConfiguration),         // .cpp:97
    FXMAPFUNC(SEL_COMMAND, MID_HOTKEY_CTRL_A_STARTSIMULATION_OPENADDITIONALELEMENTS,
              GUIApplicationWindow::onCmdStart),                     // .cpp:130
    FXMAPFUNC(SEL_UPDATE,  MID_HOTKEY_CTRL_S_STOPSIMULATION_SAVENETWORK,
              GUIApplicationWindow::onUpdStop),                      // .cpp:160
    FXMAPFUNC(FXEX::SEL_THREAD_EVENT, ID_LOADTHREAD_EVENT,
              GUIApplicationWindow::onLoadThreadEvent),              // .cpp:233
    FXMAPFUNC(SEL_KEYPRESS, 0, GUIApplicationWindow::onKeyPress),    // .cpp:228
    ...
};
FXIMPLEMENT(GUIApplicationWindow, FXMainWindow, GUIApplicationWindowMap, ...)  // .cpp:240
```

* **`SEL_COMMAND`** - button/menu click, accelerator activation.
* **`SEL_UPDATE`** - periodic poll FOX uses to enable/disable widgets.
* **`SEL_CHANGED`** - value spinners, sliders.
* **`SEL_PAINT / SEL_CONFIGURE`** - paint / resize.
* **`SEL_LEFT/MIDDLE/RIGHT BUTTONPRESS/RELEASE`, `SEL_MOTION`, `SEL_MOUSEWHEEL`** - pointer.
* **`SEL_KEYPRESS / SEL_KEYRELEASE`** - keyboard.
* **`FXEX::SEL_THREAD / SEL_THREAD_EVENT`** - custom thread‑signalling selectors used by `MFXThreadEvent`.

### 4.2 The IDs themselves - `src/utils/gui/windows/GUIAppEnum.h`

* `MID_HOTKEY_A_…` … `MID_HOTKEY_Z_…` (42‑75) - single‑letter hotkeys.
* `MID_HOTKEY_CTRL_*` (82‑135).
* `MID_HOTKEY_SHIFT_*` (171‑194).
* `MID_HOTKEY_CTRL_SHIFT_*` (201‑224).
* `MID_HOTKEY_F1` … `MID_HOTKEY_F12` (231‑276).
* `MID_RECENTFILE`, `MID_SIMSAVE`, `MID_SIMLOAD` (316‑331).
* View settings: `MID_RECENTERVIEW`, `MID_ALLOWROTATION`, `MID_ZOOM_STYLE`, `MID_DEMAND_SCALE`, `MID_SPEEDFACTOR` (379‑406).
* Object popup: `MID_CENTER`, `MID_COPY_NAME`, `MID_ADDSELECT`, `MID_REMOVESELECT`, `MID_MANIP`, `MID_DRAWROUTE`, `MID_SHOW_CURRENTROUTE`, `MID_REMOVE_OBJECT`, `MID_TOGGLE_STOP`, `MID_REACHABILITY` (449‑533).
* Locator chooser: `MID_CHOOSER_*` (577‑641).
* Interactive simulation modification: `MID_CLOSE_LANE`, `MID_CLOSE_EDGE`, `MID_ADD_REROUTER`, `MID_VIRTUAL_DETECTOR` (665‑677).

### 4.3 Menu / toolbar construction

`GUIApplicationWindow::fillMenuBar()` (`.cpp` 465‑685) builds File / Edit / Settings / Locate / Simulation / Window / Help. Example:

```cpp
GUIDesigns::buildFXMenuCommandShortcut(myFileMenu, TL("&Open Simulation..."), "Ctrl+O",
    nullptr, this, MID_HOTKEY_CTRL_O_OPENSIMULATION_OPENNETWORK);   // ~line 467
```

`buildToolBars()` (`.cpp` 686‑850) creates eight toolbars: file ops, simulation control, time LCD (`myLCDLabel`), simulation delay slider, traffic scaling, view list, gaming.

### 4.4 Hotkeys / accelerators

`src/utils/gui/shortcuts/GUIShortcutsSubSys.cpp` 31+ registers keys in the `FXAccelTable`:

```cpp
accelTable->addAccel(parseKey(KEY_a), target,
    FXSEL(SEL_COMMAND, MID_HOTKEY_A_MODE_STARTSIMULATION_ADDITIONALS_STOPS));
accelTable->addAccel(parseKey(KEY_o, KEYMODIFIER_CONTROL), target,
    FXSEL(SEL_COMMAND, MID_HOTKEY_CTRL_O_OPENSIMULATION_OPENNETWORK));
accelTable->addAccel(parseKey(KEY_F9), target,
    FXSEL(SEL_COMMAND, MID_HOTKEY_F9_EDIT_VIEWSCHEME));
```

| Key | Action | Selector |
|-----|--------|----------|
| `A` / `Ctrl+A` / `Space` | Start simulation | `MID_HOTKEY_(CTRL_)A_…` |
| `S` / `Ctrl+S`           | Stop simulation  | `MID_HOTKEY_(CTRL_)S_…` |
| `D` / `Ctrl+D`           | Single step      | `MID_HOTKEY_(CTRL_)D_…` |
| `B` / `Ctrl+B`           | Set / edit breakpoint | `MID_HOTKEY_(CTRL_)B_…` |
| `Ctrl+O`                 | Open simulation  | `MID_HOTKEY_CTRL_O_…` |
| `Ctrl+N`                 | Open network     | `MID_HOTKEY_CTRL_N_…` |
| `Ctrl+R`                 | Reload           | `MID_HOTKEY_CTRL_R_…` |
| `Ctrl+W` / `Ctrl+Q`      | Close / Quit     | `MID_HOTKEY_CTRL_W_…` / `MID_HOTKEY_CTRL_Q_CLOSE` |
| `Ctrl+I`                 | Edit viewport    | `MID_HOTKEY_CTRL_I_EDITVIEWPORT` |
| `F9`                     | Edit visualisation scheme | `MID_HOTKEY_F9_EDIT_VIEWSCHEME` |
| `F12`                    | About            | `MID_HOTKEY_F12_ABOUT` |
| `Shift+J/E/V/P`          | Locate junction/edge/vehicle/person | `MID_HOTKEY_SHIFT_*` |

Note: `Space` is dynamically rebound between Start and Stop in `onUpdStart`/`onUpdStop` (lines 1466 / 1480) so it always toggles the appropriate action.

### 4.5 Mouse and keyboard on the canvas

`src/utils/gui/windows/GUISUMOAbstractView.cpp` message map (107‑128):

```cpp
FXMAPFUNC(SEL_CONFIGURE,            0, onConfigure),
FXMAPFUNC(SEL_PAINT,                0, onPaint),
FXMAPFUNC(SEL_LEFTBUTTONPRESS,      0, onLeftBtnPress),
FXMAPFUNC(SEL_LEFTBUTTONRELEASE,    0, onLeftBtnRelease),
FXMAPFUNC(SEL_RIGHTBUTTONPRESS,     0, onRightBtnPress),
FXMAPFUNC(SEL_RIGHTBUTTONRELEASE,   0, onRightBtnRelease),
FXMAPFUNC(SEL_MIDDLEBUTTONPRESS,    0, onMiddleBtnPress),
FXMAPFUNC(SEL_MIDDLEBUTTONRELEASE,  0, onMiddleBtnRelease),
FXMAPFUNC(SEL_DOUBLECLICKED,        0, onDoubleClicked),
FXMAPFUNC(SEL_MOTION,               0, onMouseMove),
FXMAPFUNC(SEL_MOUSEWHEEL,           0, onMouseWheel),
FXMAPFUNC(SEL_LEAVE,                0, onMouseLeft),
FXMAPFUNC(SEL_KEYPRESS,             0, onKeyPress),
FXMAPFUNC(SEL_KEYRELEASE,           0, onKeyRelease),
FXMAPFUNC(SEL_COMMAND, MID_CLOSE_LANE,    onCmdCloseLane),
FXMAPFUNC(SEL_COMMAND, MID_CLOSE_EDGE,    onCmdCloseEdge),
FXMAPFUNC(SEL_COMMAND, MID_ADD_REROUTER,  onCmdAddRerouter),
FXMAPFUNC(SEL_COMMAND, MID_REACHABILITY,  onCmdShowReachability),
```

`onLeftBtnPress` (1108‑1148):

* `Ctrl+Click` → `gSelected.toggleSelection(getObjectUnderCursor())`.
* `Shift+Click` on a vehicle/person → `startTrack(id)`.
* Otherwise delegates to `myChanger->onLeftBtnPress(ptr)` (panning).
* Calls `grab()` so subsequent moves arrive even outside the canvas.

`onRightBtnPress / onRightBtnRelease` (1199‑1220):

* If the perspective changer doesn't consume the release (e.g. zoom‑rectangle was drawn), `openObjectDialogAtCursor()` is called → builds `GUIGLObjectPopupMenu` for the object under the cursor.

`onMouseWheel` (1224‑1234) and `onMouseMove` (1237‑1265) defer to `myChanger->onMouseWheel/onMouseMove`. The "Daniel" perspective changer (`src/utils/gui/windows/GUIDanielPerspectiveChanger.cpp`) implements 2D pan, zoom rectangle, mouse‑wheel zoom, optional rotation.

### 4.6 Pixel→world coordinates

```cpp
Position GUISUMOAbstractView::screenPos2NetPos(int x, int y) const {     // .cpp:200‑220
    Boundary bound = myChanger->getViewport();
    double xNet = bound.xmin() + bound.getWidth()  * x / getWidth();
    double yNet = bound.ymin() + bound.getHeight() * (getHeight() - y) / getHeight();
    if (myChanger->getRotation() != 0)
        return Position(xNet, yNet).rotateAround2D(-DEG2RAD(myChanger->getRotation()), bound.getCenter());
    return Position(xNet, yNet);
}
```

### 4.7 Right‑click popup menus

Every drawable object overrides `GUIGlObject::getPopUpMenu` (declared at `src/utils/gui/globjects/GUIGlObject.h:116‑122`) to produce a `GUIGLObjectPopupMenu` populated with items like `MID_CENTER`, `MID_COPY_NAME`, `MID_ADDSELECT`, `MID_SHOW_CURRENTROUTE`, `MID_REMOVE_OBJECT`. Vehicle popup actions live on a nested `GUIBaseVehicle::GUIBaseVehiclePopupMenu` class (`src/guisim/GUIBaseVehicle.cpp` ~354/394).

### 4.8 Dialogs

| Dialog | File | Purpose |
|--------|------|---------|
| `GUIDialog_EditViewport` | `src/utils/gui/windows/GUIDialog_EditViewport.{h,cpp}` (class 41‑88) | Numerical zoom / x / y / rotation; load‑save .xml viewport |
| `GUIDialog_ViewSettings` | `src/utils/gui/windows/GUIDialog_ViewSettings.{h,cpp}` | Edit colour scheme, label rendering, decals |
| `GUIDialog_GLObjChooser` | `src/gui/dialogs/GUIDialog_GLObjChooser.{h,cpp}` | Locate by id (junction/edge/vehicle/person …) |
| `GUIDialog_ChooserAbstract` | `src/utils/gui/windows/GUIDialog_ChooserAbstract.{h,cpp}` | Base class for choosers |
| `GUIDialog_GLChosenEditor` | `src/utils/gui/div/GUIDialog_GLChosenEditor.{h,cpp}` | Manage current selection set |

### 4.9 GUI‑state vs simulation‑state actions

| Category | Examples | Targets |
|----------|----------|---------|
| **GUI‑only** (view/state) | pan, zoom, rotation; viewport edit; colour scheme switch; show route; show best lanes; track vehicle; toggle grid; toggle junction shape; selection (Ctrl+Click); breakpoint editing | `myChanger`, `GUIVisualizationSettings*`, `gSelected`, `myBreakpoints` (GUIRunThread, but only metadata) |
| **Simulation‑modifying** | Start / Stop / Step; `MID_REMOVE_OBJECT` (delete vehicle); `MID_TOGGLE_STOP` (toggle vehicle stop); `MID_CLOSE_LANE` / `MID_CLOSE_EDGE` (rerouter inserted dynamically); `MID_ADD_REROUTER`; TraCI commands routed via `GUIRunThread` | `myRunThread->{begin,resume,stop,singleStep}()`; `MSVehicle::removeFromGrid`, `MSLane::setPermissions`, etc. |

### 4.10 Selection storage - `gSelected`

`src/utils/gui/div/GUISelectedStorage.{h,cpp}`. Storage layout (`.h` 67‑286):

```cpp
std::map<GUIGlObjectType, SingleTypeSelections> mySelections;     // .h:276
std::unordered_set<GUIGlID>                     myAllSelected;    // .h:279
struct SingleTypeSelections { std::unordered_set<GUIGlID> mySelected; }; // .h:267
```

Methods: `select / deselect / toggleSelection / isSelected / clear / getSelected`. The view consults `gSelected` during `drawGL` to apply the highlight colour.

---

## 5. Memory Architecture

### 5.1 Allocation discipline

* All network elements (`MSEdge`, `MSLane`, `MSJunction`, `MSVehicle`, `MSRoute`, …) are **heap‑allocated** with bare `new` in builders.
* The only stack‑allocated singletons of note are `FXApp application` in `guisim_main.cpp:76` and the `FXMutex` members embedded in long‑lived objects.
* Smart pointers are sparing:
  * `std::unique_ptr<MSDynamicShapeUpdater> MSNet::myDynamicShapeUpdater` (`MSNet.h:1052`) - RAII for a single optional component.
  * `std::shared_ptr<const std::vector<MSLane*>> MSEdge::myLanes` (`MSEdge.h:924`) - the lane vector is shared between the edge and read‑only consumers, but the lanes themselves are still raw pointers owned ultimately by the edge.

### 5.2 The `MSNet` singleton (microsim core)

`src/microsim/MSNet.h` 89‑1062. Singleton accessor: `MSNet* MSNet::getInstance()` (line 139), backed by `static MSNet* myInstance` (line 874).

```cpp
class MSNet {
    MSVehicleControl*          myVehicleControl;            // 897  owns vehicles
    MSTransportableControl*    myPersonControl, myContainerControl; // 899/901
    MSEdgeControl*             myEdges;                     // 903
    MSJunctionControl*         myJunctions;                 // 905
    MSTLLogicControl*          myLogics;                    // 907
    MSInsertionControl*        myInserter;                  // 909
    MSDetectorControl*         myDetectorControl;           // 911
    MSEventControl*            myBeginOfTimestepEvents;     // 913
    MSEventControl*            myEndOfTimestepEvents;       // 915
    MSEventControl*            myInsertionEvents;           // 917
    ShapeContainer*            myShapeContainer;            // 919
    MSEdgeWeightsStorage*      myEdgeWeights;               // 921
    std::map<SumoXMLTag, NamedObjectCont<MSStoppingPlace*>> myStoppingPlaces; // 1006
    std::vector<MSTractionSubstation*> myTractionSubstations;            // 1012
    std::unique_ptr<MSDynamicShapeUpdater> myDynamicShapeUpdater;        // 1052
};
```

The constructor (line 177‑180) takes ownership of the pointers passed in (vehicle control + 3 event controls). The destructor `delete`s every member.

### 5.3 Edges & lanes

`src/microsim/MSEdge.h:77` - `class MSEdge : public Named, public Parameterised`.

```cpp
std::shared_ptr<const std::vector<MSLane*>> myLanes;          // 924  - shared by const ref
MSLaneChanger* myLaneChanger;                                 // 927
MSEdgeVector mySuccessors, myPredecessors;                    // 947 / 952  - non‑owning refs
MSJunction* myFromJunction;                                   // 955
MSJunction* myToJunction;                                     // 956
std::set<MSTransportable*, ComparatorNumericalIdLess> myPersons;     // 959
std::set<MSTransportable*, ComparatorNumericalIdLess> myContainers;  // 962
static DictType myDict;       // 1040  - legacy id‑lookup
static MSEdgeVector myEdges;  // 1045  - global edge list
```

Allocation: `return new MSEdge(id, ...)` in `NLEdgeControlBuilder.cpp:272` (microsim build) - overridden in the GUI build to `return new GUIEdge(...)`.

`src/microsim/MSLane.h:84` - `class MSLane : public Named, public Parameterised`. Key members:

```cpp
typedef std::vector<MSVehicle*> VehCont;                      // 119
PositionVector myShape;                                       // 1467  - embedded, not heap
PositionVector* myOutlineShape = nullptr;                     // 1470
VehCont myVehicles, myPartialVehicles, myTmpVehicles;         // 1486 / 1498 / 1502
MSEdge* const myEdge;                                         // 1533
double myLength, myWidth;                                     // 1522 / 1525
std::vector<IncomingLaneInfo> myIncomingLanes;                // 1557
```

The lane *references* its vehicles (`myVehicles`) - it does not own them.

### 5.4 Vehicles

`src/microsim/MSVehicleControl.h:71`. The vehicle factory and registry.

```cpp
typedef std::map<std::string, SUMOVehicle*> VehicleDictType;  // 656
VehicleDictType myVehicleDict;                                // 658  - owns vehicles
typedef std::map<std::string, MSVehicleType*> VTypeDictType;
VTypeDictType   myVTypeDict;
VTypeDistDictType myVTypeDistDict;                            // 686
#ifdef HAVE_FOX
    MFXSynchQue<SUMOVehicle*, std::vector<SUMOVehicle*>> myPendingRemovals;  // 715
#else
    std::vector<SUMOVehicle*> myPendingRemovals;
#endif
```

Allocation site: `MSVehicleControl::buildVehicle` returns `new MSVehicle(...)` (`MSVehicleControl.cpp:113`). Lifecycle:

* `addVehicle()` (132) - register in `myVehicleDict`.
* `deleteVehicle()` (152) - unregister + `delete`.
* `scheduleVehicleRemoval()` (174) - defer via `myPendingRemovals` (so simulation step can finish without invalidating iterators).

### 5.5 Edge / junction control containers

`src/microsim/MSEdgeControl.h:78`:

```cpp
MSEdgeVector myEdges;                       // 266  - vector<MSEdge*>
struct LaneUsage { MSLane* lane; bool amActive; bool haveNeighbors; };
LaneUsageVector myLanes;                    // 272
std::list<MSLane*> myActiveLanes;           // 275  - currently active lanes
```

All raw pointers - `MSEdgeControl` does not own the edges; it only references them. The owners are the edge dictionary in `MSEdge` itself and `MSNet` via `myEdges` / `MSJunctionControl` for junctions.

### 5.6 GUI variants - inheritance, **not composition**

The GUI builds substitute every microsim class with a subclass that *also* inherits `GUIGlObject` (so it can be drawn). There is no separate "wrapper that holds a pointer" pattern (with one notable exception, `GUIJunctionWrapper`).

| Microsim base | GUI subclass | Header | Pattern |
|---------------|--------------|--------|---------|
| `MSNet` | `GUINet` | `src/guisim/GUINet.h:82` `class GUINet : public MSNet, public GUIGlObject` | inherit |
| `MSEdge` | `GUIEdge` | `src/guisim/GUIEdge.h:51` `class GUIEdge : public MSEdge, public GUIGlObject` | inherit |
| `MSLane` | `GUILane` | `src/guisim/GUILane.h:60` `class GUILane : public MSLane, public GUIGlObject` | inherit |
| `MSVehicle` | `GUIVehicle` | `src/guisim/GUIVehicle.h:52` `class GUIVehicle : public MSVehicle, public GUIBaseVehicle` | multiple inherit |
| `MSPerson` | `GUIPerson` | `src/guisim/GUIPerson.h` | inherit |
| `MSContainer` | `GUIContainer` | `src/guisim/GUIContainer.h` | inherit |
| `MSVehicleControl` | `GUIVehicleControl` | `src/guisim/GUIVehicleControl.h:44` `class GUIVehicleControl : public MSVehicleControl` | inherit, override `buildVehicle` |
| `MSJunction` | **`GUIJunctionWrapper`** | `src/guisim/GUIJunctionWrapper.h` | **wrapper**: holds `MSJunction& myJunction`, kept in `GUINet::myJunctionWrapper`. Junctions are not subclassed because their concrete types (`MSRightOfWayJunction`, `MSInternalJunction`, …) are too varied. |
| `MSE2Collector` etc. | `GUIE2Collector`, `GUIE3Collector`, `GUIInductLoop`, `GUIDetectorWrapper` | `src/guisim/GUIE2Collector.h` etc. | mostly inherit; some are wrappers via `GUIDetectorWrapper` |
| `MSLaneSpeedTrigger` | `GUILaneSpeedTrigger` | inherit |
| `MSTriggeredRerouter` | `GUITriggeredRerouter` | inherit |
| `MSBusStop`, `MSChargingStation`, `MSParkingArea`, `MSOverheadWire`, `MSCalibrator` | `GUIBusStop`, `GUIChargingStation`, `GUIParkingArea`, `GUIOverheadWire`, `GUICalibrator` | inherit |

Override of `buildVehicle` in the GUI:

```cpp
SUMOVehicle* GUIVehicleControl::buildVehicle(SUMOVehicleParameter* defs,
        ConstMSRoutePtr route, MSVehicleType* type,
        const bool ignoreStopErrors, ...) {
    MSVehicle* built = new GUIVehicle(defs, route, type, ...);          // GUIVehicleControl.cpp:52
    initVehicle(built, ignoreStopErrors, addRouteStops, source);
    return built;
}
```

Likewise `GUIPerson` is created by the GUI variant of `MSTransportableControl`.

`GUILane` adds rendering caches that the microsim version doesn't need:

```cpp
std::vector<double> myShapeRotations,  myShapeRotations2;  // 353/354
std::vector<double> myShapeLengths,    myShapeLengths2;    // 357/358
mutable std::vector<RGBColor> myShapeColors, myShapeColors2; // 361/362
std::vector<int> myShapeSegments;                          // 365
PositionVector myShape2; double myLengthGeometryFactor2;   // 392/393
mutable FXMutex myLock;                                    // 400
```

### 5.7 Object registry - `GUIGlObjectStorage`

`src/utils/gui/globjects/GUIGlObject.h:68‑199` and `GUIGlObjectStorage.h:49‑156`.

* `typedef unsigned int GUIGlID;`
* Each `GUIGlObject` carries `myGlID`, `myMicrosimID`, `myFullName`, `myGLObjectType`, `myAmBlocked`.
* The singleton `GUIGlObjectStorage::gIDStorage` (line 129) holds:

  ```cpp
  std::vector<GUIGlObject*>           myObjects;       // 136
  std::map<std::string, GUIGlObject*> myFullNameMap;   // 139
  GUIGlID                             myNextID;        // 142
  mutable FXMutex                     myLock;          // 145
  ```

* Lookup that protects against use‑after‑free:
  * `getObjectBlocking(id)` (line 77) increments an internal access count and returns the object;
  * the caller must call `unblockObject(id)` (line 111) when done;
  * `remove(id)` (line 97) defers actual deletion until all blockers have released.

This is critical: a GUI thread holding a `GUIVehicle*` it picked up at the cursor cannot have it deleted under it by the simulation thread mid‑draw.

### 5.8 Stack vs heap - quick summary

| Heap (always `new`) | Stack |
|---------------------|-------|
| `MSEdge`, `GUIEdge` | `FXApp application` in `guisim_main.cpp:76` |
| `MSLane`, `GUILane` | All FXMutex / std::mutex members (embedded in heap objects) |
| `MSVehicle`, `GUIVehicle`, `MSPerson`, `GUIPerson` | `OptionsCont` singleton, `gIDStorage`, `gSelected`, `gSchemeStorage` (statics) |
| `MSJunction`, `GUIJunctionWrapper` | Local `FXMutexLock locker(...)` RAII helpers |
| `MSRoute`, `MSVehicleType` | |
| `GUINet`, `MSVehicleControl` | |
| All toolbars/menus (`new` inside `dependentBuild`) | |

---

## 6. Data Sharing Between Simulation and GUI

### 6.1 Headline answer

*The GUI does not copy simulation data.* Every `GUIEdge` **is** an `MSEdge`, every `GUIVehicle` **is** an `MSVehicle`. When the renderer asks a vehicle for its position, it reads the live microsim state through the same pointer the simulation is updating. There is therefore **no separate "GUI cache"** for live state - only auxiliary geometric caches (precomputed lane rotations, halo geometry, etc.) that don't change once the network is built.

### 6.2 Push or pull?

**Pull.** Updates are not pushed object‑by‑object to the GUI. Instead:

1. The run thread executes `MSNet::simulationStep()` (mutating the network in place).
2. It releases the per‑net lock and posts a single `GUIEvent_SimulationStep` event.
3. The GUI handler eventually calls `view->update()` which schedules `onPaint`.
4. During `onPaint`, the view *pulls* current state out of the live `GUIEdge / GUILane / GUIVehicle` objects via virtual `drawGL` calls.

The only data carried through the queue is metadata: which type of event, time step, message string, end‑of‑sim reason.

### 6.3 Synchronisation boundaries

| Lock | File:line | Held during | Purpose |
|------|-----------|-------------|---------|
| `GUIRunThread::mySimulationLock` | `GUIRunThread.h:184` | the whole `simulationStep()` plus init/cleanup | coarse barrier protecting "the simulation is currently advancing" |
| `GUINet::myLock` | `GUINet.h:462` | inside `GUINet::simulationStep` (RAII via `FXMutexLock`) | overlaps with `mySimulationLock`; ensures mesoscopic and microscopic step paths don't race |
| `GUILane::myLock` | `GUILane.h:400` | `getVehiclesSecure()` / `releaseVehicles()` and inside lane drawing | makes the per‑lane vehicle list iteration consistent across run and GUI threads |
| `GUIEdge::myLock` | `GUIEdge` (analogue) | drawing persons / containers / meso vehicles on an edge | same idea, edge granularity |
| `GUIRunThread::myBreakpointLock` | `GUIRunThread.h:190` | reading/writing `myBreakpoints` | breakpoint list is mutated from GUI, read from run thread |
| `MFXSynchQue::myMutex` | `MFXSynchQue.h:196` | inside every `push_back/top/pop/empty` | event queue thread‑safety |
| `GUIGlObjectStorage::myLock` | `GUIGlObjectStorage.h:145` | id↔object translation, register/remove | id lookups during picking are atomic |
| `GUIVehicleControl::myLock` | `GUIVehicleControl.h:123` | adding/removing/iterating vehicles | mirror of the microsim vehicle dict |

The drawing functions themselves (`GUILane::drawGL`, `GUIEdge::drawGL`) lock the per‑object mutex when iterating vehicles:

```cpp
const GUILane::VehCont& GUILane::getVehiclesSecure() const {
    myLock.lock();                                   // GUILane.h:108‑125 (declaration)
    return myVehicles;
}
void GUILane::releaseVehicles() const { myLock.unlock(); }

// Drawing path:
FXMutexLock locker(myLock);                          // GUILane.cpp:161
for (MSVehicle* const veh : vehicles) { /* draw */ }
```

The run thread acquires the *same* `GUILane::myLock` inside `planMovements / executeMovements / integrateNewVehicles` (via the standard `MSLane` API), so the snapshot rendered for a lane is internally consistent.

### 6.4 What is **not** synchronised

* `mySimDelay` is read by the run thread without a lock - a `double` store is atomic on every platform SUMO targets.
* `myHalting`, `mySingle`, `myQuit` flags are also read/written without locks. They are simple booleans whose worst‑case race effect is "one extra step" or "one extra sleep" - both benign.
* During drawing, the GUI does **not** acquire `mySimulationLock`. While a single lane's drawing is consistent, vehicle position across two lanes can be observed from two different simulation half‑steps (one lane post‑move, the other still pre‑move). This is a deliberate trade‑off in favour of responsiveness - see §10.

### 6.5 Ownership boundaries summary

```
GUINet            ──── owns ──→  GUIVehicleControl   ─── owns ──→ map<id, SUMOVehicle*>
                                                                    │ (heap GUIVehicle*)
                  ──── owns ──→  MSEdgeControl       ─── refs ──→ vector<MSEdge*>          (owners: dictionaries / GUINet)
                  ──── owns ──→  MSJunctionControl   ─── refs ──→ junctions
                  ──── owns ──→  myDetectorControl, myLogics, …
                  ──── holds ─→  myEdgeWrapper/myJunctionWrapper (vector<GUIEdge*>, vector<GUIJunctionWrapper*>)
                  ──── holds ─→  myGrid (SUMORTree) ──→ pointers to the same GUIGlObjects

GUIRunThread      ──── borrows pointer ──→ GUINet (does not own)
GUILoadThread     ──── creates GUINet, hands it to GUI via GUIEvent_SimulationLoaded
GUIApplicationWindow ─ owns ──→ GUILoadThread, GUIRunThread, MFXSynchQue<GUIEvent*>
```

All the actual graph objects are reachable from `GUINet` (via control objects and dictionaries) **and** from the spatial index (`myGrid` / `SUMORTree`) **and** from `GUIGlObjectStorage::gIDStorage`. They are deleted exactly once when `GUIRunThread::deleteSim` runs `delete myNet;` (`GUIRunThread.cpp` ~310) - at which point `GUIGlObjectStorage::gIDStorage.clear()` is also called.

---

## 7. Communication Mechanism

### 7.1 Three pillars

1. **`MFXSynchQue<GUIEvent*>`** - the bounded‑less FIFO that buffers events.
2. **`FXEX::MFXThreadEvent`** (a thin wrapper around an OS pipe/event handle) - wakes the GUI's FOX event loop after a producer push.
3. **The FOX message map** - translates the wake‑up into a normal handler invocation on the GUI thread.

### 7.2 `MFXSingleEventThread` - `src/utils/foxtools/MFXSingleEventThread.{h,cpp}` (32‑79)

Inherits both `FXObject` and `FXThread`. Exposes:

```cpp
void signal();
void signal(FXuint seltype);
long onThreadSignal(FXObject*, FXSelector, void*);   // SEL_IO_READ → SEL_THREAD
long onThreadEvent (FXObject*, FXSelector, void*);   // → client->eventOccurred()
```

Implementation (`.cpp` 58‑130):
* On Unix, `pipe(event)` is created in the constructor; FOX is told to monitor the read end via `INPUT_READ`.
* On Windows, `CreateEvent()` is used.
* `signal()` writes one byte to the pipe (Unix) or sets the event handle (Windows).
* When FOX detects readability, it dispatches `SEL_IO_READ` → `onThreadSignal`, which reads the byte and re‑emits a `SEL_THREAD` selector handled by `onThreadEvent`.

`GUILoadThread` and `GUIRunThread` both inherit from `MFXSingleEventThread` (so they each have their own pipe) but **also** receive a separate `FXEX::MFXThreadEvent` from the main window - this is the actual signal channel used to wake the GUI.

### 7.3 `FXEX::MFXThreadEvent` - `src/utils/foxtools/MFXThreadEvent.{h,cpp}`

Stand‑alone object. Configured by `setTarget()` and `setSelector()` (in `dependentBuild`, lines 298‑301). When a worker calls `event.signal()`:

1. It writes to the pipe / sets the Win32 event.
2. FOX picks up the SEL_IO_READ in the GUI thread.
3. `onThreadSignal` reads, then dispatches `SEL_THREAD_EVENT` with the configured selector to the configured target → `GUIApplicationWindow::onLoadThreadEvent` or `onRunThreadEvent`.

### 7.4 The synchronised queue - `src/utils/foxtools/MFXSynchQue.h`

```cpp
template<class T, class Container = std::list<T>>
class MFXSynchQue {
    mutable FXMutex myMutex;     // 196
    Container       myItems;     // 198
    bool            myCondition; // 199 - can disable locking
public:
    void push_back(T what);      // lock / push / unlock
    T    top();                  // lock / front / unlock
    void pop();                  // lock / pop_front / unlock
    bool empty(), size(), contains(...), clear();
    Container& getContainer();   // lock; caller MUST call unlock()
    void unlock();
};
```

Note: there is no condition variable. Producer wake‑up is via the separate `FXEX::MFXThreadEvent`.

### 7.5 Event types - `src/utils/gui/events/`

```cpp
enum class GUIEventType {                         // GUIEvent.h:32‑76
    SIMULATION_LOADED,    SIMULATION_STEP,
    MESSAGE_OCCURRED,     WARNING_OCCURRED,
    ERROR_OCCURRED,       DEBUG_OCCURRED,
    GLDEBUG_OCCURRED,     STATUS_OCCURRED,
    ADD_VIEW,             CLOSE_VIEW,
    SIMULATION_ENDED,
    OUTPUT_OCCURRED,      TOOL_ENDED,
    END
};
```

Concrete payloads:

| Event | File | Payload |
|-------|------|---------|
| `GUIEvent_SimulationLoaded` | `src/gui/GUIEvent_SimulationLoaded.h:47‑92` | `GUINet*`, begin/end times, file, settings list, `osgView`, viewport flag |
| `GUIEvent_SimulationStep` | `src/utils/gui/events/GUIEvent_SimulationStep.h:34‑43` | none |
| `GUIEvent_SimulationEnded` | `src/gui/GUIEvent_SimulationEnded.h:38‑76` | `MSNet::SimulationState reason`, `SUMOTime step` |
| `GUIEvent_Message` | `src/utils/gui/events/GUIEvent_Message.h:36‑83` | `MsgType`, `string` |
| `GUIEvent_AddView` / `GUIEvent_CloseView` | `src/utils/gui/events/GUIEvent_AddView.h` etc. | caption / scheme / 3D flag |

### 7.6 Producer side (run thread)

```cpp
mySimulationLock.lock();
myNet->simulationStep();
myNet->guiSimulationStep();
mySimulationLock.unlock();
myEventQue.push_back(new GUIEvent_SimulationStep());
myEventThrow.signal();                      // GUIRunThread.cpp:193‑201
```

(`makeStep` may also push a `GUIEvent_SimulationEnded` and toggle `myHalting` - see 204‑226.)

### 7.7 Consumer side (GUI thread)

```cpp
long GUIApplicationWindow::onRunThreadEvent(FXObject*, FXSelector, void*) {
    eventOccurred();                       // GUIApplicationWindow.cpp:1784‑1787
    return 1;
}

void GUIApplicationWindow::eventOccurred() {                // .cpp:1791‑1843
    while (!myEvents.empty()) {
        GUIEvent* e = myEvents.top();
        myEvents.pop();
        switch (e->getOwnType()) {
            case GUIEventType::SIMULATION_LOADED: handleEvent_SimulationLoaded(e); break;
            case GUIEventType::SIMULATION_STEP:
                if (myRunThread->networkAvailable()) handleEvent_SimulationStep(e); break;
            case GUIEventType::MESSAGE_OCCURRED:
            case GUIEventType::WARNING_OCCURRED:
            case GUIEventType::ERROR_OCCURRED:
            case GUIEventType::STATUS_OCCURRED:   handleEvent_Message(e);          break;
            case GUIEventType::ADD_VIEW:          /* open new view */              break;
            case GUIEventType::CLOSE_VIEW:        /* close view */                 break;
            case GUIEventType::SIMULATION_ENDED:  handleEvent_SimulationEnded(e);  break;
            default: break;
        }
        delete e;
    }
    myToolBar2->forceRefresh();
    myToolBar3->forceRefresh();
}
```

`handleEvent_SimulationStep` (`.cpp` 2021‑2081) updates the time LCD, vehicle/person counters, refreshes toolbars, calls `updateChildren()`, and finally `update()` - which schedules an `onPaint` on each MDI child view.

### 7.8 Backpressure behaviour

The producer never blocks: it always pushes and continues. If the GUI thread is too slow (`mySimDelay` is small), multiple `GUIEvent_SimulationStep` events can accumulate; the GUI is implicitly safe because it drains the entire queue per wake‑up. On Windows, `handleEvent_SimulationStep` additionally throttles redraws to ~50 fps via `myLastStepEventMillis` (lines 2024‑2030). On Unix, the run thread inserts a 100 ms sleep at least once per second so the GUI can refresh even when the user has set `mySimDelay = 0` (`GUIRunThread.cpp:172‑177`).

### 7.9 Synchronicity

* GUI → simulation actions (`onCmdStart`, `onCmdStop`, `onCmdStep`) are **asynchronous in spirit**: the GUI flips a flag and returns; the run thread observes the change at the top of its next iteration.
* Simulation → GUI notifications are **asynchronous and lossy in latency, never lossy in events** - every step posts at least one event and the queue is unbounded.

### 7.10 TraCI

`TraCIServer::openSocket(execs)` is called in `GUILoadThread.cpp:175‑179`, registering GUI‑specific commands (`CMD_GET_GUI_VARIABLE`, `CMD_SET_GUI_VARIABLE`) handled by `src/gui/TraCIServerAPI_GUI.{h,cpp}`. TraCI command processing is then driven from `MSNet::simulationStep()` (lines 805‑822) - so it executes **on the run thread**, not on a separate thread. Commands that need GUI side‑effects (e.g. open a new view) post a `GUIEvent_AddView` / `GUIEvent_CloseView`; everything else mutates simulation state directly.

### 7.11 Shutdown

```cpp
GUIApplicationWindow::~GUIApplicationWindow() {              // .cpp:408‑447
    myRunThread->prepareDestruction();   // myQuit = true; myHalting = true;
    myRunThread->join();
    closeAllWindows();
    delete myRunThread;
    delete myLoadThread;
    while (!myEvents.empty()) { delete myEvents.top(); myEvents.pop(); }
}
```

`GUIRunThread::~GUIRunThread()` (`.cpp:72‑81`) sets `myQuit = true`, calls `deleteSim()` (which acquires `mySimulationLock`, calls `myNet->closeSimulation`, busy‑waits on `mySimulationInProgress`, deletes the net, clears `gIDStorage`), and then spins on `mySimulationInProgress || myNet != nullptr` to make absolutely sure no one is still touching the net.

---

## 8. Rendering / Drawing Architecture

### 8.1 The view widget chain

`GUISUMOAbstractView : FXGLCanvas` is the base GL widget. Concrete views are:

* `GUIViewTraffic` - `src/gui/GUIViewTraffic.{h,cpp}` (class at `.h:52`) - the 2D OpenGL renderer used by sumo‑gui.
* `GUIOSGView` (only when `HAVE_OSG`) - OpenSceneGraph 3D view (out of scope for this report).

Shared FXGLVisual is created once on `GUIMainWindow` (line 58) and passed to every canvas, so all views share GL textures/programs/display lists.

### 8.2 The paint trigger

```cpp
FXMAPFUNC(SEL_PAINT, 0, GUISUMOAbstractView::onPaint)        // .cpp:109

long GUISUMOAbstractView::onPaint(FXObject*, FXSelector, void*) {  // .cpp:1053‑1064
    if (!isEnabled() || !myAmInitialised) return 1;
    if (makeCurrent()) {                                     // bind GL context
        paintGL();
        makeNonCurrent();                                    // release context
    }
    myApp->handle(this, FXSEL(SEL_COMMAND, MID_RUNTESTS), nullptr);
    return 1;
}
```

`update()` from FOX schedules a `SEL_PAINT` on the next event‑loop tick. Triggers in this codebase:

* `handleEvent_SimulationStep` ends with `update()` (`.cpp:2080`).
* Zoom/pan/rotation handlers in `GUISUMOAbstractView` and `GUIDanielPerspectiveChanger`.
* Colour scheme change (`GUIViewTraffic::setColorScheme` at `.cpp:168/518`).
* Selection changes (`gSelected` notifies registered targets; see §4.10).

There is **no** continuous timer driving redraws. The architecture is purely event‑driven.

### 8.3 `paintGL()` - the per‑frame skeleton

`src/utils/gui/windows/GUISUMOAbstractView.cpp:317‑367`:

```cpp
void GUISUMOAbstractView::paintGL() {
    GLHelper::resetMatrixCounter();                          // 319
    GLHelper::resetVertexCounter();
    processPendingTextureDeletes();                          // 326

    glClearColor(myVisualizationSettings->backgroundColor);  // 334‑338
    glClear(GL_DEPTH_BUFFER_BIT | GL_COLOR_BUFFER_BIT);      // 339
    glPolygonMode(GL_FRONT_AND_BACK, GL_FILL);
    glEnable(GL_BLEND);
    glDisable(GL_LINE_SMOOTH);

    Boundary bound = applyGLTransform();                     // 350
    doPaintGL(GL_RENDER, bound);                             // 351 - virtual
    // ... legends, FPS …
    swapBuffers();                                           // 366
}
```

`applyGLTransform` (`.cpp:1973‑2003`):

```cpp
glMatrixMode(GL_PROJECTION); glLoadIdentity();
glOrtho(0, getWidth(), 0, getHeight(), -GLO_MAX-1, GLO_MAX+1);
glMatrixMode(GL_MODELVIEW); glLoadIdentity();
double scaleX = (double)getWidth()  / bound.getWidth();
double scaleY = (double)getHeight() / bound.getHeight();
glScaled(scaleX, scaleY, 1);
glTranslated(-bound.xmin(), -bound.ymin(), 0);
if (myChanger->getRotation() != 0) { translate / rotate / translate around centre }
```

Important consequences:
* Orthographic projection - no perspective foreshortening.
* `z` is used as a **layer**: every drawable is drawn with a z‑translation equal to `−GUIGlObjectType` (lanes below junctions below vehicles below labels), so depth testing yields the right occlusion order.
* The scale/translate is computed from the perspective changer's viewport, so the model‑view transform is *exactly* the network‑coords → screen‑coords mapping.

### 8.4 `GUIViewTraffic::doPaintGL` - the body

`src/gui/GUIViewTraffic.cpp:348‑395`:

```cpp
int GUIViewTraffic::doPaintGL(int mode, const Boundary& bound) {
    glRenderMode(mode);                                      // 350
    GLHelper::pushMatrix();
    glDisable(GL_TEXTURE_2D);
    glEnable(GL_BLEND);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
    glEnable(GL_DEPTH_TEST);

    drawDecals();                                            // 360
    if (myVisualizationSettings->showGrid) paintGLGrid();    // 363

    const SUMORTree& grid = GUINet::getGUIInstance()->getVisualisationSpeedUp(...);
    const float minB[2] = { (float)bound.xmin(), (float)bound.ymin() };
    const float maxB[2] = { (float)bound.xmax(), (float)bound.ymax() };
    int hits2 = grid.Search(minB, maxB, *myVisualizationSettings);   // 372

    for (auto& a : myAdditionallyDrawn) {                    // 377‑381
        a.first->drawGLAdditional(this, *myVisualizationSettings);
    }
    GLHelper::popMatrix();
    return hits2;
}
```

The single line that does most of the work is `grid.Search(...)` - see §8.5.

### 8.5 Spatial culling + drawing in one call: `SUMORTree`

`src/foreign/rtree/SUMORTree.h:60‑244`:

```cpp
class SUMORTree : private GUI_RTREE_QUAL, public Boundary {
    SUMORTree() : GUI_RTREE_QUAL(&GUIGlObject::drawGL), ... { }   // 70 - bind callback
    void Insert(...);  void Remove(...);                          // 89 / 100
    int  Search(const float min[2], const float max[2],
                const GUIVisualizationSettings& c) const;         // 114
    void addAdditionalGLObject(GUIGlObject*);                     // 122
};
```

`RTree.h:79`:

```cpp
typedef void (DATATYPENP::*Operation)(const CONTEXT&) const;
```

The R‑tree is templated such that `DATATYPE = GUIGlObject*`, `CONTEXT = GUIVisualizationSettings`, `Operation = &GUIGlObject::drawGL`. Inside `Search` (`RTree.h:1553‑1597`):

```cpp
for (int index = 0; index < a_node->m_count; ++index) {
    if (Overlap(a_rect, &a_node->m_branch[index].m_rect)) {
        DATATYPE& id = a_node->m_branch[index].m_data;
        ++a_foundCount;
        (id->*myOperation)(c);                              // calls drawGL(s)
    }
}
```

So `Search` is *both* a culling query *and* the drawing loop. Each visible object is asked to draw itself.

### 8.6 `GUIGlObject::drawGL` overrides

```cpp
virtual void drawGL(const GUIVisualizationSettings& s) const = 0;     // GUIGlObject.h:196
```

Representative implementations:

* **`GUILane::drawGL`** - `src/guisim/GUILane.cpp:546‑830+`
    * `GLHelper::pushName(getGlID())` (548) - for picking.
    * Compute scale/colour (550‑558).
    * Branch on rail / crossing / standard lane (630‑707).
    * Standard path: `GLHelper::drawBoxLines(myShape, myShapeRotations, myShapeLengths, halfWidth)` - emits `GL_QUADS` along the lane shape using the cached rotations/lengths.
    * Direction arrows, sublane markings, traffic info (715‑830).
    * Vehicles on the lane are drawn here too:
      ```cpp
      FXMutexLock locker(myLock);                    // .cpp:161
      for (MSVehicle* veh : myVehicles) /* drawGL */;
      ```
* **`GUIBaseVehicle::drawGL`** - `src/guisim/GUIBaseVehicle.cpp:662‑664`
    ```cpp
    void GUIBaseVehicle::drawGL(const GUIVisualizationSettings& s) const {
        drawOnPos(s, getVisualPosition(s.secondaryShape),
                     getVisualAngle  (s.secondaryShape));
    }
    ```
    `getVisualPosition` reads the live `MSVehicle` state (no cache).
    Colour comes from a priority chain (715‑780): emergency type → vehicle param colour → vtype colour → route colour → active colorer scheme.
* **`GUIJunctionWrapper::drawGL`** - `src/guisim/GUIJunctionWrapper.cpp:152‑216`
    * Skip if smaller than min size threshold (154‑157).
    * Compute colour, draw filled / tesselated polygon (158‑186) using `GLHelper::drawFilledPoly` / `GLHelper::drawFilledPolyTesselated`.
    * Optional ID and TLS labels (192‑215).

`GLHelper` (`src/utils/gui/div/GLHelper.{h,cpp}`) is the lowest‑level helper: `drawFilledPoly`, `drawFilledPolyTesselated`, `drawBoxLine(s)`, `drawFilledCircle`, `drawRectangle`, `pushName/popName` for picking.

### 8.7 Visualisation settings and culling thresholds

`src/utils/gui/settings/GUIVisualizationSettings.h:49‑200+`. The most important fields used in culling decisions:

```cpp
double scale;                                  // current zoom (m / pixel)
double laneMinSize, vehicleMinSize, ...;       // minimal display sizes
double laneWidthExaggeration, ...;             // visual exaggeration
GUIColorer laneColorer, vehicleColorer, junctionColorer, ...;
bool   drawForRectangleSelection;              // simplify drawing for picking
bool   drawForObjectUnderCursor;               // pixel‑accurate picking
GUIVisualizationTextSettings vehicleID, junctionID, ...;  // label visibility
RGBColor backgroundColor;
GUIVisualizationSizeSettings junctionSize;
```

Per‑object check inside `GUILane::drawGL`:

```cpp
if (color.alpha() != 0 && s.scale * exaggeration > s.laneMinSize) {
    /* draw */
}                                                // GUILane.cpp:560‑561
```

Schemes are stored in the global `gSchemeStorage` (`GUICompleteSchemeStorage`).

### 8.8 Picking and rectangle‑select

`getObjectUnderCursor(double sensitivity)` (`GUISUMOAbstractView.cpp:413‑415`) → `getObjectAtPosition(...)` (438‑476) → `getObjectsInBoundary(Boundary)` (544‑566).

`getObjectsInBoundary` is the picking primitive:

```cpp
std::vector<GUIGlID> GUISUMOAbstractView::getObjectsInBoundary(Boundary bound) {
    glSelectBuffer(NB_HITS_MAX, hits);
    glInitNames();
    myVisualizationSettings->scale = m2p(SUMO_const_laneWidth);
    Boundary oldVP = myChanger->getViewport(false);
    myChanger->setViewport(bound);
    bound = applyGLTransform(false);
    myVisualizationSettings->drawForRectangleSelection = true;
    int hits2 = doPaintGL(GL_SELECT, bound);
    myVisualizationSettings->drawForRectangleSelection = false;
    nb_hits = glRenderMode(GL_RENDER);
    /* read out hit names → GUIGlIDs */
    return result;
}
```

This is classic OpenGL `GL_SELECT` mode: the same `doPaintGL` runs, but `glRenderMode(GL_SELECT)` causes geometry to write hit records (with the names pushed via `GLHelper::pushName`) instead of fragments. Resolution of multiple hits at one cursor location is done by `getObjectAtPosition` using a priority comparator (`ComparatorClickPriority` at `GUISUMOAbstractView.h:62‑72`): higher `getClickPriority()` wins; ties broken by 2D distance.

### 8.9 What about live data during draw?

Drawing reads live data directly:

* `GUIVehicle::getVisualPosition()` reads its own `MSVehicle` state (no copy).
* `GUILane::myVehicles` is the live vehicle list, read under `GUILane::myLock`.
* `GUIEdge::myPersons / myContainers` similarly read under `GUIEdge::myLock`.

There are no per‑frame snapshots and no display lists for vehicle geometry - drawing is immediate‑mode (`glBegin`/`glVertex`/`glColor`/`glEnd` via `GLHelper`). The only retained geometry is the **static** lane/junction shapes (already in `MSLane::myShape` / junction shape, plus the precomputed rotations and lengths in `GUILane`).

### 8.10 Frame summary

```
SEL_PAINT (FOX) → onPaint
   makeCurrent()
   paintGL()
       glClear / blend / depth state
       applyGLTransform()                ← projection + model‑view from myChanger
       doPaintGL(GL_RENDER, bound)
           drawDecals
           paintGLGrid   (optional)
           grid.Search(bounds, *settings)            ← R‑tree culling + draw
                for each overlap:  obj->drawGL(s)
                    GUILane::drawGL          → GLHelper::drawBoxLines / etc.
                                              ↳ for each veh in myVehicles → veh->drawGL
                    GUIJunctionWrapper::drawGL → GLHelper::drawFilledPoly
                    GUIEdge::drawGL          → persons/containers (locks myLock)
                    GUIBaseVehicle::drawGL   → drawOnPos(...)
                    detectors / shapes / TLS / additionals
           additional drawables (myAdditionallyDrawn)
       legends + FPS
       swapBuffers()
   makeNonCurrent()
```

---

## 9. End‑to‑End Flow Diagrams

### 9.1 GUI startup

```
main()                                                     [guisim_main.cpp:53]
 ├─ XMLSubSys::init / OptionsIO / MSFrame::fillOptions     [67‑70]
 ├─ FXApp application(...)                                 [76]
 ├─ application.init(argc, argv)                           [78]
 ├─ FXGLVisual::supported() check                          [80]
 ├─ window = new GUIApplicationWindow(&application)        [85]
 │    └─ GUIMainWindow ctor                                [GUIMainWindow.cpp:54]
 │         ├─ FXMainWindow ctor
 │         ├─ myGLVisual = new FXGLVisual(VISUAL_DOUBLEBUFFER)   [.cpp:58]
 │         └─ docks, fonts, tooltips
 ├─ window->dependentBuild(false)                          [GUIApplicationWindow.cpp:284]
 │    ├─ menu bar + 8 toolbars (buildToolBars)             [293‑296]
 │    ├─ thread events                                      [298‑301]
 │    ├─ status bar                                         [303‑324]
 │    ├─ MDI client + splitter + message window             [326‑334]
 │    ├─ fillMenuBar()                                      [336]
 │    ├─ myLoadThread = new GUILoadThread(...)              [342]
 │    ├─ myRunThread  = new GUIRunThread(...)               [343]
 │    └─ myRunThread->start()                               [349]   ← idles on myHalting
 ├─ application.create()                                    [91]
 ├─ window->loadOnStartup()  if argc > 1                    [94]
 └─ application.run()                                       [98]    ← FOX event loop
```

### 9.2 Open network / config

```
User clicks File ▸ Open                                    [menu MID_HOTKEY_CTRL_O_…]
 └─ onCmdOpenConfiguration / onCmdOpenNetwork              [GUIApplicationWindow.cpp:1065 / 1085]
      ├─ FXFileDialog                                       (FOX file picker)
      ├─ recent files update
      └─ loadConfigOrNet(file)                              [.cpp:2206]
            ├─ myAmLoading = true; closeAllWindows()
            └─ myLoadThread->loadConfigOrNet(file)
                  └─ start() → GUILoadThread::run()         [GUILoadThread.cpp:84‑235]
                       ├─ OptionsCont::setByRootElement ... [.cpp:94‑149]   parses .sumocfg
                       ├─ new GUIVehicleControl                              [155]
                       ├─ new GUINet(... GUIEventControls ...)               [165]
                       ├─ new GUIEdgeControlBuilder, GUIDetectorBuilder,
                       │   NLJunctionControlBuilder, GUITriggerBuilder       [171‑175]
                       ├─ NLBuilder(...).build()                              [187‑213]
                       │    ├─ XMLSubSys::runParser(NLHandler, .net.xml)
                       │    │     └─ callbacks → builders → new GUIEdge / GUILane / …
                       │    └─ buildNet() → MSNet::closeBuilding
                       ├─ net->initGUIStructures()           [GUINet.cpp:276‑360]
                       │    ├─ build detector / calibrator wrappers
                       │    ├─ initTLMap()
                       │    ├─ myEdgeWrapper / myJunctionWrapper population
                       │    └─ myGrid (SUMORTree) Insert(boundary, ptr)
                       └─ submitEndAndCleanup(...)           [.cpp:238‑253]
                            ├─ myEventQue.push_back(new GUIEvent_SimulationLoaded(...))
                            └─ myEventThrow.signal()
                                  ↓
                                  FOX dispatches SEL_THREAD_EVENT
                                       └─ onLoadThreadEvent → eventOccurred()  [.cpp:1791]
                                            └─ handleEvent_SimulationLoaded     [.cpp:1847]
                                                 ├─ myAmLoading = false
                                                 ├─ myRunThread->init(net, begin, end)
                                                 ├─ openNewView(VIEW_2D_OPENGL)  [.cpp:2222]
                                                 │    └─ new GUISUMOViewParent
                                                 │         └─ init(canvas, net, vt)
                                                 │              └─ new GUIViewTraffic
                                                 └─ apply settings / breakpoints / delay
```

### 9.3 Start simulation

```
User clicks Play (Ctrl+A or Space)                         [accelerator → MID_HOTKEY_CTRL_A_…]
 └─ onCmdStart                                             [GUIApplicationWindow.cpp:1277]
       ├─ if (!myWasStarted) myRunThread->begin()          [GUIRunThread.cpp:271]
       └─ myRunThread->resume()                            [GUIRunThread.cpp:257]
              myHalting = false; mySingle = false          ← gate flag flipped

[Run thread] next tryStep() iteration                      [GUIRunThread.cpp:142]
       ├─ check breakpoints (locked)                       [150‑156]
       └─ makeStep()                                        [.cpp:186‑253]
              ├─ mySimulationLock.lock()
              ├─ myNet->simulationStep()                   [GUINet.cpp:242 / MSNet.cpp:794]
              ├─ myNet->guiSimulationStep()                [GUINet.cpp:235]
              ├─ mySimulationLock.unlock()
              ├─ push GUIEvent_SimulationStep
              └─ myEventThrow.signal()
                  ↓
                  onRunThreadEvent → eventOccurred()       [.cpp:1791]
                      └─ handleEvent_SimulationStep         [.cpp:2021‑2081]
                           ├─ updateTimeLCD, vehicle counters
                           ├─ updateChildren()
                           └─ update() ── schedules SEL_PAINT on each view
```

### 9.4 Drawing one frame

```
SEL_PAINT  ─→  GUISUMOAbstractView::onPaint                [.cpp:1053]
                ├─ makeCurrent()
                ├─ paintGL()                                [.cpp:317‑367]
                │    ├─ glClear / blend / depth
                │    ├─ applyGLTransform()                  [.cpp:1973]
                │    │    └─ orthographic + scale + translate from myChanger
                │    ├─ doPaintGL(GL_RENDER, bound)         [GUIViewTraffic.cpp:348]
                │    │    ├─ drawDecals
                │    │    ├─ paintGLGrid (optional)
                │    │    ├─ grid.Search(bounds, *settings) [SUMORTree.h:114]
                │    │    │    └─ for each overlap:
                │    │    │         (id->*&GUIGlObject::drawGL)(s)   [RTree.h:1553]
                │    │    │              ├─ GUILane::drawGL          [GUILane.cpp:546]
                │    │    │              │     └─ GLHelper::drawBoxLines / vehicles
                │    │    │              ├─ GUIJunctionWrapper::drawGL [GUIJunctionWrapper.cpp:152]
                │    │    │              ├─ GUIEdge::drawGL
                │    │    │              ├─ GUIBaseVehicle::drawGL    [GUIBaseVehicle.cpp:662]
                │    │    │              ├─ GUI* detectors / triggers
                │    │    │              └─ POIs / shapes / TLS overlays
                │    │    └─ myAdditionallyDrawn[*].drawGLAdditional
                │    ├─ legends + FPS
                │    └─ swapBuffers()
                └─ makeNonCurrent()
```

---

## 10. Handling Uncertainty

The architectural picture above is grounded in concrete file:line evidence; a few claims are inferences worth flagging.

### 10.1 Statements grounded directly in code

* Thread creation, message map, run/load thread `run()` bodies, event types, lock declarations and acquisitions, drawing call chain, R‑tree callback wiring - all visible in the listed files at the listed lines.
* The fact that `GUIEdge`, `GUILane`, `GUIVehicle`, `GUINet`, `GUIVehicleControl` use **inheritance** rather than composition is verified by their class declarations (e.g. `class GUIEdge : public MSEdge, public GUIGlObject` at `src/guisim/GUIEdge.h:51`).
* The fact that `GUIJunctionWrapper` is a wrapper (not a subclass) is also verified - `MSJunction` has multiple concrete subtypes which are not GUI‑aware.

### 10.2 Inferences (with reasoning)

1. **"The GUI does not copy live simulation data; it reads it directly through subclass pointers."** Inferred from:
   * `GUIVehicle` inherits `MSVehicle`, no per‑frame copy step is visible in `drawGL`, and `getVisualPosition()` is implemented on `MSVehicle`/`GUIVehicle` itself.
   * No "render queue" or "snapshot" container exists in the GUI tree (searched).
   * The R‑tree stores `GUIGlObject*` (which *are* the live objects).
2. **"There is no global lock held during drawing - drawing only acquires per‑lane / per‑edge locks, while the simulation may be advancing concurrently."** Inferred from reading `GUISUMOAbstractView::paintGL`, `GUIViewTraffic::doPaintGL`, and the per‑object `drawGL` implementations - none of them touch `mySimulationLock` or `GUINet::myLock`. The only locks acquired during drawing are `GUILane::myLock` and `GUIEdge::myLock`. Practical consequence: cross‑lane vehicle position can momentarily appear inconsistent (vehicle A pre‑move on lane X, vehicle B post‑move on lane Y).
3. **"TraCI runs on the simulation thread, not on a separate thread."** Inferred from the absence of a TraCI worker thread in the gui binary build, and the fact that `MSNet::simulationStep` itself processes TraCI commands at lines 805‑822.
4. **"`mySimDelay`, `myHalting`, `mySingle`, `myQuit` are read/written without synchronisation by design."** Inferred from declarations (no `std::atomic`, no surrounding lock) and the simplicity of their semantics (every value race is benign because the run loop re‑checks at each iteration).
5. **"Backpressure on the event queue is mostly a no‑op; the GUI drains the entire queue per wake‑up."** Direct evidence: the `while (!myEvents.empty())` drain loop in `eventOccurred` (`.cpp:1792`). The Windows‑only 50 ms throttle (`.cpp:2024‑2030`) is the one exception.

### 10.3 Things **not clear from the code** (or out of scope here)

* The exact behaviour of all TraCI commands when issued through GUI views - verifying which set commands route via `GUIEvent_AddView/CloseView` (clear) vs. directly mutate simulation state (clear) requires walking each `processSet` case.
* The 3D OSG view's rendering specifics - only its existence and selection point in `GUISUMOViewParent::init` were inspected.
* The internals of `GUIDanielPerspectiveChanger` zoom rectangle behaviour beyond the broad outline (left = pan, right = zoom rect, wheel = zoom, middle = rotate).
* Race window during `GUIRunThread::deleteSim` - the spin‑wait on `mySimulationInProgress` at `~GUIRunThread()` looks correct, but interaction with a TraCI client mid‑shutdown was not exhaustively traced.

---

### Index of Important File:Line References

| Topic | File | Lines |
|-------|------|-------|
| Main entry, FXApp, run loop | `src/guisim_main.cpp` | 53‑120 |
| Application window - message map | `src/gui/GUIApplicationWindow.cpp` | 89‑240 |
| Application window - `dependentBuild` | `src/gui/GUIApplicationWindow.cpp` | 284‑352 |
| Open file handlers | `src/gui/GUIApplicationWindow.cpp` | 1065‑1101, 2206‑2218 |
| Start/Stop/Step handlers | `src/gui/GUIApplicationWindow.cpp` | 1277‑1316, 1459‑1493 |
| Event‑queue drain | `src/gui/GUIApplicationWindow.cpp` | 1777‑1843 |
| `handleEvent_SimulationLoaded` / `_Step` | `src/gui/GUIApplicationWindow.cpp` | 1847‑1968, 2021‑2081 |
| `openNewView` | `src/gui/GUIApplicationWindow.cpp` | 2222‑2254 |
| `~GUIApplicationWindow` shutdown | `src/gui/GUIApplicationWindow.cpp` | 408‑447 |
| `GUIMainWindow` ctor + `myGLVisual` | `src/utils/gui/windows/GUIMainWindow.{h,cpp}` | h:46/192‑250; cpp:54‑89 |
| MDI child / view parent | `src/gui/GUISUMOViewParent.{h,cpp}` | h:53; cpp:78‑107 |
| GL canvas base | `src/utils/gui/windows/GUISUMOAbstractView.{h,cpp}` | h:83; cpp:107‑128, 136‑153 |
| Paint pipeline | `src/utils/gui/windows/GUISUMOAbstractView.cpp` | 317‑367, 1053‑1064, 1973‑2003 |
| Picking | `src/utils/gui/windows/GUISUMOAbstractView.cpp` | 413‑476, 544‑566 |
| `GUIViewTraffic::doPaintGL` | `src/gui/GUIViewTraffic.cpp` | 348‑395 |
| Load thread | `src/gui/GUILoadThread.{h,cpp}` | h:47‑100; cpp:84‑263 |
| Run thread | `src/gui/GUIRunThread.{h,cpp}` | h:53‑200; cpp:84‑320 |
| `MSNet::simulationStep` | `src/microsim/MSNet.cpp` | 794‑930 |
| `GUINet::simulationStep / guiSimulationStep / initGUIStructures` | `src/guisim/GUINet.cpp` | 235‑245, 276‑360 |
| Event types | `src/utils/gui/events/GUIEvent.h` | 32‑76 |
| `MFXSynchQue` | `src/utils/foxtools/MFXSynchQue.h` | 38‑207 |
| `MFXSingleEventThread` | `src/utils/foxtools/MFXSingleEventThread.{h,cpp}` | 32‑130 |
| `MFXThreadEvent` | `src/utils/foxtools/MFXThreadEvent.{h,cpp}` | 28‑145 |
| R‑tree callback | `src/foreign/rtree/RTree.h`, `SUMORTree.h` | 79, 1553‑1597; 60‑244 |
| GL drawing helpers | `src/utils/gui/div/GLHelper.{h,cpp}` | 47‑150 |
| Visualisation settings | `src/utils/gui/settings/GUIVisualizationSettings.h` | 49‑200+ |
| Selection storage | `src/utils/gui/div/GUISelectedStorage.{h,cpp}` | h:67‑286 |
| ID storage | `src/utils/gui/globjects/GUIGlObjectStorage.{h,cpp}` | h:49‑156 |
| GUI builders | `src/guinetload/GUIEdgeControlBuilder.{h,cpp}`, `GUIDetectorBuilder.{h,cpp}`, `GUITriggerBuilder.{h,cpp}` | various |
| GUI domain types | `src/guisim/GUI*.h` | class decls at ~line 50‑80 of each |
| Microsim types | `src/microsim/MS*.h` | class decls at top of each |
| App‑wide IDs | `src/utils/gui/windows/GUIAppEnum.h` | 42‑677 |
| Hotkeys | `src/utils/gui/shortcuts/GUIShortcutsSubSys.cpp` | 31+ |
