# Investigation: `pmd-kotlin:typeIs` fails on `DelegationSpecifier` nodes

## Problem

`pmd-kotlin:typeIs('java.lang.Throwable')` on a `DelegationSpecifier` node always returns `false`,
even though the node's `@TypeName` attribute is correctly populated (e.g. `java.lang.RuntimeException`).

Workaround in use: explicit `@TypeName` matching against known JDK exception base types:

```xpath
not(.//DelegationSpecifier[
    @TypeName='java.lang.RuntimeException'
    or @TypeName='java.lang.Exception'
    or @TypeName='java.lang.Throwable'
    or @TypeName='java.lang.Error'
])
```

**Limitation:** misses indirect custom exception hierarchies
(e.g. `class Foo : MyCustomException` where `MyCustomException extends RuntimeException`).

The correct fix:

```xpath
not(.//DelegationSpecifier[pmd-kotlin:typeIs('java.lang.Throwable')])
```

## What works

1. `DelegationSpecifierAnnotator.setDelegationSpecifierTypes()` IS called during parse.
2. `KotlinNodeTypeData.setTypeName(delegationSpecifierNode, "java.lang.RuntimeException")` IS set.
3. `@TypeName` attribute reads from the node's usermap — returns correct FQN.
4. Confirmed via `kotlin-type-mapper` CLI and passing unit test using `@TypeName`.

## What fails

`pmd-kotlin:typeIs('java.lang.Throwable')` on the same `DelegationSpecifier` node returns `false`.

Entry point: `AbstractKotlinTypeIsFunctionCall.call(contextNode, arguments)`:

```java
// Step 1 — reads getTypeName(node): "java.lang.RuntimeException" ← this IS non-null
String nodeType = KotlinNodeTypeData.getTypeName(node);
if (nodeType != null) {
    return ctx.isSubtypeOf(typeName, nodeType);  // ← suspect: ctx empty/wrong?
}
```

`ctx` is obtained from `KotlinTypeAnalysisContextHolder.get()`.

## Root cause hypothesis

`KotlinTypeAnalysisContextHolder` likely returns an empty/null context during XPath evaluation.

Timeline:
1. `KotlinLanguageProcessor.annotateIfPossible()` → `runSingleFileAnalysis()` during **parse**
2. `runSingleFileAnalysis()` calls `KotlinTypeAnalysisContextHolder.setGlobal(context)`
3. Visitor annotates nodes → `@TypeName` set ✓
4. XPath rule evaluates **after** parse completes
5. If `setGlobal` uses a ThreadLocal and XPath runs on a different thread — context is null → `isSubtypeOf` returns false

## Investigation steps

### 1. Locate context holder

```
../pmd/pmd-kotlin/src/main/java/net/sourceforge/pmd/lang/kotlin/rule/xpath/internal/KotlinTypeAnalysisContextHolder.java
```

Check:
- Is `setGlobal` ThreadLocal or static field?
- Is there a `clearGlobal()` call that runs before XPath evaluation?

### 2. Add diagnostic logging

In `AbstractKotlinTypeIsFunctionCall.call()` (pmd-kotlin sources, `feature/kotlin-type-mapper` branch):

```java
String nodeType = KotlinNodeTypeData.getTypeName(node);
LOG.debug("typeIs: node={}, nodeType={}, ctx={}", node.getClass().getSimpleName(), nodeType, ctx);
if (nodeType != null) {
    boolean result = ctx.isSubtypeOf(typeName, nodeType);
    LOG.debug("typeIs: isSubtypeOf({}, {}) = {}", typeName, nodeType, result);
    return result;
}
```

Run: `./mvnw test -Dtest=ImplementEqualsHashCodeOnValueObjectsTest -Dorg.slf4j.simpleLogger.log.net.sourceforge.pmd.lang.kotlin=debug`

### 3. Check `isSubtypeOf` with known types

In `KotlinTypeAnalysisContext`, add a test:

```java
// Does the context even have java.lang.Throwable in its hierarchy?
ctx.isSubtypeOf("java.lang.Throwable", "java.lang.RuntimeException")
```

If false → hierarchy not built for JDK types in single-file mode.

### 4. Check `runSingleFileAnalysis` classpath

```java
// In KotlinLanguageProcessor.runSingleFileAnalysis():
TypedAst ast = KotlinTypeMapper.fromSources(sources, classpathResolver.resolve());
```

`classpathResolver.resolve()` — does it return an empty list in unit test mode?

With an empty classpath, `ReflectionTypeHierarchy` seeds from `seedTypes` (user classes) only. JDK classes like `RuntimeException` may not be seeded → hierarchy map incomplete.

Check `KotlinAnalyzer.kt` around line 169:
```kotlin
// This seeds the reflection-based hierarchy so matchesSig can follow supertypes into
if (decl.isClassLike()) seedTypes.add(rawTypeName(decl.fqName))
```

`ServiceException` is seeded, but `RuntimeException` is only reachable if the reflection walk from `ServiceException` can find `RuntimeException` in the JVM classpath.

## Fix options

### Option A: Seed JDK exception hierarchy explicitly

In `KotlinAnalyzer` or `ReflectionTypeHierarchy`, always seed `java.lang.Throwable` (and let reflection walk the JDK):

```kotlin
seedTypes.add("java.lang.Throwable")
```

This ensures `RuntimeException → Exception → Throwable` is always in the hierarchy.

### Option B: Use system classloader for JDK types

In `ReflectionTypeHierarchy`, when a class fails to load from `classpathJars`, fall back to `ClassLoader.getSystemClassLoader()`:

```kotlin
val clazz = try {
    classLoader.loadClass(javaFqn)
} catch (e: ClassNotFoundException) {
    ClassLoader.getSystemClassLoader().loadClass(javaFqn)  // fallback for JDK types
}
```

### Option C: Fix `KotlinTypeAnalysisContextHolder` threading

If the root cause is a ThreadLocal context cleared before XPath evaluation, change `setGlobal` / `get` to use a static field (file-level lock) or ensure the context survives until rule evaluation completes.

## Files to change (in `../pmd`, branch `feature/kotlin-type-mapper`)

| File | Change |
|------|--------|
| `pmd-kotlin/.../rule/xpath/internal/KotlinTypeAnalysisContextHolder.java` | Investigate threading / lifecycle |
| `pmd-kotlin/.../rule/xpath/internal/AbstractKotlinTypeIsFunctionCall.java` | Add debug logging |
| `kotlin-type-mapper/analyzer/src/main/kotlin/ReflectionTypeHierarchy.kt` | Option B: JDK fallback |
| `kotlin-type-mapper/analyzer/src/main/kotlin/KotlinAnalyzer.kt` | Option A: seed Throwable |

## Verification

After fix, update `ImplementEqualsHashCodeOnValueObjects` rule in this project:

```xpath
(: replace @TypeName workaround :)
and not(.//DelegationSpecifier[@TypeName='java.lang.RuntimeException' or ...])

(: with proper typeIs :)
and not(.//DelegationSpecifier[pmd-kotlin:typeIs('java.lang.Throwable')])
```

Test must pass: `./mvnw test -Dtest=ImplementEqualsHashCodeOnValueObjectsTest`
