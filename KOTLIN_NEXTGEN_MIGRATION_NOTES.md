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

7 of ~103 candidate rule changes migrated and committed, all tests green:

| # | Rule | Nextgen file | Commit |
|---|------|--------------|--------|
| 1 | AvoidExcessiveClassloadingWithSaajSoap | remotingnextgen.xml | 485c0dd7 |
| 2 | SynchronizingForLocalVars | concurrentnextgen.xml (new category) | 5dad49c1 |
| 3 | AvoidCompletionServiceTake | concurrentnextgen.xml | 4cef4ea5 |
| 4 | AvoidFutureGetWithoutTimeout | concurrentnextgen.xml | 4cef4ea5 |
| 5 | AvoidCommonPoolForBlockingCalls | concurrentnextgen.xml | 4cef4ea5 |
| 6 | AvoidExecutorInvokeWithoutTimeout | concurrentnextgen.xml | 4cef4ea5 |
| 7 | AvoidParallelStreamWithCommonPool | concurrentnextgen.xml | 4cef4ea5 |

## Full remaining scope

`kotlin-type-mapper` touches three category files. Compared against the PR #687 base
(`upstream/pmd7-issue-674`):

- **common.xml**: 31 brand-new rules + 12 modified existing rules (31+12 = 43 items)
- **concurrent.xml**: entirely new category, 29 rules total (6 done, 23 remaining)
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

Remaining concurrent.xml rules: AvoidFutureJoinWithoutTimeout,
AvoidUnguardedAssignmentToNonFinalFieldsInSharedObjects, AvoidNonPrivateFieldsInSharedObjects,
AvoidCommonPoolForFutureAsync, AvoidUnguardedAssignmentToNonFinalFieldsInObjectsUsingSynchronized,
AvoidUnguardedMutableFieldsInSharedObjects, AvoidUnguardedMutableFieldsInObjectsUsingSynchronized,
AvoidUnguardedMutableInheritedFieldsInSharedObjects, NotProperlySynchronizingOnThisWhileUsingGuardedBy,
NotProperlySynchronizingOnFieldWhileUsingGuardedBy, WrongUseOfGuardedBy, AvoidStaticXmlFactories,
AvoidMutableStaticFields, AvoidParallelFlux, AvoidThreadUnsafeJaxbUsage, AvoidReactorOnEachOperator,
AvoidMDCInReactor, AvoidNonAtomicIfModifyOnConcurrentMap, AvoidIncrementOrDecrementForVolatileField,
AvoidUserDataInSharedObjects, AvoidVolatileInPrototypeScope,
AvoidMutableCollectionInParallelStreamForEach
(NB: `SetQueueCapacityForTaskExecutor` also appears under concurrent.xml in the source branch, but is
a stale duplicate — resolved below, only the remoting.xml version is migrated)

Remaining remoting.xml rules: AvoidXMLGregorianCalendar, AvoidObjectMapperAsField,
ObjectMapperCreatedForEachMethodCall, GsonCreatedForEachMethodCall,
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

## Upstream PMD improvement ideas (tracked, not yet filed)

Ideas worth proposing to the `pmd/pmd` project, discovered while doing this migration. None are
blockers — each has a reliable workaround in use today — but filing them could simplify future
`pmd-kotlin` XPath rules.

| # | Idea | Expected impact | Status |
|---|------|------------------|--------|
| 1 | Extend `@Identifier` (or an equivalent convenience attribute) to expression/reference nodes (`PrimaryExpression`, `NavigationSuffix`, `SimpleIdentifier`) wherever they resolve to a plain identifier/call name, not just declaration nodes. | Would let XPath rules match identifier/method-call text uniformly, without needing to know about Kotlin soft-keyword token types (`T-GET`, `T-SET`, etc.) or fall back to `*`-wildcard children. Removes a whole class of silent-zero-match bugs (pitfalls #2 and #3 above) across all our Kotlin rules, present and future. | Not filed |
| 2 | Ship the `pmd-kotlin:isWithin(scopeName)` function that the `kotlin-type-mapper` source branch was apparently written against (e.g. `isWithin('function-body')`, `isWithin('lambda')`) — or document officially that it's not planned, so rule authors stop assuming it exists. | Would let us drop the `not(ancestor::FunctionBody or ancestor::LambdaLiteral)` workaround (71 occurrences expected across this migration) and simplify every affected rule back to a single function call. Each occurrence is marked with `(: isWithin pattern :)` precisely so they can be grepped and reverted if this ships. | Not filed |

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
