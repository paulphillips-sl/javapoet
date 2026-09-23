---
type: deprecation-migration
title: JavaPoet Deprecation & Migration to Palantir's JavaPoet
description: Documents Square's JavaPoet deprecation notice, migration steps including Maven coordinate changes, and API differences between com.squareup.javapoet and com.palantir.javapoet packages.
tags: [javapoet, square-java-poet, palantir-javapoet, deprecation, migration, maven-coordinates]
verified:
  - by: openwiki/0.5.2
    at: 2026-09-22T02:45:45.818Z
sources:
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
generated: { by: "openwiki/0.5.2", at: "2026-09-22T02:45:45.818Z" }
---

# JavaPoet Deprecation & Migration to Palantir's JavaPoet

## Overview

As of **2020-10-10**, Square's [JavaPoet](http://github.com/square/javapoet) project is officially deprecated. The team has acknowledged their contributions but has not maintained the library since that date.

The recommended successor is **[Palantir's JavaPoet](https://github.com/palantir/javapoet)**, which continues development and supports modern Java language features.

---

## Migration Steps

### 1. Maven Coordinates

The primary change involves replacing the Maven group ID and artifact ID:

```diff
- javapoet = { module = "com.squareup:javapoet", version = "1.13.0" }
+ javapoet = { module = "com.palantir.javapoet:javapoet", version = "0.5.0" }
```

**Key changes:**

| Aspect | Square JavaPoet (`com.squareup:javapoet`) | Palantir JavaPoet (`com.palantir.javapoet:javapoet`) |
|---|---|---|
| **Group ID** | `com.squareup` | `com.palantir.javapoet` |
| **Artifact ID** | `javapoet` | `javapoet` |
| **Version scheme** | Semantic versioning (1.13.x) | Different versioning starting at 0.5.0 |

### 2. Import Statement Changes

All import paths must change from the Square package to the Palantir namespace:

```diff
- com.squareup/javapoet/*.java
+ com.palantir.javapoet/*.java
```

A bulk replacement script can automate this across a codebase:

```bash
sed -i "" 's/com.squareup.javapoet.\([A-Za-z]*\)/com.palantir.javapoet.\1/g' \
  `find . -name "*.kt" -or -name "*.java"`
```

### 3. API Changes — Fields Became Functions

Several fields in the Square JavaPoet API were converted from properties to accessor methods in Palantir's implementation:

```diff
- javaFile.packageName
+ javaFile.packageName()
```

This change affects all `TypeSpec` and related builder APIs that expose type information as fields.

---

## Architecture Context

### Square JavaPoet (Deprecated)

Square's JavaPoet was an API for generating `.java` source files from a declarative DSL. It provided:

- **Immutable specification objects** (`TypeName`, `ClassName`, `TypeSpec`, `MethodSpec`)
- **Builder APIs** with method chaining for constructing code elements
- **Code emission** via `JavaFile.writeTo()` or `JavaFile.toString()`
- **Format strings** using `$T` (types), `$S` (strings), `$L` (literals), and `$N` (names)

The library's Maven artifact was distributed under `com.squareup:javapoet` starting at version 1.0.0, with the final release being **1.13.0** in June 2020.

### Palantir JavaPoet (Active)

Palantir's fork continues development of the same code-generation API but:
- Supports newer Java language features that Square's version did not
- Maintains active issue tracking and release cycles at https://github.com/palantir/javapoet
- Retains the core DSL while adding improvements

---

## Migration Checklist

1. **Update `pom.xml`** — Replace Maven coordinates to point to `com.palantir.javapoet:javapoet:0.5.0+`
2. **Replace imports** — Run the `sed` replacement or manually update all `com.squareup/javapoet/...` references
3. **Update field accesses** — Convert any `.packageName`, `.methodName`, or other property accessors to method calls (e.g., `.packageName()` → `.packageName()`)
4. **Verify compatibility** — Run tests against the new library to catch any remaining API mismatches

---

## Related Resources

- [Square JavaPoet GitHub](http://github.com/square/javapoet) — Archived project with the deprecation notice
- [Palantir JavaPoet GitHub](https://github.com/palantir/javapoet) — Active successor repository
- [JavaPoet README migration section](http://github.com/square/javapoet/blob/master/README.md#deprecated) — Original migration guidance from Square
