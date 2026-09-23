---
type: component
title: Method & Constructor Generation
description: Documents the JavaPoet MethodSpec API for building methods, constructors, and static blocks with parameter resolution, return types, code snippet templates, and emitting to a CodeWriter.
tags: [javapoet, method-spec, code-generation, reflection, formatting]
verified:
  - by: openwiki/0.5.2
    at: 2026-09-22T02:45:45.818Z
sources:
  - id: openwiki-source-f613968dbd6aa3fcd0bacbfa
    resource: repo://src/main/java/com/squareup/javapoet/MethodSpec.java
  - id: openwiki-source-7d7f82dfa4ee7e1c6baff3c9
    resource: repo://src/main/java/com/squareup/javapoet/ParameterSpec.java
generated: { by: "openwiki/0.5.2", at: "2026-09-22T02:45:45.818Z" }
---

# Method & Constructor Generation

The `MethodSpec` class in the JavaPoet library is the central abstraction for generating method and constructor declarations during source code generation. It encapsulates all the metadata required to emit a complete method or constructor: name, modifiers, return type, parameters, annotations, exceptions, type variables, javadoc, inline code body, default values, and control flow constructs.

## Internal Representation

A `MethodSpec` instance consists of immutable fields that together represent a single method or constructor declaration:

| Field | Type | Description |
|---|---|---|
| `name` | `String` | The method name (`"<init>"` for constructors) |
| `javadoc` | `CodeBlock` | Javadoc documentation block |
| `annotations` | `List<AnnotationSpec>` | Annotations applied to the method/constructor |
| `modifiers` | `Set<Modifier>` | Java modifiers (public, private, static, abstract, etc.) |
| `typeVariables` | `List<TypeVariableName>` | Generic type parameters |
| `returnType` | `TypeName` | Return type (`null` for constructors) |
| `parameters` | `List<ParameterSpec>` | Parameter declarations in order |
| `varargs` | `boolean` | Whether the method uses varargs syntax |
| `exceptions` | `List<TypeName>` | Thrown exception types (in declaration order) |
| `code` | `CodeBlock` | The inline code body template |
| `defaultValue` | `CodeBlock` | Default value for static fields, optional parameters (Kotlin), or method defaults |

The spec is constructed via the fluent `Builder` pattern and becomes immutable upon calling `.build()`.

## Builder API for Construction

Methods and constructors are created using factory methods on the `MethodSpec.Builder`:

```
MethodSpec.methodBuilder(String name)      // Method with a named body
MethodSpec.constructorBuilder()            // Constructor (name defaults to "<init>")
```

### Builder Methods Summary

| Builder Method | Purpose |
|---|---|
| `addJavadoc(...)` | Add javadoc comment block |
| `addAnnotations(...)` | Add one or more annotation specs |
| `addModifiers(Modifier...)` | Set Java modifiers (public, static, abstract, etc.) |
| `addTypeVariables(Iterable<TypeVariableName>)` | Add generic type parameters |
| `returns(TypeName)` / `returns(Type)` | Set return type; illegal for constructors |
| `addParameters(...)` / `addParameter(...)`: | Add parameter declarations with types and names |
| `varargs()` | Mark as varargs method (last param must be an array) |
| `addExceptions(Iterable<TypeName>)` / `addException(TypeName)` | Declare thrown exceptions |
| `addCode(...)` / `addNamedCode(...)` / `beginControlFlow(...)` | Add inline code body with format tokens |
| `defaultValue(...)` | Provide a default value for the method/constructor |

All builder methods return the builder instance to enable fluent chaining.

### Constructing Parameters

Parameters are added using either pre-built `ParameterSpec` objects or directly via type/name/modifier arguments:

```java
// Named parameters with optional javadoc
methodBuilder("getTaco")
    .addParameter(TypeName.DOUBLE, "money")
    .addParameter(ParameterSpec.builder(TypeName.INT, "count")
        .addJavadoc("the number of Tacos to buy.\n").build())
```

### Constructing Annotations

Annotations can be added as `AnnotationSpec` objects, by class name, or by class reference:

```java
methodBuilder("foo")
    .addAnnotation(Override.class)
    .addAnnotation(SuppressWarnings.class)
    // or
    .addAnnotations(Arrays.asList(Override.class, SuppressWarnings.class))
```

## Emitting Code

When `MethodSpec.emit(CodeWriter codeWriter, String enclosingName, Set<Modifier> implicitModifiers)` is called, the spec produces a complete method/constructor declaration in the following order:

1. **Javadoc** — emitted as-is if present
2. **Annotations** — each annotation emits using its own format tokens (e.g., `@NonNull`, `@Override`)
3. **Modifiers** — visibility, static, abstract flags
4. **Type variables** — generic parameter declarations (`<T, V>`)
5. **Return type and name** — for constructors: `$L($Z);` (where `$Z` is the class name); for methods: `$T $L($Z)` (return type + method name)
6. **Parameters** — each emits its annotations, modifiers, type (`$T`), and name (`$L`) with comma separation between them. Varargs parameters emit an array type via `TypeName.asArray()`.
7. **Closing parenthesis** — `)`
8. **Default value** (if present) — `default <codeBlock>`
9. **Throws clause** (if exceptions exist) — `throws Exception1, Exception2`
10. **Body or terminator**:
    - **Abstract methods / native methods without code**: semicolon and newline (`;\n`)
    - **Native methods with GWT JSNI support**: inline code block followed by `;\n`
    - **All other methods/constructors**: `{\n<code body>\n}\n`

The `CodeWriter` manages indentation, control flow nesting, trailing newlines, and string literal escaping. It also tracks static imports, type variable scope, and statement-line tracking to produce syntactically correct Java source code.

## Overriding Methods

The `MethodSpec.overriding(ExecutableElement method)` static factory provides a convenience for implementing interfaces by copying metadata from a reflected method:

```java
ExecutableElement parentMethod = findFirst(methodsIn(TypeElement.class));
MethodSpec spec = MethodSpec.overriding(parentMethod).build();
```

This builder copies visibility modifiers, type parameters, return type, name, parameter types, and throws declarations. It adds an `@Override` annotation automatically. Notably:

- Annotations from the parent method are **not** copied (they must be added separately)
- Default modifiers (`default`) are removed
- The overridden method's modifier set cannot include `private`, `static`, or `final`

### Parameterized Overriding with Type Resolution

When overriding a generic interface method in a concrete type, the `overriding(ExecutableElement method, DeclaredType enclosing, Types types)` variant resolves all type parameters to their actual types:

```java
// Override Comparable<Movie>compareTo() where Movie is a concrete class
MethodSpec spec = MethodSpec.overriding(method, classType, types)
    .addStatement("return 0")
    .build();
```

This produces properly resolved return types and parameter types instead of generic placeholders.

## Invariants & Validation Rules

The `MethodSpec` enforces several Java language invariants during construction:

| Invariant | Enforcement |
|---|---|
| **Name is never null** | Checked in constructor and builder |
| **Constructors cannot have return types** | `checkState(!name.equals(CONSTRUCTOR), "constructor cannot have return type")` in `returns()` |
| **Varargs last param must be an array** | Checked during `build()`: `lastParameterIsArray()` verifies the final parameter type is an array |
| **Abstract methods cannot contain inline code** | Constructor validates: `checkArgument(code.isEmpty() || !builder.modifiers.contains(Modifier.ABSTRACT), "abstract method %s cannot have code", builder.name)` |
| **Invalid modifier combinations on override** | `overriding()` throws if the parent method has `private`, `static`, or `final` modifiers |

## Relationship Diagram

```mermaid
graph TD
    Reflection -->|get() / overriding()| Builder
    Builder -->|build()| MethodSpec
    MethodSpec -->|emit()| CodeWriter
    CodeWriter --> LineWrapper
    
    MethodSpec --> TypeName
    TypeName --> ClassName
    TypeName --> TypeMirror
    ClassName --> Modifier
```

- **`Reflection`** (Java runtime `Annotation` / `ExecutableElement`) is the source of truth for annotation values and method signatures.
- **`Builder`** constructs a mutable spec by reflecting members, selecting format tokens, and accumulating metadata.
- **`MethodSpec`** is the immutable representation consumed by the emitter.
- **`CodeWriter`** uses `$T` (the type name) and `$L` (nested annotations) format tokens to emit the annotation in context-aware syntax.

## Key Tests That Matter

The following test cases in `MethodSpecTest` cover the essential behaviors:

1. **Null argument rejection**: Verifies that null values for any builder collection parameter throw `IllegalArgumentException`.
2. **Default value filtering**: Confirms that `get(Annotation)` omits members matching their defaults, while `get(annotation, true)` includes them — analogous to constructor/parameter default behavior.
3. **Nested annotation emission**: Ensures recursive `$L` expansion produces correct nested annotation syntax (e.g., `@NonNull(@Required())`).
4. **Array member handling**: Validates that array-valued members are expanded into individual entries rather than emitting raw array literals.
5. **Builder mutation**: Tests that `toBuilder()` returns a fully mutable builder with all existing members, and that members can be added or replaced.
6. **Duplicate exceptions ignored**: Confirms the spec deduplicates exception types (e.g., adding `IOException` twice results in one declaration).
7. **Invalid parameter name rejection**: Validates that non-identifier names throw during construction.
8. **Javadoc with/without method javadoc**: Tests that parameter javadocs are emitted after the method javadoc newline, but only if present.
9. **Control flow emission**: Confirms `beginControlFlow`, `nextControlFlow`, and `endControlFlow` produce correct nested block syntax.
10. **Trailing newline handling**: Ensures code blocks without trailing newlines receive one, while existing newlines are not duplicated.

## Parameter Specification Details

The `ParameterSpec` class represents individual parameters within a method/constructor declaration:

| Field | Type | Description |
|---|---|---|
| `name` | `String` | Parameter name (must be a valid Java identifier, or `"this"`) |
| `annotations` | `List<AnnotationSpec>` | Annotations applied to the parameter |
| `modifiers` | `Set<Modifier>` | Parameter modifiers (only `FINAL` is allowed for parameters) |
| `type` | `TypeName` | Parameter type |
| `javadoc` | `CodeBlock` | Per-parameter javadoc (`@param`) |

When emitted, a parameter produces: `[annotations] [modifiers] TypeName $L name`. The varargs flag on the parent method determines whether the type is expanded to an array.

## Relationship with Related Concepts

- **`AnnotationSpec`** — Used as member values for parameters and annotations; constructed via `AnnotationSpec.builder(TypeName)` or reflected from runtime annotations.
- **`TypeName`** — Identifies any Java type (primitives, classes, generics) used in return types, parameter types, and exception declarations. Supports format tokens like `$T` and `$L`.
- **`Modifier`** — Java language modifiers (`public`, `private`, `static`, `abstract`, etc.) that control visibility and inheritance semantics of methods/constructors.
- **`CodeWriter`** — The emitter that converts a `MethodSpec` into syntactically correct Java source code, handling indentation, imports, and control flow nesting.

## Notes on Type-Level Annotations

Annotations can themselves appear as parameter values in other annotations (e.g., `@NonNull(@Required())`). The builder supports this through recursive reflection via `AnnotationSpec.get()`, which produces `$L` format tokens that the CodeWriter resolves to nested annotation syntax at emission time. This enables first-class support for annotations that take other annotations as parameters.

---
