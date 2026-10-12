# Kotlin "nextgen" migration notes — kotlin-type-mapper → nextgen overlay

Branch: `pmd7-issue-674-kotlin-type-mapper-nextgen` (based on `upstream/pmd7-issue-674`, i.e. PR #687)

## Goal

Port the new/modified Kotlin rules from the `kotlin-type-mapper` branch (stokpop fork) into the
"nextgen" overlay structure introduced in PR #687, so they:
- use real PMD 7.28.0 type-resolution XPath functions (`pmd-kotlin:typeIs`, `pmd-kotlin:hasAnnotation`,
  `pmd-kotlin:matchesSig`), and
- live in `*nextgen.xml` source files, so the standard `jpinpoint-kotlin-rules.xml` keeps working on
  older PMD versions (see README "Nextgen rules" section for the overlay mechanism itself).

Per explicit decision: **no dual safe/unsafe split** — we assume the aux classpath is always available,
so the old syntactic/heuristic fallback versions are dropped; every ported rule goes straight into a
`*nextgen.xml` file (`commonnextgen.xml`, `concurrentnextgen.xml`, `remotingnextgen.xml`).

## Status (as of this note)

32 of ~103 candidate rule changes migrated and committed, all tests green:

| # | Rule | Nextgen file | Commit |
|---|------|--------------|--------|
| 1 | AvoidExcessiveClassloadingWithSaajSoap | remotingnextgen.xml | 485c0dd7 |
| 2 | SynchronizingForLocalVars | concurrentnextgen.xml (new category) | 5dad49c1 |
| 3 | AvoidCompletionServiceTake | concurrentnextgen.xml | 4cef4ea5 |
| 4 | AvoidFutureGetWithoutTimeout | concurrentnextgen.xml | 4cef4ea5 |
| 5 | AvoidCommonPoolForBlockingCalls | concurrentnextgen.xml | 4cef4ea5 |
| 6 | AvoidExecutorInvokeWithoutTimeout | concurrentnextgen.xml | 4cef4ea5 |
| 7 | AvoidParallelStreamWithCommonPool | concurrentnextgen.xml | 4cef4ea5 |
| 8 | AvoidFutureJoinWithoutTimeout | concurrentnextgen.xml | 7f95c440 |
| 9 | AvoidNonPrivateFieldsInSharedObjects | concurrentnextgen.xml | 7f95c440 |
| 10 | AvoidMutableStaticFields | concurrentnextgen.xml | 7f95c440 |
| 11 | AvoidStaticXmlFactories | concurrentnextgen.xml | 7f95c440 |
| 12 | AvoidIncrementOrDecrementForVolatileField | concurrentnextgen.xml | 7f95c440 |
| 13 | AvoidUnguardedAssignmentToNonFinalFieldsInSharedObjects | concurrentnextgen.xml | 7f95c440 |
| 14 | AvoidCommonPoolForFutureAsync | concurrentnextgen.xml | 7f95c440 |
| 15 | AvoidUnguardedAssignmentToNonFinalFieldsInObjectsUsingSynchronized | concurrentnextgen.xml | 7f95c440 |
| 16 | AvoidUnguardedMutableFieldsInSharedObjects | concurrentnextgen.xml | 7f95c440 |
| 17 | AvoidUnguardedMutableFieldsInObjectsUsingSynchronized | concurrentnextgen.xml | 7f95c440 |
| 18 | AvoidUnguardedMutableInheritedFieldsInSharedObjects | concurrentnextgen.xml | c68248e9 |
| 19 | NotProperlySynchronizingOnThisWhileUsingGuardedBy | concurrentnextgen.xml | c68248e9 |
| 20 | NotProperlySynchronizingOnFieldWhileUsingGuardedBy | concurrentnextgen.xml | c68248e9 |
| 21 | WrongUseOfGuardedBy | concurrentnextgen.xml | c68248e9 |
| 22 | AvoidParallelFlux | concurrentnextgen.xml | c68248e9 |
| 23 | AvoidThreadUnsafeJaxbUsage | concurrentnextgen.xml | c5d1a8aa |
| 24 | AvoidReactorOnEachOperator | concurrentnextgen.xml | c5d1a8aa |
| 25 | AvoidMDCInReactor | concurrentnextgen.xml | c5d1a8aa |
| 26 | AvoidNonAtomicIfModifyOnConcurrentMap | concurrentnextgen.xml | c5d1a8aa |
| 27 | AvoidUserDataInSharedObjects | concurrentnextgen.xml | c5d1a8aa |
| 28 | AvoidVolatileInPrototypeScope | concurrentnextgen.xml | (this commit) |
| 29 | AvoidMutableCollectionInParallelStreamForEach | concurrentnextgen.xml | (this commit) |
| 30 | AvoidXMLGregorianCalendar | remotingnextgen.xml | (this commit) |
| 31 | AvoidObjectMapperAsField | remotingnextgen.xml | (this commit) |
| 32 | ObjectMapperCreatedForEachMethodCall | remotingnextgen.xml | (this commit) |

## Full remaining scope

`kotlin-type-mapper` touches three category files. Compared against the PR #687 base
(`upstream/pmd7-issue-674`):

- **common.xml**: 31 brand-new rules + 12 modified existing rules (31+12 = 43 items)
- **concurrent.xml**: entirely new category, 29 rules total (11 done, 18 remaining)
- **remoting.xml**: 31 rules total, mostly modifications of existing rules (1 done, 30 remaining)

New rules in common.xml (not yet migrated):
AvoidCalendar, AvoidCalendarDateCreation, AvoidDuplicateAssignmentsInCases, AvoidInfiniteRecursion,
AvoidLoadingAllFromFile, AvoidMutableLists, AvoidRecreatingSecurityProviders,
AvoidReflectionInToStringAndHashCode, AvoidReStreamingEnumValues, AvoidTimeUnitConfusion,
AvoidUnconditionalBuiltLogStrings, AvoidUnnecessaryStringBuilderCreation, AvoidXPathUsage,
BufferFilesNewStream, BufferFileStreaming, EqualsOperationInconsistentWithHashCode,
FieldOfHashCodeMissingInEquals, HashCodeOnlyCallsSuper, ImplementEqualsHashCodeOnValueObjects,
ImproperVariableName, InitializeComparatorOnlyOnce, LegacyTimeLibraryUsed, MissingFieldInEquals,
MissingFieldInHashCode, NonComparableMapKeys, NonComparableSetElements, OptimizeMapOrSetForEnum,
UnconditionalConcatInLogArgument, UnconditionalCreatedLogArguments, UnconditionalOperationOnLogArgument,
UnresolvedType, UsingSuppressWarnings, UsingSuppressWarningsHighRisk

Modified rules in common.xml (exist in base too — need `<exclude>` overlay treatment like
`AvoidMultipleConcatStatements`, i.e. keep the base version untouched/safe, put the type-aware rewrite
in commonnextgen.xml):
AvoidDecimalAndChoiceFormatAsField, AvoidImplicitlyRecompilingRegex,
AvoidInMemoryStreamingDefaultConstructor, AvoidMultipleConcatStatements (done, pre-existing),
AvoidRecompilingPatterns, AvoidRecompilingXPathExpression, AvoidRecreatingDateTimeFormatter,
AvoidSimpleDateFormat, AvoidStringBuffer, AvoidWideScopeXPathExpression, AvoidXPathAPIUsage,
MDCPutWithoutRemove

Remaining concurrent.xml rules: none
(NB: `SetQueueCapacityForTaskExecutor` also appears under concurrent.xml in the source branch, but is
a stale duplicate — resolved below, only the remoting.xml version is migrated)

Remaining remoting.xml rules: GsonCreatedForEachMethodCall,
HttpClientBuilderWithoutDisableConnectionState, HttpClientBuilderWithoutTimeouts,
AvoidDeprecatedHystrix, HttpClientBuilderWithoutPoolSize, NettyConnectionProviderWithoutMaxOrTimeout,
BufferingClientHttpRequestFactoryIsMemoryGreedy, AvoidHttpHostOneArgumentConstructor, AvoidJAXBUtil,
JAXBContextCreatedForEachMethodCall, AvoidReactorDebugOverhead, AvoidRecreatingHttpClient,
AvoidLeakingRetryEventConsumers, RetryCanCauseOverload, AvoidModifyingObjectMapper,
AvoidDeprecatedHttpConnectors, DefaultFeignClientWithoutTLSConnectionReuse,
AvoidHardcodedConnectionConfig, HttpClientImproperConnectionTimeouts, UseCloseableForHttpClient,
AvoidUnclearHttpRouteSecurity, AvoidDiscardingHttpClientConfig, HttpClientBuilderPoolSettingsIgnored,
AxualProducerCreatedForEachMethodCall, AvoidClientHttpRequestFactorySupplier,
HttpInterceptorNotReleasingOnException, SetQueueCapacityForTaskExecutor (the canonical, type-aware
version — see resolution below)

Note: `SetQueueCapacityForTaskExecutor` appears in both concurrent.xml and remoting.xml in
`kotlin-type-mapper`. **Resolved:** the concurrent.xml copy is a stale pre-type-aware draft (bare
`T-Identifier[@Text='ThreadPoolTaskExecutor']` string matching, no `matchesSig`); the remoting.xml copy
is the real type-aware rewrite (`pmd-kotlin:matchesSig('...ThreadPoolTaskExecutor#<init>(*)')`). The
Java equivalent rule also lives only in `category/java/remoting.xml`, confirming remoting is the
canonical category. **Decision: migrate only the remoting.xml version into `remotingnextgen.xml`;
ignore/drop the concurrent.xml duplicate.**

## Per-rule migration procedure (established, repeat for each)

1. Extract the rule + its test file from `kotlin-type-mapper` (`git show kotlin-type-mapper:<path>`).
2. If the rule is **new** (no equivalent in base): add it directly into the matching `*nextgen.xml`
   file (`commonnextgen.xml` / `concurrentnextgen.xml` / `remotingnextgen.xml`), creating the file +
   category if it doesn't exist yet.
3. If the rule **modifies an existing base rule**: do NOT touch the base file. Add the full rewritten
   rule (same `name`) into the matching `*nextgen.xml` file — the merger's overlay mechanism
   auto-generates the `<exclude>` for it (see `RulesetMerger.mergeNextgenOverlay`).
4. Fix known-bad constructs from the source branch (see "Known pitfalls" below) before testing.
5. Copy/port the test file verbatim into
   `src/test/resources/.../ruleset/<category>nextgen/xml/<Rule>.xml`, and add a stub test class in
   `src/test/java/.../ruleset/<category>nextgen/<Rule>Test.java` (`extends PmdRuleTst`, empty body).
6. Add any missing test-only Maven dependency (check `kotlin-type-mapper`'s `pom.xml` diff for the
   commit that added the rule).
7. Run `mvn -q -o test -Dtest=<Rule>Test` and iterate until green — do NOT skip this, the source
   branch has multiple confirmed bugs (see below).
8. Run `./merge kotlin` to regenerate `rulesets/kotlin/jpinpoint-kotlin-rules-nextgen.xml`.
9. Run the full Kotlin suite (`mvn -q -o test -Dtest="com.jpinpoint.perf.lang.kotlin.**"`) before
   committing, to make sure nothing else regressed.
10. Commit with a message explaining what was ported and, importantly, **what was fixed and why**
    (the source branch was written against an aspirational/future PMD API and has real bugs - don't
    silently "just copy", document corrections). No `Co-authored-by` trailer (explicit user instruction
    for this repo/session).

## Known pitfalls found so far (check for all of these on every ported rule)

1. **`pmd-kotlin:isWithin('function-body'|'lambda')` does not exist in released PMD 7.28.0.**
   It's from a proposed/experimental future API (confirmed with the user). 71 occurrences across
   common/concurrent/remoting in the source branch — expect to hit this constantly.
   - Fix: `PropertyDeclaration` is the same node type for class-level, local, and lambda-local
     variable/property declarations; only the ancestor axis distinguishes them. Replace
     `not(pmd-kotlin:isWithin('function-body') or pmd-kotlin:isWithin('lambda'))` with
     `not(ancestor::FunctionBody or ancestor::LambdaLiteral)`, and a bare
     `pmd-kotlin:isWithin('function-body')` with `ancestor::FunctionBody`.
   - **Always mark every replacement with the literal comment `(: isWithin pattern :)`** (nested
     inside a normal `(: ... :)` explanatory comment — XPath 3.1/Saxon comments nest correctly, this
     was verified). This makes all occurrences grep-able later, in case PMD ever actually ships
     `isWithin()` and we want to simplify back.

2. **`@Identifier` is not a universal "name of this identifier" attribute.** It only exists as a
   convenience attribute on *declaration* nodes (`ClassDeclaration`, `FunctionDeclaration`,
   `VariableDeclaration`/`PropertyDeclaration`). For identifier *references* inside expressions
   (e.g. `System` in `System.setProperty(...)`, or a method name in `foo.bar()`), there is no
   shorthand — you must navigate to `SimpleIdentifier/T-Identifier/@Text` (or a wildcard, see next
   point). The source branch uses `[@Identifier='xyz']` on `PrimaryExpression`/`NavigationSuffix`
   nodes in several places; this silently never matches (0 violations, not an error) and must be
   fixed to `.../SimpleIdentifier/T-Identifier[@Text='xyz']`.

3. **Kotlin soft keywords are tokenized as their own node type, not `T-Identifier` — unconditionally,
   regardless of syntactic role.** E.g. `get` is tokenized as `T-GET`, not `T-Identifier`. Verified this
   is NOT limited to actual property-accessor syntax: a `fun get() {}` declaration, a bare `get()`
   call, `map.get("x")`, and `this.get()` all produce `T-GET` for the literal text "get". PMD's Kotlin
   lexer assigns the token type based on the literal word alone ("soft keyword"), not on how it's being
   used; the parser grammar still accepts `T-GET` in identifier positions (so `get()` as an ordinary
   method call parses fine), but the resulting AST node's tag reflects the lexer token type, not the
   syntactic role. A check like `NavigationSuffix/SimpleIdentifier/T-Identifier[@Text='get']` silently
   matches nothing even though the call itself (and `matchesSig`) resolves fine. Use a wildcard
   instead: `SimpleIdentifier/*[@Text='get']`. Expect other soft keywords (`set`, possibly others) to
   have the same issue — check with `ast-dump` whenever a `@Text='...'` check unexpectedly matches 0
   results. This was found via `AvoidFutureGetWithoutTimeout` (6/6 expected violations silently
   dropped to 0).
   Note: `@Identifier` (pitfall #2) is NOT affected by this — it's a PMD-computed string attribute on
   declaration nodes, confirmed working correctly even for `fun get() {}` (`@Identifier='get'`).
   The two pitfalls are orthogonal: `@Identifier` fixes keyword-tokenization only for *declaration*
   nodes; call-site/reference matching (e.g. `future.get()`) has no `@Identifier` shorthand at all and
   still needs the `SimpleIdentifier/*[@Text='...']` wildcard fix.

   **Potential upstream PMD improvement (not yet filed):** consider proposing to the PMD project that
   `@Identifier` (or an equivalent convenience attribute) be extended to expression/reference nodes
   (`PrimaryExpression`, `NavigationSuffix`, `SimpleIdentifier`) wherever they resolve to a plain
   identifier/call name — not just declaration nodes. This would let XPath rules match identifier text
   uniformly without needing to know about soft-keyword token types (`T-GET`, `T-SET`, etc.) or fall
   back to `*`-wildcards. This is a "nice to have" QoL improvement for `pmd-kotlin`, not a blocker —
   our `SimpleIdentifier/*[@Text='...']` workaround is reliable and cheap. File as a GitHub issue on
   `pmd/pmd` if/when time allows.

4. **Always scrutinize "extra" conditions placed alongside `matchesSig()`/`typeIs()`/`hasAnnotation()`
   calls** (explicit instruction from the user, 2026-10-12). `matchesSig(...)` on its own already
   precisely identifies the exact call (receiver type + method + arity); companion attribute checks
   like `NavigationSuffix[...='methodName']` are frequently redundant leftovers from the pre-type-aware
   syntactic rule they were ported from, AND are a common source of the pitfalls above (wrong
   attribute name, keyword tokenization). When in doubt: test the rule WITHOUT the extra condition
   first (does `matchesSig` alone already give the right violations?), then decide if the extra
   condition is truly needed, and if so fix it properly rather than leaving it as dead weight.

5. **`matchesSig()` and `typeIs()` themselves are legitimate, working PMD 7.28.0 functions** — don't
   assume everything from this branch is broken; only `isWithin()` is confirmed fictional/future, and
   the `@Identifier`/keyword-tokenization issues are AST-attribute mistakes, not missing functions.

6. **`X[@Attr]='value'` vs `X[@Attr='value']` are NOT equivalent — a real authoring bug, not a PMD
   bug.** `X[@Attr]='value'` filters `X` nodes to those having `@Attr` (any value), then compares the
   resulting NODE-SET's string-value (i.e. the element's own text content) to `'value'`. For PMD's
   Kotlin AST, token/leaf nodes like `T-Identifier` are self-closing (`<T-Identifier Text='counter' />`)
   with no text content, so their string-value is always empty — the comparison silently never matches.
   The correct form is `X[@Attr='value']` (compares the attribute's value) or `X/@Attr='value'`. Found
   while porting `AvoidIncrementOrDecrementForVolatileField` (6/6 expected violations dropped to 0).
   Always double check whether an unexpected 0-match is a real PMD quirk (there are some — see e.g.
   PMD PR #7168, a genuine line-based type-attribute leak bug in `pmd-kotlin`) vs. our own XPath
   mistake; this one was ours.

7. **Rules using `hasAnnotation()`/`typeIs()`/`matchesSig()` against *non-JDK* library types need the
   real library jar on `--aux-classpath` to test correctly — otherwise they silently produce ZERO
   violations even though the XPath is correct.** PMD itself warns about this at the CLI level:
   `[WARN] No auxClasspath configured for Kotlin; type resolution will not work.` Don't mistake a
   missing-classpath false-negative for an XPath bug when iterating locally — e.g.
   `AvoidNonPrivateFieldsInSharedObjects` (checks `org.springframework.stereotype.Controller` etc.)
   needed `--aux-classpath <path-to-spring-context.jar>` to produce any violations at all; rules
   against `java.util.concurrent.*`/`javax.xml.*` types work without extra classpath since those are
   JDK built-ins always resolvable.
   PMD exposes this gap to rule authors too, via a real, confirmed-working function:
   **`pmd-kotlin:hasUnresolvedReference()`**, usable on `ImportHeader` nodes — this is exactly what
   the (not yet migrated) `UnresolvedType` common.xml rule uses
   (`//ImportHeader[pmd-kotlin:hasUnresolvedReference()]`) to self-diagnose a misconfigured classpath
   for consumers of the ruleset. Confirmed via test: flags the import when aux-classpath is missing,
   clears once the correct jar is supplied. Worth migrating early since it's cheap and helps catch
   exactly this class of silent false-negative for every other type-aware rule in the ruleset.

8. **`pmd-kotlin:modifiers()` returns a SEQUENCE of modifier strings, not a single scalar value** — a
   property/declaration can have multiple modifiers simultaneously (e.g. `private lateinit var x`
   yields both `'private'` and `'lateinit'`; confirmed via `count(pmd-kotlin:modifiers()) > 1`).
   `not(pmd-kotlin:modifiers()='private')` therefore works correctly via XPath's general-comparison
   semantics (sequence `=` string is existential — true if ANY item matches), but should be read/
   described as "modifiers() contains 'private'", not "modifiers() equals 'private'" — the `=` is
   doing containment, not scalar equality. No bug here, just a naming/mental-model clarification worth
   keeping in mind so nobody "fixes" it into a broken single-value comparison later.

9. **`pmd-kotlin:matchesSig()` can spuriously match an OUTER ancestor `PostfixUnaryExpression` for
   multi-line method chains that end in a trailing lambda** — a suspected genuine `pmd-kotlin`
   bug, not an XPath authoring mistake. Found in `AvoidFutureJoinWithoutTimeout`: for
   `return futures.filter{...}.map{it.join()}` spanning several lines, `matchesSig('...Future#join()')`
   returned `true` not only for the innermost `it.join()` node (correct) but ALSO for the entire outer
   `PostfixUnaryExpression` wrapping the whole `return`-statement chain, even though `.join()` is nested
   three levels deep inside an unrelated lambda body, not a direct suffix of that outer node at all.
   This produced 4 matches instead of the expected 2 (two real + two phantom ancestor matches). This
   looks like the same class of line/position-based type-attribution bug as PMD PR #7168 and
   `stokpop/kotlin-type-mapper#36` ("fix: property declaration position and inferred lambda
   parameters") — type/call info leaking onto a neighbouring or enclosing node for multi-line
   constructs with lambdas.
   - **Fix/mitigation**: add a direct-child (not descendant, not `[last()]`) scoping condition
     confirming the node's own immediate suffix really is the call in question, e.g.
     `PostfixUnarySuffix/NavigationSuffix/SimpleIdentifier/*/@Text = 'join'`. This restricts the match
     back down to the exact node whose own suffix is `.join()`, excluding phantom ancestor matches.
   - **This refines pitfall #4 above** (scrutinize "extra" conditions alongside `matchesSig()` as
     likely redundant): in this case the extra condition is NOT redundant — it is load-bearing and
     necessary to work around this over-matching bug. Going forward: test rules both WITH and WITHOUT
     the companion condition to empirically determine necessity, rather than assuming redundancy.
   - Not yet audited: whether this affects any other already-migrated rule using `matchesSig()` near
     chained calls with trailing lambdas — worth a dedicated audit pass later.

10. **`pmd-kotlin:hasAnnotation()` / `typeIs()` only resolve correctly when called on the *annotated
    declaration* node itself (e.g. `ClassDeclaration`, `FunctionDeclaration`), NOT on the `Annotation`
    node that represents one specific annotation usage.** Calling `hasAnnotation(...)` or `typeIs(...)`
    directly on an `Annotation` node always returns `false`/no-match, confirmed via isolated test
    (`//Annotation[pmd-kotlin:hasAnnotation(...)]` → 0 matches; `//ClassDeclaration[pmd-kotlin:hasAnnotation(...)]`
    → matches correctly on the same input). This matters whenever a rule needs to read an *argument*
    of one specific annotation (not just test for its presence) — e.g. `AvoidNonPrivateFieldsInSharedObjects`
    needed to read the string argument of `@Scope("request")`/`@Scope("prototype")` to exclude
    non-singleton-scoped classes.
    - **Fix**: use `hasAnnotation()` at the declaration level to confirm a real, resolvable annotation
      of the right fully-qualified type is present, and separately select the specific `Annotation`
      node by its literal simple name (e.g. `.//Annotation[.//SimpleIdentifier/*/@Text='Scope']`) to
      read its arguments. Combining both avoids two failure modes: (a) missing exclusion entirely
      (if you only did the name-based match with no type check, an unrelated `@Scope` from a
      different package would wrongly match too), and (b) false exclusion from an unrelated
      annotation's argument text coincidentally containing the same string (e.g.
      `@RequestMapping("request")` should NOT be mistaken for `@Scope("request")`) — tested and
      confirmed this combo correctly rejects that case.

11. **`<rule ref="some-sibling-ruleset.xml">` (used by the generated `-nextgen` overlay files to
    reference the regular merged ruleset and exclude/override select rules) does not resolve when the
    referencing ruleset is loaded via a remote URL (e.g. the IntelliJ PMD plugin loading our ruleset
    straight from its `raw.githubusercontent.com` URL)** — confirmed not an IntelliJ-plugin-specific
    bug but a `pmd-core` limitation: `RuleSetReferenceId.resolveSibling()` only ever resolves a sibling
    ref using `java.io.File` against the parent's *local filesystem* directory; it has no concept of
    resolving relative to a parent URL. When the parent is a URL, sibling resolution silently falls
    through, and the bare filename then fails to resolve via CWD/classpath/absolute-URL too, surfacing
    an error in IntelliJ that previously required manually removing the `<rule ref="...">` block.
    - **Fix applied**: changed `RulesetMerger`'s nextgen-overlay generation to fully **inline** all
      non-overridden rules from the base ruleset file(s) directly into the `-nextgen` output, instead
      of using `<rule ref="...">` + `<exclude>`. This makes the generated `-nextgen` ruleset file fully
      self-contained (no external ruleset references at all), so it works identically whether loaded
      from a local file, the Maven classpath, or a remote URL. Trade-off: the generated file is larger
      (duplicates the base ruleset's content), but this is consistent with how the regular (non-nextgen)
      merged ruleset file already works (it also fully inlines all category files).
    - Filed as upstream improvement idea #5 below (resolving sibling refs relative to a parent URL).

12. **A rule's `xpath` property must be ONE XPath expression — the source branch has several rules
    written as two disconnected top-level expressions** (e.g. `//ClassDeclaration[...]` followed by
    an unrelated `//Assignment[...]` or `//PropertyDeclaration[...]` on the next lines, with no
    operator joining them at all). This is invalid/meaningless XPath (effectively the second
    expression silently wins, or it fails to compile, depending on the engine) and was never actually
    tested as written. Found in `AvoidUnguardedAssignmentToNonFinalFieldsInSharedObjects`,
    `AvoidUnguardedAssignmentToNonFinalFieldsInObjectsUsingSynchronized`,
    `AvoidUnguardedMutableFieldsInSharedObjects`, and
    `AvoidUnguardedMutableFieldsInObjectsUsingSynchronized` — all four had a "Step 1: match the
    shared/synchronized class" expression and a "Step 2: find the field/assignment" expression as two
    separate top-level statements.
    - **Fix**: combine into a single `//ClassDeclaration[<step-1-conditions>]//Assignment[<step-2-conditions>]`
      (or `.../PropertyDeclaration[...]`) path — the same descendant-combinator pattern already proven
      working in `AvoidNonPrivateFieldsInSharedObjects`. This correctly scopes "step 2" nodes to only
      those found *within* a class matching "step 1", instead of two independent unrelated searches.

13. **`pmd-kotlin:matchesSig()` CAN resolve Kotlin's `synchronized(lock) { ... }` block despite it
    being an `inline` stdlib function with no real JVM call-site** (confirmed empirically — don't
    assume inline functions are unmatchable). Two things had to be gotten right:
    - The owner in the signature string must be the **bare package name**, not a facade class:
      `'kotlin#synchronized(_,_)'` works; `'kotlin.StandardKt#synchronized(_,_)'` and
      `'kotlin.SynchronizedKt#synchronized(_,_)'` do not (confirmed via `javap` that the generated
      bytecode method is `private static final` on `kotlin.StandardKt__SynchronizedKt` — there is no
      public JVM method to match a binary signature against at all, since the compiler copies the body
      into every call site instead of emitting a real `invokestatic`; contrast with
      `kotlin.collections.filter()`, also `inline`, which DOES emit a real public method for
      Java-interop and resolves fine under its *extension-receiver type*, e.g.
      `'kotlin.collections.Iterable#filter(_)'`, not a facade class either).
    - This works with or without an explicit `--aux-classpath` pointing at the `kotlin-stdlib` jar
      (unlike pitfall #7's non-JDK library requirement) — `kotlin.*` package resolution appears to be
      bundled with `pmd-kotlin` itself.
    - **Preferred pattern for "is this call/ancestor a call to kotlin's `synchronized` block"**:
      `ancestor::PostfixUnaryExpression[pmd-kotlin:matchesSig('kotlin#synchronized(_,_)')]` — strictly
      better than matching on the literal identifier text (`SimpleIdentifier/T-Identifier[@Text='synchronized']`),
      since it can't be fooled by an unrelated user-defined function also named `synchronized` with a
      different signature, and is immune to any future soft-keyword-tokenization surprises (pitfall #3).
    - No upstream fix needed here — this was a case of using the wrong signature-string owner, not a
      `pmd-kotlin`/`kotlin-type-mapper` bug.

14. **`FunctionValueParameter/Parameter` has no `@Identifier` attribute — another concrete instance of
    pitfall #2, found in the second 5-rule batch.** The source branch repeatedly wrote
    `ancestor::FunctionDeclaration//FunctionValueParameter/Parameter/@Identifier` to compare a
    receiver/assignment-target identifier against the enclosing function's parameter names (to
    exclude "it's just a local parameter, not the field" false positives). Since `Parameter` has no
    `@Identifier` attribute (confirmed via `ast-dump`: the name lives at
    `Parameter/SimpleIdentifier/T-Identifier/@Text`), the RHS of that `=` comparison was always an
    empty node-set, so the comparison was always `false` and the surrounding `not(...)` exclusion
    always evaluated `true` — i.e. the exclusion silently never excluded anything. This was *latent*
    in 3 already-migrated rules from the first 5-rule batch
    (`AvoidUnguardedAssignmentToNonFinalFieldsInSharedObjects`,
    `AvoidUnguardedAssignmentToNonFinalFieldsInObjectsUsingSynchronized`,
    `AvoidUnguardedMutableFieldsInSharedObjects`) — their existing test suites happened not to include
    a "receiver is a function parameter with the same name as a field" case, so it went undetected
    until a new test case in `AvoidUnguardedMutableInheritedFieldsInSharedObjects` (this batch)
    surfaced it as a false positive ("no violation: mutator on function parameter" unexpectedly
    flagged).
    - **Fix**: replace every `.../Parameter/@Identifier` with
      `.../Parameter/SimpleIdentifier/T-Identifier/@Text`. Fixed in all 4 occurrences across the
      `concurrentnextgen.xml` file (the 3 latent ones plus the one introduced this batch).
    - **Lesson reinforced**: a "found 0 violations when N expected" or (worse) a passing test suite
      with a narrow set of test cases does NOT prove an exclusion clause is correct — a clause that
      always evaluates to "not excluded" looks identical to a correctly-passing test until a test case
      specifically targets the excluded scenario. Worth deliberately adding a test case for every
      `not(@Text = ...)`-style exclusion clause, not just for the main positive/negative cases.

## Upstream PMD improvement ideas (tracked, not yet filed)

Ideas worth proposing to the `pmd/pmd` project, discovered while doing this migration. None are
blockers — each has a reliable workaround in use today — but filing them could simplify future
`pmd-kotlin` XPath rules.

| # | Idea | Expected impact | Status |
|---|------|------------------|--------|
| 1 | Extend `@Identifier` (or an equivalent convenience attribute) to expression/reference nodes (`PrimaryExpression`, `NavigationSuffix`, `SimpleIdentifier`) wherever they resolve to a plain identifier/call name, not just declaration nodes. | Would let XPath rules match identifier/method-call text uniformly, without needing to know about Kotlin soft-keyword token types (`T-GET`, `T-SET`, etc.) or fall back to `*`-wildcard children. Removes a whole class of silent-zero-match bugs (pitfalls #2 and #3 above) across all our Kotlin rules, present and future. | Not filed |
| 2 | Ship the `pmd-kotlin:isWithin(scopeName)` function that the `kotlin-type-mapper` source branch was apparently written against (e.g. `isWithin('function-body')`, `isWithin('lambda')`) — or document officially that it's not planned, so rule authors stop assuming it exists. | Would let us drop the `not(ancestor::FunctionBody or ancestor::LambdaLiteral)` workaround (71 occurrences expected across this migration) and simplify every affected rule back to a single function call. Each occurrence is marked with `(: isWithin pattern :)` precisely so they can be grepped and reverted if this ships. | Not filed |
| 3 | Fix `pmd-kotlin:matchesSig()` spuriously matching an outer ancestor `PostfixUnaryExpression` for multi-line method chains ending in a trailing lambda (see pitfall #9). | Would remove the need for a direct-child scoping condition as a workaround in rules like `AvoidFutureJoinWithoutTimeout`, and eliminate a class of silent over-matching (false positives) for any rule using `matchesSig()` near chained calls with lambda arguments. | Not filed |
| 4 | Make `pmd-kotlin:hasAnnotation()`/`typeIs()` also work when called directly on an `Annotation` node (not just on the annotated declaration node), so a specific annotation usage can be both type-confirmed and introspected for arguments in one step (see pitfall #10). | Would simplify rules that need to read one specific annotation's arguments (e.g. `@Scope("request")`) — currently requires combining a declaration-level `hasAnnotation()` check with a separate name-based `Annotation` node lookup. | Not filed |
| 5 | Support resolving a bare-filename `<rule ref="other-ruleset.xml">` relative to the URL the *referencing* ruleset itself was loaded from (not just relative to a local-filesystem sibling directory, which is all `RuleSetReferenceId.resolveSibling()` currently does via `java.io.File`). Today, a ruleset loaded from e.g. a `raw.githubusercontent.com` URL (as the IntelliJ PMD plugin does) cannot `ref` a sibling ruleset by bare filename at all — confirmed by reading `pmd-core`'s `RuleSetReferenceId.resolveSibling()` source, which only ever constructs a `java.io.File` sibling path, never resolves against a parent URL. | Would let us go back to the simpler `<rule ref="jpinpoint-kotlin-rules.xml"><exclude .../></rule>` overlay pattern for `jpinpoint-kotlin-rules-nextgen.xml` (and similarly for Java, if a nextgen overlay is ever added there) instead of fully inlining all base-ruleset rules into the overlay file. Found while fixing a real IntelliJ PMD plugin failure: the plugin loads our ruleset via its raw GitHub URL, and the bare-filename ref could not resolve, forcing users to manually strip it. We worked around it by inlining (see commit regenerating `jpinpoint-kotlin-rules-nextgen.xml`), but filing this upstream could let any PMD-ecosystem tool support the simpler reference-based overlay pattern directly. | Not filed |

## Useful tooling kept around for fast iteration

- Real PMD CLI binaries: `/tmp/pmd-bin-7.28.0` and `/tmp/pmd-bin-7.16.0` (the latter useful to confirm
  the default/non-nextgen ruleset still works on old PMD, not needed for nextgen-only work).
- Fastest iteration loop for a single rule/snippet: write a minimal single-rule ruleset XML + a `.kt`
  sample under `/tmp/...`, then
  `/tmp/pmd-bin-7.28.0/bin/pmd check -d <file.kt> -R <ruleset.xml> -f text --no-cache` (add
  `--aux-classpath <jar1>:<jar2>` if the rule needs type resolution against a library not on the JDK).
- `ast-dump` is invaluable for checking exact attribute names/node types:
  `/tmp/pmd-bin-7.28.0/bin/pmd ast-dump --file <file.kt> --language kotlin`.
- `./merge kotlin` regenerates both `rulesets/kotlin/jpinpoint-kotlin-rules.xml` (base, excludes
  `*nextgen*` source files) and `rulesets/kotlin/jpinpoint-kotlin-rules-nextgen.xml` (overlay, built
  only from `*nextgen.xml` source files) — always re-run after any nextgen source file change and
  commit the regenerated output alongside.
