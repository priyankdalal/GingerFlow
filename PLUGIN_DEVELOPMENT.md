# Gingerflow Plugin Development Guide

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Plugin Discovery](#2-plugin-discovery)
3. [Required Files](#3-required-files)
4. [NodeMetadata Reference](#4-nodemetadata-reference)
5. [PortSpec Reference](#5-portspec-reference)
6. [NodeWidgetSpec Reference](#6-nodewidgetspec-reference)
7. [NodeConfiguration (Properties Panel)](#7-nodeconfiguration-properties-panel)
8. [ExecutionContext Reference](#8-executioncontext-reference)
9. [Return Value Contract](#9-return-value-contract)
10. [Node Patterns](#10-node-patterns)
    - [Simple Node](#101-simple-node)
    - [Repeatable Node (streaming / loop body)](#102-repeatable-node-streaming--loop-body)
    - [Loop-Ending Node (collect / stop)](#103-loop-ending-node-collect--stop)
    - [Infinite / Listening Node (SSE, WebSocket)](#104-infinite--listening-node-sse-websocket)
    - [Conditional Branch Node (If / Switch)](#105-conditional-branch-node-if--switch)
    - [Stop-Branch Node](#106-stop-branch-node)
    - [Stateful Node (counter / accumulator)](#107-stateful-node-counter--accumulator)
    - [Node with Optional Dependencies](#108-node-with-optional-dependencies)
11. [VariableStore & Template Resolution](#11-variablestore--template-resolution)
12. [Cancellation](#12-cancellation)
13. [Error Handling](#13-error-handling)
14. [External Plugin (pip-installable)](#14-external-plugin-pip-installable)
15. [Complete Sample Plugin](#15-complete-sample-plugin)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)
17. [SVG Node Icons (`icons.json`)](#17-svg-node-icons-iconsjson)

---

## 1. Architecture Overview

```
gingerflowEngine
  └── NodeRegistry          – stores {node_type → class}
  └── PluginLoader          – discovers and imports plugins
        ├── filesystem scan  – app/plugins/builtin/<name>/
        └── entry_points     – group "gingerflow.plugins"

WorkflowExecutor
  └── IScheduler            – topological sort of graph
  └── NodeRegistry          – creates node instances
  └── ExecutionContext       – shared state for one run
        ├── inputs           – {node_id: {port_name: value}}
        ├── outputs          – {node_id: {port_name: value}}
        ├── properties       – {node_id: {widget_key: value}}
        ├── state            – free-form dict (cross-node)
        └── variables        – VariableStore ({{name}} templates)
```

### Execution Loop

The executor runs a **while-loop** over topologically sorted nodes. Each pass through the list executes only nodes that are "stale" (have new upstream data or have requested a repeat). This means:

- A single workflow run can call `execute()` on a node **many times**.
- A node signals it wants to run again by returning `"__repeat_node__": True`.
- The loop ends when no node is stale and no node has requested a repeat.

---

## 2. Plugin Discovery

gingerflow loads plugins via **two mechanisms** (both at startup):

### 2a. Filesystem scan (built-in plugins)

Any directory placed under `app/plugins/builtin/` that contains **both** `metadata.json` and `plugin.py` is automatically imported.

```
app/plugins/builtin/
  myplugin/
    __init__.py      ← optional but conventional
    metadata.json    ← required
    documentation.json ← optional (recommended)
    node.py          ← your node classes
    plugin.py        ← required: calls registry.register_node_type()
```

### 2b. Entry-points (external / pip-installable plugins)

Declare in your `pyproject.toml`:

```toml
[project.entry-points."gingerflow.plugins"]
my-plugin = "my_package.plugin:register"
```

The loader calls `register(registry)` automatically.

> **Deduplication:** if the same `module` path appears in both mechanisms, it is only loaded once.

---

## 3. Required Files

### `metadata.json`

```json
{
  "name": "My Plugin",
  "category": "My Category",
  "author": "Your Name",
  "version": "1.0"
}
```

| Key | Required | Description |
|-----|----------|-------------|
| `name` | yes | Human-readable plugin name shown in logs |
| `category` | yes | Default category for nodes in this plugin |
| `author` | no | Author attribution |
| `version` | no | Semver string |

> `category` in `metadata.json` is **informational only** — each node declares its own `category` in `NodeMetadata`.

---

### `documentation.json` (recommended)

Use this file to provide rich, per-node documentation rendered in the toolbox documentation modal.

```json
{
    "nodes": {
        "myplugin.my_node": {
            "documentation": "# My Node\n\n**Purpose**: Explain what this node does.\n\n- Input details\n- Output details\n\n<span style=\"color:#5a6f92\">Tip: add short usage guidance.</span>"
        },
        "myplugin.other_node": "# Other Node\n\nShort markdown doc also works as a plain string."
    }
}
```

Notes:

- Key is the exact `node_type`.
- Value can be either:
    - an object containing `documentation`, or
    - a direct markdown string.
- Markdown is supported, and HTML spans (for simple color accents) are preserved by the modal renderer.
- If a node has no entry in `documentation.json`, the UI falls back to `NodeMetadata.description`.

---

### `plugin.py`

```python
from app.core.registry import INodeRegistry
from my_plugin.node import NODES

def register(registry: INodeRegistry) -> None:
    for node_cls in NODES:
        registry.register_node_type(node_cls)
```

The `register` function receives `INodeRegistry` and must call `register_node_type(cls)` for every node class. Registering the same `node_type` string twice raises `ValueError`.

---

### `node.py`

Contains one or more `BaseNode` subclasses and a `NODES` tuple/list at the bottom.

---

### `__init__.py`

```python
from my_plugin.node import NODES
__all__ = ["NODES"]
```

Optional but keeps imports clean.

---

## 4. NodeMetadata Reference

```python
from app.core.node import NodeMetadata, PortSpec, NodeWidgetSpec

class MyNode(BaseNode):
    metadata = NodeMetadata(
        node_type   = "myplugin.my_node",   # unique dot-separated ID
        display_name= "My Node",            # shown in toolbox
        category    = "My Category",        # toolbox grouping
        documentation = "Short markdown help shown in modal.",
        icon        = "my-node",            # optional icon hint token for default icon generator
        description = "One-line summary.",  # tooltip text
        inputs      = (...),                # tuple[PortSpec, ...]
        outputs     = (...),                # tuple[PortSpec, ...]
        central_widgets = (...),            # tuple[NodeWidgetSpec, ...]
    )
```

| Field | Type | Notes |
|-------|------|-------|
| `node_type` | `str` | Must be globally unique. Convention: `"namespace.name"` |
| `display_name` | `str` | Shown in toolbox and on the node canvas |
| `category` | `str` | Toolbox tree group name |
| `documentation` | `str \| None` | Node help shown in the toolbox info modal |
| `icon` | `str \| None` | Optional icon hint used by default icon generator |
| `description` | `str \| None` | Shown in tooltip on hover |
| `inputs` | `tuple[PortSpec]` | Left-side connection ports |
| `outputs` | `tuple[PortSpec]` | Right-side connection ports |
| `central_widgets` | `tuple[NodeWidgetSpec]` | Inline controls rendered inside the node card |

### Icon behavior

- gingerflow renders default node icons in both toolbox rows and canvas headers.
- Icons are generated from node metadata (display name + optional `icon` hint).
- You do not need to ship image assets for basic usage.
- To provide **custom SVG icons** (shown in the execution Timeline panel), add an `icons.json` file to your plugin directory. See **Section 17** for the full specification.

### Documentation precedence

When both are present, per-node docs are resolved in this order:

1. `documentation.json` entry for that `node_type`
2. `NodeMetadata.documentation`
3. `NodeMetadata.description`

---

## 5. PortSpec Reference

```python
PortSpec(
    name          = "input_name",   # port identifier (used in get_input / get_output)
    data_type     = "str",          # visual label only — not enforced at runtime
    required      = False,          # if True, UI shows a required indicator
    editable      = True,           # if True, user can type a default value directly
    default_value = "",             # pre-filled value when no connection exists
)
```

### `data_type` values (visual hint only)

| Value | Meaning |
|-------|---------|
| `"any"` | Accepts / emits any type |
| `"str"` | String |
| `"int"` | Integer |
| `"float"` | Float |
| `"bool"` | Boolean |
| `"list"` | List |
| `"dict"` | Dict |
| `"bytes"` | Raw bytes |
| `"path_open"` | Shows an **Open** file-picker button when editable |
| `"path_save"` | Shows a **Save** file-picker button when editable |

### Reading inputs in `execute()`

```python
# Simple read with a default
value = context.get_input(self.id, "input_name", default="")

# Template resolution (resolves {{variable}} placeholders)
text = context.variables.resolve_template(str(context.get_input(self.id, "text", "")))

# Integer coercion pattern
raw = context.get_input(self.id, "count", 1)
count = int(float(str(raw or 1)))
```

---

## 6. NodeWidgetSpec Reference

Central widgets are rendered **inside** the node card. Values are stored in `context.properties[node_id][key]` and read with `context.get_property(self.id, "key", default)`.

```python
NodeWidgetSpec(
    label             = "My Label",      # displayed next to the widget
    ui_widget         = "entry",         # widget type (see table below)
    key               = "my_key",        # property key; auto-derived from label if None
    options           = ("a", "b"),      # for dropdown_single / dropdown_multi
    default_value     = "",              # initial value
    read_only         = False,           # disable user editing
    runtime_editable  = False,           # allow editing during execution
    file_dialog_mode  = None,            # "open" | "save" — adds file picker to entry widget
    number_min        = None,            # numeric widgets: lower bound
    number_max        = None,            # numeric widgets: upper bound
    number_step       = None,            # numeric widgets: increment step
)
```

### `ui_widget` types

| Type | Widget | `default_value` type | `get_property` returns |
|------|--------|---------------------|------------------------|
| `"entry"` | Single-line text | `str` | `str` |
| `"textarea"` | Multi-line text | `str` | `str` |
| `"number"` | Spinbox (`QSpinBox` / `QDoubleSpinBox`) | `int` / `float` | `int` / `float` |
| `"slider"` | Horizontal slider | `int` | `int` |
| `"radio_group"` | Exclusive radio options | `str` | `str` |
| `"image_render"` | Inline image preview/source widget | `str` | `str` |
| `"checkbox"` | Toggle checkbox | `bool` | `bool` |
| `"dropdown_single"` | Single-select combo | `str` (one of `options`) | `str` |
| `"dropdown_multi"` | Multi-select list | `list[str]` | `list[str]` |
| `"button"` | Click counter button | `int` (0) | `int` (increments on each click) |
| `"list_builder"` | Dynamic list of strings | `list[str]` | `list[str]` |
| `"object_builder"` | Dynamic key-value pairs | `list[dict]` | `list[{"key":…,"value":…}]` |
| `"variable_builder"` | Type+value pair | `dict` | `{"type": "string", "value": "…"}` |

### Reading widget values

```python
# String entry
text = str(context.get_property(self.id, "my_key", "default"))

# Checkbox
enabled = bool(context.get_property(self.id, "enabled", True))

# Number widget
threshold = float(context.get_property(self.id, "threshold", 0.0))

# Slider widget
percent = int(context.get_property(self.id, "percent", 50))

# Dropdown
mode = str(context.get_property(self.id, "mode", "read"))

# Button click count
clicks = int(context.get_property(self.id, "btn", 0) or 0)

# Object builder
pairs = context.get_property(self.id, "headers", [])  # list[{"key":…,"value":…}]
headers = {item["key"]: item["value"] for item in pairs if isinstance(item, dict)}
```

---

## 7. NodeConfiguration (Properties Panel)

`NodeConfiguration` is a Pydantic model whose fields appear in the **Properties** side panel (not on the node card). Use it for long-lived credentials, connection strings, or rarely-changed settings.

```python
from pydantic import Field
from app.core.node import BaseNode, NodeConfiguration

class MyConfig(NodeConfiguration):
    connection_string: str = Field(
        default="",
        description="Database DSN",
        json_schema_extra={"ui_type": "password"},
    )
    timeout: int = Field(default=30, description="Timeout in seconds")
    api_key: str = Field(
        default="",
        description="Secret API key",
        json_schema_extra={"ui_type": "password"},
    )

class MyNode(BaseNode):
    config_model = MyConfig   # ← tell gingerflow to use this config

    def execute(self, context):
        conn = self.configuration.connection_string   # read from self.configuration
        timeout = self.configuration.timeout
```

### Password masking in the Properties panel

Password masking is **explicit**. A field is masked only when it sets:

`json_schema_extra={"ui_type": "password"}`

Example:

```python
class CredentialsConfig(NodeConfiguration):
    username: str = Field(default="", description="Service username")
    password: str = Field(
        default="",
        description="Service password",
        json_schema_extra={"ui_type": "password"},
    )
    client_id: str = Field(default="", description="OAuth client ID")
    client_secret: str = Field(
        default="",
        description="OAuth client secret",
        json_schema_extra={"ui_type": "password"},
    )
```

### Credential placement guideline

Use `NodeConfiguration` (Properties panel) for credentials such as username, password, client_id, and client_secret.

Do not expose credentials as runtime `PortSpec` inputs unless the node explicitly needs per-invocation dynamic secrets.

Current built-in examples that follow this pattern include Email (SMTP/IMAP), FTP nodes, database connection strings, and Azure connection settings.

---

## 8. ExecutionContext Reference

`context` is passed to every `execute()` call and is **shared across all nodes in one run**.

```python
# --- Inputs & Outputs ---
context.get_input(node_id, port_name, default=None)   # read an incoming value
context.set_output(node_id, port_name, value)          # write (executor does this for you)
context.get_output(node_id, port_name, default=None)   # read output of any node

# --- Widget properties ---
context.get_property(node_id, key, default=None)       # read central widget value
context.set_property(node_id, key, value)              # set programmatically

# --- Cross-node state ---
context.state["my_key"] = value    # free-form dict; persists for entire run
value = context.state.get("my_key", default)

# --- Variable templates ---
context.variables.resolve_template("Hello {{name}}")   # resolves {{var}} placeholders
context.variables.set_workflow("name", "Alice")        # set a workflow-scoped variable
context.variables.set_global("name", "Alice")          # set a global variable
context.variables.get("name", default=None)            # read a variable

# --- Cancellation ---
if context.cancellation.is_cancelled:
    return {}
```

### `context.state` naming convention

Use namespaced keys to avoid collisions with other nodes:

```python
# Good
key = f"_mynode_counter_{self.id}"
context.state[key] = context.state.get(key, 0) + 1

# Bad (collides with anything else using "counter")
context.state["counter"] += 1
```

### Runtime state conventions used by built-ins

- `context.state["performance_mode"]` is a runtime-wide boolean set by the UI subprocess launch mode.
- Nodes should treat this as a contract for low-memory behavior when payload conversion would be expensive.

Example:

```python
if bool(context.state.get("performance_mode", False)):
    # Avoid optional heavyweight conversions (for example, large data-uri strings)
    pass
```

### Binary payload guidance

- Prefer binary (`bytes`) outputs for large content (images/files) and keep text conversions opt-in.
- Keep execution outputs transport-safe; UI event streams are serialized and should avoid giant unbounded blobs.

---

## 9. Return Value Contract

`execute()` must return a `dict[str, Any]`. The executor processes it as follows:

```python
def execute(self, context) -> dict[str, Any]:
    return {
        # --- Normal outputs (match a PortSpec name) ---
        "out": some_value,
        "count": 42,

        # --- Special executor signals (NOT output ports) ---
        "__repeat_node__": True,      # re-run this node in the next sweep
        "__stop_branch__": True,      # block all outgoing edges from this node
        "__active_outputs__": ["out"] # only allow listed outputs to flow downstream
    }
```

| Key | Type | Meaning |
|-----|------|---------|
| `"<port_name>"` | any | Written to `context.outputs[node_id][port_name]` and forwarded to connected nodes |
| `"__repeat_node__"` | `bool` | `True` → re-schedule this node to run again in a future sweep |
| `"__stop_branch__"` | `bool` | `True` → block ALL outgoing edges; downstream nodes are skipped |
| `"__active_outputs__"` | `str \| list[str]` | Only edges whose source port is in this set remain active; all others are blocked |

> **Omitting a port** from the return dict means that port is not updated — downstream nodes that depend only on that port will NOT be re-scheduled as stale.

---

## 10. Node Patterns

### 10.1 Simple Node

Runs once, reads inputs, writes outputs. No special signals.

```python
class UppercaseNode(BaseNode):
    metadata = NodeMetadata(
        node_type="myns.uppercase",
        display_name="Uppercase",
        category="String",
        description="Convert text to uppercase.",
        inputs=(PortSpec(name="text", data_type="str", required=True, editable=True, default_value=""),),
        outputs=(PortSpec(name="out", data_type="str", required=False),),
    )

    def execute(self, context: ExecutionContext) -> dict[str, Any]:
        text = str(context.get_input(self.id, "text", ""))
        return {"out": text.upper()}
```

---

### 10.2 Repeatable Node (streaming / loop body)

Emits one item per execution cycle and signals the executor to run it again. Downstream nodes (and nodes connected to them) are also re-scheduled each cycle.

```python
class CounterNode(BaseNode):
    metadata = NodeMetadata(
        node_type="myns.counter",
        display_name="Counter",
        category="Flow",
        description="Emit integers from start to end, one per cycle.",
        inputs=(
            PortSpec(name="start", data_type="int", required=False, editable=True, default_value="0"),
            PortSpec(name="end",   data_type="int", required=True,  editable=True, default_value="10"),
        ),
        outputs=(
            PortSpec(name="value", data_type="int",  required=False),
            PortSpec(name="index", data_type="int",  required=False),
            PortSpec(name="count", data_type="int",  required=False),
        ),
    )

    def execute(self, context: ExecutionContext) -> dict[str, Any]:
        start = int(float(str(context.get_input(self.id, "start", 0) or 0)))
        end   = int(float(str(context.get_input(self.id, "end",   10) or 10)))

        # Use context.state to persist index across repeat cycles.
        state_key = f"_counter_index_{self.id}"
        index = int(context.state.get(state_key, 0))
        count = end - start

        if index >= count:
            # Reset for next run and stop repeating.
            context.state[state_key] = 0
            return {"value": None, "index": count, "count": count, "__repeat_node__": False}

        value = start + index
        context.state[state_key] = index + 1
        more = (index + 1) < count

        return {
            "value": value,
            "index": index,
            "count": count,
            "__repeat_node__": more,   # True → keep running; False → done
        }
```

**Key rules for repeatable nodes:**

1. Store iteration state in `context.state` using a namespaced key that includes `self.id`.
2. Return `"__repeat_node__": True` while more items remain; `False` (or omit it) when done.
3. Reset state when the iterator is exhausted so the node can be re-run in a fresh execution.
4. The executor will not re-run a repeating node while any of its *descendants* are still repeating — it waits for the subtree to drain first.

---

### 10.3 Loop-Ending Node (collect / stop)

A node that **accumulates** values arriving through repeat cycles and emits a final result once the upstream repeating node has finished.

```python
class CollectNode(BaseNode):
    metadata = NodeMetadata(
        node_type="myns.collect",
        display_name="Collect",
        category="Flow",
        description="Accumulate streamed items into a single list.",
        inputs=(
            PortSpec(name="in",    data_type="any", required=False, editable=False),
            PortSpec(name="index", data_type="int", required=False, editable=False),
            PortSpec(name="count", data_type="int", required=False, editable=False),
        ),
        outputs=(
            PortSpec(name="items", data_type="any", required=False),
            PortSpec(name="total", data_type="int", required=False),
        ),
    )

    def execute(self, context: ExecutionContext) -> dict[str, Any]:
        value = context.get_input(self.id, "in")
        index = context.get_input(self.id, "index", 0)
        count = context.get_input(self.id, "count", 0)

        buf_key = f"_collect_buf_{self.id}"

        # Reset buffer when a new stream starts (index == 0).
        if index == 0:
            context.state[buf_key] = []

        buf: list = context.state.setdefault(buf_key, [])
        if value is not None:
            buf.append(value)

        is_last = (index + 1) >= count if count else True

        if not is_last:
            # Not yet complete — emit nothing downstream.
            return {}

        # Stream is done — emit the full list and clear.
        result = list(buf)
        context.state[buf_key] = []
        return {"items": result, "total": len(result)}
```

**Pattern:** connect the `index` and `count` outputs from your repeating node to the collect node's `index` and `count` inputs so it knows when the stream is finished.

---

### 10.4 Infinite / Listening Node (SSE, WebSocket)

A node that keeps running "forever" (e.g., listening to a stream) until the user stops execution or the source closes. Use `context.cancellation.is_cancelled` as the exit condition.

For resources that must stay open across repeat cycles (for example stream sockets or long-lived clients), register a cleanup callback so UI Stop and CLI interrupts can release resources gracefully:

```python
from app.runtime.resource_manager import register_cleanup_handler, unregister_cleanup_handler
```

```python
class EventStreamNode(BaseNode):
    metadata = NodeMetadata(
        node_type="myns.event_stream",
        display_name="Event Stream",
        category="Network",
        description="Connect to an SSE endpoint and emit one event per cycle.",
        inputs=(PortSpec(name="url", data_type="str", required=True, editable=True, default_value=""),),
        outputs=(
            PortSpec(name="data",  data_type="str", required=False),
            PortSpec(name="event", data_type="str", required=False),
        ),
    )

    def execute(self, context: ExecutionContext) -> dict[str, Any]:
        import requests

        url = context.variables.resolve_template(
            str(context.get_input(self.id, "url", "")).strip()
        )

        # Check cancellation before starting.
        if context.cancellation.is_cancelled:
            return {}

        conn_key  = f"_stream_conn_{self.id}"
        iter_key  = f"_stream_iter_{self.id}"

        # Re-use existing connection across repeat cycles.
        if conn_key not in context.state:
            resp = requests.get(url, stream=True, timeout=None)
            resp.raise_for_status()
            context.state[conn_key] = resp
            context.state[iter_key] = resp.iter_lines()
            register_cleanup_handler(f"sse:{self.id}", resp.close)

        line_iter = context.state[iter_key]

        # Read the next line (blocking until data arrives).
        try:
            for raw_line in line_iter:
                if context.cancellation.is_cancelled:
                    unregister_cleanup_handler(f"sse:{self.id}")
                    context.state.get(conn_key) and context.state[conn_key].close()
                    context.state.pop(conn_key, None)
                    context.state.pop(iter_key, None)
                    return {}
                if raw_line:
                    line = raw_line.decode() if isinstance(raw_line, bytes) else raw_line
                    if line.startswith("data:"):
                        return {
                            "data": line[5:].strip(),
                            "event": "message",
                            "__repeat_node__": True,   # keep listening
                        }
        except Exception:
            unregister_cleanup_handler(f"sse:{self.id}")
            context.state.get(conn_key) and context.state[conn_key].close()
            context.state.pop(conn_key, None)
            context.state.pop(iter_key, None)

        unregister_cleanup_handler(f"sse:{self.id}")
        context.state.get(conn_key) and context.state[conn_key].close()
        context.state.pop(conn_key, None)
        context.state.pop(iter_key, None)
        return {}   # stream ended
```

**Key rules for infinite nodes:**

1. Store the connection/iterator in `context.state` so it survives repeat cycles.
2. Register long-lived resources in `app.runtime.resource_manager` immediately after opening them.
3. Check `context.cancellation.is_cancelled` inside the loop and clean up state before returning.
4. Unregister cleanup handlers when your node closes the resource itself.
5. Return `"__repeat_node__": True` every time a new event is emitted.
6. Clean up the connection from `context.state` when the stream ends or is cancelled.

Runtime behavior note:

- UI Stop sends a graceful stop request first, and runtime cleanup handlers are executed before any force-terminate fallback.
- CLI Ctrl+C performs graceful cancellation and executes cleanup handlers before process exit.

---

### 10.5 Conditional Branch Node (If / Switch)

Use `"__active_outputs__"` to selectively allow only certain output ports to propagate downstream. Ports not in the active set have their outgoing edges blocked for this cycle.

```python
class IfNode(BaseNode):
    metadata = NodeMetadata(
        node_type="myns.if",
        display_name="If",
        category="Logic",
        description="Route execution based on a boolean condition.",
        inputs=(
            PortSpec(name="condition", data_type="bool", required=True, editable=True, default_value="false"),
            PortSpec(name="in",        data_type="any",  required=False, editable=False),
        ),
        outputs=(
            PortSpec(name="true",  data_type="any", required=False),
            PortSpec(name="false", data_type="any", required=False),
        ),
    )

    def execute(self, context: ExecutionContext) -> dict[str, Any]:
        raw = context.get_input(self.id, "condition", False)
        condition = raw if isinstance(raw, bool) else str(raw).lower() in {"true", "1", "yes"}
        value = context.get_input(self.id, "in")

        if condition:
            return {"true": value, "__active_outputs__": ["true"]}
        else:
            return {"false": value, "__active_outputs__": ["false"]}
```

For a **Switch** node with many branches, `__active_outputs__` accepts a list:

```python
return {
    branch_name: value,
    "__active_outputs__": [branch_name],
}
```

---

### 10.6 Stop-Branch Node

Halts all downstream execution unconditionally (e.g., an error guard or a flow terminator).

```python
class StopNode(BaseNode):
    metadata = NodeMetadata(
        node_type="myns.stop",
        display_name="Stop",
        category="Flow",
        description="Unconditionally halt this branch of execution.",
        inputs=(PortSpec(name="in", data_type="any", required=False),),
        outputs=(),
    )

    def execute(self, context: ExecutionContext) -> dict[str, Any]:
        return {"__stop_branch__": True}
```

`"__stop_branch__": True` blocks every outgoing edge from this node. Downstream nodes that have no other active incoming path will be skipped.

---

### 10.7 Stateful Node (counter / accumulator)

Nodes that need to remember state **across repeat cycles within one execution run** use `context.state`. State is NOT persisted between workflow runs.

```python
class RunningAverageNode(BaseNode):
    metadata = NodeMetadata(
        node_type="myns.running_avg",
        display_name="Running Average",
        category="Math",
        description="Compute a running average over streamed numbers.",
        inputs=(PortSpec(name="value", data_type="float", required=True, editable=False),),
        outputs=(
            PortSpec(name="average", data_type="float", required=False),
            PortSpec(name="samples", data_type="int",   required=False),
        ),
    )

    def execute(self, context: ExecutionContext) -> dict[str, Any]:
        value = float(context.get_input(self.id, "value", 0.0) or 0.0)
        sum_key = f"_ravg_sum_{self.id}"
        cnt_key = f"_ravg_cnt_{self.id}"

        total = context.state.get(sum_key, 0.0) + value
        count = context.state.get(cnt_key, 0) + 1
        context.state[sum_key] = total
        context.state[cnt_key] = count

        return {"average": total / count, "samples": count}
```

---

### 10.8 Node with Optional Dependencies

For third-party packages that may not be installed:

```python
import importlib

class MyNode(BaseNode):
    ...
    def execute(self, context: ExecutionContext) -> dict[str, Any]:
        try:
            some_lib = importlib.import_module("some_lib")
        except ImportError as exc:
            raise RuntimeError(
                "MyNode requires 'some_lib'. Install with: pip install some_lib"
            ) from exc
        # use some_lib ...
```

This pattern defers the import error until the node is actually executed, allowing gingerflow to start even if the dependency is missing. Only nodes that are actually used in a flow will fail.

---

## 11. VariableStore & Template Resolution

gingerflow supports `{{variable_name}}` templates in any string value.

```python
# In execute():
raw = context.get_input(self.id, "url", "")
resolved = context.variables.resolve_template(str(raw))
# "https://api.example.com/{{env}}/users" → "https://api.example.com/prod/users"
```

Variable names must match `[A-Za-z_][A-Za-z0-9_]*`.

```python
# Set a variable from a node (e.g., Variable node pattern)
name  = str(context.get_input(self.id, "name",  "")).strip()
value = context.get_input(self.id, "value")
context.variables.set_workflow(name, value)
```

**Scope precedence:** workflow variables override global variables.

---

## 12. Cancellation

The user can stop a running workflow at any time. Long-running nodes must check the token:

```python
def execute(self, context: ExecutionContext) -> dict[str, Any]:
    for item in big_list:
        if context.cancellation.is_cancelled:
            return {}       # return empty dict — do not raise
        process(item)
    return {"out": result}
```

> Always return `{}` (not `None`) on cancellation. Raising an exception on cancellation is unnecessary and confusing in the log.

### 12.1 Resource cleanup contract for plugin authors

If your node opens sockets, streams, file handles, or long-lived SDK clients:

1. Register a cleanup callback after opening the resource.
2. Keep local `try/finally` cleanup in the node for normal success and error paths.
3. Unregister the callback after local cleanup so runtime does not try to close it twice.

```python
from app.runtime.resource_manager import register_cleanup_handler, unregister_cleanup_handler

resource_name = f"my-node:{self.id}"
client = open_client()
register_cleanup_handler(resource_name, client.close)
try:
    ...
finally:
    client.close()
    unregister_cleanup_handler(resource_name)
```

This guarantees graceful release on normal completion, UI Stop, runtime cancellation, and CLI Ctrl+C.

---

## 13. Error Handling

- Raise any `Exception` subclass to fail the node. The executor logs it and (depending on the `RetryPolicy`) either retries or stops the workflow.
- In UI subprocess execution, failures are emitted as structured `execution_failed` events including:
    - `node_id`
    - `error`
    - `error_details` (traceback text)
- The UI errors pane shows both the concise error and traceback details.
- In CLI execution, failures print to stderr and return non-zero exit codes.
- Use `ValueError` for user-facing validation errors (bad input, missing field).
- Use `RuntimeError` for dependency or system errors.
- Use `FileNotFoundError`, `ConnectionError`, etc. for specific OS/network errors.

```python
def execute(self, context):
    path = str(context.get_input(self.id, "path", "")).strip()
    if not path:
        raise ValueError("Path must not be empty")   # shown to the user

    if not Path(path).exists():
        raise FileNotFoundError(f"File not found: {path}")

    # ... rest of logic
```

---

## 14. External Plugin (pip-installable)

To distribute a plugin as a Python package:

### Project structure

```
my_gingerflow_plugin/
  pyproject.toml
  my_gingerflow_plugin/
    __init__.py
    plugin.py
    node.py
```

### `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[project]
name = "my-gingerflow-plugin"
version = "1.0.0"
dependencies = ["gingerflow"]

[project.entry-points."gingerflow.plugins"]
my-plugin = "my_gingerflow_plugin.plugin:register"
```

After `pip install my-gingerflow-plugin`, gingerflow discovers and loads it automatically on next startup via the `gingerflow.plugins` entry point group.

---

## 14.1 Compatibility Checklist (UI + CLI)

Before shipping a plugin, validate both execution paths:

1. UI path: run workflow from desktop app and verify node outputs, branch controls, and cancellation behavior.
2. CLI path: run `python -m app.cli --compact your_workflow.fg` and verify deterministic outputs.
3. Failure path: trigger a controlled node error and confirm clear exception messages.
4. Optional dependency path: confirm missing package error message tells user exactly what to install.

Recommended CI assertion set:

1. successful execute() path
2. validation error path
3. cancellation-safe cleanup path
4. serialization compatibility for returned output types

---

## 15. Complete Sample Plugin

This example implements a fully working **HTTP File Download** plugin with three nodes:
`Download File`, `Get File Size`, and `Hash File`.

### File tree

```
app/plugins/builtin/downloader/
  __init__.py
  metadata.json
  node.py
  plugin.py
```

### `metadata.json`

```json
{
  "name": "Downloader",
  "category": "Network",
  "author": "Your Name",
  "version": "1.0"
}
```

### `__init__.py`

```python
from app.plugins.builtin.downloader.node import NODES
__all__ = ["NODES"]
```

### `plugin.py`

```python
from app.core.registry import INodeRegistry
from app.plugins.builtin.downloader.node import NODES

def register(registry: INodeRegistry) -> None:
    for node_cls in NODES:
        registry.register_node_type(node_cls)
```

### `node.py`

```python
from __future__ import annotations

import hashlib
import importlib
from pathlib import Path
from typing import Any

from app.core.context import ExecutionContext
from app.core.node import BaseNode, NodeMetadata, NodeWidgetSpec, PortSpec


class DownloadFileNode(BaseNode):
    """Download a URL to disk, emitting path, size, and HTTP status."""

    metadata = NodeMetadata(
        node_type="downloader.download",
        display_name="Download File",
        category="Network",
        description="Download a URL to a local file path.",
        inputs=(
            PortSpec(name="url",  data_type="str",       required=True,  editable=True,  default_value=""),
            PortSpec(name="dest", data_type="path_save", required=True,  editable=True,  default_value=""),
        ),
        outputs=(
            PortSpec(name="path",        data_type="str",  required=False),
            PortSpec(name="size_bytes",  data_type="int",  required=False),
            PortSpec(name="status_code", data_type="int",  required=False),
        ),
        central_widgets=(
            NodeWidgetSpec(
                label="Timeout (s)",
                key="timeout",
                ui_widget="entry",
                default_value="30",
            ),
            NodeWidgetSpec(
                label="Overwrite Existing",
                key="overwrite",
                ui_widget="checkbox",
                default_value=True,
            ),
        ),
    )

    def execute(self, context: ExecutionContext) -> dict[str, Any]:
        try:
            requests = importlib.import_module("requests")
        except ImportError as exc:
            raise RuntimeError(
                "Download File requires 'requests'. Install with: pip install requests"
            ) from exc

        url  = context.variables.resolve_template(str(context.get_input(self.id, "url",  ""))).strip()
        dest = context.variables.resolve_template(str(context.get_input(self.id, "dest", ""))).strip()
        overwrite = str(context.get_property(self.id, "overwrite", "true")).lower() in {"true", "1", "yes"}
        timeout   = float(str(context.get_property(self.id, "timeout", "30")).strip() or 30)

        if not url:
            raise ValueError("Download File: url is required")
        if not dest:
            raise ValueError("Download File: dest is required")

        dest_path = Path(dest)
        if dest_path.exists() and not overwrite:
            raise FileExistsError(f"Download File: file already exists: {dest_path}")

        dest_path.parent.mkdir(parents=True, exist_ok=True)

        response = requests.get(url, stream=True, timeout=timeout)
        response.raise_for_status()

        with dest_path.open("wb") as fh:
            for chunk in response.iter_content(chunk_size=65536):
                if context.cancellation.is_cancelled:
                    dest_path.unlink(missing_ok=True)
                    return {}
                fh.write(chunk)

        return {
            "path":        str(dest_path),
            "size_bytes":  dest_path.stat().st_size,
            "status_code": response.status_code,
        }


class GetFileSizeNode(BaseNode):
    """Return the size of a local file in bytes and a human-readable string."""

    metadata = NodeMetadata(
        node_type="downloader.file_size",
        display_name="Get File Size",
        category="Network",
        description="Return the byte size and human-readable size of a local file.",
        inputs=(
            PortSpec(name="path", data_type="path_open", required=True, editable=True, default_value=""),
        ),
        outputs=(
            PortSpec(name="bytes",  data_type="int", required=False),
            PortSpec(name="human",  data_type="str", required=False),
        ),
    )

    def execute(self, context: ExecutionContext) -> dict[str, Any]:
        raw = context.variables.resolve_template(
            str(context.get_input(self.id, "path", "")).strip()
        )
        path = Path(raw)
        if not path.exists():
            raise FileNotFoundError(f"Get File Size: not found: {path}")

        size = path.stat().st_size
        for unit in ("B", "KB", "MB", "GB", "TB"):
            if size < 1024:
                human = f"{size:.1f} {unit}"
                break
            size /= 1024
        else:
            human = f"{size:.1f} PB"

        return {"bytes": path.stat().st_size, "human": human}


class HashFileNode(BaseNode):
    """Compute a cryptographic hash of a local file."""

    metadata = NodeMetadata(
        node_type="downloader.hash_file",
        display_name="Hash File",
        category="Network",
        description="Compute MD5, SHA-1, or SHA-256 hash of a local file.",
        inputs=(
            PortSpec(name="path", data_type="path_open", required=True, editable=True, default_value=""),
        ),
        outputs=(
            PortSpec(name="hex",       data_type="str", required=False),
            PortSpec(name="algorithm", data_type="str", required=False),
        ),
        central_widgets=(
            NodeWidgetSpec(
                label="Algorithm",
                key="algorithm",
                ui_widget="dropdown_single",
                options=("sha256", "sha1", "md5"),
                default_value="sha256",
            ),
        ),
    )

    def execute(self, context: ExecutionContext) -> dict[str, Any]:
        raw = context.variables.resolve_template(
            str(context.get_input(self.id, "path", "")).strip()
        )
        path = Path(raw)
        if not path.exists():
            raise FileNotFoundError(f"Hash File: not found: {path}")

        algorithm = str(context.get_property(self.id, "algorithm", "sha256")).strip()
        h = hashlib.new(algorithm)

        with path.open("rb") as fh:
            for chunk in iter(lambda: fh.read(65536), b""):
                if context.cancellation.is_cancelled:
                    return {}
                h.update(chunk)

        return {"hex": h.hexdigest(), "algorithm": algorithm}


NODES = [DownloadFileNode, GetFileSizeNode, HashFileNode]
```

---

## 16. Quick Reference Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────────┐
│ FILE                   PURPOSE                                      │
├─────────────────────────────────────────────────────────────────────┤
│ metadata.json          Plugin name/author/version for loader        │
│ plugin.py              register(registry) → register_node_type()    │
│ node.py                BaseNode subclasses + NODES list             │
│ __init__.py            Optional re-export of NODES                  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ RETURN KEY             EFFECT                                       │
├─────────────────────────────────────────────────────────────────────┤
│ "port_name": value     Forward value to connected downstream nodes  │
│ __repeat_node__: True  Re-run this node in next sweep               │
│ __stop_branch__: True  Block ALL outgoing edges                     │
│ __active_outputs__: [] Only named outputs flow; rest are blocked    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ NODE TYPE              PATTERN                                      │
├─────────────────────────────────────────────────────────────────────┤
│ Simple                 Return outputs, no special keys              │
│ Streaming/Repeatable   Return __repeat_node__: True while looping   │
│ Loop-Ending/Collect    Accumulate in context.state; emit on last    │
│ Infinite/Listener      Loop in execute() + check cancellation       │
│ Conditional/Branch     Return __active_outputs__: [chosen_port]     │
│ Stop                   Return __stop_branch__: True                 │
│ Stateful               Store data in context.state[f"key_{self.id}"]│
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ WIDGET TYPE            READ WITH                                    │
├─────────────────────────────────────────────────────────────────────┤
│ entry / textarea       str(context.get_property(…))                 │
│ checkbox               bool(context.get_property(…, True))           │
│ number / slider        int/float(context.get_property(…))            │
│ radio_group            str(context.get_property(…))                  │
│ image_render           str(context.get_property(…))                  │
│ dropdown_single        str(context.get_property(…))                 │
│ dropdown_multi         list = context.get_property(…)               │
│ button                 int(context.get_property(…, 0))              │
│ object_builder         list[{"key":…,"value":…}]                    │
│ variable_builder       {"type":…, "value":…}                        │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ ICONS.JSON KEY         DESCRIPTION                                  │
├─────────────────────────────────────────────────────────────────────┤
│ "node.type"            Full node_type string → object with keys:    │
│   "image"              Required — inline SVG string                 │
│   "color"              Required — hex color applied to all fills    │
│ fill="none"            Preserved — transparent areas stay clear     │
│ Other fill/stroke      Replaced at render time with "color" value   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 17. SVG Node Icons (`icons.json`)

### Overview

Each plugin directory may contain an optional `icons.json` file that maps
node types to SVG icon definitions. When present, gingerflow renders these
icons at three surfaces: the **Toolbox card**, the **canvas node header**,
and the **execution Timeline** in the Process Stats panel.

The icons are rendered as `QPixmap` objects via Qt's SVG renderer. A single
`"color"` field controls the icon tint across all surfaces so you only
specify it once.

---

### Plugin directory structure

```
app/plugins/builtin/myplugin/
├── __init__.py
├── metadata.json
├── plugin.py
├── node.py
├── documentation.json   (optional)
└── icons.json           ← add this file
```

The file **must be named exactly `icons.json`** — any other name is silently
ignored by the discovery scan.

---

### File format

```json
{
  "myplugin.mynode": {
    "image": "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24'><path d='M0 0h24v24H0z' fill='none'/><path fill='currentColor' d='M12 2a10 10 0 1 0 0 20A10 10 0 0 0 12 2'/></svg>",
    "color": "#2563eb"
  },
  "myplugin.othernode": {
    "image": "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24'><path d='M0 0h24v24H0z' fill='none'/><path fill='currentColor' d='M5 3h14v18H5z'/></svg>",
    "color": "#0f766e"
  }
}
```

| Field | Required | Description |
|---|---|---|
| Key | ✓ | Full `node_type` string — must match `NodeMetadata.node_type` exactly |
| `"image"` | ✓ | Complete self-contained inline SVG string (no newlines — keep on one line) |
| `"color"` | ✓ | Hex color applied to every `fill`/`stroke` in the SVG |

**Notes:**
- `viewBox` should be square (e.g. `"0 0 24 24"` or `"0 0 16 16"`) for correct scaling.
- JSON does not allow literal newlines inside strings — keep the entire SVG on a single line. If you author the SVG in a file first, collapse it with: `(Get-Content icon.svg -Raw) -replace '\r?\n\s*', ' '`
- Duplicate node type keys are silently overwritten by the last entry loaded.

---

### SVG authoring guidelines

**Use `currentColor` for stroked icons:**
```xml
<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24'>
  <path d='M0 0h24v24H0z' fill='none'/>
  <path stroke='currentColor' stroke-width='2' fill='none' d='M5 12h14M12 5l7 7-7 7'/>
</svg>
```

**Use an explicit fill attribute for filled icons:**
```xml
<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24'>
  <path d='M0 0h24v24H0z' fill='none'/>
  <path fill='currentColor' d='M12 2a10 10 0 1 0 0 20A10 10 0 0 0 12 2'/>
</svg>
```

**Design as monochromatic.** Every `fill` and `stroke` value is replaced with
the single `"color"` at render time. Gradients and multi-colour fills will be
flattened to one colour.

---

### Color injection rules

gingerflow's color injector processes the SVG string before rendering:

| Attribute / property | Behaviour |
|---|---|
| `fill='none'` / `fill="none"` | **Preserved** — transparent areas stay clear |
| `stroke='none'` / `stroke="none"` | **Preserved** |
| `fill='currentColor'` (or double-quoted) | Replaced with resolved color |
| `stroke='currentColor'` (or double-quoted) | Replaced with resolved color |
| Any other `fill='#xxx'` value | Replaced with resolved color |
| Any other `stroke='#xxx'` value | Replaced with resolved color |
| CSS `fill: #xxx;` in `style=` attribute | Replaced with resolved color |
| No fill attribute at all | `style="fill:COLOR"` injected on the root `<svg>` |

**Color priority** (highest wins):
1. Caller-supplied `color=` argument when calling `get_pixmap()` in code.
2. `"color"` field in `icons.json` ← normal control point.
3. Built-in fallback `#607189` (muted blue-grey).

---

### Where icons appear

| Surface | Size | Color source |
|---|---|---|
| Toolbox card | 18 × 18 px | `"color"` from `icons.json` |
| Canvas node header | 18 × 18 px | `"color"` from `icons.json` |
| Execution Timeline (Process Stats) | 16 × 16 px | `"color"` from `icons.json` |

When no icon is registered for a node type, gingerflow falls back to the
initials-based coloured tile (auto-generated from the display name).

---

### Registering icons from external (pip-installable) plugins

Entry-point plugins that are not inside `app/plugins/builtin/` must register
icons programmatically inside their `register()` function:

```python
from app.ui.node_svg_icons import NodeSvgIconRegistry

def register(registry) -> None:
    registry.register_node_type(MyNode)
    NodeSvgIconRegistry.register_svg(
        node_type="myplugin.mynode",
        svg_text="<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24'>...</svg>",
        color="#4f9ef8",
    )
```

The `color` argument is optional; omitting it uses the `#607189` fallback.

---

### Discovery and caching

1. On application startup, gingerflow calls `NodeSvgIconRegistry.discover()`,
   which scans every subdirectory of `app/plugins/builtin/` for `icons.json`
   and registers all entries **before** the Toolbox is built. This ensures the
   `lru_cache` in `build_node_icon_pixmap` stores the correct SVG pixmaps.
2. External plugins call `NodeSvgIconRegistry.register_svg()` inside `register()`.
3. Rendered pixmaps are cached by `(node_type, color, size)`. The same icon at
   the same color and size is never re-rendered twice.
4. Calling `register_svg()` for a previously registered type invalidates its
   cached pixmaps so the next `get_pixmap()` call re-renders with the new data.
