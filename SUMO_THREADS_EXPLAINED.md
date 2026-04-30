# SUMO‑GUI threading, explained from scratch

Goal: by the end of this document you should understand **why** SUMO‑GUI uses threads, **what** the three threads do, **how** they communicate without stepping on each other, and **where** it all lives in the code - without assuming you've ever written multi‑threaded code before.

We'll build the picture in five parts:

1. What a thread is, and why SUMO needs more than one
2. The three threads in sumo‑gui - meet the team
3. The two tools threads use to coordinate: mutexes and queues
4. End‑to‑end: app launch → opening a file → clicking Play
5. The deep dive: how the simulation and the drawing share network data without corrupting it

---

## 1. What is a thread, and why does SUMO need more than one?

### The problem if you only have one thread

Imagine sumo‑gui had only **one thread of execution** - like a single‑track railway. Every line of code runs one at a time, in order. If you click "Play", that thread starts running the simulation:

```
[ user clicks Play ]
[ thread starts simulating step 1 ]
[ thread simulates step 2 ]
[ thread simulates step 3 ]
[ user moves the mouse - but the thread is busy simulating ]
[ thread keeps simulating ... ]
[ thread finally finishes - only now can it react to the mouse ]
```

The window would **freeze**. You couldn't pan, zoom, or click Stop, because there's no one available to listen to your mouse - the only thread is busy advancing the simulation. Operating systems even pop up "Application not responding" dialogs when this happens.

### The fix: do two things at once

A **thread** is just an independently running stream of instructions. A program with two threads has *two* streams running concurrently - your CPU switches between them many times per second (or runs them truly in parallel on different cores). Each thread has its own *call stack* (what function is it inside, what local variables) but they all share the same heap memory (the same `new` allocations, the same global variables).

So the trick used by every interactive simulator, video game, video editor, and IDE on the planet is the same:

> **One thread handles the user interface and stays responsive. Another thread does the heavy work in the background.**

In SUMO‑GUI:

* The **GUI thread** listens for clicks, redraws the window, animates buttons.
* A **simulation thread** runs the microsimulation, vehicle by vehicle.

Because they run independently, you can pan/zoom/click Stop while vehicles keep moving. The GUI thread never has to "wait its turn".

### A simple mental model

Think of two people working in the same office, sharing a single workbench full of little Lego cars (the network):

* **Alice** (GUI thread) sits at the door. When customers come in, she shows them the workbench and answers questions ("how fast is that red car going?"). She often glances at the bench to take photos for the customer.
* **Bob** (simulation thread) actually moves the Lego cars around the bench, one timestep at a time.

They *can* both work at the same time, but they need rules so Bob doesn't move a car at the exact instant Alice is photographing it (resulting in a blurry photo or a knocked‑over car). Those rules are **mutexes** and **queues**, which we'll get to in §3.

---

## 2. The three threads in sumo‑gui

SUMO‑GUI actually uses **three** threads, not two. Each has a single, focused job.

### Thread 1: the GUI thread

* Started by the OS when you launch the program.
* Lives inside the FOX‑Toolkit event loop.
* Where: `application.run()` at `src/guisim_main.cpp:98`. From this single function call until the program exits, this thread is doing nothing but reading events from FOX and running event handlers.
* Jobs: draw the window, handle keyboard/mouse, show menus, run the dialogs, redraw the network when asked.
* It is the **only** thread that may touch OpenGL. Touching GL from another thread would be a bug.

### Thread 2: the load thread (`GUILoadThread`)

* Born when the GUI thread runs `dependentBuild()` and does `myLoadThread = new GUILoadThread(...)` (`src/gui/GUIApplicationWindow.cpp:342`).
* It exists but isn't *running* yet - it's like a hired worker waiting in the break room.
* When the user picks a `.sumocfg` file, the GUI thread says "go" by calling `myLoadThread->loadConfigOrNet(file)`. That triggers the load thread's `start()` method, which spawns the actual OS thread, which runs `GUILoadThread::run()` (`src/gui/GUILoadThread.cpp:84`).
* Job: parse XML files (`.sumocfg`, `.net.xml`, additional files) and build the in‑memory network - all those `MSEdge`, `MSLane`, `MSJunction`, `GUIEdge`, `GUILane` objects.
* When done, it posts one message to the GUI ("the network is ready, here's the pointer") and the thread exits naturally.

Why is this a separate thread? Because parsing a big network can take seconds. If the GUI thread did the parsing, the window would freeze with a "loading" dialog stuck on screen. By doing it in the background, the GUI keeps drawing animations and responding to the Cancel button.

### Thread 3: the run thread (`GUIRunThread`)

* Also created in `dependentBuild()`, line 343: `myRunThread = new GUIRunThread(...)`.
* Started **immediately** by `myRunThread->start()` at line 349. So even before you've loaded a network, this thread is already alive.
* What does an alive‑but‑idle run thread do? It sits in a loop, asking itself "should I run a step?", checking a boolean flag called `myHalting`, and sleeping 50 ms when the answer is no. Look at its main loop:
    ```cpp
    FXint GUIRunThread::run() {                             // GUIRunThread.cpp:126
        while (!myQuit) {
            tryStep();                                       // sleeps 50ms when halted
        }
        deleteSim();
        return 0;
    }
    ```
* Job: run the simulation, step by step, when not halted. After each step it tells the GUI "a step happened, you might want to redraw".
* It only stops when the user closes the program (`myQuit` is set).

### Putting them on a timeline

```
time →

GUI thread:    [ event loop... event loop... event loop... event loop... ]   (forever)

Load thread:                  [ run() ]
                              ↑       ↑
                              │       └─ exits after posting "loaded" event
                              └─ starts when user picks a file

Run thread:    [ idle...idle...idle ][ step step step step ][ idle ][ step ... ]
               ↑                     ↑                       ↑
               └─ created at startup └─ user clicked Play    └─ user clicked Stop
```

The GUI thread is the longest‑lived. The load thread is one‑shot per load. The run thread is created once but spends most of its time either sleeping or running steps depending on `myHalting`.

---

## 3. How threads coordinate without crashing

Two tools, used everywhere in SUMO‑GUI:

### Tool 1: the **mutex** (a "do not disturb" sign)

A **mutex** is a lock. At any moment, *at most one* thread can be holding the lock. If thread A holds it and thread B tries to acquire it, thread B is paused by the OS until A releases it. This lets you write critical sections like:

```cpp
mutex.lock();
//   ... do stuff that must not be interrupted ...
mutex.unlock();
```

Or the safer RAII style used throughout SUMO:

```cpp
{
    FXMutexLock locker(mutex);   // locks here
    //   ... do stuff ...
}                                // unlocks automatically when 'locker' goes out of scope
```

Mental model: the workbench has a yellow ribbon. Whoever wants to touch a particular Lego car ties the ribbon around it; the other person must wait until the ribbon is removed. Without the ribbon, both people could grab the same car at the same time and break it (this is called a **race condition**).

The thing protected by the ribbon is called the **critical section**. Critical sections should be **short** - every second the ribbon is up is a second the other thread is stuck waiting.

### Tool 2: the **thread‑safe queue** (a mailbox)

If thread A wants to send thread B a notification ("I finished a step!"), it can't just call B's function directly - they're different threads, and B is busy. Instead, A drops a note in a mailbox. B checks its mailbox when convenient and reads the notes.

The mailbox itself uses a small mutex internally so that putting a note in and taking one out are safe. SUMO‑GUI's mailbox is a class called `MFXSynchQue<GUIEvent*>` (`src/utils/foxtools/MFXSynchQue.h`). It holds pointers to event objects. The GUI's mailbox is `myEvents` declared at `GUIApplicationWindow.h:465`.

A note in the mailbox alone isn't enough though, because B might not check it for a long time. So we add a **doorbell**: a special signal called `FXEX::MFXThreadEvent` (`src/utils/foxtools/MFXThreadEvent.h`) that wakes the GUI thread out of its event loop. Worker thread drops the note and rings the bell:

```cpp
myEventQue.push_back(new GUIEvent_SimulationStep());   // drop note
myEventThrow.signal();                                   // ring doorbell
```

The GUI's event loop wakes up, calls `eventOccurred()`, which empties the mailbox by calling `myEvents.top()` and `myEvents.pop()` in a loop, dispatching each note to the right handler.

### Why we need both

* **Mutex without queue:** Two threads could synchronise their access to the same data, but they can't easily *notify* each other that something happened. The other thread would have to constantly poll, wasting CPU.
* **Queue without mutex:** Notifications would arrive, but the underlying data structures (lane vehicle lists, etc.) would still be corrupted by simultaneous reads/writes.

SUMO uses both. **Mutexes protect the data; queue + doorbell deliver the notifications.**

---

## 4. End‑to‑end story

Now we put it all together. We'll walk through three moments: launching the app, opening a file, clicking Play.

### 4.1 Launch

You type `sumo-gui` in a terminal (or double‑click the icon). The OS creates a single process with one thread (the GUI thread will be this thread). It starts running `main()` in `src/guisim_main.cpp:53`.

```
guisim_main.cpp::main()
  ├── XMLSubSys::init()                        prep the XML library
  ├── MSFrame::fillOptions()                   register all CLI options
  ├── OptionsIO::getOptions()                  parse argv
  ├── FXApp application(...)                   create the FOX app object
  ├── application.init(argc, argv)             initialise the display
  ├── new GUIApplicationWindow(&application)   build the main window object
  ├── window->dependentBuild(false)            build menus + toolbars + threads ◄
  ├── application.create()                     ask FOX to actually show widgets
  └── application.run()                        ENTER FOX EVENT LOOP (never returns until quit)
```

The interesting line is `dependentBuild` at `src/gui/GUIApplicationWindow.cpp:284`. Inside it (~line 342–349):

```cpp
myLoadThread = new GUILoadThread(getApp(), this, myEvents, myLoadThreadEvent, isLibsumo);
myRunThread  = new GUIRunThread (getApp(), this, mySimDelay, myEvents, myRunThreadEvent);
... // status bar text
myRunThread->start();   //  ← OS spawns a new thread that runs GUIRunThread::run()
```

Notice what's passed to both threads:
* `myEvents` (a reference to the mailbox).
* `myLoadThreadEvent` / `myRunThreadEvent` (the two doorbells, one per worker).

So both threads can drop notes in the *same* mailbox; the doorbells are different so the GUI knows which thread rang.

After `dependentBuild` returns, `application.run()` is called and we drop into the FOX event loop. Two threads are now alive: the GUI thread (in the event loop) and the run thread (sleeping in `tryStep` because no network is loaded yet).

### 4.2 Opening a `.sumocfg`

You click **File → Open Simulation…**. The OS sends a click event to the FOX event loop, which dispatches it via the **target/selector** message map (`FXDEFMAP` table at `GUIApplicationWindow.cpp:89`). The handler bound to `MID_HOTKEY_CTRL_O_OPENSIMULATION_OPENNETWORK` is `GUIApplicationWindow::onCmdOpenConfiguration` (line 1065).

That handler runs **on the GUI thread**:

```cpp
long onCmdOpenConfiguration(...) {
    FXFileDialog opendialog(...);
    if (opendialog.execute()) {
        loadConfigOrNet(opendialog.getFilename().text());
    }
    return 1;
}
```

`loadConfigOrNet` (line 2206) closes any open views, sets `myAmLoading = true` to grey out the Open menu, and calls:

```cpp
myLoadThread->loadConfigOrNet(file);
```

That method (in `GUILoadThread.cpp:257`) just stores the filename and calls `start()`. **`start()` is the magic moment**: the OS spawns a brand‑new thread, and that new thread begins executing `GUILoadThread::run()` - `src/gui/GUILoadThread.cpp:84`. From this instant on, two threads are running important code:

| GUI thread | Load thread (newly spawned) |
|---|---|
| Returns from `loadConfigOrNet`, goes back into the FOX event loop. Window is responsive: you can move it, see "Loading..." in the status bar, click Cancel. | Starts parsing `OptionsIO::getRoot(myFile)` to read the .sumocfg, then calls `MSFrame::fillOptions()`, then the heavy work: `NLBuilder::build()`, which parses the .net.xml via SAX. |

While the load thread is working, the GUI thread keeps running the event loop, redrawing buttons, showing the busy cursor.

The load thread's `run()` does (in order):
1. Parse `.sumocfg` (`GUILoadThread.cpp:94–149`).
2. Allocate the vehicle controller: `new GUIVehicleControl()` (line 155) - note this is the GUI variant.
3. Allocate the network: `new GUINet(...)` (line 165).
4. Set up GUI builders: `new GUIEdgeControlBuilder` etc. (171–175).
5. Run the parser: `NLBuilder::build()` (213). The parser callbacks construct `GUIEdge`, `GUILane`, `GUIJunctionWrapper`, `GUIBusStop`, etc., piece by piece, and link them into the `GUINet`.
6. `net->initGUIStructures()` (`src/guisim/GUINet.cpp:276`) - populates `myEdgeWrapper`, `myJunctionWrapper`, and the spatial R‑tree `myGrid` used for fast picking and culling.
7. The crucial **handoff**:
    ```cpp
    GUIEvent* e = new GUIEvent_SimulationLoaded(net, simStartTime, simEndTime, ...);
    myEventQue.push_back(e);   //   ← drop note in the GUI's mailbox
    myEventThrow.signal();      //   ← ring the doorbell
    ```
    `myEventQue` is a reference to `GUIApplicationWindow::myEvents`. The note carries the `GUINet*` pointer so the GUI thread will know where the network lives.
8. The load thread function returns. The OS reclaims the load thread.

Meanwhile the GUI thread receives the doorbell: FOX wakes it, sees the `MFXThreadEvent` fired, dispatches it to `onLoadThreadEvent` (line 1784) which calls `eventOccurred()` (line 1791). That function runs:

```cpp
while (!myEvents.empty()) {
    GUIEvent* e = myEvents.top();
    myEvents.pop();
    switch (e->getOwnType()) {
        case GUIEventType::SIMULATION_LOADED:
            handleEvent_SimulationLoaded(e);
            break;
        // ... other cases ...
    }
    delete e;
}
```

`handleEvent_SimulationLoaded` (line 1847) extracts the `GUINet*`, hands it to the run thread (`myRunThread->init(net, begin, end)`), and calls `openNewView()` to create a `GUISUMOViewParent` MDI child holding a `GUIViewTraffic` (the OpenGL canvas).

At this point the run thread has a non‑null `myNet` pointer but is still spinning in `tryStep` with `myHalting == true`. The user sees the network on screen, frozen at time 0.

### 4.3 Clicking Play

You hit **Space** (or the Play button). FOX dispatches the action to `GUIApplicationWindow::onCmdStart` (`.cpp:1277`). Running on the GUI thread, this function does almost nothing:

```cpp
if (!myWasStarted) { myRunThread->begin(); myWasStarted = true; }
myRunThread->resume();
```

`resume()` (`GUIRunThread.cpp:257`) is just two assignments:

```cpp
mySingle  = false;
myHalting = false;
```

That's it. The GUI thread returns to the event loop. **It hasn't told the run thread to do anything; it has only flipped a flag.**

But the run thread has been spinning in `tryStep()` this whole time, sleeping 50 ms between checks. On its very next iteration it sees `myHalting == false` and enters the step path:

```cpp
void GUIRunThread::makeStep() {                         // GUIRunThread.cpp:186
    mySimulationLock.lock();
    myNet->simulationStep();           // run one timestep of microsim
    myNet->guiSimulationStep();        // update GUI counters/value-pass
    mySimulationLock.unlock();

    myEventQue.push_back(new GUIEvent_SimulationStep());
    myEventThrow.signal();             // ring the doorbell

    // ... possibly post GUIEvent_SimulationEnded ...
    sleep(mySimDelay - stepDuration);  // honour the visual delay slider
}
```

This loop continues until the user clicks Stop (which sets `myHalting = true` again), or the simulation ends, or the program quits.

Each `simulationStep` mutates the network in place: vehicles move, get inserted, get removed, traffic lights change phase. The GUI thread, woken by the doorbell, drains the event queue. For a `SIMULATION_STEP` event, it calls `handleEvent_SimulationStep` (`.cpp:2021`), which updates the time LCD and counters, then calls `update()` on each open view. `update()` schedules a `SEL_PAINT` for the next FOX iteration - at which point `paintGL` will run, walk the spatial index, and call `drawGL` on each visible object. That's where the cars on screen actually move.

---

## 5. The deep dive: how do the threads share network data without corrupting it?

This is the heart of the question. Two threads are touching the same `GUILane` objects:

* The **run thread** writes to `myVehicles` during `executeMovements` (vehicles move between lanes).
* The **GUI thread** reads `myVehicles` during `drawGL` (it iterates and renders each vehicle).

If they happen at the exact same instant - say, the run thread is appending a vehicle just as the GUI is iterating the vector - the GUI's iterator could become invalid and the program would crash.

### 5.1 No copying, no snapshots

Many programs solve this by *copying* simulation data into a "renderable snapshot" once per step. SUMO does **not**. It uses inheritance instead: every microsim class has a GUI subclass that adds drawing capability:

| microsim base | GUI subclass | header |
|---|---|---|
| `MSEdge` | `GUIEdge`  | `src/guisim/GUIEdge.h:51` |
| `MSLane` | `GUILane`  | `src/guisim/GUILane.h:60` |
| `MSVehicle` | `GUIVehicle` | `src/guisim/GUIVehicle.h:52` |
| `MSVehicleControl` | `GUIVehicleControl` | `src/guisim/GUIVehicleControl.h:44` |

The loader is rigged so that `new MSEdge(...)` is replaced by `new GUIEdge(...)` etc. (see `GUIEdgeControlBuilder::buildEdge` at `src/guinetload/GUIEdgeControlBuilder.cpp:65`). So **every "MSEdge*" pointer the run thread holds is actually a GUIEdge.** Same heap object, two views: the run thread treats it as `MSEdge` and updates traffic state; the GUI thread treats it as `GUIEdge` and draws it.

When the renderer needs a vehicle's position, `GUIVehicle::getVisualPosition()` reads `MSVehicle::myState.myPos` *directly*. Zero copy.

### 5.2 The mutex tour - five real locks

Because there's no copy, both threads must coordinate via mutexes. SUMO uses **fine‑grained** locking - many small locks, each protecting a small piece - rather than one giant lock around everything.

#### Lock 1: `GUIRunThread::mySimulationLock` (the "step in progress" sign)

```cpp
FXMutex mySimulationLock;                  // GUIRunThread.h:184
```

The run thread holds this lock for the **whole duration** of a step. The GUI thread does *not* take this lock during drawing - that's deliberate (we'll see why).

#### Lock 2: `GUINet::myLock` (mostly redundant net‑level guard)

```cpp
mutable FXMutex myLock;                    // GUINet.h:462

void GUINet::simulationStep() {
    FXMutexLock locker(myLock);            // GUINet.cpp:243 - RAII lock
    MSNet::simulationStep();
}
```

Acquired by `GUINet::simulationStep` itself. Belt‑and‑braces: it covers code paths that reach `simulationStep` not via the run thread (e.g. libsumo).

#### Lock 3: `GUILane::myLock` (THE important one)

```cpp
mutable FXMutex myLock;                    // GUILane.h:400
```

This is the synchronisation point that actually matters during drawing. `GUILane` provides a public pair of methods to *borrow* the vehicle list safely:

```cpp
const MSLane::VehCont& GUILane::getVehiclesSecure() const {  // GUILane.cpp:168
    myLock.lock();
    return myVehicles;
}
void GUILane::releaseVehicles() const {                       // GUILane.cpp:175
    myLock.unlock();
}
```

The simulation's modifications to the lane all start with the same lock. Look at `GUILane.cpp` lines 161, 182, 188, 195, 202, 209, 216, 223, 230, 237, 244 - each is a `FXMutexLock locker(myLock);` at the top of a method like `planMovements`, `executeMovements`, `integrateNewVehicles`, `swapAfterLaneChange`. Each method does its work then releases automatically when `locker` falls out of scope.

The drawing path uses the borrow pattern:

```cpp
const MSLane::VehCont& vehicles = getVehiclesSecure();   // GUILane.cpp:804 - locks
for (MSVehicle* veh : vehicles) {
    static_cast<GUIVehicle*>(veh)->drawGL(s);            // safe to read, list is frozen
}
releaseVehicles();                                       // GUILane.cpp:822 - unlocks
```

If the run thread is in the middle of `executeMovements` on the same lane, the GUI thread blocks at `getVehiclesSecure()` until the run thread is done - typically microseconds. And vice‑versa.

This is what makes the system safe: **whichever thread reaches a lane first holds it exclusively.**

#### Lock 4: `GUIEdge::myLock` (per‑edge lock for transportables and meso)

```cpp
mutable FXMutex myLock;                    // GUIEdge.h:259
```

Same idea but at the edge level. Used to protect `myPersons`, `myContainers`, and (in mesoscopic mode) the per‑segment vehicle lists. Used during drawing in `GUIEdge.cpp:376` (persons), `:384` (containers), `:401` (meso vehicles).

#### Lock 5: `GUIVehicleControl::myLock` (vehicle dictionary)

```cpp
mutable FXMutex myLock;                    // GUIVehicleControl.h:123
```

Protects the global `myVehicleDict` (the std::map of all vehicles by id). The run thread takes it when adding/removing vehicles; the GUI takes it when iterating the fleet (e.g. for the locator dialog or running statistics).

#### Plus: the queue lock

`MFXSynchQue::myMutex` (`MFXSynchQue.h:196`) protects the event mailbox itself. Held only for nanoseconds during each push/pop. This is **not** about network data - events carry only metadata.

### 5.3 Why doesn't the GUI take `mySimulationLock` during drawing?

You might think: "shouldn't the GUI hold the big simulation lock while drawing, so it sees the world frozen at one point in time?"

The designers explicitly chose **not** to do that. If the GUI held `mySimulationLock` while painting, the run thread couldn't start the next step until the frame was finished - meaning the simulation would be capped at the GUI frame rate. With slow frames or many open views, the simulation would crawl.

Instead they accept a small inconsistency: while you're rendering frame N, the simulation can already start step N+2 on lanes you've already drawn. Within any single lane, what you see is consistent (the per‑lane lock guarantees that). Across two lanes, the snapshot might be from slightly different points in simulated time. At normal step rates and frame rates this is invisible - a vehicle that just hopped across a lane boundary might be drawn on neither or both lanes for one frame.

### 5.4 A worked example - drawing one vehicle

Let's trace one vehicle from "the simulation moves it" to "it appears on screen at its new position".

```
RUN THREAD                                      GUI THREAD
────────────                                    ────────────

mySimulationLock.lock()
myNet->simulationStep()
  for each active lane:
    GUILane::executeMovements()
      FXMutexLock(GUILane::myLock)              (event loop, idle)
      veh->myState.myPos = newPos
      // unlocks here
    next lane...
mySimulationLock.unlock()

push GUIEvent_SimulationStep
signal()  ─────────────────────────────────►   wake from event loop
                                                onRunThreadEvent
                                                  eventOccurred()
                                                    drain queue
                                                      handleEvent_SimulationStep
                                                        update()  ← schedule paint
sleep(simDelay - stepDuration)
loop: makeStep again                            (FOX schedules SEL_PAINT)
                                                onPaint
                                                  paintGL
                                                    GUIViewTraffic::doPaintGL
                                                      myGrid.Search(viewport)
                                                        for each visible GUILane:
                                                          drawGL
                                                            getVehiclesSecure()  ← lock
                                                            for each veh:
                                                              GUIVehicle::drawGL
                                                                getVisualPosition()
                                                                  reads myState.myPos
                                                                  ↑ same memory the
                                                                    run thread wrote
                                                            releaseVehicles()  ← unlock
                                                      swapBuffers
                                                (back to event loop)
```

Two key points:
1. The run thread's write to `myState.myPos` and the GUI thread's read of it are **separated by `GUILane::myLock`**: either the write fully completes before the read starts, or the write hasn't happened yet and the read sees the old value. There is no torn or in‑between state, because (a) the write to a `double` is atomic on every modern CPU, and (b) we lock around the whole list mutation so we never iterate a half‑modified vector.
2. The run thread did **not** wait for the GUI to draw frame N before starting step N+1. The doorbell was a notification, not a synchronisation point.

### 5.5 What the GUI thread is *not* allowed to do

A few rules the architecture enforces:

* The GUI thread must **never** hold `GUIRunThread::mySimulationLock`. That's the run thread's lock for serialising the step itself.
* The GUI thread must **never** call OpenGL except during a paint event handler (between `makeCurrent()` and `makeNonCurrent()`).
* If you write a custom GUI overlay that needs simulation data, follow the borrow pattern: `lane->getVehiclesSecure()` ... `lane->releaseVehicles()`.

The run thread, similarly, must never call FOX widget methods or trigger redraws directly. It can only **post events**.

---

## 6. Summary in one diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              ONE PROCESS                                │
│                                                                         │
│   GUI THREAD                LOAD THREAD              RUN THREAD         │
│   (FOX event loop)          (one-shot per load)      (perpetual)        │
│   ─────────────             ─────────────────        ─────────────      │
│   onCmdOpenConfig                                                       │
│        │                                                                │
│        │ start() ────────►  GUILoadThread::run()                        │
│        │                    parse XML, build                            │
│        │                    GUINet, GUIEdge,                            │
│        │                    GUILane, GUIVehicle                         │
│        │                    push SIMULATION_LOADED                      │
│        │   ◄─── signal ──── (note dropped)                              │
│        │                    [thread exits]                              │
│   handleEvent_                                                          │
│   SimulationLoaded                                                      │
│        │                                                                │
│        │ myRunThread->init(net, ...)  ───────►  myNet = net             │
│        │                                                                │
│   onCmdStart                                                            │
│        │ myRunThread->resume()  ───────────►  myHalting = false         │
│        │                                                                │
│   (back to event loop)                          while (!myQuit)         │
│                                                   tryStep()             │
│                                                     if (!myHalting)     │
│                                                       makeStep()        │
│                                                         lock(simLock)   │
│                                                         simulationStep  │
│                                                           lock(laneLock)│
│                                                           move vehicles │
│                                                           unlock(lane)  │
│                                                         unlock(simLock) │
│                                                         push STEP_EVENT │
│                                  ◄── signal ── ─────── ring doorbell    │
│   onRunThreadEvent                                                      │
│   eventOccurred                                                         │
│   handleEvent_SimulationStep                                            │
│   update() ──► paint scheduled                                          │
│   onPaint                                                               │
│     paintGL                                                             │
│       for each visible lane:                                            │
│         lock(GUILane::myLock)  ◄── may briefly block here if run        │
│           draw vehicles            thread is also touching this lane    │
│         unlock                                                          │
│                                                                         │
│   ─── Shared state on the heap (one copy) ──────────────────────────    │
│   GUINet, MSEdgeControl, GUIEdges, GUILanes, GUIVehicles, ...           │
└─────────────────────────────────────────────────────────────────────────┘
```

### The takeaways in plain English

1. **A thread is just an independently running stream of code.** SUMO‑GUI uses three so the window stays responsive while heavy work happens in the background.
2. The **GUI thread** runs the window. The **load thread** parses XML and builds the network (one‑shot). The **run thread** runs the simulation forever, advancing one step at a time when not halted.
3. Threads share the network through plain pointers - there's only **one copy** in memory. The GUI subclasses (`GUIEdge`, `GUILane`, `GUIVehicle`) are the same objects the simulation operates on, just viewed through a richer interface that adds `drawGL`.
4. To prevent corruption when both threads touch the same data, **mutexes** are used. The most important is `GUILane::myLock` - a per‑lane lock taken by both the simulation (when it moves vehicles on that lane) and the GUI (when it draws that lane). Critical sections are tiny.
5. To **notify** the GUI of progress, the run thread drops `GUIEvent` notes in a thread‑safe **mailbox** (`MFXSynchQue`) and rings a **doorbell** (`MFXThreadEvent`). The GUI's event loop wakes, drains the mailbox, and triggers a redraw.
6. The simulation is **not synchronised** to the frame rate - the run thread keeps stepping regardless of whether the GUI has caught up. This keeps simulation throughput high. The only price is occasional cross‑lane visual inconsistency, which is invisible at normal speeds.
7. There is no fourth thread for TraCI - TraCI commands are processed by the run thread inside `MSNet::simulationStep`.

If you remember just one sentence from this document, make it:

> *Both threads work on the same heap objects; per‑lane mutexes serialise their access; events and a doorbell wake the GUI when there is something new to draw.*
