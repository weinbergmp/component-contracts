# Component Contract Spec

A **component contract** is a single JSON file that is the canonical, platform-agnostic source of truth for a design system component. It fully describes structure, data bindings, prop effects, and interaction states. A transpiler reads it and produces idiomatic code for any target platform (React, SwiftUI, Jetpack Compose, etc.).

---

## File structure

A contract file may define one or more components. Each top-level key is a component name.

```json
{
  "button-primary": {
    "meta":        { ... },
    "markup":      { ... },
    "data":        { ... },
    "events":      { ... },
    "properties":  { ... },
    "breakpoints": { ... },
    "states":      { ... }
  }
}
```

| Section       | Required | Purpose |
|---------------|----------|---------|
| `meta`        | no       | Human-readable metadata and external links (docs, Figma) |
| `markup`      | yes      | The layer tree — structure, layout, and base styles |
| `data`        | no       | The component's external data interface |
| `events`      | no       | Interactions the component emits to its parent |
| `properties`  | no       | Named props and their per-layer effects |
| `breakpoints` | no       | Viewport-size overrides for layout, style, and visibility |
| `states`      | no       | Interaction states and their per-layer effects |

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

### Layer names

Layer names — the keys used in `markup` and `children` (e.g. `surface`, `icon-left`, `label`) — are free-form identifiers chosen by the author. No names are reserved or treated specially by the schema. The only requirement is consistency: whatever name is given to a layer in `markup` must be used identically wherever that layer is referenced in `properties`, `states`, and `breakpoints`.

### Node properties

| Property        | Required | Applies to              | Description |
|-----------------|----------|-------------------------|-------------|
| `data-type`     | yes      | all nodes               | The node type (see below) |
| `src`           | no       | string, image, icon     | Data binding or literal value |
| `visible-if`    | no       | all nodes               | Conditionally renders the node based on a data field (see below) |
| `direction`     | no       | container               | Layout axis: `horizontal` or `vertical` |
| `size`          | no       | all nodes               | Sizing behavior on each axis (see below) |
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

### Sizing

The `size` property declares how a node sizes itself on each axis, independent of its style. It sits alongside `style` on the node (not inside it) because sizing affects layout geometry, not visual appearance.

```json
"surface": {
  "data-type": "container",
  "direction": "horizontal",
  "size": { "width": "fill", "height": "hug" },
  ...
}
```

| Value          | Description |
|----------------|-------------|
| `"hug"`        | Shrink to fit contents. Figma: "Hug". CSS: `width: fit-content`. SwiftUI: default frame behavior. Compose: `wrapContent`. |
| `"fill"`       | Expand to fill the available space in the parent container. Figma: "Fill". CSS: `flex: 1`. SwiftUI: `.frame(maxWidth: .infinity)`. Compose: `fillMaxWidth`. |
| number         | Fixed size in platform units (e.g. `48`). |
| `"$token"`     | Token reference resolving to a fixed size value. |

Both `width` and `height` are optional. Omitting an axis leaves sizing to the platform default.

### Typography

Text style on `string` and `input` nodes is set via the `style` map using either a composite token shorthand or individual properties.

**Composite token (recommended)** — use `font` to reference a named text style that bundles all typography properties. This maps directly to a Figma text style or a CSS class:

```json
"label": {
  "data-type": "string",
  "style": {
    "font": "$text-body-md"
  }
}
```

**Individual properties** — any text property can be set or overridden individually:

| Property          | Description |
|-------------------|-------------|
| `font`            | Composite text style token. Shorthand for all properties below. |
| `font-family`     | Typeface name or token |
| `font-size`       | Size in platform units or token |
| `font-weight`     | Weight value (`400`, `700`, etc.) or token |
| `font-style`      | `normal`, `italic` |
| `line-height`     | Absolute value, multiplier, or token |
| `letter-spacing`  | Tracking value or token |
| `text-align`      | `left`, `center`, `right`, `justify` |
| `text-decoration` | `none`, `underline`, `strikethrough` |
| `text-transform`  | `none`, `uppercase`, `lowercase`, `capitalize` |

**Override precedence** — individual properties override the corresponding value from a composite `font` token, same as the CSS `font` shorthand model:

```json
"style": {
  "font":        "$text-body-md",
  "font-weight": "$font-weight-bold"
}
```

**Figma mapping** — `font: "$text-body-md"` applies Figma text style "Body/MD" to the layer. If individual property overrides are also present, the Figma layer is detached from the text style and the specific property is set directly.

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

## `breakpoints`

Viewport-size overrides applied on top of base styles and active property values. Same shape as `states`: a map of **breakpoint name → layer IDs → overrides**. Any style property, `direction`, `visible`, or `visible-if` can be overridden per breakpoint.

```json
"breakpoints": {
  "sm": {
    "surface":     { "direction": "vertical", "padding": "$spacing-sm" },
    "icon-right":  { "visible": false }
  },
  "lg": {
    "surface":     { "padding": "$spacing-xl" }
  }
}
```

### Named breakpoints

Breakpoints follow Tailwind-style naming conventions. The ranges below are defaults — transpilers may map these to whatever pixel values their platform targets.

| Name  | Typical min-width | Notes |
|-------|-------------------|-------|
| `xs`  | 0px               | Smallest screens; rarely needed explicitly as it is the base |
| `sm`  | 480px             | Large phones, portrait |
| `md`  | 768px             | Tablets, landscape phones |
| `lg`  | 1024px            | Small desktops, landscape tablets |
| `xl`  | 1280px            | Standard desktops |
| `2xl` | 1536px            | Wide/large desktops |

Custom breakpoint names are allowed. Overrides are applied **mobile-first**: the base style applies from `xs` up, and each breakpoint overrides from its min-width upward.

### Platform translation

| Web | SwiftUI | Jetpack Compose |
|-----|---------|-----------------|
| CSS media queries / Tailwind breakpoint prefixes | `@Environment(\.horizontalSizeClass)` / `ViewThatFits` | `WindowSizeClass` adaptive layouts |

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
base style  <  property variant  <  breakpoint  <  state
```

- **Breakpoints** sit above property variants: a `sm` layout change overrides a `size` property's padding, but the component's current interaction state (e.g. `disabled`) always wins over both.
- **States** are the highest priority because they represent immediate user feedback that must be visible regardless of viewport size.
- If two sources at the same level override the same property on the same layer, the rightmost in the stack wins.

Example: `size: large` sets `padding: $spacing-lg` on `surface`. The `sm` breakpoint overrides it to `$spacing-sm`. If the component is also `disabled`, the `disabled` state's `opacity: 0.4` applies on top — targeting a different property, so no conflict. If `disabled` also set `padding`, it would win over the breakpoint value.

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

---

## Examples

### Button (primary)

A horizontal container with optional leading and trailing icons flanking a text label. Demonstrates `visible-if` for optional content, multi-layer prop effects, and platform-detected interaction states.

```json
{
  "button-primary": {
    "meta": {
      "description": "Primary call-to-action button. Use for the single most important action on a screen.",
      "category": "actions",
      "version": "1.0.0",
      "url": "https://design.example.com/components/button-primary",
      "figma": "https://www.figma.com/design/XXXXXXXXXXXX/Design-System?node-id=1-1"
    },
    "markup": {
      "surface": {
        "data-type": "container",
        "direction": "horizontal",
        "accessibility": {
          "role": "button",
          "label": "@data.text",
          "hint": "Activates the primary action"
        },
        "style": {
          "background-color": "$color-brand-active",
          "corner-radius":    "$corner-radius-md",
          "padding":          "$spacing-md",
          "cursor":           "pointer"
        },
        "children": [
          { "icon-left":  { "data-type": "image",  "src": "@data.icon", "visible-if": "@data.icon", "style": {} } },
          { "label":      { "data-type": "string", "src": "@data.text",                              "style": {} } },
          { "icon-right": { "data-type": "image",  "src": "@data.icon", "visible-if": "@data.icon", "style": {} } }
        ]
      }
    },
    "data": {
      "text": { "type": "string",       "required": true  },
      "icon": { "type": "image-source", "required": false }
    },
    "events": {
      "onPress": { "trigger": "surface", "type": "tap" }
    },
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
    },
    "states": {
      "hover":    { "surface": { "background-color": "$color-brand-hover"    } },
      "pressed":  { "surface": { "background-color": "$color-brand-pressed"  } },
      "disabled": { "surface": { "opacity": 0.4, "cursor": "not-allowed"     } },
      "focused":  { "surface": { "outline": "$focus-ring"                    } }
    }
  }
}
```

---

### Text input

A vertical stack of label, input field, and helper text. Demonstrates a nested container structure, the `input` node type, multiple emitted events, and states that affect different layers simultaneously (the `error` state changes the field border and the helper text color in one override).

```json
{
  "input-text": {
    "meta": {
      "description": "Single-line text input with an optional label and helper text.",
      "category": "forms",
      "version": "1.0.0"
    },
    "markup": {
      "root": {
        "data-type": "container",
        "direction": "vertical",
        "style": { "gap": "$spacing-xs" },
        "children": [
          {
            "label": {
              "data-type":  "string",
              "src":        "@data.label",
              "visible-if": "@data.label",
              "style": {
                "font":  "$text-label-md",
                "color": "$color-text-secondary"
              }
            }
          },
          {
            "field": {
              "data-type": "container",
              "direction": "horizontal",
              "accessibility": {
                "role":  "textfield",
                "label": "@data.label"
              },
              "style": {
                "border":           "$border-default",
                "corner-radius":    "$corner-radius-sm",
                "padding":          "$spacing-sm",
                "background-color": "$color-surface"
              },
              "children": [
                {
                  "input": {
                    "data-type": "input",
                    "src":       "@data.value",
                    "style": {
                      "flex":  1,
                      "font":  "$text-body-md",
                      "color": "$color-text-primary"
                    }
                  }
                }
              ]
            }
          },
          {
            "helper": {
              "data-type":  "string",
              "src":        "@data.helperText",
              "visible-if": "@data.helperText",
              "style": {
                "font":  "$text-label-sm",
                "color": "$color-text-secondary"
              }
            }
          }
        ]
      }
    },
    "data": {
      "label":      { "type": "string", "required": false },
      "value":      { "type": "string", "required": false },
      "helperText": { "type": "string", "required": false }
    },
    "events": {
      "onChange": { "trigger": "input", "type": "value-change" },
      "onFocus":  { "trigger": "input", "type": "focus"        },
      "onBlur":   { "trigger": "input", "type": "blur"         }
    },
    "states": {
      "focused":  {
        "field": { "border": "$border-focused", "background-color": "$color-surface-active" }
      },
      "disabled": {
        "field": { "opacity": 0.5 },
        "input": { "opacity": 0.5 }
      },
      "error": {
        "field":  { "border": "$border-error"      },
        "helper": { "color":  "$color-text-error"  }
      }
    }
  }
}
```
