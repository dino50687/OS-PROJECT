# IPC Debugger Tool

## A Comprehensive Debugging Tool for Inter-Process Communication

> **Detect Deadlocks • Race Conditions • Bottlenecks — with Real-Time Visualization**

---

## Table of Contents

1. [Project Overview](#section-1--project-overview)
2. [Operating System Concepts](#section-2--operating-system-concepts)
3. [IPC Mechanisms](#section-3--ipc-mechanisms)
4. [System Architecture](#section-4--system-architecture)
5. [Software Module Design](#section-5--software-module-design)
6. [Project Folder Structure](#section-6--project-folder-structure)
7. [Technology Stack](#section-7--technology-stack)
8. [Development Roadmap](#section-8--development-roadmap)
9. [Deadlock Detection Algorithm](#section-9--deadlock-detection-algorithm)
10. [Bottleneck Detection](#section-10--bottleneck-detection)
11. [Full Python Implementation](#section-11--full-python-implementation)
12. [GUI Implementation](#section-12--gui-implementation)
13. [Visualization Engine](#section-13--visualization-engine)
14. [Test Strategy](#section-14--test-strategy)
15. [Limitations](#section-15--limitations)
16. [Future Enhancements](#section-16--future-enhancements)

---

## SECTION 1 — PROJECT OVERVIEW

### Objective

The **IPC Debugger Tool** is a desktop application that simulates, monitors, and debugs Inter-Process Communication (IPC) mechanisms — **Pipes**, **Message Queues**, and **Shared Memory**. It provides a graphical user interface (GUI) that lets developers:

- **Create virtual processes** and connect them via IPC channels.
- **Simulate data transfers** in real time with animated visualizations.
- **Detect deadlocks** using a Wait-For Graph with cycle detection.
- **Identify race conditions** through concurrent-access pattern analysis.
- **Pinpoint bottlenecks** by measuring latency, throughput, and lock contention.

### Challenges in IPC Debugging

| Challenge | Description |
|---|---|
| **Non-determinism** | Concurrent processes execute in unpredictable order, making bugs hard to reproduce. |
| **Invisible state** | Shared memory, pipe buffers, and queue depths are not directly observable in a normal debugger. |
| **Deadlocks** | Circular waits across multiple resources cause the entire pipeline to freeze with no error message. |
| **Race conditions** | Two processes accessing the same resource without synchronization can corrupt data silently. |
| **Heisenbugs** | Adding print statements or breakpoints can change timing enough to mask the bug. |

### How the Tool Helps

1. **Visual process graph** — Processes appear as nodes; IPC channels as animated edges. You can *see* data flowing.
2. **Wait-For Graph** — The deadlock detector maintains a live directed graph. Cycles are highlighted in red instantly.
3. **Race-condition detector** — Logs every read/write to shared resources; flags unsynchronized concurrent accesses within a configurable time window.
4. **Bottleneck detector** — Monitors queue depth, pipe blocking events, and lock contention; reports when thresholds are exceeded.
5. **Structured event log** — Every IPC operation is recorded with timestamps, enabling post-mortem analysis.

### Why This Project Is Valuable for OS Study

- Demonstrates **all three major IPC mechanisms** side by side.
- Implements classic OS algorithms: **Wait-For Graph**, **cycle detection (DFS)**, **mutual exclusion**.
- Connects textbook theory (process states, synchronization primitives) to **runnable code**.
- The modular architecture itself teaches software-engineering best practices (separation of concerns, observer pattern, MVC).

---

## SECTION 2 — OPERATING SYSTEM CONCEPTS

### Inter-Process Communication (IPC)

IPC is any mechanism that allows separate processes (each with their own virtual address space) to exchange data or synchronize execution. Without IPC, processes are completely isolated by the OS memory protection model.

### Concurrent Execution

Modern OSes run multiple processes (and threads) **concurrently** — interleaving their execution on one or more CPU cores. This creates the potential for **interference** when processes share resources.

### Process Synchronization

Synchronization primitives — **mutexes**, **semaphores**, **condition variables** — ensure that concurrent accesses to shared data happen in a controlled order:

```
Process A                  Process B
──────────                 ──────────
acquire(mutex)
  write(shared_var)
release(mutex)
                           acquire(mutex)
                             read(shared_var)   ← guaranteed to see A's write
                           release(mutex)
```

### Deadlocks

A deadlock occurs when a set of processes are each waiting for a resource held by another process in the set, forming a **circular wait**:

```
P1 holds R1, waits for R2
P2 holds R2, waits for R1
→ Neither can proceed → DEADLOCK
```

Four necessary conditions (Coffman conditions):
1. **Mutual exclusion** — resources are non-sharable.
2. **Hold and wait** — a process holds at least one resource while waiting for another.
3. **No preemption** — resources cannot be forcibly taken.
4. **Circular wait** — a cycle exists in the wait-for graph.

### Race Conditions

A race condition occurs when the program's correctness depends on the **relative timing** of concurrent operations:

```
counter = 0  (shared)

Process A: temp = counter       # reads 0
Process B: temp = counter       # reads 0
Process A: counter = temp + 1   # writes 1
Process B: counter = temp + 1   # writes 1  ← should be 2!
```

### Resource Sharing

Resources shared across processes include:
- **Memory regions** (shared memory segments)
- **Files** (file descriptors, memory-mapped files)
- **Communication channels** (pipes, sockets, queues)
- **Hardware** (printers, GPUs)

Proper synchronization is required whenever a resource can be modified by more than one process.

---

## SECTION 3 — IPC MECHANISMS

### 3.1 Pipes

A **pipe** is a unidirectional (or duplex) byte-stream channel between exactly two endpoints, buffered in the kernel.

| Aspect | Detail |
|---|---|
| **Practical applications** | Parent↔child command/result channels; simple producer→consumer streaming |
| **Benefits** | Very fast (kernel buffer), simple API (`send`/`recv`), automatic flow control |
| **Constraints** | Limited to **two** endpoints; finite buffer → blocking when full; no fan-out |
| **Common errors** | Forgetting to close unused ends → hang; sending unpicklable objects; both ends calling `recv` on a half-duplex pipe |

### 3.2 Message Queues

A **message queue** is a kernel-managed FIFO that supports **multiple producers and consumers**. Messages are discrete, ordered, and buffered.

| Aspect | Detail |
|---|---|
| **Practical applications** | Task distribution (worker pools), logging pipelines, event buses |
| **Benefits** | Fan-out / fan-in; decouples producer and consumer speeds; built-in blocking semantics |
| **Constraints** | Slower than pipes (internal locking); max queue size can block producers; large payloads are expensive (pickling) |
| **Common errors** | Queue full → silent producer block; forgetting `close()`/`join_thread()` → zombie feeder thread; deadlock when joining a child before draining its queue |

### 3.3 Shared Memory

**Shared memory** maps the same physical pages into the virtual address spaces of multiple processes — the fastest possible IPC (zero kernel copy).

| Aspect | Detail |
|---|---|
| **Practical applications** | High-throughput numerical pipelines (image processing, ML), shared caches |
| **Benefits** | Zero-copy; arbitrary data layout; very low latency |
| **Constraints** | Requires **explicit locking** (mutex/semaphore); fixed size; platform-specific naming |
| **Common errors** | Forgetting the lock → data corruption; double-release of semaphore; not unlinking → resource leak; reader starvation |

---

## SECTION 4 — SYSTEM ARCHITECTURE

```
┌──────────────────────────────────────────────────────────────────┐
│                        GUI  (Tkinter)                            │
│  ┌────────────┐ ┌───────────────────────┐ ┌──────────────────┐  │
│  │  Control   │ │   Visualization       │ │   Real-Time      │  │
│  │  Panel     │ │   Canvas              │ │   Log Panel      │  │
│  │            │ │   (VisualizationPanel  │ │   (ScrolledText) │  │
│  │  - Add     │ │    + AnimationEngine)  │ │                  │  │
│  │    Process │ │                       │ │                  │  │
│  │  - IPC     │ │   ● P1 ──→ ● P2      │ │  [12:01] SEND    │  │
│  │    Type    │ │        ↘       ↓      │ │  [12:01] RECV    │  │
│  │  - Start   │ │         ● P3          │ │  [12:02] ALERT   │  │
│  │  - Stop    │ │                       │ │                  │  │
│  │  - Analyze │ │                       │ │                  │  │
│  └────────────┘ └───────────────────────┘ └──────────────────┘  │
├──────────────────────────────────────────────────────────────────┤
│                     Event Monitor (Thread)                       │
│         polls EventLogger → dispatches to Analyzers              │
├──────────┬──────────────┬──────────────┬─────────────────────────┤
│ Deadlock │ Race         │ Bottleneck   │     Event Logger        │
│ Detector │ Condition    │ Detector     │  (ring buffer + file)   │
│ (WFG+DFS)│ Detector     │ (thresholds) │                         │
├──────────┴──────────────┴──────────────┴─────────────────────────┤
│              Simulation Engine                                    │
│  ┌─────────────────────┐  ┌─────────────────────────────────┐   │
│  │ Process Simulator   │  │ Communication Simulator         │   │
│  │ (create/start/stop) │  │ (pipes, queues, shared memory)  │   │
│  └─────────────────────┘  └─────────────────────────────────┘   │
├──────────────────────────────────────────────────────────────────┤
│              IPC Layer (instrumented wrappers)                    │
│  ┌──────────────┐ ┌──────────────────┐ ┌─────────────────────┐  │
│  │ PipeCommunic.│ │ MessageQueueComm.│ │ SharedMemoryComm.   │  │
│  └──────────────┘ └──────────────────┘ └─────────────────────┘  │
├──────────────────────────────────────────────────────────────────┤
│     Python multiprocessing / shared_memory / OS Kernel           │
└──────────────────────────────────────────────────────────────────┘
```

### Module Interactions (Data Flow)

```
User clicks "Start Simulation"
    │
    ▼
GUI  ───→  ProcessSimulator.start_all()
    │          │
    │          ▼
    │      CommunicationSimulator  ──→  PipeCommunicator.send()
    │          │                         MessageQueueCommunicator.enqueue()
    │          │                         SharedMemoryCommunicator.write_bytes()
    │          │
    │          ▼  (each operation calls on_event callback)
    │      EventLogger.log(event)
    │          │
    │          ▼
    │      EventMonitor._poll()  ──→  DeadlockDetector.add_edge()
    │          │                      RaceConditionDetector.record_access()
    │          │                      BottleneckDetector.record_throughput()
    │          │
    │          ▼  (if issue found)
    │      on_alert callback  ──→  GUI.log_panel.append()
    │                               VisualizationPanel.show_deadlock_alert()
    │
    ▼
AnimationEngine  ──→  VisualizationPanel.animate_packet()
```

---

## SECTION 5 — SOFTWARE MODULE DESIGN

### 5.1 Process Simulation Engine (`engine/process_simulator.py`)

**Responsibilities:**
- Create simulated processes with unique PIDs and human-readable names.
- Start/stop processes using `multiprocessing.Process`.
- Track process state: READY → RUNNING → WAITING → TERMINATED.
- Emit events (`process_created`, `process_started`, `process_stopped`).

### 5.2 IPC Communication Manager (`engine/communication_simulator.py`)

**Responsibilities:**
- Create IPC channels (pipe, queue, shared memory) between pairs of processes.
- Provide high-level `send_via_pipe`, `enqueue`, `shm_write` helpers.
- Collect per-channel statistics for the bottleneck detector.
- Emit events for every data transfer.

### 5.3 Synchronization Manager (integrated into SharedMemoryCommunicator)

**Responsibilities:**
- Manage the `multiprocessing.Lock` guarding shared memory regions.
- Track lock acquisitions, wait times, and contentions.
- Emit `shm_lock_acquire` / `shm_lock_release` events.

### 5.4 Deadlock Detection Engine (`analyzers/deadlock_detector.py`)

**Responsibilities:**
- Maintain a directed **Wait-For Graph** (adjacency list).
- Run DFS-based **cycle detection** on demand or automatically.
- Report cycles as `DeadlockReport` objects.

### 5.5 Race Condition Detector (`analyzers/race_condition_detector.py`)

**Responsibilities:**
- Log read/write accesses with timestamps and lock status.
- Flag unsynchronized concurrent accesses within a configurable time window.
- Report races as `RaceReport` objects.

### 5.6 Bottleneck Detection Engine (`analyzers/bottleneck_detector.py`)

**Responsibilities:**
- Analyze queue depth, pipe blocking counts, and lock contention against configurable thresholds.
- Compute throughput over a sliding window.
- Report bottlenecks as `BottleneckReport` objects.

### 5.7 Event Logger (`utils/event_logger.py`)

**Responsibilities:**
- Thread-safe structured event logging (ring buffer + JSONL file).
- Provide `as_callback()` so all modules can emit events uniformly.
- Support filtering by event type and time range.

### 5.8 Event Monitor (`utils/event_monitor.py`)

**Responsibilities:**
- Run in a background thread, polling the EventLogger.
- Dispatch events to the appropriate analyzer.
- Trigger alert callbacks when issues are found.

### 5.9 Visualization Panel (`visualization/visualization_panel.py`)

**Responsibilities:**
- Render processes as color-coded circles on a Tkinter Canvas.
- Render IPC channels as dashed arrows (color = type).
- Animate data packets travelling along arrows.
- Highlight deadlocked nodes in red with alert text.

### 5.10 Animation Engine (`visualization/animation_engine.py`)

**Responsibilities:**
- Consume IPC events and translate them into canvas animations.
- Run on the Tkinter `after()` loop (no threads needed for GUI updates).
- Support configurable animation speed.

### 5.11 GUI Controller (`gui/main_window.py`)

**Responsibilities:**
- Build and manage the Tkinter main window layout.
- Handle user interactions: add process, connect channel, start/stop, analyze.
- Wire together all modules (simulator, analyzers, logger, visualization).
- Update the real-time log panel from EventLogger callbacks.

---

## SECTION 6 — PROJECT FOLDER STRUCTURE

```
ipc_debugger_tool/
│
├── main.py                              # Entry point — launches GUI
├── requirements.txt                     # Python dependencies
├── __init__.py                          # Package marker (version)
├── README.md                            # This documentation
│
├── ipc/                                 # IPC mechanism wrappers
│   ├── __init__.py
│   ├── ipc_pipes.py                     # Pipe communicator + stats
│   ├── ipc_message_queue.py             # Message queue communicator + stats
│   └── ipc_shared_memory.py             # Shared memory + lock + stats
│
├── analyzers/                           # Debugging analyzers
│   ├── __init__.py
│   ├── deadlock_detector.py             # Wait-For Graph + DFS cycle detection
│   ├── race_condition_detector.py       # Concurrent access pattern analysis
│   └── bottleneck_detector.py           # Latency / throughput / contention
│
├── engine/                              # Simulation engines
│   ├── __init__.py
│   ├── process_simulator.py             # Process creation & lifecycle
│   └── communication_simulator.py       # IPC channel orchestrator
│
├── visualization/                       # Canvas-based visualization
│   ├── __init__.py
│   ├── visualization_panel.py           # Process graph rendering
│   └── animation_engine.py              # Timed animation driver
│
├── gui/                                 # Tkinter GUI
│   ├── __init__.py
│   └── main_window.py                   # Main application window
│
├── utils/                               # Cross-cutting utilities
│   ├── __init__.py
│   ├── event_logger.py                  # Thread-safe event logger
│   └── event_monitor.py                 # Real-time event dispatcher
│
├── tests/                               # Test suite
│   ├── __init__.py
│   ├── test_ipc_pipes.py
│   ├── test_ipc_message_queue.py
│   ├── test_ipc_shared_memory.py
│   ├── test_deadlock_detector.py
│   ├── test_race_condition_detector.py
│   ├── test_bottleneck_detector.py
│   ├── test_process_simulator.py
│   ├── test_communication_simulator.py
│   ├── test_event_logger.py
│   └── test_integration.py
│
└── logs/                                # Runtime log files (auto-created)
    └── ipc_debug_YYYYMMDD_HHMMSS.jsonl
```

---

## SECTION 7 — TECHNOLOGY STACK

| Component | Technology | Justification |
|---|---|---|
| **Language** | Python 3.8+ | Rich `multiprocessing` module; rapid prototyping; widely taught in OS courses |
| **GUI Framework** | Tkinter (built-in) | Zero external dependencies; Canvas widget supports custom drawing and animation |
| **Concurrency** | `multiprocessing` | True OS-level processes (not threads); uses `Pipe`, `Queue`, `shared_memory`, `Lock`, `Value` |
| **Shared Memory** | `multiprocessing.shared_memory` (Python 3.8+) | Wraps POSIX `shm_open` / Windows `CreateFileMapping` |
| **Logging** | Custom JSONL logger | Structured events; easy to parse; minimal dependencies |
| **Testing** | `pytest` | Fixtures, parametrize, tmp_path; industry standard |
| **Platform** | macOS / Linux / Windows | `multiprocessing` is cross-platform; `shared_memory` works on all three |

### Why These Technologies?

- **Python** is the language of choice for OS courses because it provides high-level wrappers around system calls (`os.fork`, `multiprocessing.Pipe`, `shared_memory.SharedMemory`) without requiring C boilerplate.
- **Tkinter** ships with every CPython installation, so students need zero setup beyond Python itself.
- **`multiprocessing`** creates real OS processes with separate memory spaces — unlike `threading`, which is limited by the GIL and doesn't demonstrate true IPC.

---

## SECTION 8 — DEVELOPMENT ROADMAP

### Phase 1 — Environment Setup
- Install Python 3.8+.
- Clone the project.
- `pip install -r requirements.txt` (only `pytest` needed).
- Verify: `python main.py` opens the GUI.

### Phase 2 — Process Simulation Engine
- Implement `ProcessSimulator`: create, start, stop, refresh states.
- Unit tests: `test_process_simulator.py`.

### Phase 3 — IPC Mechanism Implementation
- Implement `PipeCommunicator`, `MessageQueueCommunicator`, `SharedMemoryCommunicator`.
- Add event callbacks and statistics tracking.
- Unit tests: `test_ipc_pipes.py`, `test_ipc_message_queue.py`, `test_ipc_shared_memory.py`.

### Phase 4 — Communication Simulator
- Implement `CommunicationSimulator`: channel creation, high-level send/recv.
- Integration tests: `test_communication_simulator.py`.

### Phase 5 — Deadlock Detection
- Implement `DeadlockDetector` with Wait-For Graph and DFS.
- Unit tests: `test_deadlock_detector.py`.

### Phase 6 — Race Condition & Bottleneck Detection
- Implement `RaceConditionDetector` and `BottleneckDetector`.
- Unit tests: `test_race_condition_detector.py`, `test_bottleneck_detector.py`.

### Phase 7 — Event Logger & Monitor
- Implement `EventLogger` (ring buffer + file).
- Implement `EventMonitor` (background thread, event dispatch).
- Unit tests: `test_event_logger.py`.

### Phase 8 — GUI Development
- Build `MainWindow` with control panel, canvas, log panel.
- Wire to simulation engine and analyzers.

### Phase 9 — Visualization & Animation
- Implement `VisualizationPanel` (nodes, edges, packets).
- Implement `AnimationEngine` (Tkinter `after()` loop).
- Integrate with GUI.

### Phase 10 — Integration Testing & Polish
- Run `test_integration.py` (end-to-end scenarios).
- Stress tests with 100+ messages.
- UI polish: colours, layout, responsiveness.

---

## SECTION 9 — DEADLOCK DETECTION ALGORITHM

### Wait-For Graph (WFG)

A **Wait-For Graph** `G = (V, E)` is a directed graph where:
- `V` = set of process identifiers (PIDs).
- `E` = set of directed edges `(Pi → Pj)` meaning "Pi is waiting for a resource held by Pj".

### Cycle Detection via DFS

**Pseudocode:**

```
FUNCTION detect_deadlock(graph G):
    visited = {}
    rec_stack = {}    // nodes in the current DFS recursion stack
    parent = {}

    FOR each node v in G:
        IF v not in visited:
            IF dfs(v, G, visited, rec_stack, parent):
                RETURN reconstruct_cycle(parent, v)

    RETURN "No deadlock"

FUNCTION dfs(node, G, visited, rec_stack, parent):
    visited[node] = TRUE
    rec_stack[node] = TRUE

    FOR each neighbour n of node in G:
        IF n not in visited:
            parent[n] = node
            IF dfs(n, G, visited, rec_stack, parent):
                RETURN TRUE
        ELSE IF n in rec_stack:
            // Cycle found! n is the start of the cycle.
            RETURN TRUE

    rec_stack.remove(node)
    RETURN FALSE
```

**Time complexity:** O(V + E)

### How the Tool Illustrates Deadlocks

1. Each process is a **circle** on the canvas.
2. When a process requests a lock held by another, a **directed edge** appears.
3. The edge is **dashed blue** (normal) or **solid red** (cycle detected).
4. When a cycle is found:
   - All involved nodes turn **red**.
   - A "⚠ DEADLOCK DETECTED ⚠" banner appears.
   - The event log shows the cycle: `P1 → P2 → P3 → P1`.
5. The user can click "Clear Alerts" to reset, or "Inject Deadlock" to demo again.

---

## SECTION 10 — BOTTLENECK DETECTION

### What the System Detects

| Bottleneck | How Detected | Metric |
|---|---|---|
| **Queue congestion** | `current_size > queue_depth_threshold` | Queue depth (items) |
| **Blocked pipes** | `blocked_count > pipe_block_threshold` | Block event count |
| **Lock contention** | `max_lock_wait > lock_wait_threshold` | Lock wait time (seconds) |
| **Slow consumer** | `(enqueued - dequeued) / enqueued > consumer_lag_ratio` | Consumer lag ratio |
| **High latency** | `max_latency > 1.0s` on a pipe | Max latency (seconds) |

### Metrics

- **Latency:** Time between `send()` and corresponding `recv()` for a single message. Measured per-message, aggregated as average and max.
- **Throughput:** Messages per second over a sliding window of the last 100 samples.
- **Queue depth:** Number of items currently waiting in a message queue.
- **Lock wait time:** Time a process spent blocked on `acquire()`. High values indicate contention.
- **Block count:** Number of times a pipe send/recv or queue put/get blocked.

### Thresholds (Configurable)

```python
BottleneckDetector(
    queue_depth_threshold=10,     # flag if queue > 10 items
    pipe_block_threshold=5,       # flag if pipe blocked > 5 times
    lock_wait_threshold=0.1,      # flag if lock wait > 100 ms
    consumer_lag_ratio=0.5,       # flag if consumer lags > 50%
)
```

---

## SECTION 11 — FULL PYTHON IMPLEMENTATION

All source code is organized in the folder structure above. Key files:

| File | Lines | Purpose |
|---|---|---|
| `ipc/ipc_pipes.py` | ~170 | Pipe wrapper with stats + events |
| `ipc/ipc_message_queue.py` | ~150 | Queue wrapper with stats + events |
| `ipc/ipc_shared_memory.py` | ~190 | Shared memory + lock + counter |
| `analyzers/deadlock_detector.py` | ~200 | Wait-For Graph + DFS cycle detection |
| `analyzers/race_condition_detector.py` | ~130 | Concurrent access analysis |
| `analyzers/bottleneck_detector.py` | ~190 | Threshold-based bottleneck analysis |
| `engine/process_simulator.py` | ~160 | Process creation & lifecycle |
| `engine/communication_simulator.py` | ~200 | Channel orchestration |
| `utils/event_logger.py` | ~130 | Thread-safe logger (memory + file) |
| `utils/event_monitor.py` | ~150 | Background event dispatcher |
| `visualization/visualization_panel.py` | ~260 | Canvas rendering + animations |
| `visualization/animation_engine.py` | ~100 | Timed animation driver |
| `gui/main_window.py` | ~500 | Full Tkinter GUI |
| `main.py` | ~50 | Entry point |

Each module is documented with docstrings, type hints, and inline comments.

---

## SECTION 12 — GUI IMPLEMENTATION

The GUI (`gui/main_window.py`) uses **Tkinter** with the following layout:

### Controls (Left Panel)
- **Process Name** text entry + **Add Process** button
- **IPC Type** dropdown: Pipe / Message Queue / Shared Memory
- **From (PID)** and **To (PID)** dropdowns (auto-populated)
- **Connect** button — creates an IPC channel
- **Start Simulation** / **Stop Simulation** toggle
- **Run Analysis** — triggers all three detectors
- **Inject Deadlock** — creates a deliberate circular wait
- **Send Test Data** — sends one message through all channels
- **Clear All** — resets the entire application state

### Visualization Canvas (Top Right)
- Processes rendered as colored circles in a circular layout
- IPC channels rendered as dashed arrows (blue=pipe, green=queue, orange=shm)
- Animated dots travel along arrows during data transfer
- Deadlock alerts: red nodes + red edges + banner text
- Legend bar at bottom showing color codes

### Real-Time Log Panel (Bottom Right)
- ScrolledText widget with color-coded log entries:
  - Cyan — info
  - Green — IPC events
  - Orange — warnings / race conditions
  - Red — errors / deadlocks
  - Yellow — bottlenecks
- Auto-scrolls to latest entry
- Timestamps: `[HH:MM:SS.mmm]`

### Dark Theme
The entire UI uses a Dracula-inspired dark theme (`#1e1e2e` background, `#f8f8f2` text).

---

## SECTION 13 — VISUALIZATION ENGINE

### Architecture

```
AnimationEngine
    │
    │  schedule(event)        ← called from EventLogger callback
    │
    ▼
event queue (list)
    │
    │  _process_queue()       ← called every 33ms (30 fps) via Tk.after()
    │
    ▼
VisualizationPanel
    │
    ├── animate_packet(channel, duration)
    │       Creates a small circle that moves from src to dst over N frames.
    │       Steps = duration / 16ms ≈ 60 fps granularity.
    │
    ├── update_node_state(pid, state)
    │       Changes the fill color of a process circle:
    │       running=green, waiting=yellow, terminated=gray, deadlocked=red.
    │
    └── show_deadlock_alert(cycle_pids, channel_names)
            Turns cycle nodes red, highlights edges, shows banner.
```

### Packet Animation (Python)

The packet animation in `VisualizationPanel.animate_packet()`:

1. Creates a small oval at the source node's position.
2. Computes `dx, dy` per step (total `duration_ms / 16` steps).
3. Uses `canvas.after(16, step)` to advance the oval 1 step every 16ms.
4. Deletes the oval when it reaches the destination.

This produces a smooth 60fps animation using only Tkinter's built-in event loop.

---

## SECTION 14 — TEST STRATEGY

### Test Files

| File | Tests | Category |
|---|---|---|
| `test_ipc_pipes.py` | 5 | Unit — pipe send/recv, stats, timeout, events, large payload |
| `test_ipc_message_queue.py` | 5 | Unit — enqueue/dequeue, stats, timeout, maxsize, events |
| `test_ipc_shared_memory.py` | 5 | Unit — write/read, counter, stats, bounds, events |
| `test_deadlock_detector.py` | 8 | Unit — no deadlock, 2-way cycle, 3-way, remove edge, auto-detect, 10-way |
| `test_race_condition_detector.py` | 7 | Unit — read-read safe, write-write race, window expiry, locked safe |
| `test_bottleneck_detector.py` | 7 | Unit — queue congestion, slow consumer, pipe blocked, lock contention |
| `test_process_simulator.py` | 6 | Unit — create, start/stop, events, clear |
| `test_communication_simulator.py` | 6 | Integration — pipe/queue/shm channels, stats, events |
| `test_event_logger.py` | 6 | Unit — log, recent, by-type, file persistence, callback, clear |
| `test_integration.py` | 7 | Integration — full pipeline, deadlock scenario, stress test (100 msgs) |

### Running Tests

```bash
cd ipc_debugger_tool
python -m pytest tests/ -v
```

### Test Categories

1. **Unit tests** — Each module tested in isolation with mock callbacks.
2. **Simulation tests** — `CommunicationSimulator` creates real channels and transfers data.
3. **Deadlock scenarios** — Inject 2-way, 3-way, and 10-way circular waits; verify detection.
4. **Race condition scenarios** — Concurrent read-write patterns; verify flagging.
5. **Stress tests** — 100+ messages through a pipe; verify no data loss.

---

## SECTION 15 — LIMITATIONS

1. **Simulated (not real) processes for GUI demo:** The GUI uses virtual PID counters. Real `multiprocessing.Process` is used in tests and can be wired into the simulation loop, but the GUI's "Add Process" creates visualization-only nodes for simplicity.

2. **Heuristic race detection:** The race condition detector uses a time-window heuristic — it can miss races that fall outside the window or produce false positives for unrelated accesses.

3. **Single-machine only:** All processes must run on the same machine. Distributed IPC (sockets, RPC) is not supported.

4. **macOS `shared_memory` naming:** macOS limits shared memory names to 31 characters. The tool uses short auto-generated names (`ipc_dbg_N`).

5. **No real-time OS process monitoring:** The tool simulates processes; it does not attach to or monitor externally running OS processes.

6. **Tkinter performance:** With >50 animated packets simultaneously, the Tkinter Canvas may show slight lag on older hardware.

7. **No persistence across sessions:** Closing the GUI discards all state. Log files are preserved in `logs/`.

---

## SECTION 16 — FUTURE ENHANCEMENTS

### 1. Real-Time OS Process Monitoring
Attach to actual running processes via `/proc` (Linux) or `psutil` and monitor their IPC file descriptors, pipe buffers, and shared memory segments.

### 2. Distributed IPC Debugging
Extend the tool to monitor IPC over **sockets**, **ZeroMQ**, or **gRPC** across multiple machines, with a networked event aggregator.

### 3. Advanced Visualization Dashboards
- **Timeline view** — horizontal swimlanes per process, showing send/recv events over time.
- **Resource Allocation Graph (RAG)** — separate visualization for resource-vs-process modeling.
- **Throughput charts** — real-time line graphs of messages/second per channel.

### 4. AI-Driven Deadlock Prediction
Train a model on historical wait-for-graph patterns to **predict** deadlocks before they occur, and suggest lock-ordering strategies.

### 5. Record & Replay
Save the entire event stream and replay it at variable speed, enabling deterministic debugging of non-deterministic concurrency bugs.

### 6. Integration with GDB / LLDB
Connect to native debuggers to automatically set breakpoints on `pthread_mutex_lock` and map them into the Wait-For Graph.

### 7. Multi-Language Support
Add front-ends for C/C++ (`fork`/`pipe`/`shm_open`) and Java (`ProcessBuilder`, `MappedByteBuffer`) so the visualization tool is language-agnostic.

---

## Quick Start

```bash
# 1. Clone / navigate to the project
cd ipc_debugger_tool

# 2. (Optional) Create a virtual environment
python3 -m venv venv && source venv/bin/activate

# 3. Install test dependencies
pip install -r requirements.txt

# 4. Launch the GUI
python main.py

# 5. Launch with demo data pre-loaded
python main.py --demo

# 6. Run the test suite
python -m pytest tests/ -v
```

---

*Built as a comprehensive Operating Systems project demonstrating IPC mechanisms, synchronization, and debugging techniques.*
