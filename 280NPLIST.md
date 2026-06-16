# 280NPLIST Format Specification

Version 1.0  
Format version string: `280NPLIST;1.0;`

This document specifies the 280NPLIST serialization format used by
Cappuccino's `CPKeyedArchiver` / `CPKeyedUnarchiver`. Files in this
format carry the `.cib` extension when used as Interface Builder
archives.

---

## Overview

280NPLIST is a text-based keyed archive format. It encodes an object
graph as a flat table of objects with forward references by index. The
logical structure mirrors NSKeyedArchiver but uses a compact
token-based text encoding rather than XML plist or binary plist.

The format is entirely ASCII-safe when content is ASCII. String values
are encoded as raw UTF-8 bytes; length fields count bytes, not
characters.

---

## File structure

```
<header>\n<root-value>
```

- Header: the literal string `280NPLIST;1.0;` followed by a newline
  (or immediately by the root value — the parser must accept both).
- Root value: a single encoded value, always a dictionary in practice.

The root dictionary has three mandatory keys:

| Key          | Value                                                 |
|--------------|-------------------------------------------------------|
| `$top`       | Dictionary mapping entry point names to CP$UID refs   |
| `$objects`   | Array; the flat object table                          |
| `$archiver`  | String; always `CPKeyedArchiver`                      |

A fourth optional key `$version` carries the string `100000`.

---

## Type tokens

Values are encoded with a leading type token. There is no separator
between consecutive values or between a token and its payload.

### Dictionary

```
D;<key-value-pairs…>E;
```

A key must always be a `K` token (see below). The value may be any
type. Key-value pairs appear in sequence with no delimiter. `E;`
terminates the dictionary.

### Array

```
A;<values…>E;
```

Elements may be any type. `E;` terminates the array.

### Key

```
K;<length>;<bytes>
```

`<length>` is the decimal byte count of `<bytes>`. Used only as the
key in a dictionary entry. The key name is not followed by a semicolon
after the bytes — the next token begins immediately.

Example: `K;6;CP$UID` encodes the key `CP$UID`.

### String

```
S;<length>;<bytes>
```

`<length>` is the decimal byte count of `<bytes>`. String values may
contain any UTF-8 bytes including semicolons and newlines. There is no
terminating semicolon after the bytes.

Example: `S;15;CPKeyedArchiver` encodes the string `CPKeyedArchiver`.

### Number

```
d;<length>;<digits>
```

`<length>` is the decimal character count of `<digits>`. `<digits>` is
the decimal representation of the number. Integers and floats are both
encoded this way. Negative numbers include the `-` sign in the length
count.

Examples:
- `d;1;2` → integer 2
- `d;2;-1` → integer -1
- `d;7;1048576` → integer 1048576
- `d;2;50` → integer 50 (also used for float 50.0)

### Boolean

```
T
F
```

No length or value suffix. `T` is true, `F` is false.

---

## CP$UID (object reference)

An object reference is encoded as a single-key dictionary:

```
D;K;6;CP$UID;<number>E;
```

where `<number>` is the decimal index into the `$objects` array.

The key is always `CP$UID` (6 bytes). The value is always a `d` number.

Example: `D;K;6;CP$UIDd;1;0E;` references `$objects[0]`, which is
always `$null`.

---

## The $objects array

`$objects[0]` is always the null sentinel, encoded as the string
`$null`:

```
S;5;$null
```

All other entries are either:

1. **Scalar strings** — interned string values referenced by UID
2. **Class descriptors** — dictionaries with `$classname` and `$classes`
3. **Instance dictionaries** — the encoded state of an object

### Class descriptor

```
D;
  K;10;$classname  S;<n>;<ClassName>
  K;8;$classes     A;  S;<n>;<ClassName>  S;<n>;<SuperclassName>  …  E;
E;
```

`$classes` lists the full inheritance chain from the class itself up to
and including `CPObject`. Example for `CPMenu`:

```
D;
  K;10;$classnameS;6;CPMenu
  K;8;$classesA;S;6;CPMenuS;8;CPObjectE;
E;
```

### Instance dictionary

An instance dictionary contains:

- `$class` → CP$UID reference to the class descriptor entry
- One key per encoded instance variable, using the CP archive key
  defined by `encodeWithCoder:` in the Cappuccino source

Example for a `CPMenuItem`:

```
D;
  K;6;$classD;K;6;CP$UIDd;2;55E;
  K;18;CPMenuItemTitleKeyD;K;6;CP$UIDd;3;171E;
  K;19;CPMenuItemTargetKeyD;K;6;CP$UIDd;2;57E;
  K;19;CPMenuItemActionKeyD;K;6;CP$UIDd;3;172E;
  K;20;CPMenuItemSubmenuKeyD;K;6;CP$UIDd;2;57E;
  K;17;CPMenuItemMenuKeyD;K;6;CP$UIDd;2;54E;
  K;38;CPMenuItemKeyEquivalentModifierMaskKeyD;K;6;CP$UIDd;3;173E;
E;
```

---

## String interning

Identical strings share a single entry in `$objects` and are referenced
by CP$UID throughout the archive. This applies to:

- Selector strings (action names)
- Class names
- Menu item titles
- Key names used as values (not as dictionary keys — those are inline)

In the reference CIB, commonly repeated strings such as empty string
(`S;0;`), separator title (`S;0;`), and boolean-like values appear once
and are referenced many times.

The serializer must intern strings. A map from string value to
`$objects` index is maintained during the build pass.

---

## $top dictionary

The entry point for a CIB archive:

```
K;4;$top
D;
  K;18;CPCibObjectDataKey
  D;K;6;CP$UIDd;1;2E;
E;
```

The single entry `CPCibObjectDataKey` references the
`_CPCibObjectData` instance in the object table (index 2 in the
reference archive).

---

## _CPCibObjectData

This is the root object of a CIB archive. It holds all the top-level
collections that `CPCib` reads when loading the archive.

Known keys (from the reference CIB and Cappuccino source):

| Key                               | Type       | Purpose                             |
|-----------------------------------|------------|-------------------------------------|
| `_CPCibObjectDataNamesKeysKey`    | Array      | Names for named objects             |
| `_CPCibObjectDataNamesValuesKey`  | Array      | Objects corresponding to names      |
| `_CPCibObjectDataClassesKeysKey`  | Array      | (class mapping keys, may be empty)  |
| `_CPCibObjectDataClassesValuesKey`| Array      | (class mapping values, may be empty)|
| `_CPCibObjectDataConnectionsKey`  | Array      | All outlet/action connectors        |
| `_CPCibObjectDataFrameworkKey`    | (empty)    | Framework name, usually empty       |
| `_CPCibObjectDataNextOidKey`      | Number     | Next object ID                      |
| `_CPCibObjectDataObjectsKeysKey`  | Array      | Object table keys (oids)            |
| `_CPCibObjectDataObjectsValuesKey`| Array      | Object table values                 |
| `_CPCibObjectDataOidKeysKey`      | Array      | OID → object map keys               |
| `_CPCibObjectDataOidValuesKey`    | Array      | OID → object map values             |
| `_CPCibObjectDataFileOwnerKey`    | CP$UID ref | The File's Owner proxy object       |
| `_CPCibObjectDataVisibleWindowsKey`| Array     | Windows to show on load             |

---

## Connectors

### Outlet connector (`CPCibOutletConnector`)

```
D;
  K;6;$classD;K;6;CP$UID<class-ref>E;
  K;24;_CPCibConnectorSourceKeyD;K;6;CP$UID<src>E;
  K;29;_CPCibConnectorDestinationKeyD;K;6;CP$UID<dst>E;
  K;23;_CPCibConnectorLabelKeyD;K;6;CP$UID<label>E;
E;
```

Label is the outlet property name (e.g. `delegate`, `theWindow`).

### Action connector (`CPCibControlConnector`)

Same structure. Label is the selector string (e.g.
`takeDoubleValueFrom:`).

---

## Geometry encoding

Rects, sizes, and points are encoded as JSON strings:

```
S;56;{"origin":{"x":0,"y":1},"size":{"width":96,"height":21}}
S;22;{{0, 0}, {1920, 1057}}
```

Both formats appear in the reference CIB. The JSON object format
(`{"origin":…,"size":…}`) is used for view frames and bounds. The
compact format (`{{x, y}, {w, h}}`) is used for window and screen
rects. The serializer must match these formats exactly as the runtime
parser expects them.

The Go `encoding/json` package is suitable for producing the JSON
object format. The compact format is a simple `fmt.Sprintf`.

---

## CPFont encoding

```
D;
  K;6;$classD;K;6;CP$UID<class-ref>E;
  K;13;CPFontNameKeyS;<n>;<name>
  K;13;CPFontSizeKeyd;<n>;<size>
  K;15;CPFontIsBoldKey<T|F>
  K;17;CPFontIsItalicKey<T|F>
  K;17;CPFontIsSystemKey<T|F>
  K;19;CPViewThemeClassKeyD;K;6;CP$UID<ref>E;
  K;19;CPViewThemeStateKeyD;K;6;CP$UID<ref>E;
E;
```

System font placeholder name: `_CPFontSystemFacePlaceholder` (28 bytes).

---

## _CPThemeAttribute encoding

Theme attributes appear as inline dictionaries on controls. They encode
per-state values for theme-driven attributes such as `font`,
`line-break-mode`, and `alignment`.

```
D;
  K;6;$classD;K;6;CP$UID<_CPThemeAttribute-class>E;
  K;4;nameD;K;6;CP$UID<attribute-name-string>E;
  K;12;defaultValueD;K;6;CP$UID<value>E;
  K;6;valuesD;K;6;CP$UID<CPDictionary>E;
E;
```

The `values` dictionary is a `CPDictionary` (not a raw dict) with
theme state keys (`normal`, `tableDataView`, etc.) mapped to values.

---

## CPColor encoding

```
D;
  K;6;$classD;K;6;CP$UID<CPColor-class>E;
  K;20;CPColorComponentsKeyD;K;6;CP$UID<array-of-components>E;
  K;19;CPViewThemeClassKeyD;K;6;CP$UID<ref>E;
  K;19;CPViewThemeStateKeyD;K;6;CP$UID<ref>E;
E;
```

Components are an array of four numbers (RGBA, 0.0–1.0) or a named
color string when the XIB uses a catalog color.

---

## CPTrackingArea encoding

Tracking areas appear on interactive controls. Keys:

| Key                               | Type   |
|-----------------------------------|--------|
| `CPTrackinkAreaViewRectKey`       | String (geometry) |
| `CPTrackingAreaOptionsKey`        | Number |
| `CPTrackingAreaOwnerKey`          | CP$UID |
| `CPTrackingAreaUserInfoKey`       | CP$UID (null) |
| `CPTrackingAreaReferencingViewKey`| CP$UID |
| `CPTrackingAreaWindowRect`        | String (geometry) |

Note: `CPTrackinkAreaViewRectKey` has a typo (`Trakin` not `Tracking`)
that is present in the Cappuccino source and must be preserved exactly.

---

## _CPKeyedArchiverValue

A boxed scalar used to encode primitive values (integers, floats) that
cannot be stored directly as dictionary values in the CP keyed archive.

```
D;
  K;6;$classD;K;6;CP$UID<_CPKeyedArchiverValue-class>E;
  K;15;CPValueValueKeyd;<n>;<value>
E;
```

---

## Validation

The output CIB is valid if:

1. The Cappuccino runtime can load it via `CPCib` without error.
2. The resulting UI matches the source XIB visually and functionally.

The reference pair (`MainMenu.xib` / `MainMenu.cib`) in
`toolchain_reference/Resources/` is the ground truth for output
validation. Byte-identical output is not required; functionally
equivalent output is.
