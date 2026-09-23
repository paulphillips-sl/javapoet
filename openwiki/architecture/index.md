# Files

- [Emission Engine & Formatting](emission.md) - Documents the CodeWriter class's two-pass code generation algorithm, indentation management, control flow handling, line wrapping, and zero-width newline semantics used for Java code emission.
- [File System & Directory Operations](file-system.md) - Document JavaFile's package-aware directory creation, UTF-8 enforcement, the nullAppendable pattern for first-pass import collection, and writeToPath/writeToFile return-value semantics
- [Naming Resolution & Qualification](naming.md) - Explain NameAllocator for non-conflicting name generation, always-qualified names propagation through nested types, wildcard type parameter handling, and collision resolution strategies
- [Performance & Optimization](performance.md) - Documents memoized simple name resolution, pre-computed nested type lookup, mutable builder lists for deferred import collection, and the two-pass import gathering algorithm that avoids redundant computation during CodeWriter emission.
