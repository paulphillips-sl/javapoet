---
type: component
title: Annotation Spec (Java Annotation Representation)
description: Documents the AnnotationSpec class in Javapoet, which represents a Java annotation's type and its member values, and controls how annotations are emitted as source code.
tags: [javapoet, annotation-spec, code-generation, reflection, formatting]
verified:
  - by: openwiki/0.5.2
    at: 2026-09-22T02:45:45.818Z
sources:
  - id: openwiki-source-4b55df65b812be4a63b07748
    resource: repo://src/main/java/com/squareup/javapoet/AnnotationSpec.java
generated: { by: "openwiki/0.5.2", at: "2026-09-22T02:45:45.818Z" }
---

# Annotation Spec (Java Annotation Representation)

The `AnnotationSpec` class in the Javapoet library is the central abstraction for representing Java annotations during code generation. It encapsulates both the **type** of an annotation and the **values** assigned to its members (parameters), providing a canonical representation that can be consumed by the CodeWriter emitter.

## Internal Representation

An `AnnotationSpec` instance consists of two immutable fields:

| Field | Type | Description |
|---|---|---|
| `type` | `TypeName` | The annotation's type as a `TypeName`. This is never null. |
| `members` | `Map<String, List<CodeBlock>>` | A `LinkedHashMap` mapping each member name to an ordered list of `CodeBlock` values. Empty maps produce no-member annotations (e.g., `@Singleton`). |

The `members` map is backed by a `Multiset` internally for immutability guarantees:

```java
private AnnotationSpec(Builder builder) {
    this.type = builder.type;
    this.members = Util.immutableMultimap(builder.members);
}
```

## Builder API for Construction

Annotations are constructed via the fluent `Builder` pattern. The builder accumulates members in insertion order and produces an immutable spec on `build()`.

### Static Factory Methods

| Method | Purpose |
|---|---|
| `get(Annotation)` | Reflects a runtime annotation to produce an `AnnotationSpec`, excluding default values by default. |
| `get(Annotation, boolean includeDefaults)` | Same as above but also includes members whose values match their defaults. Useful for round-tripping or diffing. |
| `get(AnnotationMirror)` | Constructs from reflection metadata without invoking methods; preserves all declared values. |

### Builder Methods

```java
Builder builder(ClassName type);        // From a known annotation class
Builder builder(Class<?> type);           // Convenience wrapper above
Builder addMember(String name, String format, Object... args);  // Format token + arguments
Builder addMember(String name, CodeBlock codeBlock);         // Pre-formatted code block
Builder addMemberForValue(String memberName, Object value);  // Auto-detects format token from value type
```

The `addMemberForValue()` method inspects the Java runtime type of the provided argument and selects the appropriate format token:

| Value Type | Format Token | Example |
|---|---|---|
| `Class<?>` | `$T.class` | `@NonNull(java.lang.String.class)` |
| Enum constant | `$T.$L` | `@Priority(java.lang.EnumConstant.CONSUMER, Priority.HIGH)` |
| `String` | `$S` | `@Name("foo")` |
| Primitive (`float`, `long`, etc.) | `$Lf`, `$LL`, etc. | `@Size(1024L)` |
| Nested annotation | `$L` (recursive) | `@NonNull(@Required)` |

### `toBuilder()`

An existing spec can be converted back into a mutable builder for adding or modifying members:

```java
AnnotationSpec spec = AnnotationSpec.builder(SomeAnnotation.class).build();
AnnotationSpec.Builder updated = spec.toBuilder();
updated.addMember("newKey", "$S", "value");
```

## Emission Format Tokens

The `emit()` method renders an annotation in one of two modes controlled by the `inline` parameter:

### Inline Mode (`inline == true`)

A single-line emission where all member values are concatenated with comma separation:

```java
// Input spec:
// @NonNull(@Required("some")(), @Size(1024L))

// Output:
@NonNull(@Required("some"), @Size(1024L))
```

The format is: `@$T(arguments)` where `$T` expands to the annotation's type (e.g., `javax.annotation.Nullable`) and arguments are emitted via `emitAnnotationValues()`.

### Non-Inline Mode (`inline == false`)

A multi-line emission that wraps member values across multiple lines when there are multiple members or complex values:

```java
// Input spec with many nested annotations
@NonNull(
    @Required("some"),
    @Size(1024L),
    @NonNull()
)
```

The structure is:

```
@$T(
    member1 = value1,
    member2 = value2,
)
```

- **Single member with a single value**: collapses to one line (`@Named("foo")`).
- **Multiple members or complex values**: expands across lines with each member on its own line.

## Member Value Rendering

When emitting values for a specific member, `emitAnnotationValues()` distinguishes three cases:

1. **Single element array / single value** (e.g., `@Size(1024L)`):
   - No curly braces are emitted.
   - The value is written directly after the member name and equals sign.

2. **Multiple values for a single member**:
   - Curly braces wrap the comma-separated list: `{value1, value2}`.
   - Used for array-valued members like `@PackageValue("com", "example")`.

3. **Nested annotation members** (e.g., `@NonNull(@Required)`):
   - The inner annotation is recursively emitted via `$L` format token resolution.

## Reflection-Based Construction (`AnnotationSpec.get()`)

The static factory method `get(Annotation)` inspects a runtime annotation using Java reflection:

### Process

1. **Retrieve declared methods**: Uses `annotation.annotationType().getDeclaredMethods()` sorted by name to ensure deterministic output ordering.
2. **Invoke each member**: Calls `method.invoke(annotation)` to retrieve the actual value.
3. **Filter defaults (when `includeDefaults == false`)**: Compares the invoked value against the method's default via `Objects.deepEquals(value, method.getDefaultValue())`. Default-valued members are omitted from the output spec.
4. **Handle special types**:
   - Arrays are expanded into individual values (one entry per array element).
   - Nested annotations trigger recursive calls to `get()`, producing `$L` format tokens.
5. **Exception handling**: Any reflection failure during invocation wraps the exception in a `RuntimeException`.

### Inclusion of Default Values

When `includeDefaults` is `true`, members whose values match their defaults are included in the spec. This enables:
- Round-tripping from source code back to a spec (since default values would otherwise be lost).
- Diffing between two specs when you need to account for defaults.

## Invariants & Error Conditions

The AnnotationSpec and its Builder enforce several invariants:

| Invariant | Enforcement |
|---|---|
| `type` is never null | Checked at builder construction via `checkNotNull(type)` |
| Member names are valid Java identifiers | Validated by `SourceVersion.isName()`; throws `IllegalArgumentException` for invalid names like `"@"` or `""`. |
| Member names are non-null | Checked in `addMemberForValue()` and `build()`. |
| Spec immutability after construction | The constructor creates an immutable view of the builder's members map via `Util.immutableMultimap()`. |

## Key Tests That Matter

The following test cases in `AnnotationSpecTest` cover the essential behaviors:

1. **Default value filtering**: Verifies that `get(Annotation)` omits members matching their defaults, and `get(annotation, true)` includes them.
2. **Nested annotation emission**: Confirms recursive `$L` expansion produces correct nested annotation syntax (e.g., `@NonNull(@Required())`).
3. **Array member handling**: Validates array-valued members are expanded into individual entries rather than emitting raw array literals.
4. **Dynamic enum arrays**: Ensures multiple enum constants for the same member produce comma-separated values within curly braces.
5. **Builder mutation**: Tests that `toBuilder()` returns a fully mutable builder with all existing members, and that members can be added or replaced.
6. **Format token dispatch**: Covers each value type (class, enum, string, primitives, nested annotations) to ensure the correct format token is selected.

## Relationship Diagram

```mermaid
graph TD
    Reflection -->|get()| Builder
    Builder -->|build()| AnnotationSpec
    AnnotationSpec -->|emit()| CodeWriter
    CodeWriter --> LineWrapper
    
    AnnotationSpec --> TypeName
    TypeName --> ClassName
    ClassName --> TypeMirror
```

- **`Reflection`** (Java runtime `Annotation`) is the source of truth for annotation values.
- **`Builder`** constructs a mutable spec by reflecting members and selecting format tokens.
- **`AnnotationSpec`** is the immutable representation consumed by the emitter.
- **`CodeWriter`** uses `$T` (the type) and `$L` (nested annotations) format tokens to emit the annotation in context-aware syntax.

## Notes on Type-Level Annotations

The AnnotationSpec can itself appear as a member value of another annotation. For example, `@NonNull(@Required())` has an outer annotation whose member is a nested `AnnotationSpec`. The builder supports this through recursive `get()` calls that produce `$L` format tokens in the emitted code. This enables first-class support for annotations that take other annotations as parameters.

---
