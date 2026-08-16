# Component Contract Spec

A **component contract** is a single JSON file that is the canonical, platform-agnostic source of truth for a design system component. It fully describes structure, data bindings, prop effects, and interaction states. A transpiler reads it and produces idiomatic code for any target platform (React, SwiftUI, Jetpack Compose, etc.).

---

## File structure

A contract file may define one or more components. Each top-level key is a component name.

```json
{
  "button-primary": {
    "meta":       { ... },
    "markup":     { ... },
    "data":       { ... },
    "events":     { ... },
    "properties": { ... },
    "states":     { ... }
  }
}
```

| Section      | Required | Purpose |
|--------------|----------|---------|
| `meta`       | no       | Human-readable metadata and external links (docs, Figma) |
| `markup`     | yes      | The layer tree — structure, layout, and base styles |
| `data`       | no       | The component's external data interface |
| `events`     | no       | Interactions the component emits to its parent |
| `properties` | no       | Named props and their per-layer effects |
| `states`     | no       | Interaction states and their per-layer effects |

---

## `meta`

Optional metadata about the component. Useful for documentation generators, design system registries, and tooling that needs to cross-reference the spec with design files or a component library site.

```json
"meta": {
  "description": "Primary call-to-action button. Use for the single most important action on a screen.",
  "category":    "actions",
  "version":     "1.0.0",
  "url":         "https://design.example.com/components/button-primary",
  "figma":       "https://www.figma.com/design/XXXXXXXXXXXX/Design-System?node-id=1-1"
}
```

| Field         | Type   | Description |
|---------------|--------|-------------|
| `description` | string | Human-readable summary of the component's purpose and usage guidance |
| `category`    | string | Organizational category (e.g. `actions`, `navigation`, `feedback`) |
| `version`     | string | Semantic version of this contract |
| `url`         | URI    | Link to the component in a documentation site or Storybook |
| `figma`       | URI    | Direct link to the Figma component or node |

---

## `markup`

The markup section contains a single named root node. Every node in the tree is a named key whose value is a node object. Children are an ordered array of single-key objects so that layer order is preserved and layer names are readable.

```json
"markup": {
  "surface": {
    "data-type": "container",
    "direction": "horizontal",
    "style": { ... },
    "children": [
      { "icon-left":  { "data-type": "image",  "src": "@data.icon" } },
      { "label":      { "data-type": "string", "src": "@data.text" } },
      { "icon-right": { "data-type": "image",  "src": "@data.icon" } }
    ]
  }
}
```

### Node properties

| Property        | Required | Applies to              | Description |
|-----------------|----------|-------------------------|-------------|
| `data-type`     | yes      | all nodes               | The node type (see below) |
| `src`           | no       | string, image, icon     | Data binding or literal value |
| `visible-if`    | no       | all nodes               | Conditionally renders the node based on a data field (see below) |
| `direction`     | no       | container               | Layout axis: `horizontal` or `vertical` |
| `accessibility` | no       | all nodes               | Semantic role, label, and hint (see below) |
| `style`         | no       | all nodes               | Base style properties |
| `children`      | no       | container, scroll, list | Ordered child nodes |
| `visible`       | no       | all nodes               | Initial visibility (default `true`) |

### Node types

| Type        | Platform equivalents | Description |
|-------------|----------------------|-------------|
| `container` | `div` / `VStack` / `HStack` / `Column` / `Row` | Layout wrapper. Use `direction` to set axis. |
| `string`    | `span` / `Text` / `Text` | Text content. Binds to a string data field. |
| `image`     | `img` / `Image` / `Image` | Raster or vector image. Binds to an image-source field. |
| `icon`      | `svg` / `Image(systemName:)` / `Icon` | Icon glyph. Binds to an icon name or image-source field. |
| `input`     | `input` / `TextField` / `TextField` | Text input. |
| `scroll`    | `div[overflow]` / `ScrollView` / `LazyColumn` | Scrollable container. |
| `list`      | mapped list / `ForEach` / `LazyColumn` | Repeating container bound to an array data field. |

### `visible-if`

Conditionally renders a node based on whether a data field is present and truthy. The value must be an `@data.<field>` binding.

```json
{ "icon-left": { "data-type": "image", "src": "@data.icon", "visible-if": "@data.icon" } }
```

The node is rendered only when the referenced data field exists and is non-null, non-empty, and non-false. This is distinct from the `visible` property (which is a static default) and from `properties`/`states` overrides (which are controlled externally). `visible-if` is purely driven by the incoming data at render time.

### `accessibility`

Declares the semantic role, accessible name, and interaction hint for a node. Transpilers use these to generate platform-appropriate attributes (`role`/`aria-label` in HTML, `.accessibilityLabel()` in SwiftUI, `contentDescription` in Compose).

```json
"accessibility": {
  "role":  "button",
  "label": "@data.text",
  "hint":  "Activates the primary action"
}
```

| Field   | Description |
|---------|-------------|
| `role`  | Semantic role. One of: `button`, `link`, `heading`, `image`, `text`, `textfield`, `checkbox`, `radio`, `switch`, `progressbar`, `list`, `listitem`, `none` |
| `label` | Accessible name. Accepts a literal string or an `@data.<field>` binding. |
| `hint`  | Short description of what happens when the user interacts with this element. |

---

## Token references

Style values may reference design tokens using the `$token-name` syntax. Tokens are resolved at transpile time from a separate token file.

```json
"style": {
  "background-color": "$color-brand-active",
  "padding":          "$spacing-md",
  "corner-radius":    "$corner-radius-md"
}
```

Any style property value that begins with `$` is treated as a token reference. Bare values are used as-is.

---

## Data binding (`src`)

The `src` property on a node wires it to external data. Bindings use the `@data.<field>` prefix, where `<field>` matches a key declared in the `data` section.

```json
{ "label": { "data-type": "string", "src": "@data.text" } }
```

Literal values (not prefixed with `@data.`) are used as-is:

```json
{ "badge": { "data-type": "string", "src": "New" } }
```

---

## `data`

Declares all external data the component accepts. Every `@data.<field>` binding in markup must resolve to a key here.

```json
"data": {
  "text": { "type": "string",       "required": true  },
  "icon": { "type": "image-source", "required": false }
}
```

### Data types

| Type           | Description |
|----------------|-------------|
| `string`       | Plain text |
| `number`       | Numeric value |
| `boolean`      | True/false flag |
| `image-source` | A URL, asset reference, or base64 image |
| `array`        | List of items (used with `list` nodes) |
| `object`       | Structured data |

---

## `events`

Declares the interactions the component emits to its parent. Each event names the layer that captures the interaction and the type of interaction that fires it. Transpilers use this to generate the correct callback props (`onClick`, `onPress`, `onChange`, etc.).

```json
"events": {
  "onPress":  { "trigger": "surface", "type": "tap"          },
  "onChange": { "trigger": "input",   "type": "value-change" }
}
```

| Field     | Description |
|-----------|-------------|
| `trigger` | The layer ID that captures this interaction. Must match a layer name in `markup`. |
| `type`    | The interaction type. One of: `tap`, `long-press`, `value-change`, `focus`, `blur`, `submit` |

---

## `properties`

Named props the component exposes. Each prop has a `default` value and a `values` map. Each value maps **layer IDs → overrides**, making it explicit exactly what changes for every prop variant.

```json
"properties": {
  "size": {
    "default": "medium",
    "values": {
      "small":  { "surface": { "padding": "$spacing-sm" } },
      "medium": { "surface": { "padding": "$spacing-md" } },
      "large":  { "surface": { "padding": "$spacing-lg" } }
    }
  },
  "icon-position": {
    "default": "left",
    "values": {
      "left":  { "icon-left": { "visible": true  }, "icon-right": { "visible": false } },
      "right": { "icon-left": { "visible": false }, "icon-right": { "visible": true  } },
      "none":  { "icon-left": { "visible": false }, "icon-right": { "visible": false } }
    }
  }
}
```

Override objects support any style property plus the non-style behavioral keys `visible`, `opacity`, and `src`.

---

## `states`

Interaction states applied on top of the resolved base + property values. Same shape as property values: a map of **layer IDs → overrides**.

```json
"states": {
  "hover":    { "surface": { "background-color": "$color-brand-hover"   } },
  "pressed":  { "surface": { "background-color": "$color-brand-pressed" } },
  "disabled": { "surface": { "opacity": 0.4, "cursor": "not-allowed"   } },
  "focused":  { "surface": { "outline": "$focus-ring"                   } }
}
```

### Built-in states

`hover`, `pressed`, `focused`, `disabled`, `selected`, `loading`, `error`

Additional custom states are allowed.

### States vs. properties

`states` and `properties` are intentionally separate sections, even though design tools like Figma model both as variant properties on a component. The distinction matters for code generation:

| | `properties` | `states` |
|---|---|---|
| Set by | Parent (passed in as a prop) | Platform (detected at runtime) |
| Transpiles to | Function parameter / component prop | Event handler, gesture modifier, or CSS pseudo-class |
| Example | `<Button size="large" />` | `:hover`, `.onHover {}`, `Indication` |

Some states (`disabled`, `selected`, `loading`) can feel prop-like since a parent often controls them. They still live in `states` because they transpile to platform-idiomatic patterns (`disabled` → `.disabled(true)` in SwiftUI, `enabled = false` in Compose) rather than plain parameters.

**For Figma handoff:** states correspond directly to Figma's variant property values. A `State` property in Figma with values `Default / Hover / Pressed / Disabled` maps cleanly to the `states` section here. The spec is the source of truth; the Figma component reflects it.

---

## Override priority

Overrides are applied in this order, with later layers winning:

```
base style  <  property variant  <  state
```

Example: if `size: small` sets `padding: $spacing-sm` on `surface`, and the component is also in a `disabled` state, both overrides apply independently — they target different properties and do not conflict. If two sources override the same property on the same layer, the rightmost in the stack wins.

---

## Validation

Contracts can be validated against `schema.json` (JSON Schema draft-07).

```bash
npx ajv validate -s schema.json -d my-component.json
```

---

## Transpiler contract

A transpiler receives a resolved contract (tokens substituted, prop defaults applied) and must:

1. Walk the markup tree depth-first.
2. Map each `data-type` to the platform's primitive (`container` + `horizontal` → `HStack`, etc.).
3. Resolve `@data.<field>` bindings to the platform's prop/parameter passing idiom.
4. Generate conditional modifiers for each prop and state, in priority order.

The spec intentionally does not prescribe how a transpiler handles idioms beyond the `direction` hint on containers. Transpilers may use additional heuristics (e.g. `list` + `array` binding → `LazyColumn`) or AI-assisted generation for edge cases.
