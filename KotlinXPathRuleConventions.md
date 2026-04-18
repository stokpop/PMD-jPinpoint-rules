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

---

## 5. `this` keyword in the Kotlin AST

**Important:** In the PMD Kotlin AST, the `this` keyword is represented as
`PrimaryExpression/SimpleIdentifier/T-Identifier[@Text='this']`, **NOT** as
`ThisExpression/T-THIS` (which is what Java uses).

`ThisExpression` does not exist in the Kotlin AST. Always use:

```xpath
(: correct — matches both 'this' and 'this@Label' :)
//T-Identifier[@Text='this']

(: WRONG — will never match anything :)
//ThisExpression
```

**`this@Label` (labeled this):** Parsed as a `Label` node containing `T-Identifier[@Text='this']`
followed by `T-AT_NO_WS`, then the label name as a separate `PrimaryExpression/SimpleIdentifier`:

```
UnaryPrefix
  Label
    SimpleIdentifier / T-Identifier[@Text='this']
    T-AT_NO_WS
  PrimaryExpression / SimpleIdentifier / T-Identifier[@Text='ClassName']
```

Both `this` and `this@Label` contain `T-Identifier[@Text='this']`, so a single check covers both.

**Scoping limitation:** `this@Outer` in an inner class refers to the outer class's `this`, not
the inner class's. XPath cannot distinguish which `this` is meant. This creates false negatives
(missed violations) but no false positives.

**Applies to:** All rules that detect `synchronized(this)` blocks or `this` references.

---

## 6. Synchronized block detection in Kotlin

Kotlin's `synchronized(lock) { ... }` is a **function call with trailing lambda**, not a
language construct like Java's `synchronized` statement.

**AST structure:**
```
PostfixUnaryExpression
  PrimaryExpression / SimpleIdentifier / T-Identifier[@Text='synchronized']
  PostfixUnarySuffix
    CallSuffix
      ValueArguments / ValueArgument / ... / T-Identifier[@Text=<lockArg>]
      AnnotatedLambda
        LambdaLiteral
          Statements / ... (code inside the synchronized block)
```

**Correct detection pattern** — check if code is inside a synchronized lambda:

```xpath
(: inside ANY synchronized block :)
ancestor::LambdaLiteral[
  ancestor::PostfixUnaryExpression[
    PrimaryExpression//T-Identifier[@Text='synchronized']
  ]
]

(: inside synchronized(this) specifically :)
ancestor::LambdaLiteral[
  ancestor::PostfixUnaryExpression[
    PrimaryExpression//T-Identifier[@Text='synchronized']
    and .//CallSuffix/ValueArguments//T-Identifier[@Text='this']
  ]
]

(: inside synchronized(lockField) specifically :)
ancestor::LambdaLiteral[
  ancestor::PostfixUnaryExpression[
    PrimaryExpression//T-Identifier[@Text='synchronized']
    and .//CallSuffix/ValueArguments//T-Identifier[@Text='lockFieldName']
  ]
]
```

**WRONG approach (broken):** The `preceding-sibling` approach does NOT work for code
inside the synchronized lambda body:

```xpath
(: BROKEN — only catches code in statements after the synchronized call, not inside it :)
ancestor::Statement[
  preceding-sibling::*[1][self::Statement]//T-Identifier[@Text='synchronized']
]
```

**Note:** Two older rules (AvoidUnguardedAssignment* lines ~212, ~432) still use the
preceding-sibling approach. The LambdaLiteral approach is correct and used by all newer rules.

**Applies to:** All rules that need to detect whether code is inside a `synchronized` block,
including the GuardedBy rules and the AvoidUnguardedMutable* rules.

---

## 7. `@GuardedBy` annotation value access

Kotlin annotation string values are accessed via `T-LineStrText`, not via Java's
`StringLiteral/@ConstValue`:

```xpath
(: match @GuardedBy("this") :)
//Annotation[.//T-Identifier[@Text='GuardedBy']]//T-LineStrText[@Text='this']

(: match @GuardedBy("lockFieldName") :)
//Annotation[.//T-Identifier[@Text='GuardedBy']]//T-LineStrText[@Text='lockFieldName']
```

**Important:** Quotes are NOT part of `@Text`. Use `@Text='this'`, not `@Text='"this"'`.

**Escaped characters:** Kotlin `"\$lock"` (literal `$lock`) is split into
`T-LineStrEscapedChar` (`\$`) and `T-LineStrText` (`lock`). The `T-LineStrText` only
contains `lock`, not `$lock`. This makes `$`-prefixed annotation values hard to match.
Lombok's `$lock`/`$LOCK` convention is Java-only and excluded from Kotlin rules.

**Non-string arguments:** `@GuardedBy(CONSTANT)` uses a variable reference instead of a
string literal. Detect with:

```xpath
//Annotation[.//T-Identifier[@Text='GuardedBy']]
  [.//ValueArguments/ValueArgument[not(.//LineStringLiteral)]]
```

---

## 8. Cross-referencing `@GuardedBy` values with field names

A common pattern in GuardedBy rules: check if the `@GuardedBy` annotation value matches
an actual field name in the class, or if a `synchronized` argument matches the `@GuardedBy` value.

**Check if @GuardedBy value IS a field name:**
```xpath
//T-LineStrText[
  @Text = ancestor::ClassBody[1]//PropertyDeclaration[not(ancestor::FunctionBody)]
    /VariableDeclaration/@Identifier
]
```

**Check if synchronized argument matches any @GuardedBy value in the class:**
```xpath
ancestor::LambdaLiteral[
  ancestor::PostfixUnaryExpression[
    PrimaryExpression//T-Identifier[@Text='synchronized']
    and .//CallSuffix/ValueArguments//T-Identifier/@Text =
      ancestor::ClassBody[1]//PropertyDeclaration[
        not(ancestor::FunctionBody)
        and .//Annotation[.//T-Identifier[@Text='GuardedBy']]
      ]//T-LineStrText/@Text
  ]
]
```

**Limitation:** When multiple fields have different `@GuardedBy` values, the simplified check
matches ANY @GuardedBy value against the synchronized argument. This can produce false negatives
with multiple different locks but avoids complex per-field cross-referencing in XPath.

---

## 9. Detecting reads vs writes to fields

**Writes (assignments):** Use `Assignment` node — the left-hand side contains the target:

```xpath
//Assignment/(DirectlyAssignableExpression|AssignableExpression)//T-Identifier[
  @Text = <fieldName>
]
```

**Reads:** Use `PrimaryExpression/SimpleIdentifier/T-Identifier` — matches standalone
identifier references:

```xpath
//PrimaryExpression[
  ancestor::FunctionBody
  and SimpleIdentifier/T-Identifier[@Text = <fieldName>]
  (: exclude function calls — identifier followed by CallSuffix :)
  and not(parent::PostfixUnaryExpression/PostfixUnarySuffix/CallSuffix)
]
```

**Limitations:**
- Read detection doesn't catch `this.property` (uses `NavigationSuffix`, not `PrimaryExpression`)
- Function names also match `PrimaryExpression` pattern but are excluded by the `CallSuffix` check
- No direct Kotlin equivalent of Java's `VariableAccess` node

**Guard with `ancestor::FunctionBody`** to restrict to code inside functions (excludes
property initializers, which are construction-time and not a thread-safety concern).
