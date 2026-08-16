# Central Widget UI Callbacks for Custom Plugins

This document explains how plugin authors can attach custom UI behavior to central widgets.

## Current implementation status

Central-widget callbacks are implemented in the Qt canvas UI. They update other widgets in the same node immediately, synchronize the visible editor with saved central-widget state, and participate in dirty tracking and undo snapshots. The mechanism is available to built-in/C++ nodes; Python plugins use the declarative action metadata because executable C++ callbacks cannot be serialized in `plugin.json`.

## Why this exists

A node can now define widget-level callback logic directly in metadata, so one widget can update others without hardcoded UI branches.

Examples:
- Button clears a log textarea
- Button computes interest from 3 input widgets and writes result
- Dropdown selection updates a textarea template

## API surface

The callback API is defined in:
- src/core/node.hpp

Key additions:
- NodeWidgetSpec::onUiEvent
- NodeWidgetActionContext
- NodeWidgetActionContext::updates

Callback signature:

```cpp
using NodeWidgetActionCallback = std::function<void(NodeWidgetActionContext&)>;
```

This is a C++ UI mechanism. Python plugins cannot serialize an `onUiEvent` lambda in `plugin.json`, but they can use the declarative `actionType` and `actionParams` fields described below.

## Context fields available in callback

Inside onUiEvent, you receive a NodeWidgetActionContext with:
- nodeId
- nodeType
- widgetKey
- triggerValue
- centralValues (pointer to current central widget values)
- inputValues (pointer to current input default values)
- configuration (pointer to node configuration map)
- centralWidgetSpecs (pointer to central widget spec vector)
- updates (write target widget values here)

Use updates to request UI/state changes:

```cpp
ctx.updates.insert("target_widget_key", newValue);
```

## Minimal example: clear logger textarea

```cpp
{
    .key = "clear_logs",
    .type = "button",
    .label = "Clear",
    .defaultValue = 0,
    .options = QStringList{},
    .readOnly = false,
    .onUiEvent = [](NodeWidgetActionContext& ctx) {
        ctx.updates.insert("message", QString());
    }
}
```

## Example: set interest via button

```cpp
{
    .key = "set_interest",
    .type = "button",
    .label = "Set Interest",
    .defaultValue = 0,
    .onUiEvent = [](NodeWidgetActionContext& ctx) {
        const QVariantMap values = ctx.centralValues ? *ctx.centralValues : QVariantMap{};

        auto toNumber = [](const QVariant& v) {
            bool ok = false;
            const double d = v.toString().toDouble(&ok);
            return ok ? d : 0.0;
        };

        const double p = toNumber(values.value("principal"));
        const double r = toNumber(values.value("rate"));
        const double t = toNumber(values.value("time_years"));
        const double si = (p * r * t) / 100.0;

        ctx.updates.insert("interest", QString::number(si, 'f', 2));
    }
}
```

## Example: dropdown updates textarea text

```cpp
{
    .key = "template_mode",
    .type = "dropdown",
    .label = "Template",
    .defaultValue = "short",
    .options = {"short", "long"},
    .onUiEvent = [](NodeWidgetActionContext& ctx) {
        const QString mode = ctx.triggerValue.toString();
        if (mode == "short") {
            ctx.updates.insert("message", "Short template selected");
        } else if (mode == "long") {
            ctx.updates.insert("message", "Long template selected with additional details");
        }
    }
}
```

## Event timing

Callbacks run on UI events emitted by central widgets (for example button click, dropdown change, checkbox toggle, entry edit finish).

The UI layer applies updates through runtime widget-sync helpers so visible widget content and node central state stay consistent.

Supported central-widget event sources include `entry` editing completion, `textarea` changes, dropdown selection, multi-select item changes, numeric and slider changes, radio selection, checkbox toggles, and button clicks. A button's `triggerValue` is its incremented click count. Programmatic updates block widget signals to avoid recursive callback loops.

## Declarative actions

`NodeWidgetSpec::actionType` provides common interactions without writing a callback.

### `set_widget_value`

```cpp
spec.actionType = "set_widget_value";
spec.actionParams = {{"targetKey", "message"}, {"value", ""}};
```

If `value` is omitted, the triggering value is written to `targetKey`.

### `set_from_option_map`

```cpp
spec.actionType = "set_from_option_map";
spec.actionParams = {
    {"targetKey", "message"},
    {"optionValueMap", QVariantMap{
        {"short", "Short text"},
        {"long", "Long text"}
    }}
};
```

The trigger value is converted to a string and looked up in `optionValueMap`. Missing options do nothing.

### `set_from_template`

```cpp
spec.actionType = "set_from_template";
spec.actionParams = {
    {"targetKey", "message"},
    {"template", "Mode: {mode}, value: {value}"}
};
```

The template is resolved from current central-widget values and the triggering value.

### `compute_simple_interest`

```cpp
spec.actionType = "compute_simple_interest";
spec.actionParams = {
    {"principalKey", "principal"},
    {"rateKey", "rate"},
    {"timeKey", "years"},
    {"targetKey", "interest"},
    {"precision", 2}
};
```

The action calculates `(principal * rate * time) / 100` and writes a formatted string to `targetKey`.

## Runtime and edit-lock notes

- Read-only widgets remain non-editable.
- Runtime edit lock still controls whether widgets are interactable.
- A callback can still write target values through updates, even for read-only display widgets.

## Best practices

- Use capture-free lambdas for onUiEvent.
- Avoid expensive logic in callbacks; keep them UI-fast.
- Guard missing keys with safe defaults.
- Keep widget keys stable and explicit.
- Keep callbacks UI-fast; do not perform blocking I/O, sleeps, or long computation in them.
- Treat `centralValues`, `inputValues`, `configuration`, and `centralWidgetSpecs` as callback-lifetime snapshots/references; do not retain their pointers.
- Use callback updates for UI state, not for downstream execution results.

## Known limitation

onUiEvent is in-process C++ metadata logic. External script plugins need a bridge layer if they cannot compile C++ lambdas directly.
