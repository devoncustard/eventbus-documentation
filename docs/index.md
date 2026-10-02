# Home

A simple, typed event bus with pattern matching, priorities, sync/async handler support, persistent storage, and a WebSocket monitor server.

## Features

- **Pattern matching**: Register handlers with glob patterns like `user.*`
- **Priorities**: Lower numbers = higher precedence
- **Async support**: Handlers can be sync or async
- **One-shot handlers**: `once()` auto-removes after first invocation
- **Fail-silent**: Failed handlers don't block others (by default)
- **Fail-fast**: Optional `fail_fast=True` to stop on first exception
- **Subscribers**: Global listeners for all events
- **Persistent storage**: Events are stored to SQLite via `EventStore`
- **Event labels**: `tuple[str, ...]` for filtering and pull queries
- **Monitor server**: Built-in WebSocket/dashboard for real-time event monitoring
- **HTTP API**: POST `/events` to emit, GET `/events?cursor=N` to pull

## Quick Start

```python
from eventbus import Event, EventBus

bus = EventBus()

def on_login(event: Event):
    print(f"User {event.payload['user']} logged in")

bus.on("user.login", on_login)
bus.emit(Event("user.login", labels=("user", "auth"), payload={"user": "devon"}))
# Output: User devon logged in
```

## Installation

```bash
pip install eventbus
```

## Usage

### Basic Events

```python
from eventbus import Event, EventBus

bus = EventBus()

def handler(event: Event):
    print(f"Got event: {event.name} with payload {event.payload}")

bus.on("user.*", handler)
bus.emit(Event("user.login", labels=("user", "auth"), payload={"user": "devon"}))
bus.emit(Event("user.logout", labels=("user",), payload={"user": "devon"}))
```

### Priority-based Execution

```python
bus.on("tick", low_priority, priority=10)
bus.on("tick", high_priority, priority=0)
# high_priority runs first
```

### One-Shot Handlers

```python
bus.once("initialization", setup_once)
# Automatically removed after first invocation
```

### Async Handlers

```python
async def async_handler(event: Event):
    await some_async_operation()

bus.on("async_event", async_handler)
await bus.emit_async(Event("async_event"))
```

### Persistent Store

```python
from eventbus import EventStore

store = EventStore("events.db", max_events=5000)
await store.start()
await store.start_writer()

event_id = store.append(event)
events, next_cursor, done = store.pull(cursor=0, labels=("user",), limit=50)

await store.close()
```

The store uses a dual-layer approach:
- **Hot layer**: In-memory ring buffer (`collections.deque`) for real-time access
- **Cold layer**: SQLite archive for persistence across restarts
- Background async writer batches events from the ring buffer to SQLite

### Monitor Server

```python
from eventbus import EventBus
from eventbus.monitor import MonitorMixin

class MyBus(MonitorMixin, EventBus):
    pass

bus = MyBus(db_path="events.db")
bus.start_monitor_sync(host="0.0.0.0", port=8765)
# Open http://192.168.1.5:8765/ in browser
```

### HTTP API

**Emit an event:**
```bash
curl -X POST http://192.168.1.5:8765/events \
  -H "Content-Type: application/json" \
  -d '{"name":"user.login","labels":["user","auth"],"payload":{"user":"devon"}}'
```
Response: `{"id": 1, "name": "user.login"}` (201 Created)

**Pull events:**
```bash
curl "http://192.168.1.5:8765/events?cursor=0&labels=user,auth&limit=50"
```
Response: `{"events": [...], "next_cursor": 42, "done": false}`

**Health check:**
```bash
curl http://192.168.1.5:8765/health
```
Response: `{}`
