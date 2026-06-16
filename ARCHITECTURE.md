# capp-nib2cib — Architecture

## Purpose

`capp-nib2cib` converts Xcode Interface Builder XIB files (`.xib`) into
Cappuccino Interface Builder files (`.cib`) for use by the Cappuccino
framework at runtime. It is a pure Go reimplementation of the legacy
`nib2cib` tool, which was written in Objective-J and ran under Node.js.

This tool is part of the `cappuccino` CLI toolchain rewrite and is
distributed as a standalone binary via GitHub Releases, independent of
npm and the Node.js/Objective-J runtime.

---

## What the conversion does

A XIB is an XML document describing an Interface Builder object graph —
windows, views, controls, menus, connections, and custom objects — in
Apple's NS* class vocabulary.

A CIB is a serialized Cappuccino keyed archive (280NPLIST format) of the
equivalent object graph expressed in Cappuccino's CP* class vocabulary.

The conversion is a structural transformation:

```
XIB (XML, NS* classes) → CP object graph → CIB (280NPLIST, CP* classes)
```

It involves:

1. Parsing the XIB XML
2. Walking the object graph, applying per-element transforms
3. Mapping NS* element types to CP* class names
4. Flipping Y coordinates (NS uses bottom-left origin; CP uses top-left)
5. Substituting fonts (IB system font references → CP font references)
6. Emitting the 280NPLIST keyed archive

No Apple tools are involved. `ibtool` and `plutil` are not required.
This was the explicit design decision: the XIB is already XML and is
parsed directly. Binary NIB format is not supported and is not a target.

---

## What the legacy tool did (and why we don't do it)

The legacy `nib2cib` routed:

```
XIB → ibtool (compile to binary NIB) → plutil (binary plist → XML plist)
    → CF$UID regex substitution → CPKeyedUnarchiver (ObjJ runtime)
    → CPKeyedArchiver (ObjJ runtime) → CIB
```

This required:
- Xcode (`ibtool`, `plutil`) to be installed
- Node.js and the Objective-J runtime
- The full Cappuccino AppKit framework loaded into the ObjJ interpreter
  (for theme lookup, font substitution, and the NS* coder categories)

The binary NIB intermediate was necessary only because the ObjJ
unarchiver expected NSKeyedArchiver XML plist format. The XIB carries
the same information and is directly parseable.

---

## Source layout

```
capp-nib2cib/
├── main.go                  entry point; delegates to cmd
├── go.mod
├── cmd/
│   └── root.go              cobra root command; subcommands registered here
├── internal/
│   ├── xib/
│   │   ├── parse.go         XIB XML parser → intermediate object graph
│   │   └── elements.go      typed structs for each XIB element kind
│   ├── transform/
│   │   ├── transform.go     orchestrates the NS→CP conversion pass
│   │   ├── classmap.go      NS* → CP* class name mapping table
│   │   ├── coords.go        Y-coordinate flip logic
│   │   ├── font.go          font name/size resolution; fontinfo integration
│   │   └── widgets/         one file per widget family (see below)
│   └── cib/
│       ├── archive.go       CP keyed archive builder (object table, UID refs)
│       └── encode.go        280NPLIST serializer
```

### Widget transform files

Each file under `internal/transform/widgets/` handles one or more
related XIB element types. The mapping knowledge — what XIB attributes
become what CP keyed archive keys — lives here, not in a parallel
directory of per-class files as in the legacy tool.

When a new Cappuccino widget is added (e.g. `CPSplitViewController`,
`CPStackView`), a new file is added here. There is no separate
`nib2cib`-side class registry to update.

Planned widget files (not exhaustive):

```
view.go              NSView → CPView (base; all views inherit this)
window.go            NSWindowTemplate → _CPCibWindowTemplate
control.go           NSControl → CPControl
button.go            NSButton → CPButton
textfield.go         NSTextField, NSSecureTextField → CPTextField
slider.go            NSSlider → CPSlider
segmented.go         NSSegmentedControl → CPSegmentedControl
tableview.go         NSTableView, NSTableColumn, NSTableHeaderView,
                     NSTableCellView → CP equivalents
outlineview.go       NSOutlineView → CPOutlineView
scrollview.go        NSScrollView, NSScroller, NSClipView → CP equivalents
splitview.go         NSSplitView → CPSplitView
                     NSSplitViewController → CPSplitViewController
stackview.go         NSStackView → CPStackView
tabview.go           NSTabView, NSTabViewItem → CPTabView, CPTabViewItem
menu.go              NSMenu, NSMenuItem → CPMenu, CPMenuItem
toolbar.go           NSToolbar and item variants → CP equivalents
box.go               NSBox → CPBox
imageview.go         NSImageView → CPImageView
progressindicator.go NSProgressIndicator → CPProgressIndicator
stepper.go           NSStepper → CPStepper
combobox.go          NSComboBox → CPComboBox
colorwell.go         NSColorWell → CPColorWell
datepicker.go        NSDatePicker → CPDatePicker
levelindicator.go    NSLevelIndicator → CPLevelIndicator
popupbutton.go       NSPopUpButton → CPPopUpButton
popover.go           NSPopover → CPPopover
browser.go           NSBrowser → CPBrowser
searchfield.go       NSSearchField → CPSearchField
tokenfield.go        NSTokenField → CPTokenField
customobject.go      NSCustomObject → _CPCibCustomObject
customview.go        NSCustomView → CPCustomView
customresource.go    NSCustomResource
classswapper.go      NSClassSwapper
visualeffect.go      NSVisualEffectView → CPVisualEffectView
viewcontroller.go    NSViewController → CPViewController
collectionview.go    NSCollectionView, NSCollectionViewItem
```

---

## The 280NPLIST format

280NPLIST is Cappuccino's own keyed archive serialization format. It is
the output format of `CPKeyedArchiver` and the input format of
`CPKeyedUnarchiver`. It is what a `.cib` file contains.

### Header

```
280NPLIST;1.0;
```

### Type tokens

| Token    | Meaning                                      |
|----------|----------------------------------------------|
| `D`…`E`  | Dictionary (key-value pairs between D and E) |
| `A`…`E`  | Array (elements between A and E)             |
| `K;n;`   | Key, followed by n bytes of key name         |
| `S;n;`   | String, followed by n bytes of string value  |
| `d;n;`   | Number literal, n bytes (decimal or float)   |
| `T`      | Boolean true                                 |
| `F`      | Boolean false                                |

### Object table structure

The archive has the same logical structure as NSKeyedArchiver:

- A top-level dictionary with keys `$top`, `$objects`, and `$archiver`
- `$objects` is an array; index 0 is always `$null`
- Objects reference each other via `CP$UID` dictionaries containing a
  numeric index into `$objects`
- Class identity is encoded as a dictionary with `$classname` and
  `$classes` (inheritance chain), stored in `$objects` and referenced
  by `$class` CP$UID

### CP$UID encoding

A UID reference is encoded as a dictionary with a single key:

```
D;K;6;CP$UID;d;n;<index>;E;
```

where `<index>` is the decimal index into the `$objects` array and `n`
is its character length.

### Example (minimal, annotated)

From `MainMenu.cib`, the top-level structure:

```
280NPLIST;1.0;
D;                              ← root dict
  K;4;$top
  D;
    K;18;CPCibObjectDataKey
    D;K;6;CP$UID;d;1;2;E;       ← reference to $objects[2]
  E;
  K;8;$objects
  A;                            ← object table
    S;5;$null;                  ← [0] always $null
    D; … E;                     ← [1] class descriptor for _CPCibObjectData
    D; … E;                     ← [2] _CPCibObjectData instance
    …
  E;
  K;9;$archiver
  S;15;CPKeyedArchiver;
E;
```

### Key observations for the serializer

- String lengths are byte counts, not character counts. UTF-8 multibyte
  characters must be measured in bytes.
- Numbers are encoded as their decimal string representation with the
  byte length of that string as the length prefix.
- Booleans `T` and `F` have no length or value suffix.
- The object table is built in two passes: first accumulate all objects
  and assign UIDs, then serialize.
- String interning: identical strings share a single `$objects` entry
  and are referenced by UID. This is important for correctness (the
  runtime expects it) and output size.

---

## NS→CP class name mapping

The authoritative mapping is in `internal/transform/classmap.go`. Key
entries (not exhaustive):

```go
"NSApplication"          → "CPApplication"
"NSView"                 → "CPView"
"NSControl"              → "CPControl"
"NSButton"               → "CPButton"
"NSTextField"            → "CPTextField"
"NSSecureTextField"      → "CPTextField"  // same CP class
"NSSlider"               → "CPSlider"
"NSMenu"                 → "CPMenu"
"NSMenuItem"             → "CPMenuItem"
"NSWindow"               → "CPWindow"
"NSPanel"                → "CPPanel"
"NSWindowTemplate"       → "_CPCibWindowTemplate"
"NSCustomObject"         → "_CPCibCustomObject"
"NSCustomView"           → "CPView"
"NSScrollView"           → "CPScrollView"
"NSTableView"            → "CPTableView"
"NSOutlineView"          → "CPOutlineView"
"NSSplitView"            → "CPSplitView"
"NSSplitViewController"  → "CPSplitViewController"
"NSStackView"            → "CPStackView"
"NSTabView"              → "CPTabView"
"NSSegmentedControl"     → "CPSegmentedControl"
"NSProgressIndicator"    → "CPProgressIndicator"
"NSBox"                  → "CPBox"
"NSImageView"            → "CPImageView"
"NSPopUpButton"          → "CPPopUpButton"
"FirstResponder"         → "CPResponder"
```

Special cases not handled by a simple rename:
- `NSWindowTemplate` encodes a window descriptor, not a window. The
  output is a `_CPCibWindowTemplate` with specific keys.
- Custom classes (user-defined `customClass` attribute in XIB) pass
  through unchanged — they are the application developer's own CP
  classes.
- `FirstResponder` maps to `CPResponder` but is a placeholder object
  with no state.

---

## Coordinate system

NS (AppKit) uses a bottom-left origin: Y increases upward.
CP (Cappuccino) uses a top-left origin: Y increases downward.

For a view with frame `(x, y, w, h)` inside a superview of height `H`:

```
cp_y = H - ns_y - h
```

This transform is applied to every view frame during the walk.
Autoresizing mask Y bits are also inverted:
- `NSViewMaxYMargin` ↔ `CPViewMinYMargin`
- `NSViewMinYMargin` ↔ `CPViewMaxYMargin`

Window coordinates use the screen rect height for the initial flip:

```
cp_window_y = screen_height - ns_window_y - window_height
```

---

## Font handling

XIB encodes fonts by family name and size (e.g. `Helvetica 12`,
`.AppleSystemUIFont 13`). CP encodes fonts with the same fields but
uses a system font placeholder name (`_CPFontSystemFacePlaceholder`)
when the IB font was the system font.

IB default system font face names:
- `.AppleSystemUIFont` (Xcode 12+)
- `LucidaGrande` (legacy)

At the default system size (13pt), a system-face font at system size
becomes a `_CPFontSystemFacePlaceholder` reference with the
`CPFontCurrentSystemSize` sentinel (-1.0), allowing the browser to
substitute the actual system font at runtime.

The legacy tool called an external `fontinfo` binary to resolve font
family names and bold/italic attributes from PostScript names. In the
Go implementation this should be handled directly: the XIB `<font>`
element encodes `metaFont="system"` for system font references and
carries explicit `name` and `size` attributes otherwise. PostScript
name → family name resolution requires `fontinfo` only for non-system
fonts specified by PostScript name rather than family name; the common
case does not need it.

---

## XIB element structure (key elements)

### `<customObject>`

```xml
<customObject id="-2" userLabel="File's Owner" customClass="NSApplication">
```

- `id` — XIB object ID (string, may be negative)
- `customClass` — the NS or user class name
- Maps to `_CPCibCustomObject` in the archive with `_CPCibCustomObjectClassName`

### `<window>`

```xml
<window title="Window" id="371">
  <windowStyleMask key="styleMask" titled="YES" closable="YES" …/>
  <rect key="contentRect" x="335" y="390" width="480" height="360"/>
  <rect key="screenRect" x="0" y="0" width="1920" height="1057"/>
  <view key="contentView" id="372"> … </view>
</window>
```

Maps to `_CPCibWindowTemplate`. Style mask bits translate:

| NS bit                        | CP bit                       |
|-------------------------------|------------------------------|
| `NSTitledWindowMask` (1)      | `CPTitledWindowMask` (1)     |
| `NSClosableWindowMask` (2)    | `CPClosableWindowMask` (2)   |
| `NSMiniaturizableWindowMask` (4) | `CPMiniaturizableWindowMask` (4) |
| `NSResizableWindowMask` (8)   | `CPResizableWindowMask` (8)  |
| `NSHUDBackgroundWindowMask` (0x2000) | `CPHUDBackgroundWindowMask` |

### `<view>`

```xml
<view key="contentView" id="372">
  <rect key="frame" x="0" y="0" width="480" height="360"/>
  <autoresizingMask key="autoresizingMask" …/>
  <subviews> … </subviews>
</view>
```

vFlags bit layout (from NSView NSCoding):

| Bit(s) | Meaning                  |
|--------|--------------------------|
| 0–5    | autoresizing mask        |
| 8      | autoresizes subviews     |
| 31     | hidden                   |

### `<connections>`

```xml
<connections>
  <outlet property="delegate" destination="450" id="451"/>
  <action selector="takeDoubleValueFrom:" target="456" id="458"/>
</connections>
```

Outlets → `CPCibOutletConnector` with source, destination, label.
Actions → `CPCibControlConnector` with source, destination, label
(label is the selector string).

---

## CLI (cobra)

The root command is `capp-nib2cib`. Subcommands mirror the pattern
established in the `cappuccino` omnibus binary:

```
capp-nib2cib convert [INPUT [OUTPUT]]   explicit conversion
capp-nib2cib watch [DIRECTORY]          watch mode
capp-nib2cib version                    print version
completion                              shell completion (cobra built-in)
```

Flags on `convert` and `watch`:

| Flag                  | Purpose                                      |
|-----------------------|----------------------------------------------|
| `--default-theme`     | Override default theme name                  |
| `-t`, `--theme`       | Additional theme (repeatable)                |
| `--config`            | Path to Info.plist for font/size override    |
| `-F`, `--framework`   | Framework to load (repeatable)               |
| `-v`, `--verbose`     | Increase verbosity (repeatable)              |
| `-q`, `--quiet`       | Suppress all output                          |
| `--no-colors`         | Disable ANSI color output                    |
| `--no-stored-options` | Ignore `.nib2cibconfig` / `nib2cib.conf`     |

Config file precedence (lowest to highest):
1. `~/.nib2cibconfig`
2. `<app_dir>/nib2cib.conf`
3. `<input_basename>.conf` (adjacent to input file)
4. Command-line flags

Input resolution: if no extension given, tries `.xib` then `.nib` (the
latter is unsupported and will error). Looks first in the given path,
then in `Resources/` adjacent to it.

Output defaults to the input path with extension replaced by `.cib`.

---

## What is explicitly out of scope

- Binary NIB (`.nib`) input. The format is obsolete and requires
  `plutil` to decode. XIB is the only supported input.
- CocoaTouch / iOS XIB files. These carry
  `com.apple.InterfaceBuilder3.CocoaTouch.XIB` in the archive type
  attribute. The converter detects this and errors with a clear message.
- Theme loading. The legacy tool loaded `.blend` theme files into the
  ObjJ runtime to look up `nib2cib-adjustment-frame` attributes. In
  this implementation, frame adjustments are encoded as static tables
  derived from the Aristo2 theme, which is the only theme relevant to
  conversion output correctness. Dynamic theme loading is not supported.
- Framework loading. The legacy tool loaded framework bundles to find
  NS class extensions defined in `NSClasses` Info.plist keys. This
  mechanism is not implemented; all supported NS→CP mappings are
  compiled in.

---

## Reference files

| File | Purpose |
|------|---------|
| `toolchain_reference/Resources/MainMenu.xib` | Reference XIB input |
| `toolchain_reference/Resources/MainMenu.cib` | Reference CIB output |

The reference pair was produced by the legacy `nib2cib` tool and is the
ground truth for output validation. The CIB serializer output must
produce a functionally equivalent archive (object identity may differ;
string interning order may differ; the runtime must load it correctly).

---

## Legacy code location

The legacy Objective-J implementation lives at:

```
CappuccinoSource/tools/nib2cib/
```

The ~90 `NS*.j` files there contain the per-class `NS_initWithCoder:`
categories and `classForKeyedArchiver` implementations. These are the
authoritative source for what XIB keys each class reads and what CP
archive keys each class writes. They are the primary reference for
implementing the widget transform files in `internal/transform/widgets/`.

The code smell in the legacy tool: the NS coder knowledge is separated
from the CP class definitions (which live in `CappuccinoSource/AppKit/`).
This means additions to AppKit (e.g. `CPSplitViewController`,
`CPStackView` added by Daniel Boehringer) have no corresponding
`nib2cib` support unless explicitly added to the tool directory. The Go
implementation does not inherit this structural defect.
