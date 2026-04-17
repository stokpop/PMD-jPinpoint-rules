# Kotlin XPath Rule Conventions

Design decisions, known patterns, and future improvement ideas for Kotlin XPath PMD rules
in `src/main/resources/category/kotlin/`.

---

## 1. Reporting at the correct line for chained method calls

**Problem:** `NavigationSuffix` inherits its `beginLine` from its parent
`PostfixUnaryExpression`, so violations reported on a `NavigationSuffix` node always
appear at the start of the full expression — not at the method call itself.

This causes wrong line numbers when the call is on a different line than the receiver, e.g.:

```kotlin
val expr = xpath           // line 10
    .compile("//book...")  // line 11 — violation should be here
```

**Rule:** When the XPath must report at the method call line, navigate to the
`T-Identifier` leaf token instead of stopping at `NavigationSuffix`:

```xpath
(: wrong — reports at line 10 :)
/PostfixUnarySuffix/NavigationSuffix[@Identifier='compile']

(: correct — reports at line 11 :)
/PostfixUnarySuffix/NavigationSuffix/SimpleIdentifier/T-Identifier[@Text='compile']
```

Keep any filter predicates on `NavigationSuffix` and add the leaf step after the closing `]`:

```xpath
/PostfixUnarySuffix/NavigationSuffix[@Identifier='compile'
    and not(... some condition ...)
]/SimpleIdentifier/T-Identifier[@Text='compile']
```

**Applies to:** Any rule whose final XPath step is a `NavigationSuffix` used as the
reported node (not in a predicate). Rules already fixed: `AvoidWideScopeXPathExpression`,
`AvoidImplicitlyRecompilingRegex` (branch 4), `MDCPutWithoutRemove`.

---

## 2. `@Identifier` attribute — structural shortcut

`@Identifier` is a custom PMD-Kotlin attribute returning the text of the direct
`SimpleIdentifier` child. It is a **filter shortcut**, not a navigation step:

```xpath
(: equivalent — @Identifier is just a filter :)
NavigationSuffix[@Identifier='put']
NavigationSuffix[SimpleIdentifier/T-Identifier/@Text='put']
```

Because it is an attribute on the parent node, the reported violation line is still the
parent's `beginLine`. See convention 1 for how to fix that.

---

## 3. Future improvement: `@IsClassField` node attribute

**Status:** Not yet implemented — deferred.

Many rules guard with `not(ancestor::FunctionBody)` to restrict to class-level fields
(not local variables). Example:

```xpath
//PropertyDeclaration[not(ancestor::FunctionBody) and pmd-kotlin:typeIs('java.lang.StringBuffer')]
```

A custom attribute `@IsClassField` (boolean, true when not inside `FunctionBody`) on
`ASTPropertyDeclaration` would make this cleaner:

```xpath
//PropertyDeclaration[@IsClassField and pmd-kotlin:typeIs('java.lang.StringBuffer')]
```

**Affected rules:** `AvoidStringBuffer`, `AvoidSimpleDateFormat`,
`AvoidDecimalAndChoiceFormatAsField`, `AvoidRecreatingDateTimeFormatter`.

**Implementation:** Add `getIsClassField()` method with `@Attribute` to
`ASTPropertyDeclaration` in pmd-kotlin, returning `getAncestors(ASTFunctionBody.class).isEmpty()`.

---

## 4. Future improvement: `pmd-kotlin:isConstantArg(node)` XPath function

**Status:** Not yet implemented — deferred.

`AvoidImplicitlyRecompilingRegex` uses a complex multi-line guard in several branches
to exclude arguments that reference function parameters (dynamic values):

```xpath
and not(.//CallSuffix/ValueArguments/ValueArgument//LineStringContent/T-LineStrRef[
   @Text = ancestor::FunctionDeclaration//Parameter/SimpleIdentifier/T-Identifier/concat('$', @Text)])
and not(.//CallSuffix/ValueArguments/ValueArgument//SimpleIdentifier/T-Identifier[
   @Text = ancestor::FunctionDeclaration//Parameter/@Identifier])
and not(PrimaryExpression[@Identifier = ancestor::FunctionBody//PropertyDeclaration/...])
```

A function `pmd-kotlin:isConstantArg(argNode)` returning `true` when the argument
expression contains no reference to function parameters, string-template parameter
interpolations, or `var` fields would replace all of this.

**Note:** Implemented as an XPath function (not attribute) because it needs to traverse
ancestor context (function parameter list) — it is not purely local to the argument node.
