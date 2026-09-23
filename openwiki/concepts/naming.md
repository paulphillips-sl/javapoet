---
type: system-concept
title: Naming & Qualification
description: Name resolution strategies (NameAllocator, always-qualified names, wildcard type parameters) that prevent identifier clashes in generated code
tags: [code-generation, types, naming]
verified:
  - by: openwiki/0.5.2
    at: 2026-09-22T02:45:45.818Z
sources:
  - id: openwiki-source-8a6417773878e412bc92fe3f
    resource: repo://src/main/java/com/squareup/javapoet/ClassName.java
  - id: openwiki-source-e41f79722fff1695afaff477
    resource: repo://src/main/java/com/squareup/javapoet/CodeBlock.java
  - id: openwiki-source-d96470ebbdcca258dc9bf91a
    resource: repo://src/main/java/com/squareup/javapoet/CodeWriter.java
  - id: openwiki-source-8d05a090e29b6eb95539f387
    resource: repo://src/main/java/com/squareup/javapoet/NameAllocator.java
  - id: openwiki-source-f1d9e07cd5f19dfbe87f3c4f
    resource: repo://src/main/java/com/squareup/javapoet/ParameterizedTypeName.java
  - id: openwiki-source-ca139b909129ba6056b11115
    resource: repo://src/main/java/com/squareup/javapoet/TypeName.java
  - id: openwiki-source-39bd05157a2b7fc932b39c99
    resource: repo://src/main/java/com/squareup/javapoet/TypeVariableName.java
  - id: openwiki-source-dff2e4f0656f620a39c2d4f8
    resource: repo://src/main/java/com/squareup/javapoet/WildcardTypeName.java
generated: { by: "openwiki/0.5.2", at: "2026-09-22T02:45:45.818Z" }
---

The javapoet library uses a layered naming and qualification system to generate valid Java source code. This page documents the mechanisms that ensure generated identifiers are unique within each scope, properly qualified for references, and correctly handle type parameters including wildcards.

## NameAllocator

### Responsibility

`NameAllocator` is responsible for assigning Java identifier names to all code elements during generation. It prevents three classes of problems:

1. **Identifier collisions**: When multiple elements require the same name (e.g., two fields or methods in different scopes), the allocator appends underscores until a unique name is produced.
2. **Java keywords**: Generated names that would conflict with reserved keywords like `public`, `class`, or `static` receive suffixes to become valid identifiers.
3. **Invalid characters**: Characters that are not allowed in Java identifiers (e.g., `-`, `space`, `_` at the start, digits) are replaced with underscores.

### Allocation Model

The allocator operates on a per-tag basis, where each tag uniquely identifies an element being named:

```java
NameAllocator nameAllocator = new NameAllocator();
nameAllocator.newName("foo", property);      // returns "foo"
nameAllocator.newName("bar", otherProperty);  // returns "bar"
// Re-allocating with the same tag gets a suffixed variant:
nameAllocator.newName("foo", property);       // returns "foo_"
```

The allocator maintains two data structures:

| Field | Type | Purpose |
|---|---|---|
| `allocatedNames` | `LinkedHashSet<String>` | Tracks all names that have been issued (prevents reuse) |
| `tagToName` | `Map<Object, String>` | Maps each tag to its allocated name for lookup via `get(tag)` |

### Collision Resolution Algorithm

When `newName(suggestion, tag)` is called:

1. Convert the suggestion string into a valid Java identifier using `toJavaIdentifier()`, which:
   - Prefixes with `_` if the first character is not a valid identifier start but is a valid part
   - Replaces any non-identifier-part characters with `_`
2. If the resulting name is a Java keyword or already allocated, append `_` and retry (up to repeated suffixing)
3. Store the mapping in `tagToName`; if the tag already exists, throw `IllegalArgumentException`

### Tag Reuse Semantics

Each tag can be associated with exactly one allocated name. If a tag is used twice for different suggestions, the second allocation throws:

```java
throw new IllegalArgumentException("tag " + tag + " cannot be used for both '" + replaced + "' and '" + suggestion + "'");
```

This invariant ensures that tags act as stable lookups throughout the lifetime of the allocator.

### Cloning for Nested Scopes

`NameAllocator.clone()` creates a deep copy with its own independent `allocatedNames` set:

- Cloned allocators share the same tag→name mapping (read-only semantics from the original)
- Each clone maintains separate allocation tracking, so names that collide in the parent are independently resolved in each child scope
- This enables correct name generation for inner classes and methods without global conflicts

```java
NameAllocator outer = new NameAllocator();
NameAllocator inner1 = outer.clone();  // independent of other clones
NameAllocator inner2 = outer.clone();
```

---

## TypeName Hierarchy

The `TypeName` abstract class represents any valid Java type reference, including primitives, `void`, and composite types. Its subclasses form the qualification hierarchy:

```
TypeName (abstract)
├── ClassName              – fully-qualified class references
├── PrimitiveTypeName      – int, double, char, etc.
│   └── Void               – static final instance for "void"
├── ParameterizedTypeName  – GenericType<T1, T2> patterns
├── TypeVariableName       – bound type parameters (e.g., <T extends String>)
└── WildcardTypeName       – ? super X / ? extends Y patterns
```

### TypeName Base Class

`TypeName` is an immutable value object that:

- Implements `equals()` and `hashCode()` based on the string representation of the type
- Supports annotation attachment via `annotated(AnnotationSpec...)`
- Provides conversion methods:
  - `box()` / `unbox()`: round-trip between primitives and their boxed counterparts
  - `withoutAnnotations()` / `isAnnotated()`: strip or check for annotations

### ClassName

`ClassName` represents fully-qualified class references with the following fields:

| Field | Purpose |
|---|---|
| `packageName` | Package name (empty string if default package) |
| `enclosingClassName` | Parent class if nested (null if top-level) |
| `simpleName` | Local name of the class |
| `canonicalName` | Full qualified name like `"java.util.Map.Entry"` |
| `reflectionName` | Binary form for JVM reflection like `"com.example.MyClass$123"` |

Key operations:

- `nestedClass(name)`: Creates a new ClassName representing an inner class with the given local name
- `topLevelClassName()`: Walks up the enclosing chain to find the root class
- `peerClass(name)`: Creates a class at the same level as this one (same package/enclosing)

### ParameterizedTypeName

`ParameterizedTypeName` represents generic types like `<T>`, `<List<String>>`, or nested generics inside a parent type.

Structure:
- `rawType` – The base class name (`ClassName`)
- `typeArguments` – List of `TypeName` arguments (cannot be primitives or `void`)
- `enclosingType` – Parent parameterized type if this is a nested generic (e.g., `Map.Entry<K,V>`)

The constructor enforces:
```java
checkArgument(!typeArguments.isEmpty() || enclosingType != null, "no type arguments");
for each argument: checkArgument(!primitive && !void);
```

Key operations:
- `nestedClass(name)`: Creates a nested inner class with the given name
- `get(rawType, args...)`: Static factory method to build a parameterized type

### TypeVariableName

`TypeVariableName` represents bound type parameters like `<T extends String>` or `<K,V>`.

Structure:
- `name` – The formal parameter name (e.g., `"T"`)
- `bounds` – List of upper and lower bounds (primitives and `void` are not allowed as bounds)

Key operations:
- `withBounds(bounds...)`: Adds additional bounds to the type variable
- `get(name, ...)`: Static factory with bound support
- `get(mirror)` / `get(typeVariable)`: Conversion from Java reflection types
- `of(name, bounds)`: Internal helper that strips `java.lang.Object` from bounds (since `<T extends Object>` is equivalent to bare `<T>`)

### WildcardTypeName

`WildcardTypeName` represents wildcard type references like `? super String` or `? extends CharSequence`.

Structure:
- `upperBounds` – Exactly one upper bound class (e.g., `String` for `? super String`)
- `lowerBounds` – Lower bounds list (e.g., `[Object]` for `? super Object`, which becomes shorthand `?`)

The constructor enforces:
```java
checkArgument(this.upperBounds.size() == 1, "unexpected extends bounds");
for each bound: checkArgument(!primitive && !void);
```

Emit behavior:
- If lower bound is empty and upper bound is `Object`, emits `?` (shorthand for `? extends Object`)
- Otherwise emits `? extends UpperBound` or `? super LowerBound`

Key operations:
- `subtypeOf(upperBound)`: Creates `? extends X` wildcard
- `supertypeOf(lowerBound)`: Creates `? super Y` wildcard
- `get(mirror)` / `get(typeVariable)`: Conversion from Java reflection types

---

## CodeWriter Type Emission

The `CodeWriter` class emits type references using the `$T` placeholder. The key mechanism is **automatic import resolution**:

When emitting a `ClassName` reference:
1. If `TypeName.emit()` detects that the next format token will be handled by the default case (not another `$...`), it defers emission
2. It checks whether the type can be statically imported based on the current `staticImports` set
3. If eligible, it emits a static import statement and then emits the member reference directly

This allows patterns like:
```java
// Format string: "$T is the type"
CodeWriter.emit(TypeName.get(String.class), ...);
// Result: "import String; String is the type"
// instead of: "String is the type" (which would require a fully-qualified name)
```

The placeholder system uses these tokens:
| Token | Emits | Notes |
|---|---|---|
| `$L` | Literal value, no escaping | Can be strings, primitives, types, annotations |
| `$N` | Name with collision avoidance | Uses `NameAllocator.get(tag)` internally |
| `$S` | String literal (wrapped in quotes) | Escapes internal double quotes as `\"` |
| `$T` | Type reference (with imports if possible) | Can be classes, type mirrors, elements |
| `$$$` | Literal dollar sign | Used for escaping `$` in format strings |

---

## Qualification Strategies

### Always-Qualified Names

When emitting a name via `$N`, the emitted text is the **fully-qualified** identifier returned by `NameAllocator.get(tag)`. This ensures:
- Inner classes are fully qualified (e.g., `Outer.Inner`)
- Nested methods carry their enclosing class context
- Generated code compiles without additional imports for nested references

The qualifier propagates through the type hierarchy: a `ParameterizedTypeName`'s `enclosingType` is emitted first, followed by a dot and the raw name. This produces correct nesting like `Map.Entry<K,V>` rather than just `Entry`.

### Wildcard Type Parameters

Wildcard types provide a way to express **unknown but bounded** types in generated code:

- `? super X` – an unknown type that is a supertype of X
- `? extends Y` – an unknown type that extends Y
- `?` – shorthand for `? extends Object` (used when the upper bound would be `Object`)

Wildcards are particularly useful in code generation because they allow expressing relationships between types without knowing their concrete form. For example, when generating a method signature that accepts any type with an interface:

```java
public static void process(? super Iterable) { ... }
```

---

## Failure Modes and Invariants

### NameAllocator Invariants

1. **Tag uniqueness**: Each tag maps to exactly one name; reuse throws `IllegalArgumentException`
2. **Allocation monotonicity**: Once a name is allocated, it cannot be reassigned to another tag
3. **Keyword avoidance**: No keyword will ever be returned as an allocated name
4. **Character safety**: All emitted names consist of valid Java identifier characters

### TypeName Invariants

1. **Primitive types** have `keyword != null` and implement `isPrimitive() == true`
2. **Boxed primitives** are subclasses of `TypeName` with `isBoxedPrimitive() == true`
3. **Type parameters** cannot be primitives or `void` (checked in constructor)
4. **Wildcard upper bounds** must have size exactly 1; lower bounds can vary

### CodeWriter Failure Conditions

- Emitting a type before its declaration may cause missing imports; the library assumes the caller provides sufficient context
- Circular dependencies in nested generics are handled by caching type variables to avoid infinite recursion
- Static import resolution is best-effort; complex cases may fall back to fully-qualified names
