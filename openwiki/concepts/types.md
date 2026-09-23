---
type: component
title: Type Specification Domain
description: Documents the hierarchical type model (classes, interfaces, enums) and how Javapoet's TypeSpec structures Java source generation from reflection metadata.
tags: [javapoet, type-specification, code-generation, reflection, formatting]
verified:
  - by: openwiki/0.5.2
    at: 2026-09-22T02:45:45.818Z
sources:
  - id: openwiki-source-4c7f11a3d4b371227b1a355b
    resource: repo://src/main/java/com/squareup/javapoet/ArrayTypeName.java
  - id: openwiki-source-8a6417773878e412bc92fe3f
    resource: repo://src/main/java/com/squareup/javapoet/ClassName.java
  - id: openwiki-source-ca139b909129ba6056b11115
    resource: repo://src/main/java/com/squareup/javapoet/TypeName.java
  - id: openwiki-source-1205045a5bab72d4678fc6da
    resource: repo://src/main/java/com/squareup/javapoet/TypeSpec.java
  - id: openwiki-source-39bd05157a2b7fc932b39c99
    resource: repo://src/main/java/com/squareup/javapoet/TypeVariableName.java
  - id: openwiki-source-dff2e4f0656f620a39c2d4f8
    resource: repo://src/main/java/com/squareup/javapoet/WildcardTypeName.java
generated: { by: "openwiki/0.5.2", at: "2026-09-22T02:45:45.818Z" }
---

# Type Specification Domain

The Javapoet library provides a complete hierarchical type model for Java's type system, enabling declarative code generation of classes, interfaces, enums, and their nested structures. The domain is organized into six core abstractions that together represent any valid Java type expression:

| Class | Role |
|---|---|
| `TypeName` | Any Java type (primitives, void, reference types, composite types) |
| `ClassName` | Fully-qualified class names with enclosure chains and package resolution |
| `TypeVariableName` | Generic type parameters (single or parameterized) |
| `WildcardTypeName` | Wildcard types (`?`, `? super T`, `? extends T`) |
| `ArrayTypeName` | Array types (`T[]`, `T[][]`, etc.) |
| `TypeSpec` | Complete type declarations (classes, interfaces, enums, annotations) |

## Architecture Overview

```mermaid
graph TD
    TypeSpec --> TypeName
    TypeName --> ClassName
    TypeName --> TypeVariableName
    TypeName --> WildcardTypeName
    TypeName --> ArrayTypeName
    
    TypeName --> TypeName
    TypeName --> TypeName
    TypeName --> TypeName
    
    ClassName --> ClassName
    ClassName --> ClassName
    
    TypeVar --> TypeName
    TypeVar --> TypeName
    
    Wildcard --> TypeName
    Wildcard --> TypeName
    
    Array -> TypeName
    Array --> TypeName
```

## Entry Points and Responsibilities

### `TypeName` — The Universal Type Identifier

`TypeName` is the foundational value object that represents **any** Java type. It serves as both a runtime identifier and an emission target for code generation.

**Key responsibilities:**
- Uniform representation of primitives, boxed types, `void`, reference classes, arrays, generics, wildcards, and type variables
- Immutable value semantics with proper `equals()`/`hashCode()` implementations
- Bidirectional conversion between reflection metadata (`java.lang.reflect.Type`) and the internal model
- Format token-based code emission via `emit(CodeWriter out)`

**Entry points:**
```
TypeName.get(Class)              — reflect a runtime class → TypeName
TypeName.get(TypeMirror)         — reflect a TypeMirror → TypeName
ClassName.get(package, name...)  — construct fully-qualified class names
ArrayTypeName.of(componentType)  — build array types from component types
ParameterizedTypeName.get(type, params) — compose generic type expressions
WildCardTypeName.subtypeOf(supertype) / supertypeOf(base) — wildcard bounds
```

**Important invariants:**
- Primitive types (`INT`, `DOUBLE`, etc.) have a non-null keyword string; all other types have `keyword == null`
- `TypeName.VOID` is the singleton representing Java's void type (not a reference type)
- Boxing/unboxing conversions are symmetric: `TypeName.INT.box().unbox() == TypeName.INT` and vice versa

**Evidence:**
- The builder API and factory methods in `repo://src/main/java/com/squareup/javapoet/TypeName.java#L18-L106` define the construction surface
- Reflection-to-model conversion in `repo://src/main/java/com/squareup/javapoet/TypeName.java#L324-L574` handles all type kinds via visitor pattern
- Boxing/unboxing in `repo://src/main/java/com/squareup/javapoet/TypeName.java#L165-L204` implements symmetric conversion

### `ClassName` — Enclosure-Aware Class Names

`ClassName` extends the base type model to handle **fully-qualified class names** with nesting chains (e.g., `java.util.Map.Entry`, `com.example.Outer.Inner`).

**Key responsibilities:**
- Maintain a chain of enclosing classes for nested types
- Resolve package names and canonical full names (`packageName + "." + simpleName`)
- Handle reflection-based name extraction from `Class<?>` objects
- Support best-effort parsing of string-based class names via `bestGuess()`

**Entry points:**
```
ClassName.get(Class)                  — extract from a runtime Class
ClassName.get(TypeElement)            — reflect a TypeElement → ClassName
ClassName.bestGuess(string)           — parse "com.example.FooBar" into hierarchy
ClassName.nestedClass(name)           — add one level of nesting
ClassName.peerClass(name)             — create a sibling class at same depth
```

**Important invariants:**
- The enclosing chain is immutable; each `ClassName` has exactly one parent (or null for top-level classes)
- Top-level classes have an empty package name (`""`) when they reside in the default package
- Enclosing chains are resolved from innermost to outermost, matching Java's nesting semantics

**Evidence:**
- Nested class construction and enclosure chain management in `repo://src/main/java/com/squareup/javapoet/ClassName.java#L54-L290`
- Reflection-based extraction in `repo://src/main/java/com/squareup/javapoet/ClassName.java#L168-L184`
- Best-effort parsing algorithm in `repo://src/main/java/com/squareup/javapoet/ClassName.java#L193-224`

### `TypeVariableName` — Generic Type Parameters with Bounds

`TypeVariableName` represents generic type parameters, including single-parameter and **parameterized** type variables (those with upper/lower bounds).

**Key responsibilities:**
- Represent type parameters like `<T>`, `<T extends Number>`, `<T, U>`
- Model bounds via `getBounds()` returning an ordered list of super/sub types
- Support recursive/bounded types (e.g., `Recursive<T extends Map<List<T>, Set<T[]>>>`)

**Entry points:**
```
TypeVariableName.get(name)                     — single-parameter type variable
TypeVariableName.get(bounds...)                 — parameterized with bounds
TypeMirror.accept(..., TypeVariableName.get())  — reflect a TypeParameter → TypeVariableName
```

**Important invariants:**
- Bounds are preserved but **not emitted during declaration**; they only appear when the type appears on the right-hand side of an equals sign or as a generic argument
- The bounds list is immutable and reflects Java's variance semantics
- Recursive types are supported by allowing the type variable to appear in its own bound definition

**Evidence:**
- Type variable construction and bound management in `repo://src/main/java/com/squareup/javapoet/TypeVariableName.java#L42-L160`
- Reflection-based extraction in `repo://src/test/java/com/squareup/javapoet/AbstractTypesTest.java#L358-409`

### `WildcardTypeName` — Wildcard Type Expressions

`WildcardTypeName` represents wildcard types with optional bounds: bare (`?`), upper-bounded (`? extends T`), or lower-bounded (`? super T`).

**Key responsibilities:**
- Model the three forms of Java wildcards
- Support reflection-based extraction from `WildcardType` mirrors
- Provide string emission for both declaration and usage contexts

**Entry points:**
```
WildcardTypeName.subtypeOf(base)     — "? extends Base"
WildcardTypeName.supertypeOf(base)   — "? super Base"
WildcardTypeName.get()                — bare wildcard "?"
TypeMirror.accept(..., WildcardTypeName.get())  — reflect → WildcardTypeName
```

**Evidence:**
- Wildcard construction and emission in `repo://src/main/java/com/squareup/javapoet/WildcardTypeName.java#L19-L78`

### `ArrayTypeName` — Array Type Expressions

`ArrayTypeName` represents Java array types (`T[]`, `T[][]`, nested arrays) with a single component type.

**Key responsibilities:**
- Represent the component type of an array expression
- Support reflection-based extraction from `ArrayType` mirrors
- Compose with `TypeName.get()` to build generic array expressions like `List<Long>[]`

**Entry points:**
```
ArrayTypeName.of(componentType)           — "Component[]"
ArrayTypeName.get(TypeMirror)               — reflect ArrayType → ArrayTypeName
ParameterizedTypeName.get(type, [args])    — composes arrays with generics
```

### `TypeSpec` — Complete Type Declarations

`TypeSpec` is the **top-level declaration type** that represents any Java declaration: class, interface, enum, annotation, or anonymous inner type. It composes all lower-level types and members into a unified specification for code generation.

**Key responsibilities:**
- Model all six kind of declarations: `CLASS`, `INTERFACE`, `ENUM`, `ANNOTATION`
- Hold references to nested structures: fields, methods, types, enum constants
- Manage annotations, modifiers, type variables, superclasses/superinterfaces
- Drive the entire code emission process via `emit(CodeWriter out)`

**Entry points:**
```java
TypeSpec.classBuilder(name)                   — create a class spec
TypeSpec.interfaceBuilder(name)               — create an interface spec  
TypeSpec.enumBuilder(name)                    — create an enum spec
TypeSpec.annotationBuilder(name)              — create an annotation spec
TypeSpec.anonymousClassBuilder(...)          — anonymous inner type
```

**Member composition:**
- `List<AnnotationSpec> annotations` — top-level annotations (e.g., `@NonNull`)
- `Set<Modifier> modifiers` — class/interface/enum modifiers
- `List<TypeVariableName> typeVariables` — generic parameters for classes/interfaces
- `TypeName superclass` / `List<TypeName> superinterfaces` — inheritance hierarchy
- `Map<String, TypeSpec> enumConstants` — enum value declarations
- `List<FieldSpec> fieldSpecs` — instance/static fields with their types and modifiers
- `CodeBlock staticBlock` / `initializerBlock` — block-level code
- `List<MethodSpec> methodSpecs` — constructors, methods, default/abstract methods
- `List<TypeSpec> typeSpecs` — nested types (classes, interfaces, enums inside)

**Important invariants:**
- Enum declarations cannot have field or method specs unless they are enum constants themselves
- Interface members must be `public abstract`, `public static final`, or `private default`
- Annotation members are always public and static; their names must be unique per annotation type
- Anonymous inner types may not have modifiers, type variables, or superinterfaces
- Enum constants require anonymous class bodies (they cannot be bare values)

**Evidence:**
- Full declaration model in `repo://src/main/java/com/squareup/javapoet/TypeSpec.java#L48-L843`
- Validation rules on `build()` in `repo://src/main/java/com/squareup/javapoet/TypeSpec.java#L759-L820`
- Code emission algorithm in `emit()` covering all member categories

## Integration with Reflection and Annotations

The type specification domain works closely with the annotation system (`AnnotationSpec`) to produce complete, compilable Java source:

1. **Reflection → Model**: `TypeName.get(Class)` and `ClassName.get(Class)` convert runtime metadata into Javapoet types
2. **Specification → Code**: `CodeWriter` emits the TypeSpec as properly-formatted Java source with correct imports, indentation, and line breaks
3. **Round-trip support**: Annotations can be re-extracted from generated code via `AnnotationSpec.get()` for diffing and testing

## Focused Tests That Matter

The type specification domain is validated through several focused test suites:

| Test Class | Focus Area | Key Assertions |
|---|---|---|
| `TypeSpecTest` | Declaration construction, member composition, code emission | Validates that classes/interfaces/enums/annotations produce correct Java source; checks import generation, modifier handling, and formatting |
| `TypeNameTest` | Type construction, boxing/unboxing, reflection extraction | Verifies primitive/box/unbox symmetry, wildcard bounds, array generics, and error type handling |
| `AbstractTypesTest` (base) | Generic types, wildcards, arrays, primitives | Tests all TypeKind visitors, nested types, recursive types, and void/null boundaries |

## Failure Modes and Edge Cases

- **Anonymous inner types** require anonymous class arguments; attempting to add modifiers or type variables throws an exception
- **Enum declarations without any enum constants** emit a trailing semicolon after the opening brace (valid Java)
- **Enums can define abstract methods** that must be implemented by at least one enum constant
- **Wildcard extraction from reflection mirrors** preserves bounds but strips default values; `get()` vs `get(annotated, true)` differ in completeness

## Relationships to Other Domains

| Domain | Relationship |
|---|---|
| Annotations (`AnnotationSpec`) | TypeSpec holds a list of AnnotationSpec for top-level annotations; fields/methods can be annotated individually |
| Code Generation (`CodeWriter`) | TypeSpec is the primary input to `emit()`; the writer translates TypeSpec structure into Java source with correct indentation and imports |
| Testing (`TypesTest`, `TypeSpecTest`) | Tests verify that reflection-based type construction produces emission identical to hand-written specifications, ensuring round-trip correctness |

## Extensibility Points

- **New kind of declaration**: Extend `Kind` enum and implement validation in `TypeSpec.Builder.build()`. Add new factory method on `TypeSpec` class.
- **Custom member types**: Extend existing builder methods (`addField`, `addMethod`) or add new ones to support domain-specific constructs (e.g., records, sealed classes).
- **Code formatting**: The `CodeWriter` class is the sole responsibility for emitting code from TypeSpec; custom formatters can be injected via the CodeWriter API.

---

## References

- `repo://src/main/java/com/squareup/javapoet/TypeSpec.java` — Complete declaration model and validation
- `repo://src/main/java/com/squareup/javapoet/ClassName.java` — Enclosure-aware class names
- `repo://src/main/java/com/squareup/javapoet/TypeName.java` — Universal type identifier with boxing/unboxing
- `repo://src/main/java/com/squareup/javapoet/TypeVariableName.java` — Generic type parameters with bounds
- `repo://src/main/java/com/squareup/javapoet/WildcardTypeName.java` — Wildcard type expressions
- `repo://src/main/java/com/squareup/javapoet/ArrayTypeName.java` — Array type composition
- `repo://src/test/java/com/squareup/javapoet/TypeSpecTest.java` — Declaration construction and emission tests
- `repo://src/test/java/com/squareup/javapoet/AbstractTypesTest.java` — Generic types, wildcards, arrays, primitives

---

**Note on claim management**: This page is newly created with no existing claims to reconcile. All material propositions above are represented as new Claims in the submission payload.
