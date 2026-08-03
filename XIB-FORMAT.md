# XIB Format Reference

This document describes the XIB XML elements relevant to `capp-nib2cib`. XIB is Apple's Interface Builder file format version 3 (`com.apple.InterfaceBuilder3.Cocoa.XIB`). It is the input format for the compiler.

Only the macOS Cocoa dialect is supported. CocoaTouch XIBs are detected and rejected.

---

## v2.0 Compiler Handling

The new Lisette compiler acts as a **blind structural transpiler**. It does not perform the semantic mappings documented in legacy references (e.g., coordinate flipping, bitfield calculations, or `NS` to `CP` class/key mapping). 

When parsing the XIB, the compiler:
1. **Extracts Elements:** Treats every XML element with an `id` attribute as a distinct object to be flattened into the output object table.
2. **Preserves Topology:** Encodes containment (parent-child relationships) using the child's `key` attribute or the wrapper element's tag name (e.g., `subviews`, `items`).
3. **Resolves References:** Recognizes `target` and `destination` attributes as cross-references to other `id`s, emitting standard `CP$UID` references.
4. **Ignores Metadata:** Silently drops elements and attributes used only for Interface Builder layout or autolayout.

---

## Top-Level Structure

```xml
<document type="com.apple.InterfaceBuilder3.Cocoa.XIB" version="3.0" …>
    <dependencies> … </dependencies>
    <objects> … </objects>
</document>
```

All objects of interest are children of `<objects>`. The `<document>` element itself becomes the root object in the flattened graph.

## Object Identity

Every object element carries an `id` attribute. IDs are strings, typically small integers assigned by Interface Builder. Negative IDs are reserved for standard proxy objects (e.g., `-1` for First Responder, `-2` for File's Owner, `-3` for Application).

Connections reference objects by these IDs via `destination` and `target` attributes.

---

## Elements Ignored by the Compiler

The following XIB elements and attributes carry IB metadata with no semantic value to the blind transpiler and are silently ignored:

- `<point key="canvasLocation" …/>`
- `<capability …/>`
- `<plugIn …/>`
- `<deployment …/>`
- `<constraints>` (Autolayout is not supported; fixed frames are used)
- `translatesAutoresizingMaskIntoConstraints` attribute
- `fixedFrame` attribute
- `verticalHuggingPriority`, `horizontalHuggingPriority` attributes
- `allowsToolTipsWhenApplicationIsInactive` attribute
- `animationBehavior` attribute

---

# Legacy Semantic Reference (v1.0)

*Note: The following mappings were performed by the legacy semantic `nib2cib` tool. The new v2.0 blind transpiler does NOT perform these calculations or mappings. This section is preserved strictly as a reference for the Cappuccino runtime team, who must now implement this logic in the runtime via `mapNativeToCapp:`.*

## Class Mappings
- `<customObject>` → `_CPCibCustomObject`
- `<menu>` → `CPMenu`
- `<menuItem>` → `CPMenuItem`
- `<window>` → `_CPCibWindowTemplate`
- `<view>` → `CPView`
- `<slider>` → `CPSlider`
- `<textField>` → `CPTextField`
- `<font>` → `CPFont`
- `<color>` → `CPColor`

## Connections
- `<outlet>` → `CPCibOutletConnector` (`property` → label, `destination` → target)
- `<action>` → `CPCibControlConnector` (`selector` → label, `target` → target)

## Coordinate System
The XIB Y coordinate is in screen space (bottom-left origin). The legacy compiler performed a flip: `cp_y = screenRect.height - contentRect.y - contentRect.height`. Y-axis autoresizing mask bits were also inverted. This must now be handled by the runtime.

## Bitfield Calculations (Legacy)
The legacy compiler calculated integer bitfields from boolean XML attributes. This must now be handled by the runtime.

### Window Style Mask (`<windowStyleMask>`)
| Attribute       | NS bit | CP bit |
|-----------------|--------|--------|
| `titled`        | 1      | 1      |
| `closable`      | 2      | 2      |
| `miniaturizable`| 4      | 4      |
| `resizable`     | 8      | 8      |
| `utilityPanel`  | 16     | (panel)|
| `texturedBackground` | 256 | 256  |

### Autoresizing Mask (`<autoresizingMask>`)
| Attribute       | Bit |
|-----------------|-----|
| `flexibleMinX`  | 1   |
| `widthSizable`  | 2   |
| `flexibleMaxX`  | 4   |
| `flexibleMinY`  | 8   |
| `heightSizable` | 16  |
| `flexibleMaxY`  | 32  |

### Modifier Mask (`<modifierMask>`)
| Attribute | Bit value  |
|-----------|------------|
| `command` | 1048576    |
| `shift`   | 131072     |
| `option`  | 524288     |
| `control` | 262144     |

## Key Mappings (Legacy)
The legacy compiler mapped XIB attributes to specific `CP...Key` strings. The v2.0 compiler preserves the original XIB attribute names. The runtime must map them.

- `<slider>`: `doubleValue` → `CPControlValueKey`, `maxValue` → `CPSliderMaxValueKey`, etc.
- `<textField>`: `editable` → `CPTextFieldIsEditableKey`, `lineBreakMode` → `CPTextFieldLineBreakModeKey`, etc.
- `<font>`: `metaFont="system"` → `.AppleSystemUIFont` at system size.
- `<color>`: Catalog colors retain the name as a string; the runtime resolves it.

## System Menu Names
| XIB `systemMenu` | CP `CPMenuNameKey`       |
|------------------|--------------------------|
| `main`           | `_CPMainMenu`            |
| `apple`          | `_CPApplicationMenu`     |
| `window`         | `_CPWindowsMenu`         |
| `recentDocuments`| `_CPRecentDocumentsMenu` |
