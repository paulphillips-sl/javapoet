---
type: api-version-history
title: Versioning & API Evolution
description: Documents JavaPoet's semantic version progression from 1.0.0 through 1.14.0-SNAPSHOT, highlighting key additions (type annotations, static imports, charsets) and breaking changes (Java 8 requirement).
tags: [java-poet, java-api-evolution, source-generation]
verified:
  - by: openwiki/0.5.2
    at: 2026-09-22T02:45:45.818Z
sources:
  - id: openwiki-source-ca6cb4b1a14fd7969dfae3ec
    resource: repo://CHANGELOG.md
generated: { by: "openwiki/0.5.2", at: "2026-09-22T02:45:45.818Z" }
---

# JavaPoet Versioning & API Evolution

## Overview

[JavaPoet](http://github.com/palantir/javapoet) is a Java API for generating `.java` source files. The library evolved from the original [JavaWriter](https://github.com/square/javawriter) project through a comprehensive rewrite and rebranding, culminating in the builder-based **JavaPoet** API that powers modern annotation processing workflows.

The version history spans multiple major redesigns:
- **JavaWriter 1.x–2.x** (2013–2015): Stream-based API with deprecated int-modifier methods
- **JavaPoet 1.0.0** (2015–present): Immutable value objects, builder pattern, type-aware import resolution

## JavaWriter → JavaPoet Transition

The most significant architectural shift occurred at **[JavaPoet 1.0.0](http://github.com/square/javapoet/blob/master/CHANGELOG.md#JavaPoet-1-0-0)** (2015-01-28), a complete rewrite that restructured the codebase from streaming-based generation to immutable value objects with builders:

| Feature | JavaWriter 2.x | JavaPoet 1.0 |
|---|---|---|
| API style | Streaming, imperative methods | Immutable objects, method chaining |
| Modifiers | Int flags (deprecated in 1.0) | `Modifier` enum (`PUBLIC`, `STATIC`, etc.) |
| Imports | Manual | Automatic based on type references |
| Type model | None | Full type system (`TypeName`, `TypeSpec`, etc.) |

The project was renamed from `com.squareup.javawriter` → `com.squareup.javapoet` to allow coexistence of the old API and new builder-based APIs during migration. The rename was also intentional: **JavaPoet** emphasizes its role as a code-generation _poet_ (a creative author) rather than a writer.

## Semantic Version Progression

### 1.x Major Versions

#### JavaPoet 1.0.0 *(2015-01-28)*
Initial release of the rewritten library:
- Complete rewrite from JavaWriter's streaming API to immutable value objects and builders
- Full type system with `TypeName`, `TypeSpec`, `TypeVariableName`
- Automatic import generation from referenced types
- **Breaking change:** Requires Java 8 or newer

#### JavaPoet 1.1.0 *(2015-05-25)*
- Eager validation of argument types (`$T`, `$N`)
- Varargs method support via `MethodSpec.varargs(boolean)`
- `AnnotationSpec.get()` and `MethodSpec.overriding()` using `javax.lang.model` API
- Java 8 `DEFAULT` modifier
- `TypeSpec.toBuilder()` for constructing from existing objects

#### JavaPoet 1.2.0 *(2015-07-04)*
- Positional argument indexes (`$1T`, `$2N`) for reusing format arguments
- Javadoc on enum constants
- Class initializer blocks via `addStaticBlock()`
- `MethodSpec.overriding()` preserves annotations (changed behavior in 1.8)

#### JavaPoet 1.3.0 *(2015-09-20)*
- `NameAllocator` API for non-conflicting name generation
- Annotation support on enum values
- Fix: Avoid infinite recursion in `TypeName.get(TypeMirror)`
- Qualified names for same-file simple name collisions

#### JavaPoet 1.4.0 *(2015-11-13)*
- Type annotations via `TypeName.annotated()` (e.g., `@Nullable String`)
- Equality/hashing on all main classes (`AnnotationSpec`, `CodeBlock`, `FieldSpec`, etc.)
- `NameAllocator.clone()` for inner scope refinement
- **Breaking change:** Int-based modifier methods removed

#### JavaPoet 1.5.x *(2016)*
- Annotated `TypeName` equality
- Performance improvements to `CodeWriter.resolve()` via pre-computed nested simple names

#### JavaPoet 1.6.x *(2016)*
- `CodeBlock.of()` factory method revival
- Type builder methods accepting `ClassName`
- `TypeName.annotated()` for adding annotations to types
- `TypeVariableName.withBounds()` for generic type bounds
- Instance initializer blocks (`addInitializerBlock`)
- `TypeName.box()` / `unbox()` convenience APIs

#### JavaPoet 1.7.0 *(2016-04-26)*
- Enclosing types support (`Outer<String>.Inner`)
- `TypeName.isBoxedPrimitive()` helper
- Fix: `TypeVariableName` self-reference stack overflow prevention

#### JavaPoet 1.8.0 *(2016-11-09)*
- Line wrapping with `$W` (Wrappable Whitespace) character
- Named arguments in code blocks (`$argumentName:L`)
- Javadoc support on `TypeSpec`, `MethodSpec`, `FieldSpec` via `addJavadoc()`
- Method comments via `addComment()`

#### JavaPoet 1.9.0 *(2017-05-13)*
- Fix: Handle anonymous inner classes in `ClassName.get()`

#### JavaPoet 1.10.0 *(2018-01-27)**
| Category | Changes |
|---|---|
| **Breaking change** | Requires Java 8 or newer |
| New | `$Z` optional newline (zero-width space) for line wrapping at 100-char boundaries |
| New | `CodeBlock.join()` and `joining()` delimiters |
| New | `CodeBlock.Builder.isEmpty()` |
| New | `addStatement(CodeBlock)` overloads for code blocks and methods |

#### JavaPoet 1.11.x *(2018)*
- Fix: Prevent `TypeName.get(ErrorType)` from masking other errors

#### JavaPoet 1.12.x *(2020)*
| Category | Changes |
|---|---|
| New | `JavaFile.writeToPath()` and `.writeToFile()` returning paths |
| New | `TypeSpec.alwaysQualify()` to avoid nested type name clashes |
| New | `CodeBlock.Builder` overloads accepting `CodeBlock`s for control flow |
| New | Mutable lists on all builder types |
| New | `CodeBlock.clear()` |
- **Charset support** (added in 1.12.0): Custom `Charset` passed to `JavaFile.writeTo(Path, Charset)`

#### JavaPoet 1.13.0 *(2020-06-18)
| Category | Changes |
|---|---|
| New | Explicit receiver parameters in methods (e.g., for interface method simulation) |

### 1.14.0-SNAPSHOT
Development version after the major rewrite completion and post-rewrite stabilization work.

## Key API Additions by Version

### Type System Enhancements
- **`TypeName.annotated()`** (1.4.0): Attach annotations to types directly (`@Nullable String`)
- **Type annotations on parameters**: Works for both top-level types and type parameters (`List<@Nullable String>`)
- **`TypeName.isBoxedPrimitive()` / `box() / unbox()`** (1.6.0): Convenience APIs for boxed/unboxed conversions
- **`TypeName.annotated()` with multiple annotations**: Supports annotation lists

### Import Mechanisms
- **Automatic imports** (1.0.0+): Types referenced in code blocks are imported automatically based on the class hierarchy
- **Static imports** (`import static`) (1.4.0): Collected from method calls and field references via `addStaticImport()`
- **Always qualify types** (`alwaysQualify` set) (1.12.0): Prevent nested type name clashes in generated code

### Encoding & File I/O
- **UTF-8 invariant** (1.4.0+): Always writes UTF-8 regardless of host charset, preventing platform-dependent encoding issues
- **Custom charset support** (1.12.0): `JavaFile.writeTo(Path, Charset)` accepts arbitrary character encodings

### Generator API Evolution
| Era | Style | Key Characteristics |
|---|---|---|
| JavaWriter 1.x–2.x | Streaming | `emit`, `addParameter`, `setCompressingTypes` |
| JavaPoet 1.0+ | Immutable builders | `MethodSpec.methodBuilder()`, `TypeSpec.classBuilder()` |

## Breaking Changes Summary

### Version 1.0.0 *(2015-01-28)*
**Major breaking change:** Requires **Java 8 or newer**. The rewrite eliminated support for Java 7 and earlier features:
- Removed int-based modifier methods (replaced with `Modifier` enum)
- Removed deprecated static import APIs from previous versions
- Rebuilt all type models

### Version 1.4.0 *(2015-11-13)*
**Major breaking change:** Int-based modifier methods removed entirely:
```diff
- addModifiers(Modifier.PUBLIC, Modifier.FINAL)  // OK - enum
+ setModifiers(0x3F)  // Deprecated → removed in 1.4.0
```

### Version 1.8.0 *(2016-11-09)*
**Behavior change:** `MethodSpec.overriding()` no longer retains annotations from overridden method:
```diff
// JavaPoet 1.7 and earlier:
MethodSpec overridden = original.overriding();  // Annotations preserved

// JavaPoet 1.8+:
MethodSpec overridden = original.overriding();  // Must add annotations separately
```

### Version 1.9.0 *(2017-05-13)*
**Bug fix:** Anonymous inner classes no longer cause infinite recursion in `TypeName.get()`.

## Deprecation Notice

> ⚠️ **As of 2020-10-10, Square's JavaPoet project is deprecated.** We're proud of our work but we haven't kept up on maintaining it. We recommend using [Palantir's JavaPoet](https://github.com/palantir/javapoet), which continues development and supports modern Java language features.

### Migration to Palantir's JavaPoet

#### Maven coordinates update
```diff
- javapoet = { module = "com.squareup:javapoet", version = "1.13.0" }
+ javapoet = { module = "com.palantir.javapoet:javapoet", version = "0.5.0" }
```

#### Import changes
Replace `com.squareup.javapoet.*` with `com.palantir.javapoet.*`:
```diff
- sed -i "" \
  's/com.squareup.javapoet.\([A-Za-z]*\)/com.palantir.javapoet.\1/g' `find . -name "*.kt" -or -name "*.java"`
```

#### Field → Method changes
Several fields became accessor methods:
```diff
- javaFile.packageName
+ javaFile.packageName()
```

## Encoding & File Operations

### UTF-8 Invariant

JavaPoet enforces **UTF-8 as the sole encoding** for all file writing operations. This is a deliberate design choice that prevents platform-dependent encoding issues:

| Operation | Encoding | Notes |
|---|---|---|
| `writeTo()` | UTF-8 | Default, always enforced |
| `writeToFile()` / `writeToPath()` | UTF-8 | Returns Path/File for confirmation |
| `writeTo(Filer)` | UTF-8 | Via `SimpleJavaFileObject` wrapper |

Custom charsets are supported via the optional parameter in `JavaFile.writeTo(Path, Charset)`, but UTF-8 is the default and recommended encoding.

## Version Tracking Reference

```
1.0.0  (2015-01-28)   Initial rewrite to JavaPoet
1.1.0  (2015-05-25)   Eager validation, varargs, DEFAULT modifier
1.2.0  (2015-07-04)   Positional arguments, enum Javadoc
1.3.0  (2015-09-20)   NameAllocator, anonymous classes
1.4.0  (2015-11-13)   Type annotations, static imports (BREAKING: int modifiers removed)
1.5.x  (2016)         Performance improvements
1.6.0  (2016-03-19)   CodeBlock factory, type bounds
1.7.0  (2016-04-26)   Enclosing types
1.8.0  (2016-11-09)   Line wrapping ($W), named arguments (BREAKING: overridng annotations)
1.9.0  (2017-05-13)   Anonymous inner class fix
1.10.0 (2018-01-27)   $Z newline, CodeBlock joining (BREAKING: Java 8 required)
1.11.x (2018)         TypeName.get(ErrorType) regression fix
1.12.x (2020)         Path-returning writes, alwaysQualify, mutable builders, charset support
1.13.0 (2020-06-18)   Explicit receiver parameters
1.14.0-SNAPSHOT (2020+) Development version after major rewrite completion
```

---

**Status:** [Deprecated — use Palantir's JavaPoet](https://github.com/palantir/javapoet)

**Related pages:**
- [JavaPoet Deprecation & Migration to Palantir's JavaPoet](/openwiki/integrations/deprecation.md)
<!-- openwiki: broken internal link [/openwiki/integrations/maven.md] file "/openwiki/integrations/maven.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [Maven integration for JavaPoet](/openwiki/integrations/maven.md)
<!-- openwiki: broken internal link [/openwiki/operations/building-projects.md] file "/openwiki/operations/building-projects.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [Building projects with JavaPoet](/openwiki/operations/building-projects.md)

**Version history:** See [CHANGELOG.md](http://github.com/square/javapoet/blob/master/CHANGELOG.md) for the complete changelog.
