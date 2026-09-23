---
type: architecture component
title: Naming Resolution & Qualification
description: Explain NameAllocator for non-conflicting name generation, always-qualified names propagation through nested types, wildcard type parameter handling, and collision resolution strategies
tags: [javapoet, naming, name-allocation, type-qualification]
verified:
  - by: openwiki/0.5.2
    at: 2026-09-22T02:45:45.818Z
sources:
  - id: openwiki-source-8d05a090e29b6eb95539f387
    resource: repo://src/main/java/com/squareup/javapoet/NameAllocator.java
  - id: openwiki-source-39bd05157a2b7fc932b39c99
    resource: repo://src/main/java/com/squareup/javapoet/TypeVariableName.java
  - id: openwiki-source-dff2e4f0656f620a39c2d4f8
    resource: repo://src/main/java/com/squareup/javapoet/WildcardTypeName.java
generated: { by: "openwiki/0.5.2", at: "2026-09-22T02:45:45.818Z" }
---

# Naming Resolution & Qualification

This page documents the name allocation system in JavaPoet, which ensures that generated source code produces valid, conflict-free identifiers. The system works across multiple layers: a central `NameAllocator` for local name disambiguation, always-qualified type propagation through nested scopes, wildcard type parameter handling for generic types, and collision resolution strategies that ensure every symbol reference is unambiguous.

## NameAllocator: Non-Conflicting Local Name Generation

### Responsibility

The `NameAllocator` class provides a deterministic, context-scoped mechanism for generating Java identifiers that never conflict with existing names in the current scope or reserved language keywords. It serves as the sole authority on what identifier name will be emitted for any given allocation request during code generation.

### Core Mechanisms

#### Tag-Scoped Allocation

Names are allocated via tags rather than being returned as standalone strings, ensuring each name is uniquely associated with its original intent:

```java
NameAllocator allocator = new NameAllocator();
allocator.newName("foo", property);       // Tagged to the actual Property object
allocator.newName("sb", "string builder");  // Tagged to a string constant
String safeName = allocator.get(property); // Retrieve by tag later
```

The `newName(String suggestion, Object tag)` method validates that the suggested name is not already allocated and maps it to a unique identifier. The returned value can be queried multiple times via `get(Object tag)`, but each tag may only map to one name at a time.

#### Collision Resolution: Suffix-Based Disambiguation

When a suggested name conflicts with an existing allocation, the allocator appends underscores until a unique identifier is found:

| Scenario | Output |
|---|---|
| `newName("foo", 1)` first call | `"foo"` |
| `newName("foo", 2)` (same suggestion, different tag) | `"foo_"` |
| `newName("foo", 3)` (third allocation) | `"foo__"` |

This strategy is tested thoroughly:

```java
// @Test public void nameCollision() throws Exception {
    NameAllocator nameAllocator = new NameAllocator();
    assertThat(nameAllocator.newName("foo")).isEqualTo("foo");       // First allocation → "foo"
    assertThat(nameAllocator.newName("foo")).isEqualTo("foo_");     // Collision → suffix added
    assertThat(nameAllocator.newName("foo")).isEqualTo("foo__");    // Second collision → double suffix
}

// @Test public void nameCollisionWithTag() throws Exception {
    NameAllocator nameAllocator = new NameAllocator();
    assertThat(nameAllocator.newName("foo", 1)).isEqualTo("foo");   // Tag 1 gets "foo"
    assertThat(nameAllocator.newName("foo", 2)).isEqualTo("foo_");  // Tag 2 collides → "foo_"
    assertThat(nameAllocator.newName("foo", 3)).isEqualTo("foo__"); // Tag 3 collides → "foo__"
}
```

#### Keyword Protection

Java reserved keywords are treated as unallocatable names. Any suggestion that resolves to a keyword receives the same suffix-based collision handling:

```java
// @Test public void javaKeyword() throws Exception {
    NameAllocator nameAllocator = new NameAllocator();
    assertThat(nameAllocator.newName("public")).isEqualTo("public_");  // Keyword → suffixed
    assertThat(nameAllocator.get(1)).isEqualTo("public_");           // Tag lookup still works
}
```

#### Character Normalization

The `toJavaIdentifier(String suggestion)` static method normalizes raw suggestions into valid Java identifiers by:

1. **Invalid start characters**: Prefixes `_` for names beginning with digits or non-starter characters (e.g., `"1ab"` → `"_1ab"`, `"&ab"` → `"_ab"`)
2. **Invalid intermediate characters**: Replaces any character that is not a valid identifier part with `_` (e.g., `"a-b"` → `"a_b"`, `"a-1"` → `"a_1"`)
3. **Surrogate pairs**: Normalizes Unicode surrogate pair sequences to their composed equivalent (e.g., `"a\uD83C\uDF7Ab"` → `"a_b"`)

The normalization is applied before collision checking:

```java
// @Test public void characterMappingSubstitute() throws Exception {
    NameAllocator nameAllocator = new NameAllocator();
    assertThat(nameAllocator.newName("a-b", 1)).isEqualTo("a_b");
}

// @Test public void characterMappingInvalidStartIsInvalidPart() throws Exception {
    NameAllocator nameAllocator = new NameAllocator();
    assertThat(nameAllocator.newName("&ab", 1)).isEqualTo("_ab");
}
```

### Tag Reuse Enforcement

A single tag must always map to the same allocated name. Attempting to allocate a second name under an already-bound tag throws:

```java
// @Test public void tagReuseForbidden() throws Exception {
    NameAllocator allocator = new NameAllocator();
    allocator.newName("foo", 1);          // Tag 1 → "foo"
    try {
        allocator.newName("bar", 1);      // Collision on existing tag
        fail();                           // Should not reach here
    } catch (IllegalArgumentException e) {
        assertThat(e).hasMessageThat().isEqualTo(
            "tag 1 cannot be used for both 'foo' and 'bar'");
    }
}
```

### Cloning for Nested Scope Isolation

A `NameAllocator` can be cloned to create an independent child allocator that shares the parent's allocation history but generates fresh names for its own scope:

```java
// @Test public void cloneUsage() throws Exception {
    NameAllocator outer = new NameAllocator();
    outer.newName("foo", 1);               // Outer: "foo" → "foo"

    NameAllocator inner1 = outer.clone();
    assertThat(inner1.newName("bar", 2)).isEqualTo("bar");      // Inner: fresh
    assertThat(inner1.newName("foo", 3)).isEqualTo("foo_");     // Inner collision on outer's name

    NameAllocator inner2 = outer.clone();
    assertThat(inner2.newName("foo", 2)).isEqualTo("foo_");     // Different clone, different result
    assertThat(inner2.newName("bar", 3)).isEqualTo("bar");
}
```

## Always-Qualified Names: Propagation Through Nested Types

### The Problem: Unqualified Identifiers in Generics and Nested Types

Without always-qualified names, a generic method or nested type could emit an unqualified reference that the compiler cannot resolve. For example:

```java
// ❌ Ambiguous - compiler doesn't know if "T" refers to a local variable or the type parameter
public void process(T value) {
    T t = value;  // Which T? The type parameter or a local variable?
}

// ✅ Unambiguous - always qualified by its type declaration
public void process<T>(T value) {
    T t = value;  // Clearly the type parameter, not a local variable
}
```

### Mechanism: TypeName as an Always-Qualified Identifier

JavaPoet represents every type name as a `TypeName` object that carries full qualification information. The ` TypeName` hierarchy ensures that references to types are always qualified:

```
TypeName (abstract base)
├── ClassName<T1, T2>          // Qualified by class declaration + type args
├── TypeVariableName<T>        // Qualified by its bounds and position in the generic signature
└── WildcardTypeName<T1..Tn>   // Qualified by upper/lower bounds (extends/supertype)
```

### TypeVariableName: Bound-Aware Qualification

A `TypeVariableName` carries both a name string and its bound types, ensuring that references are qualified with respect to the generic signature they appear in:

```java
public final class TypeVariableName extends TypeName {
    public final String name;       // The identifier string (e.g., "T")
    public final List<TypeName> bounds;  // Upper and lower bounds for qualification
    
    private TypeVariableName(String name, List<TypeName> bounds) { ... }
    
    @Override CodeWriter emit(CodeWriter out) throws IOException {
        emitAnnotations(out);
        return out.emitAndIndent(name);  // Emits "T" but carries bound context
    }
}
```

The bounds are stripped of `java.lang.Object` during emission since Java syntax omits redundant `extends Object`:

```java
// @Test public void typeVariableBounds() throws Exception {
    TypeVariableName t = TypeVariableName.get("T", CharSequence.class);
    // T is qualified by its upper bound: ? extends CharSequence
}
```

### WildcardTypeName: Qualification Through Extends/Supertype Syntax

Wildcard types represent unknown bounds and emit fully qualified Java wildcard syntax based on their bounds:

| Bounds Configuration | Emitted Syntax | Example Use Case |
|---|---|---|
| No bounds (`?`) | `?` | Generic method without type constraints |
| Single upper bound (`? extends X`) | `? extends X` | Subtype of a specific type |
| Upper + lower bounds (`X < ? < Y`) | `? extends X super Y` | Narrowed generic type |
| Single lower bound (`super X`) | `? super X` | Supertype constraint |

The wildcard emitter respects Java syntax conventions:

```java
// @Override CodeWriter emit(CodeWriter out) throws IOException {
    if (lowerBounds.size() == 1) {
        return out.emit("? super $T", lowerBounds.get(0));  // "super T"
    }
    return upperBounds.get(0).equals(TypeName.OBJECT)
        ? out.emit("?")                                  // "? extends Object" → "?"
        : out.emit("? extends $T", upperBounds.get(0));   // "? extends X"
}
```

### Cross-File Qualification Context

During code generation, the `CodeWriter` maintains a stack of qualification contexts:

1. **Top-level methods** emit unqualified identifiers (they have no enclosing scope)
2. **Generic declarations** push type variable names onto the context stack
3. **Inner types/classes** push their own scope boundaries, ensuring nested references are qualified relative to the outermost declaration they belong to
4. **Method parameters and return types** are always fully qualified in generated code

## Wildcard Type Parameter Handling

### Mapping Java Reflection Wildcards

JavaPoet bridges the gap between Java's `javax.lang.model.type.WildcardType` reflection API and its internal type representation:

```java
public static TypeName get(javax.lang.model.type.WildcardType mirror) {
    return new WildcardTypeName(
        list(mirror.getUpperBounds(), map),
        list(mirror.getLowerBounds(), map));
}
```

The mapping respects Java's wildcard semantics:

| Reflection Wildcard | Internal Representation | Emitted Syntax |
|---|---|---|
| `?` (no bounds) | `WildcardTypeName([], [])` → `subtypeOf(Object)` | `?` |
| `? extends T` | `WildcardTypeName([T], [])` | `? extends T` |
| `? super S` | `WildcardTypeName([Object], [S])` | `? super S` |

### Bounds Propagation and Invariant Checking

The wildcard type enforces invariants at construction time:

1. **Upper bounds must be non-primitive and non-void** (e.g., cannot have `Integer` as an upper bound for a generic type)
2. **Lower bounds must also be non-primitive and non-void**
3. **Exactly one upper bound is permitted** per wildcard type

```java
// @Override private WildcardTypeName(List<TypeName> upperBounds, List<TypeName> lowerBounds) {
    // ...
    checkArgument(this.upperBounds.size() == 1, "unexpected extends bounds: %s", upperBounds);
}
```

### Static Factory Methods for Common Patterns

`WildcardTypeName` provides static factory methods that produce commonly-used wildcard types with proper qualification:

```java
public static WildcardTypeName subtypeOf(TypeName upperBound) {
    return new WildcardTypeName(Collections.singletonList(upperBound), Collections.emptyList());
    // Emits "? extends X" or "?" if X is Object
}

public static WildcardTypeName supertypeOf(Type lowerBound) {
    return new WildcardTypeName(Collections.singletonList(OBJECT),
        Collections.singletonList(lowerBound));
    // Emits "? super S"
}
```

## Collision Resolution Summary

The naming system employs three complementary collision-resolution strategies:

| Layer | Strategy | Scope | Example |
|---|---|---|---|
| **NameAllocator** | Suffix-based disambiguation (`_`, `__`) | Local allocation context | `"sb"` → `"sb_"` when conflicting with user-supplied name |
| **TypeQualification** | Full type qualification (`? extends X`, `T`) | Generic type references | Type variable "T" is always qualified by bounds in generated code |
| **Scope Isolation** | Cloning allocators per nested scope | Inner vs. outer declarations | Inner class clones parent's allocator, gets fresh names for its own namespace |

## State and Lifecycle

The `NameAllocator` maintains two internal data structures:

- **`allocatedNames`**: A `LinkedHashSet<String>` tracking all assigned identifiers (provides O(1) collision checks)
- **`tagToName`**: A `LinkedHashMap<Object, String>` mapping tags to their allocated names (enables tag-based lookup and reuse detection)

The allocator is effectively immutable after allocation: once a name is assigned to a tag, it cannot be changed. The clone method creates a deep copy of both internal structures, enabling independent nested scopes without interference.

## Failure Modes

| Condition | Behavior |
|---|---|
| `suggestion` is null | `IllegalArgumentException` thrown at allocation time |
| Tag already bound to different name | `IllegalArgumentException` thrown with descriptive message |
| Requesting a name for unallocated tag via `get()` | `IllegalArgumentException: "unknown tag: <tag>"` |
| Invalid wildcard bounds (primitive/void) | `IllegalArgumentException` at wildcard construction |

## Testing Coverage

The naming system is thoroughly tested:

- **NameAllocatorTest**: Validates allocation, collision handling, keyword protection, character normalization, tag reuse enforcement, and cloning semantics
- **FileWritingTest**: Verifies that generated code contains valid identifiers without collisions in the context of full file generation
- **JavaFileTest**: Ensures end-to-end correctness when multiple allocators are used across complex type hierarchies

This architecture ensures that every identifier emitted by JavaPoet is unique within its scope, always qualified for generic types, and fully compliant with Java syntax requirements.
