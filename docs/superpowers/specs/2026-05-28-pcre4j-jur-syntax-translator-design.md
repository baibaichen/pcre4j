# pcre4j Java→PCRE2 Syntax Translator — Design Spec

**Date:** 2026-05-28
**Branch:** `chang/compat-test` (continued)
**Companion data:** `compat-test/build/reports/compat/{raw.jsonl,report.md}` from sibling spec `2026-05-28-pcre4j-jur-compat-verification-design.md`

## Goal

Reduce the 275 / 859 (32%) failures observed by the compat-test harness by adding a Java-regex → PCRE2 syntax translation layer inside `org.pcre4j.regex.Pattern.compile()`. Target: **≥ 73% of current failures eliminated** (compat rate 68% → ~91% on `.txt` data).

Only `pcre4j` is modified. PCRE2 source is untouched. The translator must work against the system libpcre2 (10.42), not relying on `PCRE2_ALT_EXTENDED_CLASS` (10.45+).

## Scope (in)

The translator must handle, in declared order of fix value:

1. **Block-property normalization** — `\p{InXxx}` / `\P{InXxx}` → `\p{Xxx}` / `\P{Xxx}` (using PCRE2's `Block:` namespace) or, when not directly named, an equivalent code-point range. 64 failures.
2. **Script-property normalization** — `\p{IsXxx}` / `\P{IsXxx}` → `\p{Xxx}` / `\P{Xxx}`. 12 failures.
3. **Short-alias property table** — `L1`, `IsASCII`, `IsLC`, `IsPf`, `IsP`, `IsL`, etc. → PCRE2 equivalent or expanded class. 6 failures.
4. **`\p{javaXxx}` Java-specific properties** — `javaWhitespace`, `javaDigit`, `javaLowerCase`, … → POSIX-or-equivalent PCRE2 class. ~unknown count but classifier reports presence; covered as best-effort with a documented mapping table.
5. **Nested character-class union** — `[abc[def]]` / `[a-c[d-f[g-i]]]` / `[^abc[def]]` → flattened to a single PCRE2 character class via a tiny class-body parser. 43 + most of the ~120 behavior-diffs.
6. **Character-class intersection** — `[A&&B]` / `[A&&[B]]` / `[A&&B&&C]` → expanded using De Morgan: `[A]` ∩ `[B]` = `[^[^A][^B]]` (PCRE2 understands negation inside class). 24 failures.
7. **Composite of the above** — `[\p{L}&&[\P{InGreek}]]` etc. — works because translator runs property normalization first, then class normalization.

## Scope (out)

- Lone surrogate ranges `[\x{d800}-\x{dbff}]` — incompatible with `PCRE2_UTF`; no clean fix without dropping UTF (would break supplementary handling). Document as known limitation.
- `\b{g}` grapheme boundary, `\X` grapheme cluster, `\R` line-break — PCRE2 has these directly under different feature flags; deferred (not in current failure set significantly).
- Replacement string syntax (`$1`, named back-refs) — handled in `Matcher.appendReplacement`, separate concern.
- `Pattern.CANON_EQ` semantics — current `Normalizer.NFD` workaround stays.
- `(?U)` inline flag — Java's `(?U)` ≈ `UNICODE_CHARACTER_CLASS` is already handled via `PCRE2_UCP`. We will leave the inline `(?U)` rewrite to a single string substitution → `(*UCP)` callout block (if needed) — but defer if it doesn't show as a top failure.

## Non-functional requirements

- **Zero PCRE2 source changes.** No new C code.
- **Backward compatibility.** Patterns that already work must continue to work. Round-trip test: every pattern that currently passes in the compat harness must still pass after translator activation.
- **Translator off-switch.** A boolean: `Pattern.compile(regex, flags)` always translates; system property `pcre4j.regex.translate=false` disables (escape hatch for users who want raw PCRE2 syntax).
- **Failure mode.** If translation produces a pattern that PCRE2 rejects, the `PatternSyntaxException` must surface the **original** Java pattern and a reasonable offset (best-effort mapping back from translated → original offset; OK to fall back to offset 0).
- **`Pattern.pattern()`** must return the original (untranslated) input, matching JDK semantics.
- **No new dependencies.** Stdlib only.

## Architecture

```
org.pcre4j.regex/
  Pattern.java                       (modified: call translator before Pcre2Code.create)
  translate/
    JavaRegexTranslator.java         (public API: translate(String, int) -> String)
    PropertyMap.java                 (data: Java property name → PCRE2 equivalent)
    ClassBodyParser.java             (parses Java class bodies, emits PCRE2-flat class)
    ClassNode.java                   (AST: Literal | Range | Class | Property | Intersection | Negation | Union)
    ClassRenderer.java               (AST → PCRE2 class-body string with De Morgan for &&)
```

### Translator flow

```
input: javaPattern, flags
  1. PropertyRewriter      — single-pass scan, rewrite \p{...}/\P{...} tokens (outside [...] is also valid)
  2. ClassBodyParser       — when scanner enters `[`, parse the full class body up to matching `]`
                              (tracking nesting, []], escapes, embedded \p{...}, &&)
                              → emits a ClassNode AST
  3. ClassRenderer         — collapses nested Union to a single PCRE2 class-body
                              for Intersection nodes, applies:
                                  A ∩ B  →  [^[^<renderA>][^<renderB>]]
                              recursively until rendering is a single character class string
  4. Re-stitch              — non-class parts of the pattern are copied verbatim;
                              translated class strings are spliced back in
  5. return: pcre2Pattern (and a side-channel offset map if we choose to surface errors better)
```

### Single-file scope (preserve testability)

- `JavaRegexTranslator` is the orchestrator and the public entry point.
- `PropertyMap` is a static `Map<String,String>` plus `apply(String name) -> String` helper. It also encodes:
  - Block-prefix strip: `In<X>` → if `Block:<X>` known by PCRE2 → keep as `Block:<X>`; else keep as `<X>` (PCRE2 accepts script names without prefix). Best-effort with a fallback list.
  - Script-prefix strip: `Is<X>` → `<X>`.
  - Java-specific aliases: `javaWhitespace` → `[\p{Zs}\t\n\x0B\f\r]` (POSIX whitespace minus some). Use the JDK's `Character.isWhitespace()` definition documented here: https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Character.html#isWhitespace(int)
- `ClassBodyParser` is the only nontrivial component. Tokens it must handle:
  - Literals (with `\` escapes pass-through)
  - Backslash escapes (`\d`, `\w`, `\s`, `\D`, `\W`, `\S`, `\xNN`, `\x{...}`, `\u{...}`, `\p{...}`, `\P{...}`, `\\`)
  - Ranges (`a-z`, `\x00-\xff`)
  - Nested class (`[...]`)
  - Intersection operator (`&&`)
  - Negation (`^` only valid as first char of a class)
  - Posix class shorthand `[:alpha:]` (rare in JDK tests but accept it as named-class)

### De Morgan justification

PCRE2 (10.42+) allows class **negation** inside a class only via `[^...]` at the top level; we can simulate intersection by:

```
A && B  ==  ¬(¬A ∪ ¬B)
```

This becomes a class of the form `[^[^A][^B]]`. The outer class is negated; inside there are two negated nested classes. **But PCRE2 10.42 does NOT support nested classes either.** So we must combine: render `A`, `B` as char-set lists (sequence of code-point ranges), compute the negation manually (range complement over the Unicode space when `UTF` is on, ASCII space otherwise), compute the union, and emit a single flat class.

**Set algebra implementation:**
- Represent each class body after parse as a sorted disjoint `List<Range>` of code-point ranges
- Union, intersection, negation, subtraction all become standard interval-set operations
- Render the final `List<Range>` as a single PCRE2 `[r1r2r3…]` literal, plus the top-level `^` if needed
- Special leaves (`\d`, `\w`, `\s`, `\p{X}`) cannot be expanded to ranges without knowing Unicode tables; KEEP them as PCRE2 tokens when the class is pure union; for **intersection** with such a leaf, fall back to two strategies:
  - If pcre2 supports `PCRE2_ALT_EXTENDED_CLASS` at runtime (10.45+), use it (Approach C activation point)
  - Else, expand the property to a range list **for a known subset** of properties via a bundled, generated table (`\d` = `0-9`; `\w` = `[0-9A-Z_a-z]` in non-UCP, full Unicode in UCP; etc.); for unknown properties we cannot expand → emit a translator warning and skip the intersection optimization (record passes through, that pattern remains a failure)

This is the most complex piece. To keep scope manageable for the first delivery:

- **Phase 1** (this spec, must-have): Property rewrite + nested-union flattening (NO intersection support). Covers ~107 / 275 failures = 40% reduction.
- **Phase 2** (this spec, should-have): Intersection support via the De Morgan + range-set algebra for **literals and ranges only**; when a class contains `\p{}` / `\d` / `\w` / `\s` and is in an intersection, leave it untranslated (counts as known limitation). Adds ~20 / 275 → 47% reduction.
- **Phase 3** (this spec, nice-to-have): Property expansion table generator for `\d`/`\w`/`\s`/`\p{ASCII}` to enable intersection with those. Adds the remaining intersection coverage. Brings total to ~73% reduction.

## Data flow

```
User code:
  Pattern.compile("a[\\p{InGreek}&&[^Ͱ]]c", 0)
    ↓
Pattern.compile saves regex="a[\\p{InGreek}&&[^Ͱ]]c"  (returned by pattern())
    ↓
compiledRegex = JavaRegexTranslator.translate(regex, flags)
                = "a[\\p{Greek}&&[^\\x{0370}]]c"      (Phase 1+2: property OK, intersection skipped)
                = "a[\\x{0370}-\\x{03FF}&&[^\\x{0370}]]c" then collapsed by Phase 3 to:
                  "a[\\x{0371}-\\x{03FF}]c"
    ↓
Pcre2Code.create(compiledRegex, ...)
```

## Testing

### Unit tests
- `PropertyMapTest` — 30+ entries covering JDK property table
- `ClassBodyParserTest` — round-trip tests for nested classes, intersections, escapes, edge cases (empty class `[]`, single-char `[a]`, `[\]]`, `[^]`, `[a-]`, `[--]`, `[&&]`)
- `RangeSetTest` — union/intersection/complement on code-point ranges
- `JavaRegexTranslatorTest` — 50+ golden tests: each input pattern + expected translated PCRE2 pattern + the input that should match
  - Includes "do not translate" cases: `\d+`, `[a-z]`, plain text — must pass through unchanged
  - Includes "translation preserves pattern() result" tests

### Integration: regression via compat-test harness
- Re-run `:compat-test:compatReport` after each Phase commit.
- **Gate per phase**: total `pass` count strictly increases; `fail` count strictly decreases; no previously-passing record becomes failing.
- Phase 1 target: pass ≥ 660 (was 584; +12 percentage points)
- Phase 2 target: pass ≥ 700 (+16 pp)
- Phase 3 target: pass ≥ 770 (+22 pp, total compat ~90%)

### No-regression on existing modules
- `:regex:test` must still pass (it has its own JDK-API smoke tests).
- `:jna:test` (when enabled) must still pass.

## Failure modes

- **Translation throws** → wrap in `PatternSyntaxException(originalPattern, "translator: " + msg, 0)`.
- **Translated pattern PCRE2-rejected** → catch `Pcre2CompileException`, rethrow as `PatternSyntaxException` with **original** pattern string in the message; offset best-effort.
- **Pattern uses a feature we explicitly skipped (e.g., `\p{IsLC}` not in alias table)** → fall through to PCRE2 unchanged; user gets a PCRE2 error (current behavior, no regression).

## Backout

Behind a system property `pcre4j.regex.translate=false` the translator is bypassed. If a release-blocking bug is found, users can opt out.

## Done criteria

- [ ] `:compat-test:compatReport` shows pass rate ≥ 91% on the 3 `.txt` sources (target Phase 3) or ≥ 81% (Phase 2 fallback if Phase 3 deferred)
- [ ] `:regex:test` and `:jna:test` (where buildable) still pass
- [ ] No previously-passing case becomes failing in `raw.jsonl` diff
- [ ] System property `pcre4j.regex.translate=false` round-trips to previous behavior (verified by re-running compat harness with the flag set and getting back the original 275 failures)
- [ ] All new translator classes have unit-test coverage; integration coverage via the compat harness
- [ ] `Pattern.pattern()` returns the original pre-translation string (verified by test)

## Out-of-scope (deferred)

- pcre2 version detection + auto-enable of `PCRE2_ALT_EXTENDED_CLASS` (Approach C from brainstorming). May be added later as a perf/correctness improvement once the system upgrades libpcre2 to ≥10.45.
- Translator optimization (caching, single-pass scanner). Acceptable to be O(n²) for first cut; patterns are short.
- Surfacing translation in `Pattern.toString()` / debug output. Not needed for compat correctness.
