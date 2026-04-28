# SUMO GUI Documentation

This repository contains documentation for understanding SUMO GUI behavior, flow, and data handling.

---

## Files

### 1. End-to-End Flow

* [SUMO_END_TO_END_FLOW.md](SUMO_END_TO_END_FLOW.md)


Covers:

* Application startup
* Thread model (GUI / Load / Run)
* Loading network / `.sumocfg`
* Simulation start / stop / step
* Event flow between threads

---

### 2. GUI Architecture

* [SUMO_GUI_ARCHITECTURE.md](SUMO_GUI_ARCHITECTURE.md)


Covers:

* GUI initialization
* Window and view structure
* Thread setup
* Event system
* Rendering pipeline
* User interaction handling

---

### 3. Network Data & Sharing

* [SUMO_NETWORK_DATA_SHARING.md](SUMO_NETWORK_DATA_SHARING.md)


Covers:

* Memory structure
* Ownership hierarchy
* Data sharing between simulation and GUI
* Synchronization (locks, threads)

---

## Suggested Order

1. `SUMO_END_TO_END_FLOW.md`
2. `SUMO_GUI_ARCHITECTURE.md`
3. `SUMO_NETWORK_DATA_SHARING.md`

---
