---
type: tooling
title: Getting Started
description: Learn JavaPoet's core concepts through its main entry types and APIs. This page explains the primary data models, code emission pipeline, and formatting conventions used throughout the library.
tags: [java-poet, code-generation, getting-started]
verified:
  - by: openwiki/0.5.2
    at: 2026-09-22T02:45:45.818Z
sources:
  - id: openwiki-source-2355f81d7cf522f8dbdaabd4
    resource: repo://pom.xml
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
  - id: openwiki-source-d96470ebbdcca258dc9bf91a
    resource: repo://src/main/java/com/squareup/javapoet/CodeWriter.java
  - id: openwiki-source-9607265cd9da174456c1f38d
    resource: repo://src/main/java/com/squareup/javapoet/JavaFile.java
  - id: openwiki-source-1205045a5bab72d4678fc6da
    resource: repo://src/main/java/com/squareup/javapoet/TypeSpec.java
generated: { by: "openwiki/0.5.2", at: "2026-09-22T02:45:45.818Z" }
---

# Getting Started

JavaPoet is a **Java API for generating `.java` source files**. It lets you build Java classes, methods, fields, annotations, and constructors programmatically — without writing boilerplate or dealing with syntax trees directly.

The library is organized around a small set of core types that work together to produce valid, well-formatted Java source code:

- **[TypeSpec](/openwiki/concepts/types.md)** — The primary domain model for classes, interfaces, and enums
<!-- openwiki: broken internal link [/openwiki/workflows/code-synthesis.md] file "/openwiki/workflows/code-synthesis.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- **[MethodSpec](/openwiki/workflows/code-synthesis.md)** — Methods, constructors, and static blocks built with parameter resolution
<!-- openwiki: broken internal link [/openwiki/concepts/files.md] file "/openwiki/concepts/files.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- **[JavaFile](/openwiki/concepts/files.md)** — File system operations that handle package-based directory creation, UTF-8 enforcement, and import emission ordering

## Code Format Strings

Most of JavaPoet's API uses string templates rather than AST nodes. The template syntax is inspired by `String.format()` but with its own placeholders:

| Placeholder | Emits | Example |
|---|---|---|
| `$L` | **Literal** value (no escaping) | `.addStatement("int total = $L", 42)` → `int total = 42;` |
| `$S` | String with quotes and escaping | `.addStatement("return $S", "hello")` → `return "hello";` |
| `$T` | **Type name** (triggers import) | `.addStatement("return new $T()", Date.class)` → emits the class + import |
| `$N` | Another generated declaration's name | `.addStatement("$T.sort($N, $L)", Collections.class, strings)` |

## The Emission Pipeline

### 1. Build Your Domain Model

Start with `TypeSpec`, which represents a Java type (class, interface, or enum):

```java
MethodSpec main = MethodSpec.methodBuilder("main")
    .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
    .returns(void.class)
    .addParameter(String[].class, "args")
    .addStatement("$T.out.println($S)", System.class, "Hello, JavaPoet!")
    .build();

TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
    .addMethod(main)
    .build();
```

### 2. Assemble a JavaFile

Wrap your type in a `JavaFile` and configure options:

```java
JavaFile javaFile = JavaFile.builder("com.example.helloworld", helloWorld)
    .skipJavaLangImports(true)
    .build();
```

- **Package-aware directory creation**: `writeTo()` creates the full package hierarchy automatically.
- **UTF-8 enforcement**: All files are written in UTF-8 by default.
- **NullAppendable pattern**: The first-pass import collector uses a no-op output, so imports can be computed without writing anything to disk.

### 3. Emit the Source

```java
javaFile.writeTo(System.out);          // Write to stdout
javaFile.writeToFile("/tmp/hello.java"); // Write to file system
String source = javaFile.toString();    // Get as string
```

## Key Concepts

- **[Code Writer Two-Pass Algorithm](/openwiki/architecture/emission.md)** — The emitter collects all needed imports in a first pass (using the `nullAppendable` pattern), then writes clean code with resolved imports in a second pass. Line wrapping (`$W`) and zero-width newlines (`$Z`) control formatting within multi-line statements.

- **[Name Allocation & Qualification](/openwiki/architecture/naming.md)** — The `NameAllocator` ensures unique identifiers throughout nested types, propagates always-qualified names to prevent collisions with local declarations, and resolves wildcard type parameters without ambiguity.

<!-- openwiki: broken internal link [/openwiki/architecture/failure-handling.md] file "/openwiki/architecture/failure-handling.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- **[Failure Handling](/openwiki/architecture/failure-handling.md)** — JavaPoet performs eager argument validation (`checkArgument`, `checkState`) at builder construction time. Recursive type variable resolution is protected against stack overflow, and malformed reflection metadata is handled gracefully with informative errors.

## Quick Examples

### Generating a class with fields and methods:

```java
FieldSpec android = FieldSpec.builder(String.class, "android")
    .addModifiers(Modifier.PRIVATE, Modifier.FINAL)
    .build();

MethodSpec main = MethodSpec.methodBuilder("main")
    .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
    .addsParameter(android)
    .addStatement("$N.set($L)", android, "Hello")
    .build();

TypeSpec app = TypeSpec.classBuilder("App")
    .addField(android)
    .addMethod(main)
    .build();

JavaFile.builder("com.example.app", app).build().writeTo(System.out);
```

### Generating code with static imports:

```java
ClassName hoverboard = ClassName.get("com.mattel", "Hoverboard");
MethodName createNimbus = TypeName.get(hoverboard, "createNimbus");

MethodSpec beyond = MethodSpec.methodBuilder("beyond")
    .addStatement("$T result = new $T<>()", listOfHoverboards)
    .addStatement("result.add($T.createNimbus(2000))", hoverboard)
    .build();

JavaFile.builder("com.example.helloworld", helloWorld)
    .addStaticImport(hoverboard, "createNimbus")
    .build()
    .writeTo(System.out);
```

## Architecture Overview

The library follows a layered architecture: reflection metadata ingestion → `TypeSpec`/`MethodSpec`/`AnnotationSpec` domain models → `CodeWriter` emission engine → `JavaFile` file system operations. Each layer is independently testable and has a single responsibility, making the codebase easy to extend and maintain.

## Common Pitfalls

- **Import resolution**: Types not visible in the current scope will trigger imports, but nested types (like `java.util.Map.Entry`) may require explicit qualification. Use `alwaysQualify` to prevent accidental conflicts.
- **Type variable collisions**: When emitting code with type variables, ensure you're using `$T` correctly; unbound wildcards can produce ambiguous output if not handled by the `NameAllocator`.
- **Empty packages**: Package paths starting with empty strings or trailing dots will cause directory creation failures in `JavaFile.writeTo()`.

## Next Steps

<!-- openwiki: broken internal link [/openwiki/concepts/code-generation.md] file "/openwiki/concepts/code-generation.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- **[Code Generation & Emission](/openwiki/concepts/code-generation.md)** — Deep dive into how `JavaFile` and `CodeWriter` transform type specs into valid Java source, including import resolution and formatting.
<!-- openwiki: broken internal link [/openwiki/architecture/build-flow.md] file "/openwiki/architecture/build-flow.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- **[Build Flow & Pipeline](/openwiki/architecture/build-flow.md)** — Trace the complete build pipeline: checkout → Maven compile/test → GitHub Actions CI → Sonatype publish.
<!-- openwiki: broken internal link [/openwiki/integrations/config.md] file "/openwiki/integrations/config.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- **[Configuration & Build Settings](/openwiki/integrations/config.md)** — Understand `pom.xml` properties, Maven plugins, and CI environment variables that configure the JavaPoet build lifecycle.

---

*JavaPoet is licensed under Apache 2.0.*
