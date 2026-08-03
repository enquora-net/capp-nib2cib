# 280NPLIST Format Specification

This document specifies the 280NPLIST serialization format used by Cappuccino's `CPKeyedArchiver` / `CPKeyedUnarchiver`. Files in this format carry the `.cib` extension.

## Overview

280NPLIST is a text-based keyed archive format. It encodes an object graph as a flat table of objects with forward references by index. The logical structure mirrors NSKeyedArchiver but uses a compact token-based text encoding rather than XML or binary plists.

The format is entirely ASCII-safe when content is ASCII. String values are encoded as raw UTF-8 bytes; length fields count bytes, not characters.

## File Structure

```
<header>\n<root-value>
```

- **Header:** The literal string `280NPLIST;<version>;` followed by a newline (or immediately by the root value). v1.0 used `1.0`. The new blind transpiler outputs `2.0`.
- **Root value:** A single encoded dictionary.

The root dictionary has three mandatory keys:

| Key          | Value                                                 |
|--------------|-------------------------------------------------------|
| `$top`       | Dictionary mapping entry point names to CP$UID refs   |
| `$objects`   | Array; the flat object table                          |
| `$archiver`  | String; always `CPKeyedArchiver`                      |

A fourth optional key `$version` carries a version integer (e.g., `100000` for v1.0).

## Wire Format (Type Tokens)

Values are encoded with a leading type token. There is no separator between consecutive values or between a token and its payload.

### Dictionary
`D;<key-value-pairs…>E;`
A key must always be a `K` token. The value may be any type. Key-value pairs appear in sequence with no delimiter. `E;` terminates the dictionary.

### Array
`A;<values…>E;`
Elements may be any type. `E;` terminates the array.

### Key
`K;<length>;<bytes>`
`<length>` is the decimal byte count of `<bytes>`. Used only as the key in a dictionary entry. The next token begins immediately after the bytes.

### String
`S;<length>;<bytes>`
`<length>` is the decimal byte count. String values may contain any UTF-8 bytes. There is no terminating semicolon after the bytes.

### Number
`d;<length>;<digits>`
`<length>` is the decimal character count. Integers and floats are both encoded this way. Negative numbers include the `-` sign in the length count.

### Boolean
`T` (true) or `F` (false). No length or value suffix.

## CP$UID (Object Reference)

An object reference is encoded as a single-key dictionary:
`D;K;6;CP$UID;<number>E;`
where `<number>` is the decimal index into the `$objects` array.

Example: `D;K;6;CP$UIDd;1;0E;` references `$objects[0]`.

## The $objects Array

`$objects[0]` is always the null sentinel, encoded as the string `$null` (`S;5;$null`).

All other entries are:
1. **Scalar strings** — interned string values referenced by UID.
2. **Class descriptors** — dictionaries with `$classname` and `$classes`.
3. **Instance dictionaries** — the encoded state of an object.

### Class Descriptor
```
D;
  K;10;$classname  S;<n>;<ClassName>
  K;8;$classes     A;  S;<n>;<ClassName>  S;<n>;<SuperclassName>  …  E;
E;
```

### Instance Dictionary
Contains a `$class` key pointing to a CP$UID reference of a class descriptor, followed by one key per encoded instance variable.

## String Interning

Identical strings share a single entry in `$objects` and are referenced by CP$UID throughout the archive. This applies to selector strings, class names, and key names used as values.

**v2.0 Compiler Requirement:** The Lisette serializer must intern strings. A map from string value to `$objects` index must be maintained during the build pass to ensure compact output.

---

# v2.0 Blind Transpiler Schema

The new Lisette compiler acts as a strict 1:1 mirror of the input XML structure. It possesses zero semantic knowledge of AppKit. The following rules define how XML concepts map to the 280NPLIST wire format:

1. **Root Entry (`$top`):** The `$top` dictionary contains a single key (e.g., `root`) pointing to the `CP$UID` of the root `<document>` element. The compiler does not fabricate a `_CPCibObjectData` shell.
2. **Class Names:** The `$classname` and `$classes` array are populated verbatim with the lowercase XIB tag name (e.g., `$classname = "menu"`, `$classname = "document"`). The runtime's `mapNativeToCapp:` is responsible for mapping `"menu"` to `[CPMenu class]`.
3. **Attribute Typing:** All XML attribute values are serialized as `S` (String) tokens. The compiler performs no lexical type inference (e.g., it does not emit `d` for numbers or `T`/`F` for booleans). The runtime is responsible for parsing strings into integers, floats, or booleans as needed.
4. **Object Topology:** Containment references (an `id`-bearing element inside another element) are encoded as standard `CP$UID` references. The local key used in the instance dictionary is either the child's `key` attribute or the wrapper element's tag name.

---

# Legacy v1.0 Schema Reference

*Note: The following structures are produced by the legacy semantic `nib2cib` tool. The new v2.0 blind transpiler will NOT generate these exact structures. This section is preserved strictly as a reference for the Cappuccino runtime team to understand v1.0 backward compatibility.*

## v1.0 $top Dictionary
The entry point for a v1.0 CIB archive:
```
K;4;$top
D;
  K;18;CPCibObjectDataKey
  D;K;6;CP$UIDd;1;2E;
E;
```
References the `_CPCibObjectData` instance in the object table.

## v1.0 _CPCibObjectData
The root object of a v1.0 archive. Holds top-level collections read by `CPCib`.
Keys include: `_CPCibObjectDataNamesKeysKey`, `_CPCibObjectDataConnectionsKey`, `_CPCibObjectDataObjectsKeysKey`, `_CPCibObjectDataFileOwnerKey`, `_CPCibObjectDataVisibleWindowsKey`, etc.

## v1.0 Connectors
### Outlet connector (`CPCibOutletConnector`)
Keys: `_CPCibConnectorSourceKey`, `_CPCibConnectorDestinationKey`, `_CPCibConnectorLabelKey`.
### Action connector (`CPCibControlConnector`)
Same structure. Label is the selector string.

## v1.0 Specific Encodings
- **Geometry:** Encoded as JSON strings (`{"origin":...}`) or compact strings (`{{x,y},{w,h}}`).
- **CPFont:** Encoded via `CPFontNameKey`, `CPFontSizeKey`, etc.
- **CPColor:** Encoded via `CPColorComponentsKey` (Array of RGBA numbers).
- **_CPKeyedArchiverValue:** A boxed scalar used to encode primitive values that cannot be stored directly.
- **Typo Preservation:** `CPTrackinkAreaViewRectKey` has a typo present in the Cappuccino source that must be preserved by the runtime.

## Validation
The reference pair (`MainMenu.xib` / `MainMenu.cib`) in `toolchain_reference/Resources/` is the ground truth for v1.0 output validation. For v2.0, output is valid if the Cappuccino 2.0 runtime can load it via `CPCib` without error and apply `mapNativeToCapp:` successfully.
