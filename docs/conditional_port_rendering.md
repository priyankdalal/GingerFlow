# Conditional Port Rendering Implementation

## Overview
Implemented conditional rendering for input and output ports in NodeItem. When no input ports are defined, the input column is not rendered. Similarly, when no output ports are defined, the output column is not rendered. When neither are defined, the node displays only the header.

## Current platform context

This rendering behavior applies to built-in nodes, Python plugin nodes, Stream nodes, and Custom grouped nodes because all of them expose the same `NodeMetadata` port contract. Custom nodes derive their visible boundary ports from the selected subflow; when a saved custom definition is unavailable, its inner nodes can be expanded and rendered individually.

## Changes Made

### 1. Layout Computation (node_item.cpp - computeLayout())
- **Port height calculation**: Modified to return 0 if no ports exist
  - `portsHeight` now only includes minimum height if there are actual ports
  - `minWithoutCentral` adjusted to not add padding when there are no ports

- **Column width allocation**: Dynamic based on port existence
  - **Both inputs and outputs**: Split space equally with column gap
  - **Only inputs**: Use full width for input column
  - **Only outputs**: Use full width for output column
  - **No ports**: No space allocated, empty rectangles

- **Central widget positioning**: Fixed to work when no ports exist
  - When ports exist: Central widget positioned below ports with spacing
  - When no ports: Central widget positioned directly after header

### 2. Paint Method (node_item.cpp - paint())
- **Input label rendering**: Skip if input column is empty
  - Check `if (m_inputsRect.isEmpty()) break;`
- **Output label rendering**: Skip if output column is empty
  - Check `if (m_outputsRect.isEmpty()) break;`
- **Separator line**: Only draw if there are ports AND central widgets
  - Check `if (hasCentralWidgets() && !m_centralCollapsed && !m_portsRect.isEmpty())`

### 3. Port Handles (node_item.cpp - buildPortHandles())
- Handles are only created if corresponding specs exist
- Input spec loop only executes if `m_inputSpecs` is not empty
- Output spec loop only executes if `m_outputSpecs` is not empty
- `__enabled__` handle is always created (special control)

## Test Cases Added (test_node_shell.cpp)

### 1. `outputsOnlyNoInputColumn()`
- Creates node with only output ports
- Verifies output port center is accessible
- Verifies no input column is rendered

### 2. `inputsOnlyNoOutputColumn()`
- Creates node with only input ports
- Verifies input port center is accessible
- Verifies no output column is rendered

### 3. `noPortsHeaderOnly()`
- Creates node with no ports and no central widgets
- Verifies node still has minimum size (header-only)

## Rendering Behavior

### Node with Both Ports
```
┌─────────────────────────┐
│ Header (30px)           │
├──────────┬──────────────┤
│ Inputs   │   Outputs    │
│ (split)  │   (split)    │
├──────────┴──────────────┤
│  [Central Widget Area]  │
└─────────────────────────┘
```

### Node with Only Inputs
```
┌─────────────────────────┐
│ Header (30px)           │
├─────────────────────────┤
│ Inputs (full width)     │
├─────────────────────────┤
│  [Central Widget Area]  │
└─────────────────────────┘
```

### Node with Only Outputs
```
┌─────────────────────────┐
│ Header (30px)           │
├─────────────────────────┤
│ Outputs (full width)    │
├─────────────────────────┤
│  [Central Widget Area]  │
└─────────────────────────┘
```

### Node with No Ports (Header-Only)
```
┌─────────────────────────┐
│ Header (30px)           │
├─────────────────────────┤
│  [Central Widget Area]  │
│      OR Empty           │
└─────────────────────────┘
```

## Configuration Examples

### Outputs-Only Node
```cpp
NodeMetadata meta;
meta.type = "processor";
meta.displayName = "Data Processor";
meta.outputs = {
    PortSpec {.name = "result", .label = "Result", ...}
};
```

### Inputs-Only Node
```cpp
NodeMetadata meta;
meta.type = "trigger";
meta.displayName = "Event Trigger";
meta.inputs = {
    PortSpec {.name = "event", .label = "Event", ...}
};
```

### Header-Only Node (Info Display)
```cpp
NodeMetadata meta;
meta.type = "info";
meta.displayName = "Status Display";
// No inputs, outputs, or central widgets
```

## Implementation Notes

- Empty rect checks use `isEmpty()` method which returns true for 0-width/0-height rectangles
- Port handles are never created if specs are empty (natural loop behavior)
- Labels are skipped during paint but row rects are still calculated (non-critical)
- Central widget spacing automatically adjusts based on port existence
- Minimum node dimensions (280x140) are still enforced

## Backward Compatibility

✅ All existing nodes with both input and output ports work unchanged
✅ Layout and rendering priorities remain the same
✅ Signal/slot connections for ports work only if ports exist
✅ Column width distribution is seamless for all configurations
