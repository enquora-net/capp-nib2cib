# Cappuccino Runtime v2.0 CIB Loading Plan

## Background

The new `capp-nib2cib` Lisette compiler acts as a strict 1:1 blind transpiler. It performs zero semantic interpretation of the XIB structure. It parses the XML, flattens the object graph, and serializes it into the `280NPLIST` v2.0 format.

Because the compiler no longer performs `NS` to `CP` class mapping, coordinate flipping, bitmask packing, or theme resolution, **all of these responsibilities must now be handled by the Cappuccino runtime during deserialization.**

This document outlines the required adaptations to the `AppKit` and `Foundation` frameworks to successfully load and instantiate v2.0 `.cib` files.

---

## 1. CPKeyedUnarchiver Adaptations

The unarchiver is the entry point for the `.cib` data. It must be updated to detect the v2.0 header and adapt its decoding strategy.

### 1.1. Version Detection
- The archive header will now read `280NPLIST;2.0;`.
- The unarchiver must parse this version string. If `2.0` is detected, it must enable the v2.0 decoding paths outlined below. If `1.0` is detected, it must fall back to the existing legacy behavior to ensure backward compatibility.

### 1.2. Class Resolution (`mapNativeToCapp:`)
In v1.0, the `$classname` in the archive was already the Cappuccino class name (e.g., `CPMenu`). In v2.0, the `$classname` is the verbatim XIB tag name (e.g., `menu`, `window`, `customObject`).
- The unarchiver must intercept class instantiation. Before calling `alloc` on a class, it must consult a mapping table (or a `mapNativeToCapp:` class method) to resolve the native XIB tag to the appropriate Cappuccino class.
- Example: `$classname = "menu"` resolves to `[CPMenu class]`.
- Example: `$classname = "document"` resolves to a generic container or is handled specially by `CPCib` (see Section 2).

### 1.3. Type Coercion (Strict String Output)
The v2.0 compiler serializes *all* XML attribute values as `280NPLIST` String (`S`) tokens. It does not perform lexical type inference.
- `decodeIntForKey:`, `decodeFloatForKey:`, and `decodeBoolForKey:` must be updated to transparently handle the case where the underlying archive value is a String.
- If a number is expected but a String is found, the unarchiver must parse the string (e.g., `parseInt("192")`, `"YES" === true`).
- This ensures the compiler remains simple while keeping the `initWithCoder:` implementations clean.

---

## 2. CPCib Root Object Handling

In v1.0, the `$top` dictionary pointed to a `_CPCibObjectData` object, which acted as a semantic container holding arrays of objects, connections, and the File's Owner.

In v2.0, the `$top` dictionary points directly to the root `<document>` element of the XIB. 
- `CPCib` must be updated to recognize this new root structure.
- Instead of looking for `_CPCibObjectData` keys, the runtime must inspect the `document` object's children (which will be encoded as members under the `objects` key).
- The runtime must identify the File's Owner (typically the `<customObject>` with `id="-2"`) and the visible windows (typically `<window>` elements) dynamically from this flat list, rather than relying on pre-separated arrays.

---

## 3. Per-Class Semantic Transformations

The heavy lifting of XIB-to-Cappuccino translation now happens in the `initWithCoder:` methods of the framework classes. The following logic, previously performed at build time, must be implemented in the runtime.

### 3.1. Coordinate System Flipping (`CPView`, `CPWindow`)
- XIBs use a bottom-left origin coordinate system. Cappuccino uses a top-left origin.
- During decoding, `CPView` and `CPWindow` must read their `frame` or `contentRect` attributes (which will be generic dictionaries or strings in v2.0) and apply the Y-axis flip: `cp_y = superview_height - ns_y - height`.
- Autoresizing mask Y-axis bits (`flexibleMinY`, `flexibleMaxY`) must also be inverted during decoding.

### 3.2. Bitmask Packing (`CPWindow`, `CPMenuItem`, etc.)
- XIBs represent masks (like window style masks or modifier masks) as distinct boolean XML attributes (e.g., `titled="YES"`, `closable="YES"`).
- In v2.0, these will appear as separate dictionary keys on the instance.
- The class's decoder must read these individual boolean strings and pack them into the integer bitmasks expected by the Cappuccino runtime (e.g., `CPTitledWindowMask | CPClosableWindowMask`).

### 3.3. Font Resolution (`CPFont`)
- XIBs use `<font metaFont="system"/>` or `<font name=".AppleSystemUIFont" size="13"/>`.
- The `CPFont` decoder must detect these specific string values. If a system font is requested, it must substitute the Cappuccino system font placeholder (`_CPFontSystemFacePlaceholder`) and the `CPFontCurrentSystemSize` sentinel.

### 3.4. Color Resolution (`CPColor`)
- XIBs use catalog colors (e.g., `<color name="controlTextColor" catalog="System" .../>`).
- The `CPColor` decoder must read the `name` string and resolve it to the appropriate Cappuccino theme color at runtime, enabling dynamic Light/Dark mode theme swapping.

---

## 4. Connections (`CPCibConnector`)

In v1.0, the compiler explicitly created `CPCibOutletConnector` and `CPCibControlConnector` objects.
In v2.0, the compiler blindly outputs `<outlet>` and `<action>` elements as generic objects in the archive.
- The runtime must detect these generic objects during unarchiving.
- If an object's tag is `outlet`, the runtime must promote it to a `CPCibOutletConnector`, mapping its `property` and `destination` attributes.
- If an object's tag is `action`, the runtime must promote it to a `CPCibControlConnector`, mapping its `selector` and `target` attributes.

---

## 5. Validation Strategy

1.  **Reference Pair:** Use the `MainMenu.cib` generated by the new Lisette compiler as the primary test case.
2.  **Runtime Loading:** The Cappuccino runtime should be able to load the v2.0 `MainMenu.cib` without throwing errors.
3.  **Visual Equivalence:** The loaded UI (Main Menu and Window with a Slider and Text Field) must be visually and functionally equivalent to the legacy v1.0 `.cib`.
4.  **Theme Swapping:** Verify that changing the runtime theme (e.g., Light to Dark) correctly updates the UI, proving that theme attributes are no longer baked into the archive.
