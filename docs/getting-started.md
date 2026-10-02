# Getting Started

## Installation

Install eventbus via pip:

```bash
pip install eventbus
```

For development:

```bash
pip install eventbus[dev]
```

## Core Concepts

### Events

Events are immutable data carriers with a name, labels, and optional payload:

```python
from eventbus import Event

event = Event(
    name="user.login",
    labels=("user", "auth"),
    payload={"user": "devon", "ip": "127.0.0.1"}
)
```

- **`name`**: Event identifier (e.g., `"user.login"`)
- **`labels`**: Tuple of label strings for filtering (default: empty tuple)
- **`payload`**: Dictionary of arbitrary data (default: empty dict)
- **`timestamp`**: Wall-clock timestamp (seconds since epoch)
- **`mono_timestamp`**: Monotonic timestamp for ordering

### Event Bus

The `EventBus` is the central hub for all event routing:

```python
from eventbus import EventBus

bus = EventBus()
```

Key features:
- **Pattern matching**: Glob patterns with `*` wildcards (via `fnmatch`)
- **Priorities**: Lower numbers execute first
- **Async support**: Both sync and async handlers
- **Fail-silent**: Failed handlers don't block others (default)
- **Fail-fast**: Optional `fail_fast=True` to stop on first exception

### Persistent Store

The `EventStore` provides persistent event storage with dual-layer architecture:

```python
from eventbus import EventStore

store = EventStore(db_path="events.db", max_events=5000)
await store.start()
await store.start_writer()

# Append events (returns event ID)
event_id = store.append(event)

# Pull events by cursor with optional label filtering
events, next_cursor, done = store.pull(
    cursor=0,
    labels=("user", "auth"),
    limit=50
)

await store.close()
```

**Architecture:**
- **Hot layer**: In-memory ring buffer (`collections.deque`) for real-time access
- **Cold layer**: SQLite on-disk archive for persistence across restarts
- Background async writer batches events from ring buffer to SQLite
- Events auto-evict from the ring buffer when at capacity (`max_events`)

### Monitor Server

The `MonitorMixin` adds a real-time monitoring server:

```python
from eventbus import EventBus
from eventbus.monitor import MonitorMixin

class MyBus(MonitorMixin, EventBus):
    pass

bus = MyBus(db_path="events.db")
bus.start_monitor_sync(host="0.0.0.0", port=8765)
```

Endpoints:
- **Dashboard**: `http://localhost:8765/`
- **WebSocket**: `ws://localhost:8765/ws` (real-time event streaming)
- **Events API**: `GET /events?cursor=0&labels=user,auth&limit=50`
- **Post Event**: `POST /events` with JSON body
- **Health**: `GET /health`

## Basic Usage

### Registering Handlers

```python
from eventbus import EventBus

bus = EventBus()

def on_login(event):
    print(f"User {event.payload['user']} logged in")

bus.on("user.login", on_login)
```

### Emitting Events

```python
from eventbus import Event, EventBus

bus = EventBus()

bus.emit(Event("user.login", labels=("user", "auth"), payload={"user": "devon"}))
```

### Pattern Matching

```python
bus.on("user.*", user_handler)  # Matches user.login, user.logout, etc.
bus.on("system.*.restart", system_handler)  # Matches system.boot.restart, etc.
```

### Subscribers

```python
bus.subscribe(lambda event: print(f"All events: {event.name}"))
bus.unsubscribe(subscriber_callback)
```

## Next Steps

- [API Reference](api-reference.md) — Full documentation of all classes and methods
- [Monitor Server](monitor-server.md) — Real-time event monitoring and HTTP API
