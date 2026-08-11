# Task lifecycle reference (language-agnostic rules, Python example)

How to implement a taskable agent that receives and executes tasks via the Lattice Tasks API.
The lifecycle rules here apply across languages and protocols; the concrete code snippets use the
Python SDK because it is a compact example.

## Overview

A taskable agent does two things in parallel:
1. **Publishes itself** as a live entity with a `task_catalog` advertising what tasks it accepts
2. **Streams task events** via `client.tasks.stream_as_agent(...)` and drives each task through
   its status lifecycle

Both must be running simultaneously — a common pattern is a background thread for entity
heartbeat and the main thread for the task stream loop.

## The task stream loop

`stream_as_agent` is a gRPC server-streaming call. It delivers `execute_request` and
`cancel_request` events as the server sends them. The stream can disconnect (server-side timeout,
network hiccup) — wrap the call in a reconnect loop so the agent never permanently misses events:

```python
import threading
from anduril import Principal, System

def run_task_stream(client, agent_entity_id):
    while True:
        try:
            for event in client.tasks.stream_as_agent(
                principal=Principal(system=System(entity_id=agent_entity_id))
            ):
                event_type = event.type if hasattr(event, "type") else None

                if hasattr(event, "execute_request") and event.execute_request:
                    req = event.execute_request
                    task_id = req.task.task_id
                    # Spawn a worker thread per task so the stream loop never blocks
                    t = threading.Thread(target=execute_task, args=(client, agent_entity_id, task_id), daemon=True)
                    t.start()

                elif hasattr(event, "cancel_request") and event.cancel_request:
                    task_id = event.cancel_request.task_id
                    # Signal the matching worker to stop
                    cancel_event = active_tasks.get(task_id)
                    if cancel_event:
                        cancel_event.set()

        except Exception as e:
            print(f"[task stream] disconnected ({e}), reconnecting...")
            time.sleep(2)
```

Key points:
- **Reconnect on any exception.** The stream is long-lived by design; a disconnect is expected,
  not an error.
- **One worker thread per task.** Don't block the stream loop on task execution or you'll miss
  concurrent events.
- **Thread-safe cancellation.** Use a `threading.Event` per active task; the stream loop sets it,
  the worker polls it.

## Worker thread: status lifecycle

The worker drives the task from `STATUS_SENT` through to a terminal state. Every update needs a
strictly increasing `status_version` — own the counter yourself, starting at 1.

```python
active_tasks = {}   # task_id -> threading.Event (cancellation signal)

def execute_task(client, agent_entity_id, task_id):
    cancel_event = threading.Event()
    active_tasks[task_id] = cancel_event
    author = Principal(system=System(entity_id=agent_entity_id))
    version = 1

    def update(status):
        nonlocal version
        client.tasks.update_task_status(
            task_id=task_id,
            status=status,
            status_version=version,
            author=author,
        )
        version += 1

    try:
        update("STATUS_EXECUTING")

        # Do the work — poll cancel_event so cancellations are handled promptly
        elapsed = 0
        while elapsed < EXECUTE_DURATION_SECONDS:
            if cancel_event.is_set():
                update("STATUS_DONE_NOT_OK")
                return
            time.sleep(0.2)
            elapsed += 0.2

        update("STATUS_DONE_OK")

    except Exception as e:
        print(f"[task {task_id}] error: {e}")
        update("STATUS_DONE_NOT_OK")
    finally:
        active_tasks.pop(task_id, None)
```

## Status lifecycle rules

- **`STATUS_EXECUTING`** — report this immediately when you pick up the task. Operators see this
  as confirmation the agent is working.
- **`STATUS_DONE_OK`** — the happy-path terminal state. Only report this if the task completed
  without cancellation or error.
- **`STATUS_DONE_NOT_OK`** — the failure/cancellation terminal state. Use this for cancelled
  tasks, errors, and anything that didn't complete successfully.
- **Terminal states are final.** Once you report `STATUS_DONE_OK` or `STATUS_DONE_NOT_OK`, do
  not send further updates for that task.
- **`status_version` must strictly increase.** Lattice ignores any update where the version is
  not greater than what it has already seen. Start at 1 and increment on every update.

Never leave a task permanently in `STATUS_EXECUTING` — it will appear stuck to operators forever.
Cancellations in particular must reach a terminal state.

## Wiring it together

```python
import time, threading, os
from anduril import Lattice, Principal, System, TaskCatalog, TaskDefinition

AGENT_ENTITY_ID = os.environ["AGENT_ENTITY_ID"]
EXECUTE_DURATION_SECONDS = float(os.environ.get("EXECUTE_DURATION_SECONDS", "10"))
active_tasks = {}

client = Lattice(
    base_url=f"https://{os.environ['LATTICE_ENDPOINT']}",
    client_id=os.environ["LATTICE_CLIENT_ID"],
    client_secret=os.environ["LATTICE_CLIENT_SECRET"],
    headers={"anduril-sandbox-authorization": f"Bearer {os.environ['SANDBOXES_TOKEN']}"},
    timeout=300,
)

def publish_entity():
    """Heartbeat loop — keep the agent entity live."""
    from datetime import datetime, timedelta, timezone
    from anduril import Aliases, Ontology, MilView, Provenance, Location, Position
    while True:
        now = datetime.now(timezone.utc)
        client.entities.publish_entity(
            entity_id=AGENT_ENTITY_ID,
            is_live=True,
            expiry_time=(now + timedelta(seconds=30)).isoformat(),
            aliases=Aliases(name="My Agent"),
            ontology=Ontology(template="TEMPLATE_ASSET", platform_type="agent"),
            mil_view=MilView(disposition="DISPOSITION_FRIENDLY", environment="ENVIRONMENT_LAND"),
            provenance=Provenance(
                integration_name="my-agent",
                data_type="agent",
                source_update_time=now.isoformat(),
            ),
            task_catalog=TaskCatalog(task_definitions=[
                # Example only. Built-in: anduril.tasks.v2.Investigate, .VisualId, etc.
                # Custom: <your-org>.tasks.v1.<YourTaskName>.
                TaskDefinition(task_specification_url="type.googleapis.com/anduril.tasks.v2.Investigate")
            ]),
        )
        time.sleep(5)

if __name__ == "__main__":
    threading.Thread(target=publish_entity, daemon=True).start()
    run_task_stream(client, AGENT_ENTITY_ID)
```

## Validating the task lifecycle end-to-end

For a taskable agent, "read it back" means more than confirming the entity is live — the thing
that actually has to work is the **task loop**, so prove it: with your agent running, dispatch a
task and confirm its status progresses to a terminal state. A process that starts without crashing
tells you nothing: a common failure mode is a task agent that registers fine but never picks up
routed tasks, leaving them stuck at `STATUS_SENT`.

Dispatch the task and watch its status from a **separate driver script** — its own freshly
constructed client, run as a distinct process from your agent — so it proves the server-side
lifecycle rather than your agent code's view of it. With the agent running, in another shell:

```python
# validate_task.py — a SEPARATE process from your agent. Constructs its own client.
import os, time
from anduril import Lattice, Principal, System, GoogleProtobufAny

c = Lattice(
    base_url=f"https://{os.environ['LATTICE_ENDPOINT']}",
    client_id=os.environ["LATTICE_CLIENT_ID"],
    client_secret=os.environ["LATTICE_CLIENT_SECRET"],
    headers={"anduril-sandbox-authorization": f"Bearer {os.environ['SANDBOXES_TOKEN']}"},
    timeout=60,
)
agent_id = os.environ["AGENT_ENTITY_ID"]

# 1. Confirm the agent entity is live and advertises the task URL.
e = c.entities.get_entity(entity_id=agent_id)
print("live:", e.is_live, "catalog:", e.task_catalog.task_definitions)

# 2. Send a task assigned to the agent, then poll its status a few times.
# `type` below must match a type_url actually advertised in step 1, not this example value.
task = c.tasks.create_task(
    display_name="validate",
    specification=GoogleProtobufAny(type="type.googleapis.com/anduril.tasks.v2.Investigate", objective={"entity_id": agent_id}),
    author=Principal(system=System(service_name="validator")),
    relations={"assignee": Principal(system=System(entity_id=agent_id))},
    is_executed_elsewhere=False,
)
task_id = task.version.task_id
for _ in range(10):
    t = c.tasks.get_task(task_id=task_id)
    print("status:", t.status.status, "version:", t.status.status_version)
    time.sleep(1)
```

The status should move `STATUS_SENT` → `STATUS_EXECUTING` → `STATUS_DONE_OK`. If it stays at
`STATUS_SENT`, the stream loop isn't receiving or processing events — check the stream
subscription, the `principal` argument's `entity_id`, and whether the publish loop is running so
the entity is live *before* you open the stream.
