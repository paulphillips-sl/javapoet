---
type: architecture
title: Performance & Optimization
description: Documents memoized simple name resolution, pre-computed nested type lookup, mutable builder lists for deferred import collection, and the two-pass import gathering algorithm that avoids redundant computation during CodeWriter emission.
tags: [javapoet-performance, code-emission, optimization]
verified:
  - by: openwiki/0.5.2
    at: 2026-09-22T02:45:45.818Z
sources:
  - id: openwiki-source-d96470ebbdcca258dc9bf91a
    resource: repo://src/main/java/com/squareup/javapoet/CodeWriter.java
  - id: openwiki-source-898f4f1120e1bdadc2d200e9
    resource: repo://src/main/java/com/squareup/javapoet/LineWrapper.java
generated: { by: "openwiki/0.5.2", at: "2026-09-22T02:45:45.818Z" }
---

# Performance & Optimization

The `CodeWriter` class in Javapoet performs the bulk of Java source code generation from an abstract syntax tree. The implementation deliberately prioritizes **runtime performance** over structural elegance at several key points, including memoized simple-name resolution, pre-computed nested-type name lookup, mutable builder lists that defer import collection to a final pass, and a single-traversal two-pass algorithm for gathering imports.

## Memoized Simple Name Resolution (`importableTypes`)

### Responsibility
The `LinkedHashMap` field `importableTypes` memoizes the mapping from a simple type name (e.g., `"Entry"`) to its full `ClassName`. This structure is populated lazily during the single source-tree traversal and used by `lookupName()` to resolve types when they cannot be resolved through existing imports, nesting context, or same-package scope.

### Mechanism
- **Lazy population:** Types are added on-demand as `lookupName()` encounters them in contexts where resolution fails. This avoids enumerating every type declaration upfront.
- **First-wins collision policy:** When two distinct types share the same simple name in different packages, the first one inserted into `importableTypes` wins, and subsequent collisions are silently ignored (via `LinkedHashMap.put` returning the existing value). The winner is determined by traversal order — typically source-file declaration order.
- **Same-package optimization:** Types within the current package are excluded from import suggestions by being tracked separately in `referencedNames`. This avoids unnecessary import statements for intra-package references and reduces the size of the final import list.

### Observables
- The map is populated as a side effect during traversal; its contents at emission time determine which types become import candidates.
- After all code has been emitted, `suggestedImports()` returns the importable types minus any that are referenced within the same package.

### Tests
- Verify that types correctly identified as needing imports appear in the suggested list.
- Verify that intra-package references are excluded from import suggestions.
- Verify that first-wins collision policy produces deterministic ordering (insertion order preserved).

## Pre-computed Nested Type Name Lookup (`stackClassName`)

### Responsibility
The `resolve()` method uses a pre-computed approach to find the shortest type name suffix that uniquely identifies a target class within the current nesting context. The `stackClassName(int stackDepth, String simpleName)` helper materializes this by building a fully qualified reference from the top-level class down through the nested-type chain at the specified depth.

### Mechanism
- **Bottom-up search:** Given a target type, resolution walks outward from its enclosing classes toward the root, checking at each level whether the local simple name resolves to the intended type via `resolve()`.
- **Shortest suffix wins:** The first (innermost) level that produces a matching resolved type determines the shortest usable name. For example, if resolving `"Entry"` matches `Map.Entry` but not `java.util.Map.Entry`, the result is `"Entry"`.
- **Nesting-aware resolution:** The `typeSpecStack` tracks all enclosing types during traversal. Resolution checks nested types in reverse order (innermost first) so that `Map.Entry` can be resolved as just `"Entry"` when inside a `Map` block, but falls back to the fully qualified name outside it.

### Observables
- Name resolution is **deterministic** for a fixed set of imports and nesting context.
- The algorithm is single-pass per type reference — no repeated lookups are performed.

## Mutable Builder Lists (`typeSpecStack`)

### Responsibility
The `typeSpecStack` is a mutable list that tracks the current nesting context (classes, inner types, etc.) during code emission. It is mutated by `pushType()` and `popType()`, mirroring the structural nesting of the AST being emitted.

### Mechanism
- **Stack discipline:** The stack grows when emitting an entry point for a nested type and shrinks when exiting it. This enables `lookupName()` to know which types are locally scoped versus globally imported.
- **Bounded mutations:** Stack operations are paired (`pushType` / `popType`) within corresponding AST nodes, ensuring the stack depth never exceeds the actual nesting level of the emitted code.

### Observables
- The stack depth reflects the current type-nesting level during emission.
- All resolution logic that depends on scope (e.g., distinguishing between a local inner class and an imported top-level class) queries this stack at emission time.

## Two-Pass Import Collection Algorithm

### Responsibility
Import suggestion generation is deferred until after all code has been traversed, using a two-phase algorithm:
1. **First pass (traversal):** As each type is encountered during code emission, check whether it can be resolved via existing imports or the current nesting context. If not, add its simple name to `importableTypes`.
2. **Second pass (suggestion computation):** After traversal completes, filter `importableTypes` by removing names in `referencedNames` to produce the final import list.

### Mechanism
- **Single source-tree traversal:** CodeWriter emits code and collects imports simultaneously in one pass over the AST. Import candidates are buffered during traversal and materialized only after all nodes have been visited.
- **No redundant computation:** Types are checked against existing imports, nesting context, and same-package scope exactly once per occurrence. The memoization in `importableTypes` ensures subsequent references to the same type do not trigger additional resolution work.
- **Lazy import generation:** Import statements are not emitted during traversal. They are computed as a side effect after the entire tree is processed, via `suggestedImports()`.

### Observables
- The suggested imports map contains only types that require import statements (those not resolvable through context or same-package scope).
- Import ordering preserves insertion order (first use wins), which aligns with typical IDE-generated output.

## Relationship Between Components

```mermaid
graph TD
    CodeWriter --> LineWrapper
    CodeWriter --> ClassName
    CodeWriter --> TypeSpec
    CodeWriter --> AnnotationSpec
    CodeWriter --> CodeBlock
    
    LineWrapper --> RecordingAppendable
    RecordingAppendable --> Appendable
    
    CodeWriter --name resolution--> importableTypes
    CodeWriter --nesting context--> typeSpecStack
    CodeWriter --deferred imports--> suggestedImports()
```

- **`CodeWriter`** orchestrates the single-traversal emission loop: it emits code, collects import candidates via `importableTypes`, and resolves types through `lookupName()`.
- **`LineWrapper`** sits between `CodeWriter` and the final output, handling soft-wrapping ($W), zero-width newlines ($Z), indentation emission, and deferred formatting token resolution.
- **All formatting tokens** are emitted through `emitAndIndent()`, which ensures lazy whitespace handling and statement-aware continuation indentation (double-indent for multi-line statement continuations).

## Failure Modes & Invariants

| Condition | Behavior |
|---|---|
| Invalid indentation | Attempting to unindent more levels than were previously entered throws an `IllegalArgumentException`. |
| Double package declaration | Setting a package after one has already been declared throws via state check. |
| Mismatched statement tokens | A `$]` without a matching `$[` (or vice versa) causes validation failures. |
| Static import resolution failure | When a deferred type is followed by a dot-separated member access, static import resolution fails if no applicable static import exists; the caller must provide appropriate imports. |
| Closed LineWrapper | Writing to a `LineWrapper` after calling `close()` throws immediately. |

## Key Invariants

1. **Two-pass is not truly sequential:** Both traversal and import collection interleave during a single AST walk. Import candidates are buffered during traversal, but final suggestions are only computed after the full traversal via `suggestedImports()`.
2. **Indentation state is consistent at line boundaries:** The `indentLevel`, `statementLine`, and `trailingNewline` fields remain coherent when newlines are emitted.
3. **Name resolution is deterministic:** Given a fixed set of imports and nesting context, the same type always resolves to the same shortest available name.
4. **Insertion order is preserved:** The `LinkedHashMap` in `importableTypes` preserves insertion order, which influences import statement ordering in the final suggestion.

## Configuration & Options

| Option | Description | Default |
|---|---|---|
| `indent` | Whitespace string used for indentation (e.g., `"  "` or `"\t"`) | `"  "` |
| `columnLimit` | Maximum line length before wrapping occurs | 100 characters |
| `javadoc` | Whether we're emitting Javadoc documentation | `false` |
| `comment` | Whether we're emitting single-line comments | `false` |
| `staticImports` | Set of statically imported class names to use for member resolution | empty set |
| `alwaysQualify` | Types that must always be fully qualified, never shortened | empty set |

The `javadoc` flag changes comment prefix formatting: Javadoc uses `" * "` instead of `"// "`, and blank lines within Javadoc blocks are properly indented.

## Focused Tests That Matter

- **Import suggestion accuracy:** Verify that types correctly identified as needing imports are suggested, and types resolvable through context/imports are excluded.
- **Indentation correctness:** Test that blocks have proper indentation, multi-line statements use double-indentation for continuations, and no trailing whitespace is emitted.
- **Line wrapping:** Confirm that `$W` and `$Z` produce expected wrap behavior at the column limit, including handling of statement continuations across wraps.
- **Name resolution:** Ensure `lookupName()` correctly resolves types through imports, nesting context, and same-package scope, falling back to fully-qualified names when necessary.
- **Static import interaction:** Verify deferred type resolution with `$.` members produces correct static import usage or falls back to explicit qualification.

## Source References

- [CodeWriter.java](repo://src/main/java/com/squareup/javapoet/CodeWriter.java) — Main emission class implementing the two-pass algorithm, indentation management, and format token handling.
- [LineWrapper.java](repo://src/main/java/com/squareup/javapoet/LineWrapper.java) — Soft-wrapping implementation for line wrapping ($W), zero-width newlines ($Z), and indentation emission.
- [CodeWriterTest.java](repo://src/test/java/com/squareup/javapoet/CodeWriterTest.java) — Unit tests validating import suggestions, indentation formatting, and control flow handling.
- [LineWrapperTest.java](repo://src/test/java/com/squareup/javapoet/LineWrapperTest.java) — Tests for the line wrapping mechanism including soft-wrapping behavior.
