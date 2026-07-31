# capp-nib2cib

Converts  Cocoa Interface Builder XIB files to Cappuccino CIB files.
This is a from-scratch Lisette reimplementation of the asset pipeline. It replaces the legacy Objective-J/Node.js tool and completely removes dependencies on Mac-specific tools (ibtool, plutil) and semantic interpretation.
## Architecture
The legacy nib2cib was a semantic interpreter. It performed coordinate flipping, NS to CP class name mapping, and font substitution at build time, baking these decisions into the .cib file. This created structural drift between the build tool and the framework, and permanently locked UI geometry to specific themes.
This implementation is a blind structural transpiler. It operates as a linear, multi-pass compiler pipeline written in Lisette. It possesses zero semantic knowledge of AppKit. It parses the XIB XML tree, flattens it into an object table, resolves string IDs to integer CP$UID references, and emits the new CAPP2PLIST text format as a one-to-one mirror of the XIB structure.
All semantic mapping (class resolution, coordinate flipping, theme application) is deferred entirely to the Cappuccino 2.0 runtime via the mapNativeToCapp: method on CP classes. This eliminates the structural condition that produced drift in the first place.
## Documentation
Document	Contents
ARCHITECTURE.md	Design decisions, pipeline definition, CLI, scope
AGENTS.md	Implementation constraints for LLM-assisted development
280NPLIST.md	Legacy CIB format specification (for runtime backward compat)
XIB-FORMAT.md	XIB input element reference
Reference material

toolchain_reference/Resources/ contains a matched XIB/CIB pair produced by the legacy tool. The legacy .cib uses 280NPLIST; the new tool will output CAPP2PLIST.
## Status
Pre-implementation. Scaffolding phase.
| Document | Contents |
|----------|----------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Design decisions, source layout, pipeline, CLI, scope |
| [280NPLIST.md](280NPLIST.md) | CIB output format specification |
| [XIB-FORMAT.md](XIB-FORMAT.md) | XIB input element reference |

## Reference material
`toolchain_reference/Resources/` contains a matched XIB/CIB pair
produced by the legacy tool. This is the ground truth for output
validation.
The legacy Objective-J implementation is at
`CappuccinoSource/tools/nib2cib/` in the Cappuccino source tree. The
`NS*.j` files there are the authoritative per-class source for what XIB
attributes map to what CP archive keys.
