# capp-nib2cib

Converts Xcode Interface Builder XIB files to Cappuccino CIB files.

Pure Go reimplementation of the legacy `nib2cib` Objective-J tool.
No Xcode, no `ibtool`, no `plutil`, no Node.js, no npm dependency.

## Documentation

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

## Status

Pre-implementation. Documentation phase.
