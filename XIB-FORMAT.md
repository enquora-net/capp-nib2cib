# XIB Format Reference

This document describes the XIB XML elements relevant to nib2cib
conversion. XIB is Apple's Interface Builder file format version 3
(`com.apple.InterfaceBuilder3.Cocoa.XIB`). It is the input format for
`capp-nib2cib`.

Only the macOS Cocoa dialect is supported. CocoaTouch XIBs are
detected and rejected.

---

## Detection

```xml
<document type="com.apple.InterfaceBuilder3.Cocoa.XIB" …>
```

CocoaTouch (rejected):
```xml
<document type="com.apple.InterfaceBuilder3.CocoaTouch.XIB" …>
```

---

## Top-level structure

```xml
<document type="…" version="3.0" …>
    <dependencies> … </dependencies>
    <objects> … </objects>
</document>
```

All objects of interest are children of `<objects>`.

---

## Object identity

Every object element carries an `id` attribute. IDs are strings,
typically small integers assigned by Interface Builder. Negative IDs
are reserved:

| ID  | Role            |
|-----|-----------------|
| -1  | First Responder |
| -2  | File's Owner    |
| -3  | Application     |

Connections reference objects by these IDs via `destination` and
`target` attributes.

---

## Element reference

### `<customObject>`

A placeholder for objects not directly instantiated by IB.

```xml
<customObject id="-2" userLabel="File's Owner" customClass="NSApplication">
  <connections> … </connections>
</customObject>
```

Attributes:
- `id` — object identity
- `customClass` — NS or user-defined class name
- `userLabel` — display label (not used in conversion)

Maps to `_CPCibCustomObject` in the archive.

---

### `<menu>`

```xml
<menu title="AMainMenu" systemMenu="main" id="29" userLabel="MainMenu">
  <items>
    <menuItem title="File" id="83">
      <menu key="submenu" title="File" id="81">
        <items> … </items>
      </menu>
    </menuItem>
    …
  </items>
</menu>
```

Attributes:
- `title` — menu title
- `systemMenu` — `main`, `apple`, `window`, `recentDocuments` (special names)
- `key` — when nested, the key in the parent (`submenu`)

System menu name mapping:

| XIB `systemMenu` | CP `CPMenuNameKey`       |
|------------------|--------------------------|
| `main`           | `_CPMainMenu`            |
| `apple`          | `_CPApplicationMenu`     |
| `window`         | `_CPWindowsMenu`         |
| `recentDocuments`| `_CPRecentDocumentsMenu` |

Maps to `CPMenu`.

---

### `<menuItem>`

```xml
<menuItem title="Save" keyEquivalent="s" id="75">
  <connections>
    <action selector="saveDocument:" target="-1" id="362"/>
  </connections>
</menuItem>
```

Separator items:
```xml
<menuItem isSeparatorItem="YES" id="236">
  <modifierMask key="keyEquivalentModifierMask" command="YES"/>
</menuItem>
```

Attributes:
- `title` — display title
- `keyEquivalent` — single character key
- `isSeparatorItem` — YES if separator
- `tag` — integer tag

`<modifierMask>` child encodes the key equivalent modifier mask:

| Attribute | Bit value  |
|-----------|------------|
| `command` | 1048576    |
| `shift`   | 131072 (added to command = 1179648) |
| `option`  | 524288 (added = 1572864) |
| `control` | 262144     |

Default (command only) = 1048576.

Maps to `CPMenuItem`.

---

### `<window>`

```xml
<window title="Window" allowsToolTipsWhenApplicationIsInactive="NO"
        autorecalculatesKeyViewLoop="NO" releasedWhenClosed="NO"
        animationBehavior="default" id="371">
  <windowStyleMask key="styleMask" titled="YES" closable="YES"
                   miniaturizable="YES"/>
  <windowPositionMask key="initialPositionMask" leftStrut="YES"
                      bottomStrut="YES"/>
  <rect key="contentRect" x="335" y="390" width="480" height="360"/>
  <rect key="screenRect" x="0" y="0" width="1920" height="1057"/>
  <view key="contentView" id="372"> … </view>
</window>
```

`<windowStyleMask>` boolean attributes → NS style mask bits:

| Attribute       | NS bit | CP bit |
|-----------------|--------|--------|
| `titled`        | 1      | 1      |
| `closable`      | 2      | 2      |
| `miniaturizable`| 4      | 4      |
| `resizable`     | 8      | 8      |
| `utilityPanel`  | 16     | (panel)|
| `texturedBackground` | 256 | 256  |
| `docModal`      | 64     | CPDocModalWindowMask |
| `hudBackground` | 0x2000 | CPHUDBackgroundWindowMask |

The `contentRect` Y coordinate is in screen space (bottom-left origin).
The flip: `cp_y = screenRect.height - contentRect.y - contentRect.height`.

`autorecalculatesKeyViewLoop` → `_CPCibWindowTemplateWindowAutorecalculatesKeyViewLoopKey` (bool).

Maps to `_CPCibWindowTemplate`.

---

### `<view>`

```xml
<view key="contentView" id="372">
  <rect key="frame" x="0" y="0" width="480" height="360"/>
  <autoresizingMask key="autoresizingMask" widthSizable="YES"
                    heightSizable="YES"/>
  <subviews> … </subviews>
</view>
```

`<autoresizingMask>` boolean attributes → NS autoresizing mask bits:

| Attribute       | Bit |
|-----------------|-----|
| `flexibleMinX`  | 1   |
| `widthSizable`  | 2   |
| `flexibleMaxX`  | 4   |
| `flexibleMinY`  | 8   |
| `heightSizable` | 16  |
| `flexibleMaxY`  | 32  |

Y-axis bits are inverted on output (see ARCHITECTURE.md §Coordinate system).

`autoresizesSubviews` defaults YES in NS (bit 256 of vFlags). Check for
explicit `autoresizesSubviews="NO"` attribute on the `<view>` element.

`hidden="YES"` → bit 31 of vFlags.

Maps to `CPView`.

---

### `<slider>`

```xml
<slider verticalHuggingPriority="750" fixedFrame="YES"
        translatesAutoresizingMaskIntoConstraints="NO" id="452">
  <rect key="frame" x="192" y="143" width="96" height="21"/>
  <autoresizingMask key="autoresizingMask" flexibleMinX="YES" …/>
  <sliderCell key="cell" continuous="YES" state="on" alignment="left"
              maxValue="100" doubleValue="50" tickMarkPosition="above"
              sliderType="linear" id="453">
    <font key="font" size="12" name="Helvetica"/>
  </sliderCell>
  <connections>
    <action selector="takeDoubleValueFrom:" target="456" id="458"/>
  </connections>
</slider>
```

Key `<sliderCell>` attributes:
- `doubleValue` → `CPControlValueKey`
- `maxValue` → `CPSliderMaxValueKey`
- `minValue` (default 0) → `CPSliderMinValueKey`
- `tickMarkPosition` → `CPSliderTickMarkPositionKey` (0=below, 1=above)
- `sliderType` → `CPSliderTypeKey` (0=linear, 1=circular)
- `continuous` → `CPControlSendActionOnKey`

Maps to `CPSlider`.

---

### `<textField>`

```xml
<textField verticalHuggingPriority="750" fixedFrame="YES"
           translatesAutoresizingMaskIntoConstraints="NO" id="456">
  <rect key="frame" x="192" y="191" width="96" height="22"/>
  <autoresizingMask key="autoresizingMask" …/>
  <textFieldCell key="cell" scrollable="YES" lineBreakMode="clipping"
                 selectable="YES" editable="YES"
                 sendsActionOnEndEditing="YES" state="on"
                 borderStyle="bezel" drawsBackground="YES" id="457">
    <font key="font" metaFont="system"/>
    <color key="textColor" name="controlTextColor" catalog="System"
           colorSpace="catalog"/>
    <color key="backgroundColor" name="textBackgroundColor"
           catalog="System" colorSpace="catalog"/>
  </textFieldCell>
</textField>
```

`<textFieldCell>` attributes:
- `editable` → `CPTextFieldIsEditableKey`
- `selectable` → `CPTextFieldIsSelectableKey`
- `drawsBackground` → `CPTextFieldDrawsBackgroundKey`
- `borderStyle` — `bezel` → bezeled+editable theme state
- `sendsActionOnEndEditing` → `CPControlSendsActionOnEndEditingKey`
- `lineBreakMode` → `CPTextFieldLineBreakModeKey`
  - `clipping` = 2
  - `wordWrapping` = 0
  - `charWrapping` = 1
  - `truncatingHead` = 3
  - `truncatingTail` = 4
  - `truncatingMiddle` = 5
- `alignment` → `CPTextFieldAlignmentKey`
  - `left` = 0, `center` = 2, `right` = 1, `natural` = 4

`<font key="font" metaFont="system"/>` → system font at system size.
`<font key="font" size="12" name="Helvetica"/>` → named font.

Maps to `CPTextField`.

---

### `<font>`

```xml
<font key="font" metaFont="system"/>
<font key="font" size="13" name="LucidaGrande"/>
<font key="font" size="12" name="Helvetica"/>
```

`metaFont` values:
- `system` → `.AppleSystemUIFont` at system size (13pt)
- `smallSystem` → system font at small size (11pt)
- `label` → system font at label size

Named fonts carry `name` (PostScript or family name) and `size`.

When the name is `.AppleSystemUIFont` or `LucidaGrande` at size 13.0
with no bold/italic, this maps to the CP system font placeholder.

Maps to `CPFont`.

---

### `<color>`

```xml
<color key="textColor" name="controlTextColor" catalog="System"
       colorSpace="catalog"/>
<color key="backgroundColor" red="1" green="1" blue="1" alpha="1"
       colorSpace="custom" customColorSpace="sRGB"/>
```

Catalog colors (colorSpace="catalog") reference named system colors.
The conversion retains the name as a string; the runtime resolves it.

Custom colors carry RGBA components as float strings (0.0–1.0).

Maps to `CPColor`.

---

### `<connections>`

```xml
<connections>
  <outlet property="delegate" destination="450" id="451"/>
  <action selector="orderFrontStandardAboutPanel:" target="-2" id="142"/>
</connections>
```

- `<outlet>` → `CPCibOutletConnector`
  - `property` → label (outlet name)
  - `destination` → target object id
  - Source is the enclosing element's id

- `<action>` → `CPCibControlConnector`
  - `selector` → label (action selector)
  - `target` → target object id
  - Source is the enclosing element's id

---

### `<rect>`

```xml
<rect key="frame" x="192" y="143" width="96" height="21"/>
<rect key="screenRect" x="0" y="0" width="1920" height="1057"/>
<rect key="contentRect" x="335" y="390" width="480" height="360"/>
```

All attributes are floats. Y coordinate is in the NS coordinate system
(bottom-left origin) unless otherwise noted.

---

### `<size>`

```xml
<size key="minSize" width="100" height="100"/>
```

---

### `<point>`

```xml
<point key="canvasLocation" x="-203" y="129"/>
```

Canvas location is IB metadata and is ignored during conversion.

---

## Elements not converted

The following XIB elements carry IB metadata with no CIB equivalent
and are silently ignored:

- `<point key="canvasLocation" …/>`
- `<capability …/>`
- `<plugIn …/>`
- `<deployment …/>` (the deployment target patch was only needed for
  the ibtool path; it is irrelevant when parsing XIB directly)
- `translatesAutoresizingMaskIntoConstraints` attribute (autolayout
  constraint; Cappuccino does not use autolayout)
- `fixedFrame` attribute
- `verticalHuggingPriority`, `horizontalHuggingPriority` attributes
- `allowsToolTipsWhenApplicationIsInactive` attribute
- `animationBehavior` attribute

---

## Autolayout

XIBs created with `useAutolayout="YES"` (Xcode 8+) may contain
`<constraints>` elements. Cappuccino does not implement autolayout.
Constraints are ignored. Fixed frames (`fixedFrame="YES"`) are the
supported layout mechanism.
