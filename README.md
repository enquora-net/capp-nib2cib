# capp-nib2cib

Converts Cocoa Interface Builder XIB files to Cappuccino CIB files.

This is a from-scratch Lisette reimplementation of the asset pipeline. It replaces the legacy Objective-J/Node.js tool and completely removes dependencies on Mac-specific tools (ibtool, plutil) and semantic interpretation.

## Architecture

The legacy nib2cib was a semantic interpreter. It performed coordinate flipping, NS to CP class name mapping, and font substitution at build time, baking these decisions into the .cib file. This created structural drift between the build tool and the framework, and permanently locked UI geometry to specific themes.

This implementation is a **blind structural transpiler**. It operates as a linear, multi-pass compiler pipeline written in Lisette. It possesses zero semantic knowledge of AppKit. It parses the XIB XML tree, flattens it into an object table, resolves string IDs to integer CP$UID references, and emits version 2.0 of the 280NPLIST text format as a strict 1:1 mirror of the input structure.

**The zero-knowledge boundary is absolute**. The compiler performs no key translation, no class name mapping, and no value coercion. It outputs exactly the keys and values it reads from the input. This keeps the toolchain flexible enough to support other input formats (e.g., GNUStep GORM) in the future without duplicating semantic logic, as modern browser JavaScript is more than capable of handling these transformations at runtime.

All semantic mapping (class resolution, coordinate flipping, theme application) is deferred entirely to the Cappuccino 2.0 runtime via the mapNativeToCapp: method on CP classes. This eliminates the structural condition that produced drift in the first place.

## File System Layout

Following standard Cocoa bundle conventions, the compiler emits compiled .cib archives into the project's ./Resources directory. No limitation is imposed on where source .xib files may reside within the source tree.

## Documentation

Document	Contents
| [ARCHITECTURE.md](ARCHITECTURE.md) | Design decisions, source layout, pipeline, CLI, scope |
| [280NPLIST.md](280NPLIST.md) | CIB output format specification |
| [XIB-FORMAT.md](XIB-FORMAT.md) | XIB input element reference |
## Reference Material

toolchain_reference/Resources/ contains a matched XIB/CIB pair produced by the legacy tool. The legacy .cib uses 280NPLIST v1.0; the new tool will output 280NPLIST v2.0.

The legacy Objective-J implementation is located at CappuccinoSource/tools/nib2cib/ in the Cappuccino source tree. The NS*.j files there document the legacy semantic mappings, which are now deferred to the Cappuccino runtime.

## Status

Work-in-progress[](XIB-FORMAT.md)
