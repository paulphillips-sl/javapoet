# JavaPoet

This repository has a generated code wiki in `openwiki/` with architecture and
concept documentation grounded to specific source line ranges.

** Consult the wiki before reading source files. 
** It is far cheaper in context than the source, and its claims are verified against the code.

Page map:

- `openwiki/architecture/emission.md` — CodeWriter two-pass algorithm, indentation, `$W`/`$Z` tokens, line wrapping
- `openwiki/architecture/naming.md` — name resolution
- `openwiki/architecture/file-system.md` — JavaFile output and layout
- `openwiki/architecture/performance.md` — performance characteristics
- `openwiki/concepts/types.md` — TypeName hierarchy
- `openwiki/concepts/methods.md` — MethodSpec
- `openwiki/concepts/naming.md` — NameAllocator
- `openwiki/concepts/annotations.md` — AnnotationSpec
- `openwiki/testing/method-coverage.md` — test structure
- `openwiki/quickstart.md` — core entry types

State which wiki page you used. If the wiki contradicts the source, trust the source and say so explicitly.
