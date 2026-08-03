# Architecture

This document defines the architectural structure and constraints for the Lisette reimplementation of `capp-nib2cib`.

## Purpose

`capp-nib2cib` converts Xcode Interface Builder XIB files (`.xib`) into Cappuccino Interface Builder files (`.cib`) for use by the Cappuccino framework at runtime. It is a pure Lisette reimplementation of the legacy `nib2cib` tool, which was written in Objective-J and ran under Node.js. It is distributed as a standalone binary, independent of npm, Node.js, and Mac-specific tools (`ibtool`, `plutil`).

## 1. The Blind Transliterator Constraint

The original `nib2cib` architecture reflected the performance constraints of web browsers in 2008 by performing semantic transformation during compilation. Class mapping, geometry adjustments, theme-dependent corrections, font substitution, and other interpretation were executed by the build tool before serialization. This duplicated object encoding knowledge between the compiler and the Cappuccino runtime, creating two independent sources of truth that had to evolve in lockstep.

The Lisette implementation eliminates this architectural coupling by treating `capp-nib2cib` as a purely structural transpiler. The compiler performs no semantic interpretation of the Interface Builder document. Instead, all semantic behavior is defined by the runtime and applied during deserialization. Serialization knowledge therefore resides exclusively with the runtime classes that own it, allowing both first-party and third-party components to participate in the compilation pipeline without requiring compiler modifications.

This architectural separation also removes several longstanding limitations of the original design. Theme-dependent adjustments are evaluated at runtime rather than being baked into generated archives, enabling runtime theme changes such as light and dark mode. Likewise, because the compiler no longer depends upon AppKit-specific semantic assumptions, it can accommodate alternative Interface Builder dialects, including GNUstep GORM documents, without requiring architecture-specific transformation logic.

### Permitted Operations
- Parsing XML syntax.
- Flattening an object graph into an array.
- Deduplicating strings (interning).
- Resolving string identifiers to integer indices (`CP$UID`).
- Emitting `280NPLIST` v2.0 tokens.

### Forbidden Operations (Handled at Runtime)
- Coordinate flipping (Y-axis translation).
- Theme application or geometry adjustment.
- `NS` to `CP` class name mapping.
- Property value transformation or bitmask packing.
- Font substitution.

## 2. Pipeline Architecture

The application is structured as a linear, multi-pass compiler pipeline. Execution flow is governed by an iterated enum utilizing Lisette's `#[iterate]` attribute. Pipeline phases must not be combined. Each phase operates as a pure function that consumes the intermediate representation (IR) output of the preceding phase.

```rust
#[iterate]
pub enum Nib2CibPhase
{
    ParseXibXml,
    FlattenObjectGraph,
    InternStrings,
    ResolveUidReferences,
    Emit280nPlist,
}
```

## 3. Implementation Constraints

Development must proceed top-down according to the following order:
1. Define the `Nib2CibPhase` enum.
2. Define the Algebraic Data Types (ADTs) for the intermediate representation.
3. Implement the execution harness driving the phase iteration.
4. Implement individual phase logic starting with `ParseXibXml`.

### Language Idioms
- Source code is written in Lisette, an ML family language employing Rust-derived syntax with Go tab-indentation conventions, compiling natively to idiomatic Go.
- All block formatting must strictly adhere to the Allman style.
- Exhaustive pattern matching must be used for branching logic; `if`/`else` chains are prohibited where `match` applies.
- Error propagation relies on the `Result` type and the `?` operator.
- Shared mutable state across pipeline phases is strictly prohibited.

## 4. Source Layout

The source layout mirrors the blind compiler pipeline. There are no widget-specific or class-specific transform files.

```
capp-nib2cib/
├── main.lis                  entry point; delegates to cmd
├── cmd/
│   └── root.lis              cobra root command; subcommands registered here
├── pipeline/
│   ├── phases.lis            Nib2CibPhase enum definition
│   ├── parse_xib_xml.lis     XIB XML parser → intermediate object graph
│   ├── flatten_graph.lis     Walks tree, flattens to object table (Zero semantic logic)
│   ├── intern_strings.lis    Deduplicates strings into the object table
│   ├── resolve_uid.lis       Resolves string IDs to CP$UID references
│   └── emit_280nplist.lis    280NPLIST v2.0 text serializer
└── ir/
    └── types.lis             ADTs for the intermediate representation
```

## 5. Command Line Interface (CLI)

The root command is `capp-nib2cib`. Subcommands mirror the pattern established in the Cappuccino omnibus binary:

```
capp-nib2cib convert [INPUT [OUTPUT]]   explicit conversion
capp-nib2cib watch [DIRECTORY]          watch mode
capp-nib2cib version                    print version
completion                              shell completion (cobra built-in)
```

### Flags (on `convert` and `watch`)
| Flag                  | Purpose                                      |
|-----------------------|----------------------------------------------|
| `--default-theme`     | Override default theme name                  |
| `-t`, `--theme`       | Additional theme (repeatable)                |
| `--config`            | Path to Info.plist for font/size override    |
| `-F`, `--framework`   | Framework to load (repeatable)               |
| `-v`, `--verbose`     | Increase verbosity (repeatable)              |
| `-q`, `--quiet`       | Suppress all output                          |
| `--no-colors`         | Disable ANSI color output                    |
| `--no-stored-options` | Ignore `.nib2cibconfig` / `nib2cib.conf`      |

Config file precedence (lowest to highest):
1. `~/.nib2cibconfig`
2. `<app_dir>/nib2cib.conf`
3. `<input_basename>.conf` (adjacent to input file)
4. Command-line flags

Input resolution: if no extension given, tries `.xib` then `.nib` (the latter is unsupported and will error). Looks first in the given path, then in `Resources/` adjacent to it. Output defaults to the input path with extension replaced by `.cib`, emitted to the project's `./Resources` directory.

## 6. Scope and Limitations

- **Binary NIBs:** Binary NIB (`.nib`) input is obsolete and requires `plutil` to decode. XIB is the only supported input.
- **CocoaTouch:** iOS XIB files (`com.apple.InterfaceBuilder3.CocoaTouch.XIB`) are detected and rejected.
- **Theme Loading:** Dynamic theme loading is not supported in the compiler. Frame adjustments and theming are deferred entirely to the runtime.
- **Framework Loading:** The compiler does not load framework bundles. All elements are processed generically.

---

# Runtime Semantic Contract (Legacy Reference)

*Note: The following mappings were performed at build-time by the legacy semantic `nib2cib` tool. Because no documentation exists for this logic anywhere else, this section serves as the authoritative specification for the Cappuccino runtime team. The new v2.0 blind transpiler does NOT perform these calculations. The runtime must now implement this logic dynamically during `CPCib` loading via `mapNativeToCapp:`.*

## NS → CP Class Name Mapping
The runtime must map standard AppKit classes to their Cappuccino equivalents. Custom classes (user-defined `customClass` attribute in XIB) pass through unchanged.
- `NSApplication` → `CPApplication`
- `NSView` → `CPView`
- `NSControl` → `CPControl`
- `NSButton` → `CPButton`
- `NSTextField` / `NSSecureTextField` → `CPTextField`
- `NSSlider` → `CPSlider`
- `NSMenu` → `CPMenu`
- `NSMenuItem` → `CPMenuItem`
- `NSWindow` → `CPWindow`
- `NSPanel` → `CPPanel`
- `NSWindowTemplate` → `_CPCibWindowTemplate`
- `NSCustomObject` → `_CPCibCustomObject`
- `NSCustomView` → `CPView`
- `NSScrollView` → `CPScrollView`
- `NSTableView` → `CPTableView`
- `NSOutlineView` → `CPOutlineView`
- `NSSplitView` → `CPSplitView`
- `NSStackView` → `CPStackView`
- `NSTabView` → `CPTabView`
- `NSSegmentedControl` → `CPSegmentedControl`
- `NSProgressIndicator` → `CPProgressIndicator`
- `NSBox` → `CPBox`
- `NSImageView` → `CPImageView`
- `NSPopUpButton` → `CPPopUpButton`
- `FirstResponder` → `CPResponder` (placeholder with no state)

## Coordinate System Flipping
NS (AppKit) uses a bottom-left origin (Y increases upward). CP (Cappuccino) uses a top-left origin (Y increases downward).
For a view with frame `(x, y, w, h)` inside a superview of height `H`:
`cp_y = H - ns_y - h`
Window coordinates use the screen rect height for the initial flip:
`cp_window_y = screen_height - ns_window_y - window_height`
Autoresizing mask Y bits are also inverted: `NSViewMaxYMargin` ↔ `CPViewMinYMargin`, `NSViewMinYMargin` ↔ `CPViewMaxYMargin`.

## Font Handling
XIB encodes fonts by family name and size. At the default system size (13pt), a system-face font (e.g., `.AppleSystemUIFont` or `LucidaGrande`) becomes a `_CPFontSystemFacePlaceholder` reference with the `CPFontCurrentSystemSize` sentinel (-1.0), allowing the browser to substitute the actual system font at runtime.

## Legacy Code Location
The legacy Objective-J implementation lives at `CappuccinoSource/tools/nib2cib/`. The ~90 `NS*.j` files there contain the per-class `NS_initWithCoder:` categories and `classForKeyedArchiver` implementations. These are the authoritative source for what XIB keys each class reads and what CP archive keys each class writes, and serve as the primary reference for implementing the runtime `mapNativeToCapp:` logic.

