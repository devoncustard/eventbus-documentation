# Monitor Server

The MonitorMixin adds a WebSocket-based monitoring server to your EventBus with persistent storage.

## Basic Usage

```python
from eventbus import EventBus
from eventbus.monitor import MonitorMixin

class MyBus(MonitorMixin, EventBus):
    pass

bus = MyBus(db_path="events.db", max_events=5000)
bus.start_monitor_sync(host="0.0.0.0", port=8765)
```

Then open [http://192.168.1.5:8765/](http://192.168.1.5:8765/) in your browser.

## Dashboard Features

- **Real-time event streaming**: Events appear instantly via WebSocket
- **Pattern filtering**: Filter events by glob pattern (e.g., `user.*`)
- **Search filtering**: Search within event names and payloads
- **Pause/Resume**: Pause the feed to examine specific events
- **Clear**: Clear the event list
- **Connection status**: Visual indicator of connection state

## HTTP API

### GET /events — Pull events by cursor

Query parameters:
- `cursor` (int, default `0`): Return events with ID greater than this value
- `labels` (str, optional): Comma-separated labels for intersection filter (all must match)
- `limit` (int, default `50`, max `500`): Maximum events to return

Response JSON:
```json
{
    "events": [
        {
            "id": 42,
            "name": "user.login",
            "labels": ["user", "auth"],
            "payload": {"user": "devon"},
            "ts": 1727520000.123
        }
    ],
    "next_cursor": 43,
    "done": false
}
```

**Examples:**

```bash
# Get first 50 events
curl "http://192.168.1.5:8765/events?cursor=0&limit=50"

# Get events after ID 42
curl "http://192.168.1.5:8765/events?cursor=42"

# Get events with both 'user' AND 'auth' labels
curl "http://192.168.1.5:8765/events?cursor=0&labels=user,auth&limit=50"
```

### POST /events — Emit an event

Request body (JSON):
```json
{
    "name": "user.login",
    "labels": ["user", "auth"],
    "payload": {
        "user": "devon",
        "ip": "192.168.1.5"
    }
}
```

- **`name`** (required): Event identifier
- **`labels`** (optional): Array of label strings (default: `[]`)
- **`payload`** (optional): Arbitrary JSON object (default: `{}`)

Response:
```json
{"id": 43, "name": "user.login"}
```

**Example:**

```bash
curl -X POST http://192.168.1.5:8765/events \
  -H "Content-Type: application/json" \
  -d '{"name":"system.health","payload":{"status":"ok"}}'
```

### GET /health — Health check

Response: `{}` (HTTP 200)

## WebSocket Protocol

### Connection

Connect to the WebSocket endpoint:

```
ws://192.168.1.5:8765/ws
```

### Per-Client Filtering

Clients can specify pattern filters via query parameters:

```
ws://192.168.1.5:8765/ws?pattern=user.*
```

Only matching events will be delivered to that client.

### Message Format

Messages are newline-delimited JSON:

```json
{
    "id": 43,
    "name": "user.login",
    "labels": ["user", "auth"],
    "payload": {
        "user": "devon",
        "ip": "127.0.0.1"
    },
    "ts": 1727520000.123
}
```

### Fields

| Field    | Type   | Description                          |
|----------|--------|--------------------------------------|
| `id`     | int    | Unique event ID (monotonically increasing) |
| `name`   | string | Event name                           |
| `labels` | array  | List of label strings                |
| `payload`| object | Event payload dictionary             |
| `ts`     | float  | Wall-clock timestamp (seconds since epoch) |

## Server Methods

### `start_monitor(host, port)`

Start the monitor server asynchronously.

**Parameters:**
- `host` (str): Host to bind to (default: `"0.0.0.0"`)
- `port` (int): Port to listen on (default: `8765`)

**Example:**

```python
import asyncio

async def main():
    bus = MyBus()
    await bus.start_monitor("127.0.0.1", 8765)

asyncio.run(main())
```

### `stop_monitor()`

Stop the monitor server and clean up resources (closes TCP server, broadcast loop, and event store).

**Example:**

```python
await bus.stop_monitor()
```

### `start_monitor_sync(host, port)`

Start the monitor server synchronously on a background thread. Returns immediately.

**Example:**

```python
bus = MyBus()
bus.start_monitor_sync(host="0.0.0.0", port=8765)
```

## Event Store Integration

The MonitorMixin automatically creates an `EventStore` instance with:

- **`db_path`**: Path to SQLite database (default: `"events.db"`)
- **`max_events`**: Ring buffer capacity in events (default: `5000`)

The store:
- Starts automatically when `start_monitor()` is called
- Appends every event emitted through the bus
- Provides cursor-based pull via the `/events` API
- Closes gracefully on `stop_monitor()`

## Labels

Events carry an ordered `tuple[str, ...]` of labels for filtering:

```python
# Emit with labels
bus.emit(Event("user.login", labels=("user", "auth"), payload={"user": "devon"}))

# Pull only events with both 'user' AND 'auth' labels
store.pull(cursor=0, labels=("user", "auth"), limit=50)
```

The pull API uses intersection logic — all specified labels must be present in the event.
