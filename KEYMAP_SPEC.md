# Keymap YAML specification

This page documents the YAML-format keymap representation that is output by `keymap parse` and used by `keymap draw`.

At the root, three key values are expected, which are detailed in respective sections. A typical keymap will have the following structure:

```yaml
layout:      # physical layout specs, optional
  ...
layers:      # ordered mapping of layer name to contents
  layer_1:   # list of (lists of) key specs
    - [Q, W, ...]
    ...
  layer_2:
    ...
combos:      # list of combo specs, optional
  - ...
draw_config: # config overrides for drawing, optional
  - ...
```

## `layout`

This field provides information about the physical layout of the keyboard, i.e., the location and sizes of individual keys.
Following physical layout parameters ([mentioned in the README](README.md#producing-the-svg)) can be specified either in the command line or under this field definition as key-value pairs:

- **`qmk_keyboard`** (equivalent to `-k`/`--qmk-keyboard` on the command line):
  Specifies the path of the keyboard to retrieve from, in the QMK repository
- **`qmk_layout`** (equivalent to `-l`/`--qmk-layout` on the command line):
  Specifies the layout macro to be used for the QMK keyboard, defaults to first one specified if not used
- **`ortho_layout`** (equivalent to `-o`/`--ortho-layout` on the command line):
  Specifies a mapping of parameters to values to generate an ortholinear physical layout, with schema:

  | field name   | type                     | default value | description                                                                                              |
  | ------------ | ------------------------ | ------------- | -------------------------------------------------------------------------------------------------------- |
  | `split`      | `bool`                   | `False`       | whether the layout is a split keyboard or not, affects a few other options below                         |
  | `rows`       | `int`                    | required      | how many rows are in the keyboard, excluding the thumb row if split                                      |
  | `columns`    | `int`                    | required      | how many columns are in the keyboard, only applies to one half if split                                  |
  | `thumbs`     | `int \| "MIT" \| "2x2u"` | `0`           | the number thumb keys per half if split; for non-splits can only take special values `MIT` or `2x2u`[^1] |
  | `drop_pinky` | `bool`                   | `False`       | whether the pinky (outermost) columns have one fewer key, N/A for non-splits                             |
  | `drop_inner` | `bool`                   | `False`       | whether the inner index (innermost) columns have one fewer key, N/A for non-splits                       |

[^1]: Corresponding to bottom row arrangements of a single `2u` key, or two neighboring `2u` keys, respectively.

> **Note**
>
> If these parameters are specified in both command line and under the `layout` section, the former will take precedence.

## `layers`

This field is an ordered mapping of layer names to a list of `LayoutKey` specs that represent the keys on that layer.
A `LayoutKey` can be defined with either a string value or with a mapping with the following fields:

| field name (alias) | type  | default value | description                                                                                                                                                                                                                    |
| ------------------ | ----- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `tap` (`t`)        | `str` | `""`          | the tap action of a key, drawn on the center of the key; spaces will be converted to line breaks[^2]                                                                                                                           |
| `hold` (`h`)       | `str` | `""`          | the hold action of a key, drawn on the bottom of the key                                                                                                                                                                       |
| `shifted` (`s`)    | `str` | `""`          | the "shifted" action of a key, drawn on the top of the key                                                                                                                                                                     |
| `type`             | `str` | `"main"`[^3]  | the styling of the key that corresponds to the [SVG class](keymap_drawer/config.py#L62). predefined types are `held` (a red shading to denote held down keys) and `ghost` (a gray shading to denote optional keys in a layout) |

[^2]: You can prevent line breaks by using double spaces `"  "` to denote a single non-breaking space.
[^3]: Equivalent to default styling but can be overridden in the SVG config if desired.

Using a string value such as `"A"` for a key spec is equivalent to defining a mapping with only the tap field, i.e., `{tap: "A"}`.
It is meant to be used as a shortcut for keys that do not need `hold` or `type` fields.

`layers` field also flattens any lists that are contained in its value: This allows you to semantically divide keys to "rows," if you prefer to do so.
The two layers in the following example are functionally identical:

```yaml
layers:
  flat_layer: ["7", "8", "9", "4", "5", "6", "1", "2", "3", { t: "0", h: Fn }]
  nested_layer:
    - ["7", "8", "9"]
    - ["4", "5", "6"]
    - ["1", "2", "3"]
    - { t: "0", h: Fn }
```

## `combos`

This is an optional field that contains a list of combo specs, each of which is a mapping that can have the following fields:

| field name (alias)    | type                                              | default value | description                                                                                                                                                                       |
| --------------------- | ------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `key_positions` (`p`) | `list[int]`                                       | required      | list of key indices that trigger the combo[^4]                                                                                                                                    |
| `key` (`k`)           | `LayoutKey`[^5]                                   | required      | key produced by the combo when triggered, `type` field will be ignored                                                                                                            |
| `layers` (`l`)        | `list[str]`                                       | `[]`[^6]      | list of layers the combo can trigger on, specified using layer names in `layers` field                                                                                            |
| `align` (`a`)         | `"mid" \| "top" \| "bottom" \| "left" \| "right"` | `"mid"`       | where to draw the combo: `mid` draws on the mid-point of triggering keys' center coordinates, or to the `top`/`bottom`/`left`/`right` of the triggering keys                      |
| `offset` (`o`)        | `float`                                           | `0.0`         | additional offset to `top`/`bottom`/`left`/`right` positioning, specified in units of key width/height: useful for combos that would otherwise overlap                            |
| `dendron` (`d`)       | `null \| bool`                                    | `null`        | whether to draw dendrons going from combo to triggering key coordinates, default is to draw for non-`mid` alignments and draw for `mid` if key coordinates are far from the combo |
| `type`                | `str`                                             | `"combo"`     | the styling of the key that corresponds to the [SVG class](keymap_drawer/config.py#L62), see `LayoutKey` definition above                                                         |

[^4]: Key indices start from `0` on the first key position and increase by columns and then rows, corresponding to their ordering in the `layers` field. This matches the `key-positions` property in ZMK combo definitions.
[^5]: Just like for keys in a layer under the `layers` field, `key` field can be specified with a string value as a shortcut, or a mapping (where the `type` field will be ignored).
[^6]: The default value of empty list corresponds to all layers in the keymap, similar to the `layers` property in ZMK.

## `draw_config`

This optional field lets you override [config parameters](README.md#customization) for SVG drawing.
This way you can specify drawing configuration for a specific layout and store in the keymap specification.
It is a mapping from field names in [`DrawConfig` class](keymap_drawer/config.py) to values.
