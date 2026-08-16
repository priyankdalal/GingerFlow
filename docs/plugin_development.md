# GingerFlow Plugin Development Guide

This guide covers everything you need to write, package, install, and debug a GingerFlow Python plugin from scratch.

## Current implementation snapshot

The current plugin runtime supports ZIP installation, hot registration, plugin-local icons, Python nodes executed through a workflow-scoped worker, per-node timeout control with `0` meaning no GingerFlow timeout, listener-style long-running nodes, scoped resource cleanup, hidden execution control keys, and declarative central-widget actions. The core application currently does not collect or transmit plugin data, workflow data, credentials, execution payloads, or telemetry to the GingerFlow developer; any external transmission is caused by the workflow, plugin code, or explicitly configured service.

---

## Table of Contents

1. [Overview](#overview)
2. [Quick Start — Demo Plugin](#quick-start--demo-plugin)
3. [Plugin ZIP Structure](#plugin-zip-structure)
4. [Plugin Icons: icons.json](#plugin-icons-iconsjson)
5. [Manifest: plugin.json](#manifest-pluginjson)
   - [Top-level fields](#top-level-fields)
   - [Node definition fields](#node-definition-fields)
   - [Port definition fields](#port-definition-fields)
   - [Config definition fields](#config-definition-fields)
6. [Writing the Python Node](#writing-the-python-node)
   - [Function signature](#function-signature)
   - [Inputs and config](#inputs-and-config)
   - [Return value](#return-value)
   - [Raising errors](#raising-errors)
   - [Importing helpers](#importing-helpers)
7. [Data Types](#data-types)
8. [Config Widget Types](#config-widget-types)
9. [End-to-End Example: Text Transformer](#end-to-end-example-text-transformer)
10. [End-to-End Example: HTTP Request](#end-to-end-example-http-request)
11. [Python Environment Setup](#python-environment-setup)
    - [First-run auto-setup](#first-run-auto-setup)
    - [Resolution priority](#resolution-priority)
    - [Settings dialog](#settings-dialog)
    - [Virtual environments](#virtual-environments)
12. [Installing a Plugin](#installing-a-plugin)
    - [Install flow](#install-flow)
    - [Where plugins are stored](#where-plugins-are-stored)
    - [Dependency auto-install](#dependency-auto-install)
13. [Execution Model](#execution-model)
14. [Debugging](#debugging)
    - [Error messages reference](#error-messages-reference)
15. [Central Widget Event Callbacks](#central-widget-event-callbacks)
16. [Packaging Checklist](#packaging-checklist)

---

## Overview

GingerFlow's plugin system lets you add custom workflow nodes written in **Python 3**. Each plugin is a **ZIP archive** that GingerFlow extracts and loads at runtime. During a workflow run, Python plugin nodes execute through a workflow-scoped long-lived Python worker that receives framed JSON requests and returns JSON responses.

No C++ knowledge, recompilation, or restarts are required. New nodes appear in the Toolbox **immediately** after installation.

For resource release semantics (file/socket/db/process handles), see `docs/resource_lifecycle_and_cleanup.md`.

---

## Quick Start — Demo Plugin

A fully working demo plugin is included in the repository at `dist/gingerflow_demo.zip`. It contains four nodes (HTTP fetch, date parsing, text statistics, template formatting) and demonstrates `requirements.txt` auto-install.

```
dist/
├── gingerflow_demo.zip      ← ready to install
├── gingerflow_demo/         ← source files
│   ├── plugin.json
│   ├── requirements.txt     (requests, python-dateutil)
│   └── nodes/
│       ├── http_node.py
│       ├── date_node.py
│       ├── text_node.py
│       └── template_node.py
└── build_plugin.ps1         ← rebuild the ZIP: .\dist\build_plugin.ps1
```

To install: **View → Plugin Manager → Install Plugin from ZIP…** → select `dist/gingerflow_demo.zip`.

---

## Plugin ZIP Structure

```
my_plugin.zip
├── plugin.json           ← required manifest (MUST be at the ZIP root)
├── icons.json            ← optional icon map for this plugin's node types
├── my_node.py            ← Python implementation files
├── helper_utils.py       ← optional support modules
└── requirements.txt      ← optional pip dependencies
```

**Critical rules:**

- `plugin.json` **must** be at the **root** of the ZIP, not inside any sub-folder. GingerFlow validates this after extraction and rejects plugins with wrong structure.
- Python files referenced in `plugin.json` via `entry` must use paths relative to the plugin directory root.
- Sub-directories are allowed for organising helper modules (e.g. `nodes/my_node.py`).
- `icons.json` is optional. If omitted, GingerFlow falls back to `app.default` for all node icons from that plugin.

**Packaging command (Windows PowerShell):**

```powershell
# From inside the plugin source directory — exports contents, not the folder itself
Compress-Archive -Path my_plugin\* -DestinationPath my_plugin.zip

# Verify structure (plugin.json must appear at root, not under my_plugin/)
$z = [System.IO.Compression.ZipFile]::OpenRead("my_plugin.zip")
$z.Entries | Select-Object FullName
$z.Dispose()
```

```bash
# Linux / macOS
cd my_plugin && zip -r ../my_plugin.zip .
unzip -l ../my_plugin.zip  # plugin.json must appear without a path prefix
```

---

## Plugin Icons: icons.json

Plugin authors can ship a plugin-local `icons.json` at the plugin root. GingerFlow loads this file (if present) and uses it to render icons in the toolbox, node header, and process timeline.

### File shape

`icons.json` must be a JSON object keyed by node type.

```json
{
  "app.default": {
    "image": "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24'><rect x='3' y='3' width='18' height='18' rx='4' fill='currentColor'/></svg>",
    "color": "#4c6ea9"
  },
  "my_plugin.transform": {
    "image": "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24'><path fill='currentColor' d='M4 4h16v16H4z'/></svg>",
    "color": "#2f9e66"
  }
}
```

Supported fields per icon entry:

- `image` (preferred), `svg`, or `icon`: SVG string content.
- `color` (preferred) or `iconColor`: color used to replace `currentColor` in the SVG.

### Resolution and fallback rules

- If a node type has a matching key in any loaded icon map, that icon is used.
- If a node type has no icon entry, GingerFlow uses `app.default`.
- If an external plugin does not provide `icons.json`, all its node types render with `app.default`.

Best practice:

- Always include `app.default` in your plugin `icons.json` so your plugin has a consistent fallback style.
- Keep icon keys exactly equal to `nodes[].type` values in `plugin.json`.

---

## Manifest: plugin.json

The manifest is a UTF-8 JSON file that describes the plugin and all its nodes.

### Top-level fields

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | **Yes** | Unique plugin ID. Used as the installation directory name. Use only letters, digits, `_`, `-`, `.`. |
| `version` | string | No | Semantic version, e.g. `"1.2.0"`. Defaults to `"0.0.0"`. Shown in Plugin Manager. |
| `author` | string | No | Author name. Shown in Plugin Manager and node metadata. |
| `description` | string | No | Short summary shown in Plugin Manager. |
| `nodes` | array | **Yes** | List of node definitions. |

### Node definition fields

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | **Yes** | Globally unique node type ID, e.g. `"myplugin.do_thing"`. Pattern: `<plugin_name>.<node_name>`. Must not clash with built-in types. |
| `display_name` | string | No | Human-readable name in the Toolbox and on the canvas node. Defaults to `type`. |
| `category` | string | No | Toolbox category group. Defaults to `"Python Plugins"`. |
| `description` | string | No | Tooltip / detail text. |
| `entry` | string | **Yes** | Path to the Python file relative to the plugin directory root, e.g. `"my_node.py"` or `"nodes/transform.py"`. |
| `function` | string | No | Callable name inside the entry module. Defaults to `"execute"`. |
| `inputs` | array | No | Input port definitions. |
| `outputs` | array | No | Output port definitions. |
| `config` | array | No | Inspector configuration field definitions. |

### Port definition fields

Used in both `inputs` and `outputs`.

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | **Yes** | Internal key: matches the key in `inputs` dict passed to the function, or the key in the returned dict. |
| `label` | string | No | Display label on the canvas port. Defaults to `name`. |
| `data_type` | string | No | Type hint on the port. See [Data Types](#data-types). Defaults to `"any"`. |
| `required` | boolean | No | If `true`, the canvas marks this port as mandatory. Defaults to `false`. |

### Config definition fields

Config fields appear in the Properties panel (Inspector) when the node is selected on canvas.

| Field | Type | Required | Description |
|---|---|---|---|
| `key` | string | **Yes** | Key in the `config` dict passed to the Python function. |
| `label` | string | No | Label in the Inspector. Defaults to `key`. |
| `type` | string | No | Widget type: `"String"`, `"Password"`, or `"Enum"`. Defaults to `"String"`. |
| `default` | string | No | Default value. |
| `enum_values` | array | No | Required when `type` is `"Enum"`. Populates the dropdown. |

---

## Writing the Python Node

### Function signature

```python
def execute(inputs: dict, config: dict) -> dict:
    ...
```

GingerFlow calls the function with two positional arguments:

- **`inputs`** — `dict[str, Any]` — values on input ports, keyed by port `name`.
- **`config`** — `dict[str, str]` — Inspector values, keyed by config `key`. All values are **strings**; cast as needed.

The function **must return a `dict`** whose keys match the output port `name`s. Extra keys are ignored. Missing keys produce `None` on connected downstream ports.

### Inputs and config

```python
def execute(inputs: dict, config: dict) -> dict:
    text   = inputs.get("text", "")          # None if port not connected → use default
    mode   = config.get("mode", "upper")     # always str
    repeat = int(config.get("repeat", "1"))  # cast as needed
    ...
```

Unconnected input ports deliver `None`. Always guard with `.get()` or `or ""`.

### Return value

```python
return {
    "result": transformed_text,
    "count":  len(transformed_text),
}
```

Values must be JSON-serialisable (`str`, `int`, `float`, `bool`, `list`, `dict`, `None`). Non-serialisable objects are coerced to `str()` automatically.

### Raising errors

Any unhandled exception stops the workflow and shows the full Python traceback in the Output console.

```python
def execute(inputs: dict, config: dict) -> dict:
    path = inputs.get("file_path")
    if not path:
        raise ValueError("file_path input is required")
    ...
```

### Importing helpers

The plugin directory is prepended to `sys.path`, so sibling modules import without any path manipulation:

```
my_plugin/
├── plugin.json
├── transform.py      ← entry file
└── utils.py          ← helper
```

```python
# transform.py
from utils import clean_text

def execute(inputs, config):
    return {"result": clean_text(inputs.get("text", ""))}
```

Third-party packages must be installed in the Python environment GingerFlow is configured to use. Declare them in `requirements.txt` — GingerFlow installs them automatically during plugin install.

---

## Data Types

Informational hints displayed on canvas ports. Not enforced at runtime.

| `data_type` | Meaning |
|---|---|
| `"any"` | Accepts any value (default) |
| `"string"` | Text |
| `"number"` | Integer or float |
| `"boolean"` | `true` / `false` |
| `"list"` | Python list / JSON array |
| `"object"` | Python dict / JSON object |
| `"path_select"` | File path (opens a file picker on compatible port widgets) |
| `"path_save"` | Save file path |

---

## Config Widget Types

| `type` | Inspector widget | Notes |
|---|---|---|
| `"String"` | Single-line text field | Default |
| `"Password"` | Masked text field with show/hide eye toggle | Passed as plain text to Python |
| `"Enum"` | Drop-down list | Requires `enum_values` array |

---

## End-to-End Example: Text Transformer

### Directory structure

```
text_transformer/
├── plugin.json
└── transform.py
```

### plugin.json

```json
{
  "name": "text_transformer",
  "version": "1.0.0",
  "author": "Jane Doe",
  "description": "Common text transformation nodes.",
  "nodes": [
    {
      "type": "text_transformer.transform",
      "display_name": "Text Transform",
      "category": "Text Transformer",
      "description": "Convert text to upper, lower, title case, or reverse it.",
      "entry": "transform.py",
      "function": "execute",
      "inputs": [
        {"name": "text", "label": "Input Text", "data_type": "string", "required": true}
      ],
      "outputs": [
        {"name": "result", "label": "Result", "data_type": "string"},
        {"name": "length", "label": "Length", "data_type": "number"}
      ],
      "config": [
        {
          "key": "mode", "label": "Mode", "type": "Enum",
          "default": "upper", "enum_values": ["upper", "lower", "title", "reverse"]
        },
        {
          "key": "__python_timeout_ms", "label": "Execution Timeout (ms, 0 = no timeout)", "type": "String",
          "default": "0"
        }
      ]
    }
  ]
}
```

### transform.py

```python
def execute(inputs: dict, config: dict) -> dict:
    text = str(inputs.get("text") or "")
    mode = config.get("mode", "upper")

    if mode == "upper":   result = text.upper()
    elif mode == "lower": result = text.lower()
    elif mode == "title": result = text.title()
    elif mode == "reverse": result = text[::-1]
    else: raise ValueError(f"Unknown mode: {mode!r}")

    return {"result": result, "length": len(result)}
```

---

## End-to-End Example: HTTP Request

Demonstrates `requirements.txt` auto-install and multiple functions in one entry file.

### requirements.txt

```
requests>=2.28.0
```

### plugin.json (excerpt)

```json
{
  "name": "http_tools",
  "version": "1.0.0",
  "author": "Your Name",
  "description": "HTTP GET and POST nodes.",
  "nodes": [
    {
      "type": "http_tools.get",
      "display_name": "HTTP GET",
      "category": "HTTP Tools",
      "entry": "http_nodes.py",
      "function": "http_get",
      "inputs": [
        {"name": "url",     "label": "URL",     "data_type": "string", "required": true},
        {"name": "headers", "label": "Headers", "data_type": "object"}
      ],
      "outputs": [
        {"name": "body",        "label": "Body",        "data_type": "string"},
        {"name": "status_code", "label": "Status Code", "data_type": "number"},
        {"name": "json",        "label": "JSON",        "data_type": "object"}
      ],
      "config": [
        {"key": "timeout",             "label": "Timeout (s)",        "type": "String", "default": "30"},
        {"key": "verify_ssl",          "label": "Verify SSL",         "type": "Enum",   "default": "true",
         "enum_values": ["true", "false"]},
        {"key": "__python_timeout_ms", "label": "Execution Timeout (ms)", "type": "String", "default": "0"}
      ]
    }
  ]
}
```

### http_nodes.py

```python
try:
    import requests
except ImportError:
    raise RuntimeError("Install 'requests': pip install requests")


def http_get(inputs: dict, config: dict) -> dict:
    url     = inputs.get("url") or ""
    headers = inputs.get("headers") or {}
    timeout = float(config.get("timeout", 30))
    verify  = config.get("verify_ssl", "true") == "true"

    if not url:
        raise ValueError("'url' input is required")

    resp = requests.get(url, headers=headers, timeout=timeout, verify=verify)

    parsed_json = None
    try:
        parsed_json = resp.json()
    except Exception:
        pass

    return {"body": resp.text, "status_code": resp.status_code, "json": parsed_json}
```

---

## Python Environment Setup

### First-run auto-setup

On the **first launch** after installation, GingerFlow automatically:

1. Scans system `PATH` for a Python 3 interpreter.
2. Creates a virtual environment at `%APPDATA%\GingerFlow\GingerFlow\python_venv` (Windows) or `~/.local/share/GingerFlow/GingerFlow/python_venv` (Linux/macOS).
3. Saves both paths to settings.

This happens silently during the splash screen. No manual action is required if Python is already on `PATH`.

### Resolution priority

GingerFlow resolves the Python interpreter in this exact order:

| Priority | Source |
|---|---|
| 1 | `GINGERFLOW_PYTHON` environment variable |
| 2 | Active virtual environment (configured in Settings) |
| 3 | Configured interpreter search paths (checked in order) |
| 4 | System `PATH` fallback |

The active virtual environment always wins over manually listed interpreter paths.

### Settings dialog

Open **View → Settings…** to configure Python and runtime paths.

The dialog has a tree on the left and an editor on the right:

**Environment → Python Interpreter Locations**

A list of directories or direct executable paths. Each entry is checked in order until a valid interpreter is found. Entries can be:
- A directory containing `python` or `python3` (e.g. `C:\Python312\`)
- A direct path to a Python executable (e.g. `C:\Python312\python.exe`)

The **Detect Python** button tests all configured entries against the current (unsaved) list and shows which interpreter would be resolved, or reports "Not found" if nothing matches.

**Environment → Python Virtual Environments**

Manage multiple virtual environments. The ★ active environment is used for all plugin execution and dependency installation.

- **Create New Venv…** — scans your configured interpreter paths, offers a selection dialog if multiple are found, asks for a parent directory, then runs `python -m venv <dir>/gingerflow_venv` in a live progress window.
- **Add Existing…** — register a virtual environment you already created elsewhere.
- **★ Set as Active** — mark the selected row as the environment GingerFlow uses at runtime.
- **🗑 Remove** — removes the entry from the list (does not delete files from disk).

**Environment → Runtime Library Paths**

Directories or direct binary paths for external tools (e.g. `psql`, `mongosh`, `ffmpeg`). Plugins that need external binaries look here before falling back to `PATH`.

### Virtual environments

Creating and using a dedicated virtual environment is strongly recommended. It isolates plugin dependencies from the system Python.

```powershell
# Windows — manual creation (or use View → Settings → Create New Venv…)
python -m venv C:\venvs\gingerflow_plugins
C:\venvs\gingerflow_plugins\Scripts\pip install requests python-dateutil

# Then add it in View → Settings → Python Virtual Environments → Add Existing…
```

```bash
# Linux / macOS
python3 -m venv ~/venvs/gingerflow_plugins
~/venvs/gingerflow_plugins/bin/pip install requests python-dateutil
# Add it in View → Settings → Python Virtual Environments → Add Existing…
```

**Override via environment variable** (highest priority, bypasses all settings):

```powershell
$env:GINGERFLOW_PYTHON = "C:\venvs\gingerflow_plugins\Scripts\python.exe"
```

---

## Installing a Plugin

### Install flow

1. Open **View → Plugin Manager…**
2. Click **Install Plugin from ZIP…**
3. Browse to your `.zip` file and click **Open**.
4. GingerFlow extracts the ZIP into its own subdirectory inside the plugin root.
5. It validates that `plugin.json` is present — if not, extraction is rolled back with a clear error.
6. It loads the manifest and registers all node types declared in it.
7. A success dialog lists the exact node type IDs that were registered.
8. If a `requirements.txt` is present, a **live terminal-style dialog** opens running `pip install -r requirements.txt`. Output streams in real time.
9. New nodes appear in the **Toolbox** immediately — no restart required.

If any step fails, an error dialog explains what went wrong and what to check. The window never closes silently on failure.

### Where plugins are stored

```
%APPDATA%\GingerFlow\GingerFlow\plugins\      (Windows)
~/.local/share/GingerFlow/GingerFlow/plugins/  (Linux/macOS)

plugins/
├── my_plugin/            ← one subdirectory per plugin (named after the ZIP)
│   ├── plugin.json
│   ├── requirements.txt
│   └── nodes/
│       └── *.py
└── gingerflow_runner.py  ← GingerFlow's Python bridge — do not edit or delete
```

The directory name is taken from the ZIP filename without the `.zip` extension. The ZIP filename therefore determines the plugin directory name — name it accordingly.

To **uninstall**: select the plugin in Plugin Manager → **Uninstall**. Files are deleted immediately. Nodes disappear from the Toolbox after restart.

### Dependency auto-install

When a `requirements.txt` is present, GingerFlow automatically runs:

```
<active_python> -m pip install -r requirements.txt
```

using whichever Python interpreter is active (virtual environment preferred). Output appears in a live progress dialog with green/red status on completion.

If no Python is found, a warning explains how to configure one in Settings. If pip reports errors, the warning includes the manual install command.

---

## Execution Model

```
GingerFlow Worker Process (gingerflow_worker.exe)
│
├─ Loads all installed Python plugins at startup
│
├─ Workflow executes
│   └─ Node "demo.text_stats" is reached
│       └─ PythonNode::execute() called
│           ├─ Resolves Python interpreter (venv → config paths → PATH)
│           ├─ Verifies gingerflow_runner.py exists in plugin root
│           ├─ Builds JSON payload:
│           │     { "plugin_dir":  "…/plugins/gingerflow_demo",
│           │       "entry":       "nodes/text_node.py",
│           │       "function":    "execute",
│           │       "inputs":      { "text": "Hello world" },
│           │       "config":      { "words_per_minute": "238" } }
│           ├─ Spawns: python gingerflow_runner.py
│           ├─ Writes payload JSON → stdin of the subprocess
│           ├─ Waits until the node timeout; default `0` means no deadline
│           └─ Reads result JSON ← stdout
│               { "outputs": { "word_count": 2, … } }
│
└─ Outputs propagated to downstream nodes
```

**Important behaviours:**

| Behaviour | Detail |
|---|---|
| One worker per workflow run | Python nodes reuse one long-lived worker within a run; no state is shared across runs |
| Per-request timeout | If a request exceeds timeout, the session is marked failed and the worker is terminated |
| `sys.stdout` is reserved | Use `sys.stderr` for debug prints; they appear in the Output console |
| `sys.stdin` is consumed | Do not read from stdin in your function |
| JSON only | All values must be JSON-serialisable; others are coerced to `str()` |
| Both desktop and worker | The worker process (`gingerflow_worker.exe`) loads the same plugins as the desktop, so workflows run correctly |

Resource cleanup for file/socket/db/process handles is scope-aware (node, branch, workflow). See `docs/resource_lifecycle_and_cleanup.md`.

Every Python node exposes `__python_timeout_ms` in the Inspector. It defaults to `0`, which means no timeout and no automatic worker kill for that node. Set a positive value in milliseconds to enforce a deadline. Blank values are also treated as no timeout for compatibility with older workflows; negative or non-numeric values are invalid.

---

## Debugging

### Configure timeout for one node

Every Python node exposes `Execution Timeout (ms, 0 = no timeout)` in Properties.

- The default is `0`, meaning no timeout and no automatic worker kill.
- Set `Execution Timeout (ms, 0 = no timeout)` to a positive integer (for example `120000` for 2 minutes).
- Set it to `0` when the node is a long-running listener or intentionally unbounded operation.
- Blank values are treated as `0` for compatibility with older workflows.
- Invalid values (negative, non-numeric) fail the node with a clear validation error.

This override applies only to that specific node execution request. It is separate from timeouts used inside your Python libraries, such as `requests`, database drivers, or socket APIs.

### Print debug output

Write to `stderr` — it is captured and shown in the **Output** panel:

```python
import sys

def execute(inputs, config):
    print(f"[debug] inputs: {inputs}", file=sys.stderr)
    print(f"[debug] config: {config}", file=sys.stderr)
    ...
```

### Unit-test without GingerFlow

The function is a plain Python callable:

```python
# test_transform.py
from transform import execute

result = execute({"text": "hello"}, {"mode": "upper"})
assert result == {"result": "HELLO", "length": 5}, result
print("OK")
```

### Simulate the runner protocol

```powershell
# Windows PowerShell
$payload = @{
    plugin_dir = "C:\Users\you\AppData\Roaming\GingerFlow\GingerFlow\plugins\my_plugin"
    entry      = "transform.py"
    "function" = "execute"
    inputs     = @{ text = "hello" }
    config     = @{ mode = "upper" }
} | ConvertTo-Json -Compress

$payload | python "$env:APPDATA\GingerFlow\GingerFlow\plugins\gingerflow_runner.py"
# Expected: {"outputs": {"result": "HELLO", "length": 5}}
```

### Diagnose the Python path

Add a temporary diagnostic node to your plugin:

```python
def execute(inputs, config):
    import sys, os
    return {
        "executable": sys.executable,
        "version":    sys.version,
        "sys_path":   sys.path[:5],   # first 5 entries
    }
```

### Error messages reference

| Message | Cause | Fix |
|---|---|---|
| `Python interpreter not found for node '…'` | No Python resolved from any source | View → Settings → Python Interpreter Locations or set `GINGERFLOW_PYTHON` |
| `Failed to start Python process. Command: …` | Binary not executable or path wrong | Check interpreter path in Settings; verify file exists and is executable |
| `GingerFlow runner script missing: …` | `gingerflow_runner.py` was deleted | Reinstall the plugin; the runner is recreated on first plugin load |
| `Python node timed out` | A positive `__python_timeout_ms` deadline expired | Set the node value to `0` for no GingerFlow deadline; still use library-level timeouts for network/database calls |
| `Python node returned invalid JSON` | `sys.stdout` was written to directly | Use `sys.stderr` for all debug prints |
| `Python node raised: Traceback…` | Unhandled exception in your function | Full traceback is shown — fix the Python code |
| `ModuleNotFoundError: No module named 'requests'` | Package not installed in active Python env | Run `pip install requests` in the active virtual environment |
| `plugin.json not found after extraction` | ZIP has wrong structure — `plugin.json` is inside a sub-folder | Re-package: `Compress-Archive -Path my_plugin\* …` (contents, not folder) |
| `No nodes could be registered` | `entry` paths wrong, `type` field empty, or duplicate type ID | Check `entry` matches actual file name; ensure `type` is globally unique |
| `Could not start the process` | `Expand-Archive` or Python not found during install | Check PowerShell is available; check Python path in Settings |

---

## Packaging Checklist

Before distributing your plugin:

- [ ] `plugin.json` is at the **root** of the ZIP (not inside a named sub-folder).
- [ ] Every `entry` path in the manifest matches the actual relative file path inside the ZIP.
- [ ] Every `type` value follows `<plugin_name>.<node_name>` and is globally unique (no clash with built-ins).
- [ ] All outputs are JSON-serialisable (`str`, `int`, `float`, `bool`, `list`, `dict`, `None`).
- [ ] `sys.stdout` is never written to directly in your `execute` functions.
- [ ] All inputs are guarded with `.get("key") or default` — unconnected ports pass `None`.
- [ ] Third-party dependencies are listed in `requirements.txt` at the ZIP root.
- [ ] The ZIP filename matches the `name` field in `plugin.json` (it becomes the install directory name).
- [ ] Tested with **Detect Python** in Settings to confirm GingerFlow will find the right interpreter.
- [ ] Verify ZIP structure: `$z.Entries | Select-Object FullName` — `plugin.json` must appear without a path prefix.

---

*GingerFlow Plugin System — built-in nodes: author GingerFlow v1.0.0 | external plugins: author and version from plugin.json*

---

## Complete Plugin Runtime Contract

This section is the authoritative reference for plugin authors. It documents the values that cross the C++/Python boundary, the Inspector fields created automatically, the hidden executor keys, listener behavior, cleanup rules, and the limits that matter for large or long-running nodes.

### What GingerFlow creates for each Python node

For every node declared in `plugin.json`, GingerFlow creates a native node wrapper with:

| Native value | Source | Effect |
|---|---|---|
| Node type | `nodes[].type` | Global registry key and saved-workflow identifier. |
| Display name | `nodes[].display_name` | Toolbox and canvas title. |
| Category | `nodes[].category` | Toolbox group. |
| Description | `nodes[].description` | Node help text and metadata. |
| Entry module | `nodes[].entry` | Python file loaded by the worker. |
| Callable | `nodes[].function` or `execute` | Function invoked for each execution. |
| Input ports | `nodes[].inputs` | Values collected into the `inputs` dictionary. |
| Output ports | `nodes[].outputs` | Values accepted from the returned dictionary. |
| Inspector fields | `nodes[].config` | Values collected into the `config` dictionary. |
| Timeout field | Automatic | `__python_timeout_ms`, default `0`. |

The automatically added timeout field is not necessary in `plugin.json`. Do not declare a second field with the same key. If an older workflow contains a blank timeout, it is treated as `0`.

### Exact manifest contract

The manifest must be a JSON object encoded as UTF-8. The parser requires a top-level `name` and a top-level `nodes` array. Each usable node requires a non-empty `type` and `entry`; malformed node entries without those values are skipped.

```json
{
  "name": "event_tools",
  "version": "1.0.0",
  "author": "Example Team",
  "description": "Long-running event and file processing nodes.",
  "nodes": [
    {
      "type": "event_tools.listener",
      "display_name": "Event Listener",
      "category": "Stream",
      "description": "Wait for events until the workflow is cancelled.",
      "entry": "nodes/listener.py",
      "function": "listen",
      "inputs": [],
      "outputs": [
        {"name": "event", "label": "Event", "data_type": "object"}
      ],
      "config": [
        {"key": "endpoint", "label": "Endpoint", "type": "String", "default": ""}
      ]
    }
  ]
}
```

#### Top-level properties

- `name`: required plugin identifier. Use a stable, globally unique value. It becomes part of the installed plugin path and should not be changed after publishing.
- `version`: optional display/version value. Use semantic versioning such as `1.2.0`.
- `author`: optional author or organization name.
- `description`: optional Plugin Manager summary.
- `nodes`: required array of node objects.

#### Node properties

- `type`: required and globally unique. Use `plugin_name.node_name`; the value is persisted in `.fg` workflow files.
- `display_name`: optional human-readable title; defaults to `type`.
- `category`: optional Toolbox group; defaults to `Python Plugins`.
- `description`: optional node description.
- `entry`: required path relative to the installed plugin directory. Forward slashes are recommended on every platform.
- `function`: optional Python callable name; defaults to `execute`.
- `inputs`: optional input-port array.
- `outputs`: optional output-port array.
- `config`: optional Inspector configuration array.

#### Port properties

- `name`: required internal key. It must be unique within the node's inputs or outputs.
- `label`: optional visible label; defaults to `name`.
- `data_type`: optional type hint. It is used by the canvas and optional strict validation; default is `any`.
- `required`: optional boolean. Missing required values fail when strict validation is enabled.

#### Config properties

- `key`: required key passed to Python in `config`.
- `label`: optional Inspector label; defaults to `key`.
- `type`: `String`, `Password`, or `Enum`; defaults to `String`.
- `default`: optional default value. Manifest values are read as strings.
- `enum_values`: array of strings used by `Enum`.

Password values are masked in the Inspector but are passed to Python as ordinary strings. Do not print them, return them in outputs, or write them to logs.

### Python call boundary

The worker invokes exactly one callable at a time:

```python
def execute(inputs: dict, config: dict) -> dict:
    ...
```

- `inputs` contains declared input names. An unconnected input is normally present with a `None` value.
- `config` contains declared Inspector fields and their defaults. Convert strings explicitly with `int`, `float`, or a boolean helper.
- The callable should return a dictionary whose keys match declared output names.
- A non-dictionary return is wrapped as `{"result": value}`.
- Unknown output keys are carried through the worker response but are not valid declared ports for graph routing.
- Values cross the boundary as JSON. Use strings, numbers, booleans, lists, dictionaries, and `None`.
- Bytes, open files, sockets, generators, threads, and class instances are not portable values. Return a path, summary, or JSON-safe metadata instead.

```python
def as_bool(value, default=False):
    if value is None:
        return default
    return str(value).strip().lower() in {"1", "true", "yes", "on"}


def execute(inputs, config):
    text = str(inputs.get("text") or "")
    limit = int(config.get("limit", "100"))
    return {"text": text[:limit], "length": min(len(text), limit)}
```

### Automatically reserved Inspector key

`__python_timeout_ms` is reserved by GingerFlow and automatically appears in every Python node's Properties panel.

| Value | Meaning |
|---|---|
| `0` | No GingerFlow request timeout; the worker is not killed by the node deadline. This is the default. |
| Positive integer | Maximum request duration in milliseconds. On expiry, the Python worker is killed and the node fails. |
| Blank | Treated as `0` for compatibility with older workflows. |
| Negative or non-numeric | Node validation failure. |

This setting does not cancel a request inside `requests`, a database driver, or a socket. Add library-level timeouts when appropriate. Conversely, a listener that must wait indefinitely should use `0` and rely on workflow cancellation or the Stop/Cancel control.

### Listener and long-running node rules

GingerFlow does not create a separate Python thread per listener. The long-lived Python worker processes requests serially, and one listener call occupies the worker until it returns or is cancelled.

For a listener:

1. Set `__python_timeout_ms` to `0`.
2. Use a library or loop that can observe shutdown/cancellation where possible.
3. Avoid blocking the worker with multiple unrelated listeners in one workflow; use separate workflows or an external multiplexer.
4. Do not assume a Python `finally` block is guaranteed after a hard process kill.
5. Release sockets, files, database cursors, and subprocesses in normal cancellation and error paths.
6. Return bounded event payloads. Do not return an ever-growing list.

The current Python protocol has no asynchronous callback channel. A listener can emit one result per node execution, request a repeat with `__repeat_node__`, or keep the call open. A long-running call that never returns cannot update downstream nodes until it returns.

### Hidden executor output keys

These keys control the C++ executor and are not ordinary output ports:

| Key | JSON value | Effect |
|---|---|---|
| `__repeat_node__` | boolean | Schedules the same node again after the current graph pass. |
| `__stop_branch__` | boolean | Blocks downstream edges for the current branch. |
| `__active_outputs__` | string or list of strings | Only named output edges remain active; other output edges are blocked. |

Example:

```python
def execute(inputs, config):
    index = int(config.get("index", "0"))
    values = inputs.get("values") or []
    if index >= len(values):
        return {"done": True, "__repeat_node__": False}
    return {
        "item": values[index],
        "index": index + 1,
        "done": False,
        "__repeat_node__": True,
    }
```

Do not declare these names as ordinary output ports. They are interpreted by the executor and removed from normal output routing.

Repeat limits are configured by native repeat-capable nodes through `Max Iterations (0 = infinite)`. Python nodes can request repeats, but they do not automatically receive that native Inspector field unless the plugin exposes its own counter/configuration and stops returning `__repeat_node__`.

### Resource registration and cleanup

The runner exposes this helper to plugin modules:

```python
gingerflow_register_resource(resource, scope="node", scope_id=None)
```

Valid scopes are:

- `node`: released after the current Python node finishes.
- `branch`: released when a branch is stopped.
- `workflow`: released when execution ends or the worker shuts down.

Registered resources are closed by calling `flush()` and then `close()` when those methods exist. Cleanup exceptions are intentionally suppressed so one broken resource does not prevent other resources from being released.

```python
def execute(inputs, config):
    import socket

    sock = socket.create_connection((inputs["host"], int(inputs.get("port") or 80)))
    gingerflow_register_resource(sock, "node")
    sock.sendall(b"PING")
    return {"connected": True}
```

Use module-level `gingerflow_cleanup(scope, scope_id, reason)` for custom cleanup logic. It is called for loaded plugin modules during scoped cleanup. Cleanup hooks must be idempotent and must not raise.

### Worker protocol and hidden implementation details

The C++ session writes length-prefixed JSON frames to the worker's stdin:

```text
<decimal byte length>\n
<UTF-8 JSON payload>
```

The worker returns the same framing. Requests contain fields such as:

```json
{
  "id": 1,
  "action": "execute",
  "plugin_dir": "...",
  "entry": "nodes/example.py",
  "function": "execute",
  "inputs": {},
  "config": {},
  "node_id": "example_1",
  "node_type": "example.transform",
  "workflow_id": "__workflow__"
}
```

The runner reserves `stdout` for protocol frames. GingerFlow redirects ordinary plugin stdout to stderr, but plugins should still use `print(..., file=sys.stderr)` for diagnostics. The default maximum frame is 16 MiB; large files must be passed by path or processed in bounded chunks rather than returned as one JSON value.

The runner caches imported modules and callable functions during a workflow-scoped worker session. Do not rely on module import being repeated for every node execution. Store per-execution state in inputs/config or explicitly reset module state at workflow boundaries.

### Cancellation, failure, and retries

- Workflow cancellation is checked while waiting for a Python response.
- Cancellation kills the Python worker process and marks the node as cancelled.
- A positive request timeout kills the worker when the deadline expires.
- A Python traceback is returned as node failure text.
- A failed Python node can be retried by the executor retry policy; design side effects to be idempotent or use an idempotency key.
- A worker killed by timeout or cancellation cannot safely continue serving later requests; the session is considered broken for that run.
- Cleanup is attempted at node, branch, workflow, and shutdown boundaries, but hard process termination can prevent Python cleanup code from running.

### Large files and streaming

Do not pass a complete large file as a Python string, byte array, list, or JSON object. Use a path and process bounded chunks:

```python
def execute(inputs, config):
    source = inputs["path"]
    destination = inputs["output_path"]
    chunk_size = int(config.get("chunk_size", "4194304"))

    with open(source, "rb") as src, open(destination, "wb") as dst:
        while True:
            chunk = src.read(chunk_size)
            if not chunk:
                break
            dst.write(chunk)

    return {"path": destination}
```

For CSV, JSONL, and event data, emit bounded lists and return only the current batch. Avoid accumulating all batches in module globals or in a returned output.

### Security and correctness checklist

- Treat input paths, URLs, commands, and credentials as untrusted configuration.
- Never build shell commands by string concatenation; use argument arrays with `subprocess.run`.
- Never log passwords, tokens, cookies, or connection strings.
- Validate file paths before opening or deleting them.
- Set explicit timeouts in third-party network/database calls even when GingerFlow timeout is `0`.
- Close or register every external resource.
- Make side effects idempotent if retries are enabled.
- Bound memory use and output sizes.
- Return declared, JSON-safe outputs.
- Test cancellation, timeout, malformed inputs, missing dependencies, and repeated execution.

### Minimal production plugin template

```text
event_tools/
├── plugin.json
├── icons.json
├── requirements.txt
├── nodes/
│   ├── __init__.py
│   └── listener.py
└── tests/
    └── test_listener.py
```

`plugin.json`:

```json
{
  "name": "event_tools",
  "version": "1.0.0",
  "author": "Example Team",
  "description": "Event listener nodes.",
  "nodes": [
    {
      "type": "event_tools.listener",
      "display_name": "Event Listener",
      "category": "Stream",
      "entry": "nodes/listener.py",
      "function": "execute",
      "outputs": [{"name": "event", "data_type": "object"}],
      "config": [
        {"key": "endpoint", "type": "String", "default": ""},
        {"key": "poll_seconds", "type": "String", "default": "1"}
      ]
    }
  ]
}
```

`nodes/listener.py`:

```python
import time


def execute(inputs, config):
    endpoint = config.get("endpoint", "").strip()
    poll_seconds = max(0.1, float(config.get("poll_seconds", "1")))
    if not endpoint:
        raise ValueError("endpoint is required")

    # For a real listener, replace this with a bounded poll or receive call.
    # Set __python_timeout_ms to 0 in the Inspector and stop the workflow to cancel.
    time.sleep(poll_seconds)
    return {"event": {"endpoint": endpoint, "received": True}}
```

This template illustrates the contract; it does not create an asynchronous listener. A production listener must define its own reconnect, backoff, cancellation, deduplication, and resource-release behavior.

---

## Central Widget Event Callbacks

GingerFlow also has a UI callback mechanism for making one central widget react to another widget in the same node. This is separate from Python execution: it runs immediately in the canvas UI when a widget event occurs and can update other central-widget values.

### Important plugin boundary

`onUiEvent` is a C++ metadata callback. It cannot be declared as a Python function or serialized in `plugin.json`, because a JSON manifest cannot carry an executable C++ lambda. Use it in built-in nodes or other C++ node providers. Python plugins can still use the supported declarative `actionType` fields described below, because those actions are represented as ordinary manifest metadata.

The implementation surfaces are:

- `NodeWidgetSpec::onUiEvent`
- `NodeWidgetActionContext`
- `NodeWidgetActionContext::updates`
- `NodeWidgetSpec::actionType`
- `NodeWidgetSpec::actionParams`

### Events that trigger callbacks

Callbacks run after the central widget has accepted a user change. Depending on the widget, this includes:

- `entry`: editing finishes.
- `textarea`: text changes.
- `dropdown` and `dropdown_single`: selection changes.
- `dropdown_multi`: checked selection changes.
- `number`: numeric value changes.
- `radio_group`: a radio option becomes selected.
- `slider`: slider value changes.
- `checkbox`: checked state changes.
- `button`: button is clicked; its trigger value is the incremented click count.
- `image_render`: path edits or file selection changes update the widget value, but do not use the generic action callback for every preview refresh.
- `list_builder`, `object_builder`, and `variable_builder`: their value changes update central state; use their owning node behavior for more complex reactions.

Programmatic synchronization uses signal blocking, so a callback update does not recursively trigger the same widget callback.

### C++ callback context

The callback signature is:

```cpp
using NodeWidgetActionCallback = std::function<void(NodeWidgetActionContext&)>;
```

The context contains:

| Field | Meaning |
|---|---|
| `nodeId` | ID of the node on the canvas. |
| `nodeType` | Registered node type, such as `math.interest`. |
| `widgetKey` | Key of the widget that caused the event. |
| `triggerValue` | New value from the triggering widget. |
| `centralValues` | Pointer to a snapshot of current central-widget values. |
| `inputValues` | Pointer to current inline input/default values. |
| `configuration` | Pointer to the node configuration map. |
| `centralWidgetSpecs` | Pointer to all central-widget specifications for the node. |
| `updates` | Output map. Insert target widget keys and values here. |

The pointers are valid only while the callback executes. Copy values you need; do not store the pointers or references for later use.

### C++ example: one widget updates another

```cpp
NodeWidgetSpec mode;
mode.key = "mode";
mode.type = "dropdown_single";
mode.label = "Mode";
mode.defaultValue = "short";
mode.options = {"short", "long"};
mode.onUiEvent = [](NodeWidgetActionContext& ctx) {
  if (ctx.widgetKey != "mode") return;

  if (ctx.triggerValue.toString() == "short") {
    ctx.updates.insert("message", "Short template selected");
  } else {
    ctx.updates.insert("message", "Long template selected with more detail");
  }
};

NodeWidgetSpec message;
message.key = "message";
message.type = "textarea";
message.label = "Message";
message.defaultValue = "";

m.centralWidgets = {mode, message};
```

When the user changes `mode`, GingerFlow writes the `message` update through its runtime widget-sync path. The visible editor, central-value map, undo/dirty integration, and later node serialization remain synchronized.

### C++ example: read several widget values

```cpp
NodeWidgetSpec calculate;
calculate.key = "calculate";
calculate.type = "button";
calculate.label = "Calculate";
calculate.defaultValue = 0;
calculate.onUiEvent = [](NodeWidgetActionContext& ctx) {
  const QVariantMap values = ctx.centralValues ? *ctx.centralValues : QVariantMap{};
  const double principal = values.value("principal").toDouble();
  const double rate = values.value("rate").toDouble();
  const double years = values.value("years").toDouble();
  ctx.updates.insert("interest", QString::number(principal * rate * years / 100.0, 'f', 2));
};
```

Callbacks should be fast and deterministic. Do not perform network requests, file scans, sleeps, or Python execution in a UI callback. Put expensive work in `execute()` or a worker and update the widget from a controlled runtime event.

### Declarative actions for plugin metadata

For metadata-driven nodes, `actionType` and `actionParams` provide common widget interactions without a C++ lambda.

#### `set_widget_value`

Copies a fixed value, or the trigger value when `value` is omitted, into another central widget:

```cpp
NodeWidgetSpec clear;
clear.key = "clear";
clear.type = "button";
clear.label = "Clear";
clear.defaultValue = 0;
clear.actionType = "set_widget_value";
clear.actionParams = {{"targetKey", "message"}, {"value", ""}};
```

#### `set_from_option_map`

Maps the selected option to a target value:

```cpp
mode.actionType = "set_from_option_map";
mode.actionParams = {
  {"targetKey", "message"},
  {"optionValueMap", QVariantMap{
    {"short", "Short template"},
    {"long", "Long template"}
  }}
};
```

The trigger value is converted to a string and used as the map key. If the key is absent, no update occurs.

#### `set_from_template`

Resolves a template using current central-widget values and the trigger value:

```cpp
mode.actionType = "set_from_template";
mode.actionParams = {
  {"targetKey", "message"},
  {"template", "Mode: {mode}, selected: {value}"}
};
```

Use stable widget keys in templates. Missing values resolve to an empty string according to the existing template resolver behavior.

#### `compute_simple_interest`

The built-in convenience action calculates simple interest and writes a formatted string:

```cpp
calculate.actionType = "compute_simple_interest";
calculate.actionParams = {
  {"principalKey", "principal"},
  {"rateKey", "rate"},
  {"timeKey", "years"},
  {"targetKey", "interest"},
  {"precision", 2}
};
```

`precision` is clamped to the supported range by the UI action implementation. Missing or invalid numeric values are treated as zero.

### Update rules and limitations

- `updates` keys must be central-widget keys; unknown keys are stored in central state but have no visible widget to update.
- Updates are applied after the callback returns, in map iteration order.
- Callback updates are runtime UI changes and are not node execution outputs.
- A callback does not call `execute()` automatically.
- A callback does not send values to downstream ports by itself.
- To affect execution, the updated central value must be read by the node during its next `execute()` call or be copied into an input through normal workflow logic.
- Read-only display widgets can still be targets of callback updates.
- Runtime edit locks prevent ordinary editing but do not prevent callback synchronization.
- Keep widget keys stable: changing them breaks saved central-widget values and action targets in existing workflows.

For a complete C++ callback reference, see [central_widget_callbacks.md](central_widget_callbacks.md).


---

## Table of Contents

1. [Overview](#overview)
2. [Plugin ZIP Structure](#plugin-zip-structure)
3. [Manifest: plugin.json](#manifest-pluginjson)
   - [Top-level fields](#top-level-fields)
   - [Node definition fields](#node-definition-fields)
   - [Port definition fields](#port-definition-fields)
   - [Config definition fields](#config-definition-fields)
4. [Writing the Python Node](#writing-the-python-node)
   - [Function signature](#function-signature)
   - [Inputs and config](#inputs-and-config)
   - [Return value](#return-value)
   - [Raising errors](#raising-errors)
   - [Importing helpers](#importing-helpers)
5. [Data Types](#data-types)
6. [Config Widget Types](#config-widget-types)
7. [End-to-End Example: Text Transformer](#end-to-end-example-text-transformer)
8. [End-to-End Example: HTTP Request](#end-to-end-example-http-request)
9. [Installing a Plugin](#installing-a-plugin)
10. [Python Interpreter Setup](#python-interpreter-setup)
11. [Execution Model](#execution-model)
12. [Debugging](#debugging)
13. [Packaging Checklist](#packaging-checklist)

---

## Overview

GingerFlow's plugin system lets you add custom workflow nodes written in **Python 3**. Each plugin is a **ZIP archive** that GingerFlow extracts and loads at runtime. During a workflow run, Python plugin nodes execute through a workflow-scoped long-lived Python worker that receives framed JSON requests and returns JSON responses.

No C++ knowledge, recompilation, or restarts are needed to install a plugin. New nodes appear in the Toolbox immediately after installation.

---

## Plugin ZIP Structure

```
my_plugin.zip
├── plugin.json           ← required manifest
├── my_node.py            ← one or more Python implementation files
├── helper_utils.py       ← optional support modules (importable by your nodes)
└── requirements.txt      ← optional: list pip packages your nodes need
```

Rules:
- The ZIP **must** contain `plugin.json` at its root (not inside a sub-folder).
- All Python files referenced in `plugin.json` must be at paths relative to the ZIP root.
- Sub-directories are allowed for organising helper modules.
- `requirements.txt` is informational — GingerFlow does **not** auto-install pip packages. Install them into your Python environment beforehand.

---

## Manifest: plugin.json

The manifest is a UTF-8 JSON file that describes the plugin and all the nodes it provides.

### Top-level fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | **Yes** | Unique plugin identifier. Used as the directory name after extraction. Use only letters, digits, underscores, hyphens, and dots. |
| `version` | string | No | Semantic version, e.g. `"1.2.0"`. Defaults to `"0.0.0"`. |
| `author` | string | No | Displayed in Plugin Manager. |
| `description` | string | No | Short summary shown in Plugin Manager details. |
| `nodes` | array | **Yes** | List of node definitions (see below). |

### Node definition fields

Each entry in `nodes` defines one node type.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | **Yes** | Globally unique node type ID, e.g. `"myplugin.do_thing"`. Recommended pattern: `<plugin_name>.<node_name>`. |
| `display_name` | string | No | Human-readable name shown in the Toolbox and on the canvas node. Defaults to `type`. |
| `category` | string | No | Toolbox category group, e.g. `"My Plugin"`. Defaults to `"Python Plugins"`. |
| `description` | string | No | Tooltip / detail text for this node. |
| `entry` | string | **Yes** | Path to the Python file, relative to the plugin directory, e.g. `"my_node.py"` or `"nodes/transform.py"`. |
| `function` | string | No | Name of the callable inside the entry module. Defaults to `"execute"`. |
| `inputs` | array | No | List of input port definitions. |
| `outputs` | array | No | List of output port definitions. |
| `config` | array | No | List of inspector configuration fields. |

### Port definition fields

Used in both `inputs` and `outputs` arrays.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | **Yes** | Internal key used to look up the value in `inputs` dict / set in the returned dict. |
| `label` | string | No | Display label on the canvas port. Defaults to `name`. |
| `data_type` | string | No | Type hint shown on the port. See [Data Types](#data-types). Defaults to `"any"`. |
| `required` | boolean | No | If `true`, the canvas highlights this port as mandatory. Defaults to `false`. |

### Config definition fields

Config fields appear in the Properties panel (right-hand Inspector) when the node is selected.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `key` | string | **Yes** | Dictionary key passed in `config` to the Python function. |
| `label` | string | No | Label shown in the Inspector. Defaults to `key`. |
| `type` | string | No | Widget type: `"String"`, `"Password"`, or `"Enum"`. Defaults to `"String"`. |
| `default` | string | No | Default value pre-filled in the widget. |
| `enum_values` | array of strings | No | Required when `type` is `"Enum"`. Populates the dropdown. |

---

## Writing the Python Node

### Function signature

```python
def execute(inputs: dict, config: dict) -> dict:
    ...
```

GingerFlow always calls the function with exactly two positional arguments:

- **`inputs`** — `dict[str, Any]` — values arriving on input ports, keyed by port `name`.
- **`config`** — `dict[str, str]` — values from the Inspector panel, keyed by config `key`. All values are strings; cast as needed.

The function **must return a `dict`** whose keys match the output port `name`s declared in the manifest. Extra keys are ignored. Missing keys result in `None` on the connected downstream port.

### Inputs and config

```python
def execute(inputs: dict, config: dict) -> dict:
    text   = inputs.get("text", "")          # may be None if port not connected
    mode   = config.get("mode", "upper")     # always a str
    repeat = int(config.get("repeat", "1"))  # cast as needed
    ...
```

If an input port is not connected, its value in `inputs` is `None` (not absent from the dict). Always use `.get()` with a sensible default.

### Return value

```python
return {
    "result": transformed_text,
    "count":  len(transformed_text),
}
```

Values can be any JSON-serialisable Python type: `str`, `int`, `float`, `bool`, `list`, `dict`, or `None`. Non-serialisable objects are converted to their `str()` representation automatically.

If your function returns a non-dict value it is wrapped as `{"result": <value>}`. It is better to return a dict explicitly.

### Raising errors

Raise any exception to signal node failure. GingerFlow catches it, formats the full traceback, and reports it as a node execution error in the Output console. The workflow execution stops at the failed node.

```python
def execute(inputs: dict, config: dict) -> dict:
    path = inputs.get("file_path")
    if not path:
        raise ValueError("file_path input is required")
    ...
```

### Importing helpers

Your plugin directory is automatically prepended to `sys.path`, so you can import sibling modules freely:

```
my_plugin/
├── plugin.json
├── transform.py      ← entry file
└── utils.py          ← helper
```

```python
# transform.py
from utils import clean_text   # works without any path manipulation

def execute(inputs, config):
    return {"result": clean_text(inputs.get("text", ""))}
```

Third-party packages (e.g. `requests`, `pandas`) must be installed in the Python environment that GingerFlow uses. See [Python Interpreter Setup](#python-interpreter-setup).

---

## Data Types

These are informational hints displayed on canvas ports. GingerFlow does not enforce types at runtime.

| `data_type` value | Meaning |
|---|---|
| `"any"` | Accepts any value (default) |
| `"string"` | Text |
| `"number"` | Integer or float |
| `"boolean"` | `true` / `false` |
| `"list"` | Python list / JSON array |
| `"object"` | Python dict / JSON object |
| `"path_select"` | File path (opens a file picker on compatible port widgets) |
| `"path_save"` | Save file path |

---

## Config Widget Types

| `type` value | Inspector widget | Notes |
|---|---|---|
| `"String"` | Single-line text field | Default |
| `"Password"` | Masked text field with show/hide toggle | Value passed as plain text to Python |
| `"Enum"` | Drop-down list | Requires `enum_values` array |

---

## End-to-End Example: Text Transformer

This plugin provides one node that transforms text in various ways.

### Directory after extraction

```
text_transformer/
├── plugin.json
└── transform.py
```

### plugin.json

```json
{
  "name": "text_transformer",
  "version": "1.0.0",
  "author": "Jane Doe",
  "description": "Nodes for common text transformations.",
  "nodes": [
    {
      "type": "text_transformer.transform",
      "display_name": "Text Transform",
      "category": "Text Transformer",
      "description": "Converts text to upper-case, lower-case, title-case, or reverses it.",
      "entry": "transform.py",
      "function": "execute",
      "inputs": [
        {
          "name": "text",
          "label": "Input Text",
          "data_type": "string",
          "required": true
        }
      ],
      "outputs": [
        {
          "name": "result",
          "label": "Result",
          "data_type": "string"
        },
        {
          "name": "length",
          "label": "Length",
          "data_type": "number"
        }
      ],
      "config": [
        {
          "key": "mode",
          "label": "Transform Mode",
          "type": "Enum",
          "default": "upper",
          "enum_values": ["upper", "lower", "title", "reverse"]
        },
        {
          "key": "__python_timeout_ms",
          "label": "Execution Timeout (ms, 0 = no timeout)",
          "type": "String",
          "default": "0"
        }
      ]
    }
  ]
}
```

### transform.py

```python
def execute(inputs: dict, config: dict) -> dict:
    text = str(inputs.get("text") or "")
    mode = config.get("mode", "upper")

    if mode == "upper":
        result = text.upper()
    elif mode == "lower":
        result = text.lower()
    elif mode == "title":
        result = text.title()
    elif mode == "reverse":
        result = text[::-1]
    else:
        raise ValueError(f"Unknown mode: {mode!r}")

    return {
        "result": result,
        "length": len(result),
    }
```

### Packaging

```
# On Windows PowerShell:
Compress-Archive -Path text_transformer\* -DestinationPath text_transformer.zip
```

```
# On Linux/macOS:
cd text_transformer && zip -r ../text_transformer.zip .
```

> **Important:** The ZIP must contain the files at its root, not inside an extra parent folder. Verify with `unzip -l text_transformer.zip` — you should see `plugin.json`, not `text_transformer/plugin.json`.

---

## End-to-End Example: HTTP Request

This plugin shows a more realistic use case with a third-party library.

### plugin.json

```json
{
  "name": "http_tools",
  "version": "1.0.0",
  "author": "Your Name",
  "description": "Make HTTP GET and POST requests from your workflow.",
  "nodes": [
    {
      "type": "http_tools.get",
      "display_name": "HTTP GET",
      "category": "HTTP Tools",
      "description": "Performs an HTTP GET request and returns the response body.",
      "entry": "http_nodes.py",
      "function": "http_get",
      "inputs": [
        {"name": "url",     "label": "URL",     "data_type": "string", "required": true},
        {"name": "headers", "label": "Headers", "data_type": "object"}
      ],
      "outputs": [
        {"name": "body",        "label": "Body",        "data_type": "string"},
        {"name": "status_code", "label": "Status Code", "data_type": "number"},
        {"name": "json",        "label": "JSON",        "data_type": "object"}
      ],
      "config": [
        {"key": "timeout",        "label": "Timeout (s)",        "type": "String", "default": "30"},
        {"key": "verify_ssl",     "label": "Verify SSL",         "type": "Enum",   "default": "true", "enum_values": ["true", "false"]},
        {"key": "__python_timeout_ms", "label": "Execution Timeout (ms)", "type": "String", "default": "0"}
      ]
    },
    {
      "type": "http_tools.post",
      "display_name": "HTTP POST",
      "category": "HTTP Tools",
      "description": "Performs an HTTP POST request with a JSON body.",
      "entry": "http_nodes.py",
      "function": "http_post",
      "inputs": [
        {"name": "url",     "label": "URL",  "data_type": "string", "required": true},
        {"name": "payload", "label": "Body", "data_type": "object"}
      ],
      "outputs": [
        {"name": "body",        "label": "Body",        "data_type": "string"},
        {"name": "status_code", "label": "Status Code", "data_type": "number"}
      ],
      "config": [
        {"key": "timeout", "label": "Timeout (s)", "type": "String", "default": "30"},
        {"key": "__python_timeout_ms", "label": "Execution Timeout (ms, 0 = no timeout)", "type": "String", "default": "0"}
      ]
    }
  ]
}
```

### http_nodes.py

```python
import json

try:
    import requests
except ImportError:
    raise RuntimeError(
        "The 'requests' package is not installed. "
        "Run: pip install requests"
    )


def http_get(inputs: dict, config: dict) -> dict:
    url     = inputs.get("url")
    headers = inputs.get("headers") or {}
    timeout = float(config.get("timeout", 30))
    verify  = config.get("verify_ssl", "true") == "true"

    if not url:
        raise ValueError("'url' input is required")

    response = requests.get(url, headers=headers, timeout=timeout, verify=verify)

    parsed_json = None
    try:
        parsed_json = response.json()
    except Exception:
        pass

    return {
        "body":        response.text,
        "status_code": response.status_code,
        "json":        parsed_json,
    }


def http_post(inputs: dict, config: dict) -> dict:
    url     = inputs.get("url")
    payload = inputs.get("payload") or {}
    timeout = float(config.get("timeout", 30))

    if not url:
        raise ValueError("'url' input is required")

    response = requests.post(url, json=payload, timeout=timeout)

    return {
        "body":        response.text,
        "status_code": response.status_code,
    }
```

---

## Installing a Plugin

1. In GingerFlow, open **View → Plugin Manager…**
2. Click **Install Plugin from ZIP…**
3. Browse to your `.zip` file and click **Open**.
4. GingerFlow extracts the ZIP and immediately registers the new nodes.
5. The new nodes appear in the **Toolbox** under the category you specified — no restart needed.

To **uninstall** an external plugin, select it in the Plugin Manager list and click **Uninstall**. The plugin files are deleted from disk. A restart is required for the nodes to disappear from the Toolbox in the current session.

Built-in GingerFlow nodes are shown in the Plugin Manager for reference but cannot be uninstalled.

---

## Python Interpreter Setup

GingerFlow locates Python at runtime in this order:

1. The value of the `GINGERFLOW_PYTHON` environment variable (set this to an absolute path to use a specific interpreter or virtual environment).
2. `python3` on `PATH`.
3. `python` on `PATH`.

### Using a virtual environment

```powershell
# Windows PowerShell
python -m venv C:\venvs\gingerflow
C:\venvs\gingerflow\Scripts\pip install requests pandas  # install your deps

# Then set before launching GingerFlow:
$env:GINGERFLOW_PYTHON = "C:\venvs\gingerflow\Scripts\python.exe"
```

```bash
# Linux / macOS
python3 -m venv ~/venvs/gingerflow
~/venvs/gingerflow/bin/pip install requests pandas

export GINGERFLOW_PYTHON=~/venvs/gingerflow/bin/python3
```

### Verifying which Python is used

Add a diagnostic node to your plugin:

```python
def execute(inputs, config):
    import sys
    return {"python": sys.executable, "version": sys.version}
```

---

## Execution Model

Understanding how your node runs helps you write correct, efficient code.

```
GingerFlow Desktop
│
├─ Workflow runs
│   └─ Node "my_plugin.transform" is reached
│       └─ PythonNode::execute() called (C++ side)
│           ├─ Builds JSON payload:
│           │     { "plugin_dir": "...", "entry": "transform.py",
│           │       "function": "execute",
│           │       "inputs":  { "text": "hello" },
│           │       "config":  { "mode": "upper"  } }
│           ├─ Spawns: python gingerflow_runner.py
│           ├─ Writes payload JSON → stdin
│           ├─ Waits until the node timeout; `0` means no deadline
│           └─ Reads result JSON ← stdout
│               { "outputs": { "result": "HELLO", "length": 5 } }
│
└─ Outputs propagated to downstream nodes
```

Key behaviours:

- **One worker per workflow run.** Python node calls in the same run share one process; workflow boundaries isolate state.
- **Per-request timeout.** If a request times out, GingerFlow terminates the worker session and reports an execution error.
- **stdout is reserved.** Do not write anything to `sys.stdout` directly. Use `sys.stderr` for debug prints; they appear in the GingerFlow Output console under the Errors tab.
- **stdin is consumed.** Do not read from `sys.stdin` in your function.
- **JSON serialisation.** All inputs, config values, and outputs must be JSON-serialisable. Non-serialisable objects are coerced to their `str()` representation via `json.dumps(..., default=str)`.

For resource cleanup semantics (node/branch/workflow cleanup, graceful stop vs forced kill), see `docs/resource_lifecycle_and_cleanup.md`.

---

## Debugging

### Print debug information

Write to `stderr` — it is captured and shown in the **Output** panel:

```python
import sys

def execute(inputs, config):
    print(f"[debug] inputs received: {inputs}", file=sys.stderr)
    ...
```

### Test your function without GingerFlow

Since the function is a plain Python callable, you can unit-test it directly:

```python
# test_transform.py
from transform import execute

def test_upper():
    result = execute({"text": "hello"}, {"mode": "upper"})
    assert result == {"result": "HELLO", "length": 5}

test_upper()
print("All tests passed.")
```

```powershell
python test_transform.py
```

### Simulate the runner protocol manually

```powershell
$payload = @{
    plugin_dir    = "C:\path\to\my_plugin"
    entry         = "transform.py"
    "function"    = "execute"
    inputs        = @{ text = "hello" }
    config        = @{ mode  = "upper" }
} | ConvertTo-Json -Compress

$payload | python "C:\Users\<you>\AppData\Roaming\GingerFlow\GingerFlow\plugins\gingerflow_runner.py"
```

Expected output:
```json
{"outputs": {"result": "HELLO", "length": 5}}
```

### Common errors

| Error message | Likely cause |
|---|---|
| `Python interpreter not found` | Python is not on PATH; set `GINGERFLOW_PYTHON` |
| `Python node timed out` | A positive node timeout expired; use `0` for no GingerFlow deadline |
| `Python node returned invalid JSON` | You printed to stdout — use stderr for debug output |
| `ModuleNotFoundError: No module named 'requests'` | Package not installed in the Python env GingerFlow uses |
| `Cannot open plugin.json` | The manifest is inside a sub-folder in the ZIP; restructure so it is at the root |
| `Entry file not found: nodes/my_node.py` | The `entry` path in plugin.json does not match the actual file path inside the ZIP |

---

## Packaging Checklist

Before distributing your plugin, verify:

- [ ] `plugin.json` is at the **root** of the ZIP (not inside a named folder).
- [ ] Every `entry` file path in the manifest matches the actual path inside the ZIP.
- [ ] Every `type` value follows the `<plugin_name>.<node_name>` pattern and is unique.
- [ ] All outputs returned by your function are JSON-serialisable.
- [ ] Dependencies are documented in `requirements.txt`.
- [ ] `sys.stdout` is never written to directly inside your `execute` functions.
- [ ] The function handles `None` inputs gracefully (unconnected ports pass `None`).
- [ ] A `requirements.txt` lists all pip dependencies your plugin needs.

---

*GingerFlow Plugin System — built-in author: GingerFlow v1.0.0 | External plugins: author and version defined in plugin.json*
