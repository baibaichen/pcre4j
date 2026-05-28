# pcre4j ↔ `java.util.regex` 兼容性实测验证 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a `:compat-test` Gradle subproject that runs JDK 21u `java/util/regex` tests + data files against `org.pcre4j.regex` (oracle = `java.util.regex`, SUT = pcre4j) and emits a categorized compat report.

**Architecture:** New Gradle subproject in the worktree, depends on `:regex` + `:ffm`. Two test drivers: (1) `.txt` data-driven via JUnit 5 parameterized tests producing `MatchProbe` pairs into `raw.jsonl`; (2) imported `.java` JDK tests adapted to JUnit 5 reflectively driven, assertion failures captured into `raw.jsonl`. A separate `compatReport` task classifies failures by pattern-string regex matching and renders `report.md`.

**Tech Stack:** Java 21 (preview FFM), Gradle 8 Kotlin DSL, JUnit 5 Jupiter, pcre4j FFM backend, openjdk/jdk21u test sources (GPLv2+CE).

**Spec:** `docs/superpowers/specs/2026-05-28-pcre4j-jur-compat-verification-design.md`

**Worktree:** `/root/SourceCode/pcre4j-compat-test` on branch `chang/compat-test`.

**Env prefix for every Gradle command:**
```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 \
PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH \
./gradlew <task> -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```

**Commit conventions:** all commits use `git commit -s` (DCO sign-off mandatory) and include trailer `Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>`. Types: `(chore)` for build setup, `(test)` for test infra/imports, `(docs)` for README/notice.

---

## File Structure (overview)

```
compat-test/
├── build.gradle.kts                    # M1
├── README.md                           # M5
├── LICENSE.NOTICE                      # M1
└── src/
    ├── main/java/org/pcre4j/compat/
    │   ├── MatchProbe.java             # M2
    │   ├── Hit.java                    # M2
    │   ├── Group.java                  # M2
    │   ├── Outcome.java                # M2
    │   ├── Probes.java                 # M2
    │   ├── ComparisonRecorder.java     # M2
    │   ├── TxtCaseRunner.java          # M3
    │   └── report/
    │       ├── RawRecord.java          # M5
    │       ├── Classifier.java         # M5
    │       └── ReportRenderer.java     # M5
    └── test/
        ├── java/org/pcre4j/compat/
        │   ├── MatchProbeTest.java     # M2
        │   ├── ProbesTest.java         # M2
        │   ├── ComparisonRecorderTest.java # M2
        │   ├── TxtCaseRunnerTest.java  # M3
        │   ├── TxtCompatTest.java      # M3
        │   ├── report/ClassifierTest.java  # M5
        │   ├── report/ReportRendererTest.java # M5
        │   └── imported/               # M4
        │       ├── README.md
        │       ├── RegExTestRunner.java
        │       ├── RegExTest.java
        │       ├── NamedGroupsTests.java
        │       ├── NamedGroupsTestsRunner.java
        │       ├── SplitWithDelimitersTest.java
        │       ├── POSIX_ASCII.java
        │       ├── POSIX_Unicode.java
        │       ├── ImmutableMatchResultTest.java
        │       └── NegativeArraySize.java
        └── resources/imported/
            ├── TestCases.txt
            ├── BMPTestCases.txt
            ├── SupplementaryTestCases.txt
            └── GraphemeTestCases.txt
```

---

## M1 — Subproject skeleton

### Task M1.1: Register `:compat-test` in settings

**Files:**
- Modify: `/root/SourceCode/pcre4j-compat-test/settings.gradle.kts`

- [ ] **Step 1: Add include line**

Edit settings.gradle.kts to append after the `include(":native:all")` line:

```kotlin
include(":compat-test")
```

- [ ] **Step 2: Verify Gradle recognizes it (will fail until build.gradle.kts exists, but settings should parse)**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew projects
```
Expected: lists `+--- Project ':compat-test'` (Gradle is tolerant of missing build.gradle.kts at `projects` step).

### Task M1.2: Create `compat-test/build.gradle.kts`

**Files:**
- Create: `/root/SourceCode/pcre4j-compat-test/compat-test/build.gradle.kts`

- [ ] **Step 1: Create the build script**

```kotlin
/*
 * Copyright (C) 2026 Oleksii PELYKH
 *
 * This file is a part of the PCRE4J. The PCRE4J is free software: you can redistribute it and/or modify it under the
 * terms of the GNU Lesser General Public License as published by the Free Software Foundation, either version 3 of the
 * License, or (at your option) any later version.
 *
 * This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied
 * warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU Lesser General Public License for more
 * details.
 *
 * You should have received a copy of the GNU Lesser General Public License along with this program. If not, see
 * <https://www.gnu.org/licenses/>.
 */
plugins {
    `java-library`
    checkstyle
    id("pcre4j-native-test")
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

val libs = the<org.gradle.accessors.dm.LibrariesForLibs>()

dependencies {
    testImplementation(platform(libs.junit.bom))
    testImplementation(libs.junit.jupiter)
    testRuntimeOnly(libs.junit.platform.launcher)

    testImplementation(project(":regex"))
    testImplementation(project(":ffm"))
    testImplementation(project(":jna"))
}

configurations.all {
    resolutionStrategy {
        failOnVersionConflict()
    }
}

tasks.withType<JavaCompile>().configureEach {
    options.encoding = "UTF-8"
    options.compilerArgs.add("--enable-preview")
}

tasks.test {
    useJUnitPlatform()
    jvmArgs("--enable-preview")
    // Surface pcre4j.test.backends from -Pcompat.backend
    val backend = (project.findProperty("compat.backend") as String?) ?: "ffm"
    systemProperty("pcre4j.test.backends", backend)
    // Forward heap-friendly defaults for the large RegExTest.java reflective run
    maxHeapSize = "2g"
}

checkstyle {
    toolVersion = libs.versions.checkstyle.get()
}

tasks.withType<Checkstyle>().configureEach {
    // Imported JDK test sources do not follow pcre4j checkstyle profile.
    exclude("org/pcre4j/compat/imported/**")
}
```

- [ ] **Step 2: Verify compile succeeds (empty sources still produce a valid task graph)**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:compileJava :compat-test:compileTestJava -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```
Expected: `BUILD SUCCESSFUL`. (No sources yet → both tasks NO-SOURCE.)

- [ ] **Step 3: Verify root build is not affected**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew build -x test -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```
Expected: `BUILD SUCCESSFUL`, `:compat-test:jar` runs (NO-SOURCE) but no failure. If `checkModuleDependencies` complains (because `:compat-test` not in `allowedProjectDependencies`), inspect output and add a note to skip — but per spec, the map silent-skips missing entries.

### Task M1.3: LICENSE.NOTICE for imported JDK sources

**Files:**
- Create: `/root/SourceCode/pcre4j-compat-test/compat-test/LICENSE.NOTICE`

- [ ] **Step 1: Write notice**

```
This subproject (:compat-test) contains source files copied from the OpenJDK
project (https://github.com/openjdk/jdk21u) under:

    GNU General Public License, version 2, with the Classpath Exception
    (GPLv2 + CE)

The imported files live under:
    src/test/java/org/pcre4j/compat/imported/
    src/test/resources/imported/

Their original headers are preserved verbatim and identify each file's
upstream source (`jdk-21+<build>` or commit SHA noted in
src/test/java/org/pcre4j/compat/imported/README.md).

The rest of :compat-test (everything outside the two `imported/` directories
listed above) is part of PCRE4J and is licensed under LGPL-3.0-or-later, the
same as the rest of the repository.

This subproject is NOT published to Maven Central nor distributed with
PCRE4J releases. It is an internal compatibility regression tool.
```

- [ ] **Step 2: Commit M1**

```
cd /root/SourceCode/pcre4j-compat-test
git add settings.gradle.kts compat-test/
git commit -s -m "(chore) compat-test: add :compat-test subproject skeleton

Adds a new Gradle subproject for empirical compat testing of pcre4j
against java.util.regex. Skeleton only — sources land in subsequent
commits.

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## M2 — Core probe & recorder (TDD)

### Task M2.1: `Outcome`, `Group`, `Hit`, `MatchProbe` records

**Files:**
- Create: `compat-test/src/main/java/org/pcre4j/compat/Outcome.java`
- Create: `compat-test/src/main/java/org/pcre4j/compat/Group.java`
- Create: `compat-test/src/main/java/org/pcre4j/compat/Hit.java`
- Create: `compat-test/src/main/java/org/pcre4j/compat/MatchProbe.java`
- Test: `compat-test/src/test/java/org/pcre4j/compat/MatchProbeTest.java`

- [ ] **Step 1: Write failing test**

```java
package org.pcre4j.compat;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class MatchProbeTest {
    @Test
    void okProbe_carriesAllFields() {
        var hit = new Hit(0, 3, "abc", List.of(new Group(null, 0, 3, "abc")));
        var p = new MatchProbe(new Outcome.Ok(), true, true, List.of(hit));
        assertTrue(p.compile() instanceof Outcome.Ok);
        assertEquals(Boolean.TRUE, p.matchesFull());
        assertEquals(1, p.findAll().size());
    }

    @Test
    void compileErrorProbe_hasNullMatchFields() {
        var p = new MatchProbe(new Outcome.SyntaxError("bad"), null, null, List.of());
        assertTrue(p.compile() instanceof Outcome.SyntaxError se && se.message().equals("bad"));
        assertNull(p.matchesFull());
    }
}
```

- [ ] **Step 2: Implement minimal types**

`Outcome.java`:
```java
package org.pcre4j.compat;

public sealed interface Outcome {
    record Ok() implements Outcome {}
    record SyntaxError(String message) implements Outcome {}
}
```

`Group.java`:
```java
package org.pcre4j.compat;

public record Group(String name, int start, int end, String text) {}
```

`Hit.java`:
```java
package org.pcre4j.compat;

import java.util.List;

public record Hit(int start, int end, String text, java.util.List<Group> groups) {
    public Hit { groups = List.copyOf(groups); }
}
```

`MatchProbe.java`:
```java
package org.pcre4j.compat;

import java.util.List;

public record MatchProbe(Outcome compile, Boolean matchesFull, Boolean lookingAt, List<Hit> findAll) {
    public MatchProbe { findAll = List.copyOf(findAll); }
}
```

- [ ] **Step 3: Run tests**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:test --tests "org.pcre4j.compat.MatchProbeTest" -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```
Expected: PASS, 2 tests.

- [ ] **Step 4: Commit**

```
git add compat-test/src/main/java/org/pcre4j/compat/ compat-test/src/test/java/org/pcre4j/compat/MatchProbeTest.java
git commit -s -m "(test) compat-test: add MatchProbe value types

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task M2.2: `Probes` — oracle + SUT probes

**Files:**
- Create: `compat-test/src/main/java/org/pcre4j/compat/Probes.java`
- Test: `compat-test/src/test/java/org/pcre4j/compat/ProbesTest.java`

- [ ] **Step 1: Write failing tests**

```java
package org.pcre4j.compat;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ProbesTest {
    @Test
    void oracle_matchesSimpleDigits() {
        var p = Probes.oracle("\\d+", "abc123def456", 0);
        assertTrue(p.compile() instanceof Outcome.Ok);
        assertEquals(Boolean.FALSE, p.matchesFull());
        assertEquals(Boolean.FALSE, p.lookingAt());
        assertEquals(2, p.findAll().size());
        assertEquals("123", p.findAll().get(0).text());
        assertEquals("456", p.findAll().get(1).text());
    }

    @Test
    void oracle_syntaxErrorRecorded() {
        var p = Probes.oracle("(unclosed", "x", 0);
        assertTrue(p.compile() instanceof Outcome.SyntaxError);
        assertNull(p.matchesFull());
    }

    @Test
    void sut_matchesSimpleDigits() {
        var p = Probes.sut("\\d+", "abc123def456", 0);
        assertTrue(p.compile() instanceof Outcome.Ok);
        assertEquals(2, p.findAll().size());
        assertEquals("123", p.findAll().get(0).text());
    }

    @Test
    void sut_syntaxErrorRecorded() {
        var p = Probes.sut("(unclosed", "x", 0);
        assertTrue(p.compile() instanceof Outcome.SyntaxError);
    }

    @Test
    void sut_runtimeErrorTreatedAsCompileError_orPropagated() {
        // pcre4j may throw IllegalStateException at compile or match time for some inputs;
        // any RuntimeException during probe must be caught and stored.
        var p = Probes.sut("\\p{InNoSuchBlock}", "x", 0);
        assertNotNull(p);
    }
}
```

- [ ] **Step 2: Implement `Probes`**

```java
package org.pcre4j.compat;

import java.util.ArrayList;
import java.util.List;

public final class Probes {

    private Probes() {}

    public static MatchProbe oracle(String pattern, String input, int flags) {
        java.util.regex.Pattern p;
        try {
            p = java.util.regex.Pattern.compile(pattern, flags);
        } catch (java.util.regex.PatternSyntaxException e) {
            return new MatchProbe(new Outcome.SyntaxError(e.getMessage()), null, null, List.of());
        } catch (RuntimeException e) {
            return new MatchProbe(new Outcome.SyntaxError("runtime:" + e.getClass().getSimpleName() + ":" + e.getMessage()),
                    null, null, List.of());
        }
        return runWithCompiled(input, () -> p.matcher(input));
    }

    public static MatchProbe sut(String pattern, String input, int flags) {
        org.pcre4j.regex.Pattern p;
        try {
            p = org.pcre4j.regex.Pattern.compile(pattern, flags);
        } catch (java.util.regex.PatternSyntaxException e) {
            return new MatchProbe(new Outcome.SyntaxError(e.getMessage()), null, null, List.of());
        } catch (RuntimeException e) {
            return new MatchProbe(new Outcome.SyntaxError("runtime:" + e.getClass().getSimpleName() + ":" + e.getMessage()),
                    null, null, List.of());
        }
        return runWithCompiled(input, () -> p.matcher(input));
    }

    private interface MatcherFactory {
        java.util.regex.MatchResult build();
    }

    @SuppressWarnings("unchecked")
    private static MatchProbe runWithCompiled(String input, java.util.function.Supplier<?> matcherSupplier) {
        Object m = matcherSupplier.get();
        try {
            Boolean matchesFull = (Boolean) m.getClass().getMethod("matches").invoke(m);
            // reset for lookingAt
            m.getClass().getMethod("reset").invoke(m);
            Boolean lookingAt = (Boolean) m.getClass().getMethod("lookingAt").invoke(m);
            m.getClass().getMethod("reset").invoke(m);
            List<Hit> hits = new ArrayList<>();
            while ((Boolean) m.getClass().getMethod("find").invoke(m)) {
                int start = (int) m.getClass().getMethod("start").invoke(m);
                int end = (int) m.getClass().getMethod("end").invoke(m);
                String text = (String) m.getClass().getMethod("group").invoke(m);
                int gc = (int) m.getClass().getMethod("groupCount").invoke(m);
                List<Group> groups = new ArrayList<>(gc);
                for (int i = 1; i <= gc; i++) {
                    Object gStart = m.getClass().getMethod("start", int.class).invoke(m, i);
                    Object gEnd = m.getClass().getMethod("end", int.class).invoke(m, i);
                    Object gText = m.getClass().getMethod("group", int.class).invoke(m, i);
                    groups.add(new Group(null, (int) gStart, (int) gEnd, (String) gText));
                }
                hits.add(new Hit(start, end, text, groups));
                if (end == start) {
                    // zero-width safety
                    if (end >= input.length()) break;
                    m.getClass().getMethod("region", int.class, int.class).invoke(m, end + 1, input.length());
                }
            }
            return new MatchProbe(new Outcome.Ok(), matchesFull, lookingAt, hits);
        } catch (RuntimeException e) {
            return new MatchProbe(new Outcome.SyntaxError("runtime-match:" + e.getClass().getSimpleName() + ":" + e.getMessage()),
                    null, null, List.of());
        } catch (Exception e) {
            Throwable c = e.getCause() != null ? e.getCause() : e;
            return new MatchProbe(new Outcome.SyntaxError("runtime-match:" + c.getClass().getSimpleName() + ":" + c.getMessage()),
                    null, null, List.of());
        }
    }
}
```

Note: using reflection lets both oracle and SUT share one path; both expose the same Matcher surface. If reflection becomes a perf problem in M3, refactor to two typed paths.

- [ ] **Step 3: Run tests**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:test --tests "org.pcre4j.compat.ProbesTest" -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```
Expected: 5 tests pass. If `sut` test fails because pcre4j needs `Pcre4j.setup(...)` first, add a static initializer block in `Probes` that calls `org.pcre4j.Pcre4j.setup(new org.pcre4j.ffm.Pcre2(...))` guarded by `pcre4j.test.backends` system property.

- [ ] **Step 4: If backend setup needed, add bootstrap**

If Step 3 fails with "no backend configured" or NPE on Pcre4j, prepend to `Probes` (verify exact API via `grep -n "Pcre4j.setup" lib/src/main/java`):

```java
static {
    String backend = System.getProperty("pcre4j.test.backends", "ffm");
    if (backend.equals("ffm")) {
        org.pcre4j.Pcre4j.setup(new org.pcre4j.ffm.Pcre2());
    } else {
        org.pcre4j.Pcre4j.setup(new org.pcre4j.jna.Pcre2());
    }
}
```

Re-run Step 3.

- [ ] **Step 5: Commit**

```
git add compat-test/src/main/java/org/pcre4j/compat/Probes.java compat-test/src/test/java/org/pcre4j/compat/ProbesTest.java
git commit -s -m "(test) compat-test: add Probes for oracle/SUT match capture

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task M2.3: `ComparisonRecorder` — JSONL writer

**Files:**
- Create: `compat-test/src/main/java/org/pcre4j/compat/ComparisonRecorder.java`
- Test: `compat-test/src/test/java/org/pcre4j/compat/ComparisonRecorderTest.java`

- [ ] **Step 1: Write failing test**

```java
package org.pcre4j.compat;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import java.nio.file.*;
import java.util.List;
import java.util.concurrent.*;
import static org.junit.jupiter.api.Assertions.*;

class ComparisonRecorderTest {
    @Test
    void writesOneLinePerRecord(@TempDir Path dir) throws Exception {
        Path out = dir.resolve("raw.jsonl");
        try (var rec = new ComparisonRecorder(out)) {
            rec.record("TestCases.txt", 1, "\\d+", "abc123", 0,
                    Probes.oracle("\\d+", "abc123", 0),
                    Probes.sut("\\d+", "abc123", 0));
        }
        List<String> lines = Files.readAllLines(out);
        assertEquals(1, lines.size());
        assertTrue(lines.get(0).startsWith("{"));
        assertTrue(lines.get(0).contains("\"source\":\"TestCases.txt\""));
        assertTrue(lines.get(0).contains("\"pattern\":\"\\\\d+\""));
    }

    @Test
    void isThreadSafe(@TempDir Path dir) throws Exception {
        Path out = dir.resolve("raw.jsonl");
        int n = 200;
        try (var rec = new ComparisonRecorder(out)) {
            var pool = Executors.newFixedThreadPool(8);
            var futures = new ArrayList<Future<?>>();
            for (int i = 0; i < n; i++) {
                final int idx = i;
                futures.add(pool.submit(() -> rec.record("t.txt", idx, "x", "y", 0,
                        Probes.oracle("x", "y", 0), Probes.sut("x", "y", 0))));
            }
            for (var f : futures) f.get();
            pool.shutdown();
        }
        assertEquals(n, Files.readAllLines(out).size());
    }
}
```

Add the import `import java.util.ArrayList;` to the test file.

- [ ] **Step 2: Implement recorder (hand-rolled JSON — no extra deps)**

```java
package org.pcre4j.compat;

import java.io.BufferedWriter;
import java.io.IOException;
import java.nio.file.*;
import java.util.List;

public final class ComparisonRecorder implements AutoCloseable {
    private final BufferedWriter writer;
    private final Object lock = new Object();

    public ComparisonRecorder(Path out) throws IOException {
        Files.createDirectories(out.getParent());
        this.writer = Files.newBufferedWriter(out, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
    }

    public void record(String source, int caseIndex, String pattern, String input, int flags,
                       MatchProbe oracle, MatchProbe sut) {
        String json = "{"
                + "\"source\":" + jsonString(source) + ","
                + "\"caseIndex\":" + caseIndex + ","
                + "\"flags\":" + flags + ","
                + "\"pattern\":" + jsonString(pattern) + ","
                + "\"input\":" + jsonString(input) + ","
                + "\"oracle\":" + probeToJson(oracle) + ","
                + "\"sut\":" + probeToJson(sut)
                + "}";
        synchronized (lock) {
            try {
                writer.write(json);
                writer.newLine();
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    }

    @Override public void close() throws IOException { writer.close(); }

    static String probeToJson(MatchProbe p) {
        var sb = new StringBuilder("{");
        if (p.compile() instanceof Outcome.Ok) {
            sb.append("\"compile\":\"ok\"");
        } else if (p.compile() instanceof Outcome.SyntaxError se) {
            sb.append("\"compile\":\"err\",\"err\":").append(jsonString(se.message()));
        }
        sb.append(",\"matches\":").append(p.matchesFull());
        sb.append(",\"lookingAt\":").append(p.lookingAt());
        sb.append(",\"findAll\":[");
        for (int i = 0; i < p.findAll().size(); i++) {
            if (i > 0) sb.append(",");
            sb.append(hitToJson(p.findAll().get(i)));
        }
        sb.append("]}");
        return sb.toString();
    }

    static String hitToJson(Hit h) {
        var sb = new StringBuilder("{");
        sb.append("\"start\":").append(h.start());
        sb.append(",\"end\":").append(h.end());
        sb.append(",\"text\":").append(jsonString(h.text()));
        sb.append(",\"groups\":[");
        for (int i = 0; i < h.groups().size(); i++) {
            if (i > 0) sb.append(",");
            var g = h.groups().get(i);
            sb.append("{\"start\":").append(g.start())
                    .append(",\"end\":").append(g.end())
                    .append(",\"text\":").append(jsonString(g.text())).append("}");
        }
        sb.append("]}");
        return sb.toString();
    }

    static String jsonString(String s) {
        if (s == null) return "null";
        var sb = new StringBuilder("\"");
        for (int i = 0; i < s.length(); i++) {
            char c = s.charAt(i);
            switch (c) {
                case '\\' -> sb.append("\\\\");
                case '"' -> sb.append("\\\"");
                case '\n' -> sb.append("\\n");
                case '\r' -> sb.append("\\r");
                case '\t' -> sb.append("\\t");
                case '\b' -> sb.append("\\b");
                case '\f' -> sb.append("\\f");
                default -> {
                    if (c < 0x20) sb.append(String.format("\\u%04x", (int) c));
                    else sb.append(c);
                }
            }
        }
        sb.append("\"");
        return sb.toString();
    }
}
```

- [ ] **Step 3: Run tests, expect PASS**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:test --tests "org.pcre4j.compat.ComparisonRecorderTest" -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```

- [ ] **Step 4: Commit**

```
git add compat-test/src/main/java/org/pcre4j/compat/ComparisonRecorder.java compat-test/src/test/java/org/pcre4j/compat/ComparisonRecorderTest.java
git commit -s -m "(test) compat-test: add ComparisonRecorder JSONL writer

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## M3 — `.txt` data-driven runner

### Task M3.1: Download the 4 `.txt` files (pin to commit SHA)

**Files:**
- Create: `compat-test/src/test/resources/imported/TestCases.txt`
- Create: `compat-test/src/test/resources/imported/BMPTestCases.txt`
- Create: `compat-test/src/test/resources/imported/SupplementaryTestCases.txt`
- Create: `compat-test/src/test/resources/imported/GraphemeTestCases.txt`
- Create: `compat-test/src/test/java/org/pcre4j/compat/imported/README.md`

- [ ] **Step 1: Resolve the jdk21u HEAD SHA at the moment of import**

```
SHA=$(curl -sS https://api.github.com/repos/openjdk/jdk21u/commits/master | grep -m1 '"sha"' | cut -d'"' -f4)
echo "$SHA"
```
Save this SHA — it becomes the citation in the imported README.

- [ ] **Step 2: Download each file pinned to that SHA**

```
mkdir -p compat-test/src/test/resources/imported
for f in TestCases.txt BMPTestCases.txt SupplementaryTestCases.txt GraphemeTestCases.txt; do
  curl -sS -o "compat-test/src/test/resources/imported/$f" \
    "https://raw.githubusercontent.com/openjdk/jdk21u/$SHA/test/jdk/java/util/regex/$f"
done
wc -l compat-test/src/test/resources/imported/*.txt
```
Expected: each file > 50 lines, no `404: Not Found` content. Inspect first 5 lines of each manually:

```
head -5 compat-test/src/test/resources/imported/TestCases.txt
```

- [ ] **Step 3: Write the imported README**

`compat-test/src/test/java/org/pcre4j/compat/imported/README.md`:

```markdown
# Imported OpenJDK test sources

All files in this directory and in `../../../../../resources/imported/` are
copied from https://github.com/openjdk/jdk21u/tree/<SHA>/test/jdk/java/util/regex
under GPLv2 + Classpath Exception.

**Upstream commit SHA:** `<SHA-from-Step-1>`
**Date imported:** 2026-05-28

## Files

| File | Modifications |
| --- | --- |
| `TestCases.txt` | none |
| `BMPTestCases.txt` | none |
| `SupplementaryTestCases.txt` | none |
| `GraphemeTestCases.txt` | none |
| `RegExTest.java` | import switched from `java.util.regex` to `org.pcre4j.regex` (see RegExTestRunner.java); methods reflecting on JDK private API marked `@Disabled` |
| `NamedGroupsTests.java` | same as RegExTest |
| `SplitWithDelimitersTest.java` | same |
| `POSIX_ASCII.java` / `POSIX_Unicode.java` | unmodified (helper truth tables) |
| `ImmutableMatchResultTest.java` | replaced `jdk.test.lib.RandomFactory` with `new Random(0xC0FFEEL)`; import switch |
| `NegativeArraySize.java` | import switch |

See `../../../../../../LICENSE.NOTICE` for the full license notice.
```

Fill in `<SHA-from-Step-1>` with the actual SHA, both in the body and in the title placeholder.

- [ ] **Step 4: Commit imports**

```
git add compat-test/src/test/resources/imported/ compat-test/src/test/java/org/pcre4j/compat/imported/README.md
git commit -s -m "(test) compat-test: import .txt data files from openjdk/jdk21u@<SHA>

GPLv2+CE source; see compat-test/LICENSE.NOTICE.

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task M3.2: `TxtCaseRunner` — parser

**Files:**
- Create: `compat-test/src/main/java/org/pcre4j/compat/TxtCaseRunner.java`
- Test: `compat-test/src/test/java/org/pcre4j/compat/TxtCaseRunnerTest.java`

The OpenJDK `.txt` format (per `RegExTest.processFile`):
- blank line or line starting with `//` = skip
- pattern line, then input line, then expectation line:
  - `true <captures...>` — match expected, captures are group strings (the special token `EMPTY` denotes "")
  - `false` — no match expected
- pattern may contain `\n`, `\t`, etc. — used literally as Java string would be after `Pattern.compile("\\n")`-style escapes (i.e., the file contains literal backslashes that the regex engine interprets)
- input lines may contain `\n` escape sequence that needs literal interpretation (see `RegExTest.processFile` for exact behavior)

Re-read the upstream `processFile` to confirm escape handling before implementation:

```
curl -sS "https://raw.githubusercontent.com/openjdk/jdk21u/$SHA/test/jdk/java/util/regex/RegExTest.java" | grep -n -A 80 "static void processFile"
```

- [ ] **Step 1: Write failing parser test**

```java
package org.pcre4j.compat;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class TxtCaseRunnerTest {
    @Test
    void parsesTriple() {
        String body = """
                // comment
                \\d+
                abc123
                true 123

                ^x$
                y
                false
                """;
        List<TxtCaseRunner.Case> cases = TxtCaseRunner.parse(body);
        assertEquals(2, cases.size());
        assertEquals("\\d+", cases.get(0).pattern());
        assertEquals("abc123", cases.get(0).input());
        assertTrue(cases.get(0).expectedMatch());
        assertEquals("y", cases.get(1).input());
        assertFalse(cases.get(1).expectedMatch());
    }
}
```

- [ ] **Step 2: Implement parser**

```java
package org.pcre4j.compat;

import java.util.ArrayList;
import java.util.List;

public final class TxtCaseRunner {

    public record Case(int index, String pattern, String input, boolean expectedMatch, List<String> expectedGroups) {}

    private TxtCaseRunner() {}

    public static List<Case> parse(String body) {
        List<Case> out = new ArrayList<>();
        String[] lines = body.split("\\R", -1);
        int idx = 0;
        int i = 0;
        while (i < lines.length) {
            String l = lines[i];
            if (l.isBlank() || l.startsWith("//")) { i++; continue; }
            // Need at least 3 lines: pattern / input / expectation
            if (i + 2 >= lines.length) break;
            String pattern = l;
            String input = lines[i + 1];
            String exp = lines[i + 2].trim();
            boolean match = exp.startsWith("true");
            List<String> groups = new ArrayList<>();
            if (match) {
                String tail = exp.substring("true".length()).trim();
                if (!tail.isEmpty()) {
                    for (String g : tail.split("\\s+")) {
                        groups.add(g.equals("EMPTY") ? "" : g);
                    }
                }
            }
            out.add(new Case(idx++, pattern, input, match, List.copyOf(groups)));
            i += 3;
        }
        return out;
    }
}
```

Note: This is the **minimal** parser to make tests green. Step 4 will reconcile any escape-handling differences against the actual `.txt` data.

- [ ] **Step 3: Run parser test, expect PASS**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:test --tests "org.pcre4j.compat.TxtCaseRunnerTest" -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```

- [ ] **Step 4: Validate against real data — load TestCases.txt, count cases, ensure parser doesn't throw**

Add this to `TxtCaseRunnerTest`:

```java
@Test
void parsesRealTestCasesTxt() throws Exception {
    String body = new String(getClass().getResourceAsStream("/imported/TestCases.txt").readAllBytes());
    var cases = TxtCaseRunner.parse(body);
    assertTrue(cases.size() > 50, "expected > 50 cases, got " + cases.size());
    // Reasonable sanity: at least one true and one false case
    assertTrue(cases.stream().anyMatch(TxtCaseRunner.Case::expectedMatch));
    assertTrue(cases.stream().anyMatch(c -> !c.expectedMatch()));
}
```

Run; if it fails, inspect the actual file structure (especially comment markers, whether `true`/`false` is on its own line) and refine the parser accordingly. Re-check that the line count matches the file's expected case count by running upstream RegExTest's `processFile` mental model.

- [ ] **Step 5: Commit**

```
git add compat-test/src/main/java/org/pcre4j/compat/TxtCaseRunner.java compat-test/src/test/java/org/pcre4j/compat/TxtCaseRunnerTest.java
git commit -s -m "(test) compat-test: add TxtCaseRunner parser for .txt fixtures

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task M3.3: `TxtCompatTest` — parameterized end-to-end

**Files:**
- Create: `compat-test/src/test/java/org/pcre4j/compat/TxtCompatTest.java`

- [ ] **Step 1: Write the parameterized test**

```java
package org.pcre4j.compat;

import org.junit.jupiter.api.*;
import org.junit.jupiter.params.*;
import org.junit.jupiter.params.provider.*;
import java.io.IOException;
import java.nio.file.*;
import java.util.*;
import java.util.stream.*;

class TxtCompatTest {

    private static ComparisonRecorder RECORDER;
    private static Path OUT;

    @BeforeAll
    static void open() throws IOException {
        OUT = Path.of("build/reports/compat/raw.jsonl");
        // Append mode: each parameterized class run writes; we recreate per @BeforeAll
        RECORDER = new ComparisonRecorder(OUT);
    }

    @AfterAll
    static void close() throws IOException { RECORDER.close(); }

    static Stream<Arguments> cases() throws IOException {
        String[] files = {"TestCases.txt", "BMPTestCases.txt", "SupplementaryTestCases.txt", "GraphemeTestCases.txt"};
        List<Arguments> args = new ArrayList<>();
        for (String name : files) {
            String body = new String(TxtCompatTest.class.getResourceAsStream("/imported/" + name).readAllBytes());
            for (TxtCaseRunner.Case c : TxtCaseRunner.parse(body)) {
                args.add(Arguments.of(name, c));
            }
        }
        return args.stream();
    }

    @ParameterizedTest(name = "{0}#{1}")
    @MethodSource("cases")
    void compare(String source, TxtCaseRunner.Case c) {
        MatchProbe oracle = Probes.oracle(c.pattern(), c.input(), 0);
        MatchProbe sut = Probes.sut(c.pattern(), c.input(), 0);
        RECORDER.record(source, c.index(), c.pattern(), c.input(), 0, oracle, sut);
        // Intentionally NOT asserting equality — this harness records, the report classifies.
        // The test exists so JUnit drives the parameterized run.
    }
}
```

- [ ] **Step 2: Run end-to-end**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:test --tests "org.pcre4j.compat.TxtCompatTest" -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```
Expected: BUILD SUCCESSFUL (no assertions inside `compare`). `compat-test/build/reports/compat/raw.jsonl` exists and has > 1000 lines.

- [ ] **Step 3: Sanity-eyeball first 20 records**

```
head -20 compat-test/build/reports/compat/raw.jsonl | jq -r '"\(.source) #\(.caseIndex): pattern=\(.pattern) oracle=\(.oracle.compile)/\(.oracle.matches) sut=\(.sut.compile)/\(.sut.matches)"'
```

Expected: diffs visible on patterns predicted by spec §4.2 (e.g., `\p{InGreek}`, `\p{javaWhitespace}`, `[a-z&&[def]]`). If `jq` not installed, use `head -20 ... | sed -n '...' ` instead.

- [ ] **Step 4: Commit**

```
git add compat-test/src/test/java/org/pcre4j/compat/TxtCompatTest.java
git commit -s -m "(test) compat-test: add parameterized .txt compat runner

Each .txt case writes one JSONL record to build/reports/compat/raw.jsonl.
No assertions — recording-only harness; report task classifies.

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## M4 — Import & adapt 7 `.java` JDK tests

Source URL template (uses SHA pinned in M3.1):
```
https://raw.githubusercontent.com/openjdk/jdk21u/$SHA/test/jdk/java/util/regex/<file>
```

### Task M4.1: Copy 5 unmodified-style files

**Files:**
- Create: `compat-test/src/test/java/org/pcre4j/compat/imported/POSIX_ASCII.java`
- Create: `compat-test/src/test/java/org/pcre4j/compat/imported/POSIX_Unicode.java`
- Create: `compat-test/src/test/java/org/pcre4j/compat/imported/NegativeArraySize.java`
- Create: `compat-test/src/test/java/org/pcre4j/compat/imported/SplitWithDelimitersTest.java`
- Create: `compat-test/src/test/java/org/pcre4j/compat/imported/ImmutableMatchResultTest.java`

- [ ] **Step 1: Download all five**

```
for f in POSIX_ASCII.java POSIX_Unicode.java NegativeArraySize.java SplitWithDelimitersTest.java ImmutableMatchResultTest.java; do
  curl -sS -o "compat-test/src/test/java/org/pcre4j/compat/imported/$f" \
    "https://raw.githubusercontent.com/openjdk/jdk21u/$SHA/test/jdk/java/util/regex/$f"
done
```

- [ ] **Step 2: Patch package + import**

For each `.java` file:
1. Replace `package XX;` (if any) with `package org.pcre4j.compat.imported;`. If file has no package declaration (default-package test), prepend `package org.pcre4j.compat.imported;` at top after the license header.
2. Replace every `import java.util.regex.Pattern;`, `Matcher;`, `MatchResult;`, `PatternSyntaxException;` with the `org.pcre4j.regex` counterpart **only for classes pcre4j re-exports**. Leave `PatternSyntaxException` as `java.util.regex.PatternSyntaxException` (pcre4j re-throws the JDK class).

Run search for each file's imports:
```
grep -n "^import" compat-test/src/test/java/org/pcre4j/compat/imported/*.java
```

Edit each file individually with the `edit` tool.

3. For `ImmutableMatchResultTest.java`: replace `import jdk.test.lib.RandomFactory;` and any `RandomFactory.getRandom()` call with `new java.util.Random(0xC0FFEEL)`. If the file uses `@org.testng.annotations.Test` (TestNG), convert each annotation to `@org.junit.jupiter.api.Test` and replace `Assert.assertEquals(actual, expected)` with `org.junit.jupiter.api.Assertions.assertEquals(expected, actual)` (note argument order swap).

- [ ] **Step 3: Compile-only check**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:compileTestJava -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```
Fix any compile error one-by-one. If a method uses `java.util.regex.Pattern.class.getDeclaredField/Method` or accesses `sun.*` / `jdk.internal.*`, surround the offending block in a try/catch and replace the body with `org.junit.jupiter.api.Assumptions.abort("requires JDK internals: " + e.getMessage());` — or, if the entire test method is internals-only, replace the body with that abort call.

- [ ] **Step 4: Run them**

```
./gradlew :compat-test:test --tests "org.pcre4j.compat.imported.POSIX_*" --tests "org.pcre4j.compat.imported.NegativeArraySize" --tests "org.pcre4j.compat.imported.SplitWithDelimitersTest" --tests "org.pcre4j.compat.imported.ImmutableMatchResultTest" -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```

Expected: some tests fail (the whole point of the harness). The `:compat-test:test` task should NOT be considered failing — we want Gradle to record the results. To allow the build to proceed despite test failures, append `--continue` to the gradlew invocation **and** add `ignoreFailures = true` to the `tasks.test {}` block in `compat-test/build.gradle.kts`:

```kotlin
tasks.test {
    useJUnitPlatform()
    jvmArgs("--enable-preview")
    ignoreFailures = true
    // ...existing code...
}
```

After change, re-run.

- [ ] **Step 5: Commit imports + adapter changes + ignoreFailures**

```
git add compat-test/src/test/java/org/pcre4j/compat/imported/ compat-test/build.gradle.kts
git commit -s -m "(test) compat-test: import 5 small jdk21u regex tests

Switches imports to org.pcre4j.regex; jdk-internal-dependent test
bodies replaced with Assumptions.abort. Test task set to
ignoreFailures so the harness records failures instead of halting.

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task M4.2: Import `NamedGroupsTests.java` with legacy-style runner

`NamedGroupsTests.java` is in legacy static-`main` style. We need a thin JUnit 5 runner that invokes its `main`.

**Files:**
- Create: `compat-test/src/test/java/org/pcre4j/compat/imported/NamedGroupsTests.java`
- Create: `compat-test/src/test/java/org/pcre4j/compat/imported/NamedGroupsTestsRunner.java`

- [ ] **Step 1: Download & patch imports per M4.1 Step 2**

```
curl -sS -o "compat-test/src/test/java/org/pcre4j/compat/imported/NamedGroupsTests.java" \
  "https://raw.githubusercontent.com/openjdk/jdk21u/$SHA/test/jdk/java/util/regex/NamedGroupsTests.java"
```
Apply package + import-switch edits.

- [ ] **Step 2: Write the runner**

```java
package org.pcre4j.compat.imported;

import org.junit.jupiter.api.Test;

class NamedGroupsTestsRunner {
    @Test
    void runMain() throws Throwable {
        NamedGroupsTests.main(new String[0]);
    }
}
```

- [ ] **Step 3: Compile + run**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:test --tests "org.pcre4j.compat.imported.NamedGroupsTestsRunner" -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```

If `main` throws on failure (most JDK legacy tests do), the JUnit run will record one fat failure. That's acceptable for M4 — M5's report renderer will surface `NamedGroupsTests` as `legacy-runner-failed`.

- [ ] **Step 4: Commit**

```
git add compat-test/src/test/java/org/pcre4j/compat/imported/NamedGroupsTests.java compat-test/src/test/java/org/pcre4j/compat/imported/NamedGroupsTestsRunner.java
git commit -s -m "(test) compat-test: import NamedGroupsTests with legacy runner

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task M4.3: Import `RegExTest.java` with reflective per-method runner

**Files:**
- Create: `compat-test/src/test/java/org/pcre4j/compat/imported/RegExTest.java`
- Create: `compat-test/src/test/java/org/pcre4j/compat/imported/RegExTestRunner.java`

- [ ] **Step 1: Download & patch imports**

```
curl -sS -o "compat-test/src/test/java/org/pcre4j/compat/imported/RegExTest.java" \
  "https://raw.githubusercontent.com/openjdk/jdk21u/$SHA/test/jdk/java/util/regex/RegExTest.java"
```

This file is ~190 KB. Apply: prepend package, switch `import java.util.regex.{Pattern,Matcher,MatchResult}` to `org.pcre4j.regex.{Pattern,Matcher,MatchResult}`. **Keep** `import java.util.regex.PatternSyntaxException`.

- [ ] **Step 2: Identify methods to disable**

```
grep -nE "(getDeclaredField|getDeclaredMethod|getDeclaredConstructor|setAccessible|ObjectOutputStream|Serializable)" compat-test/src/test/java/org/pcre4j/compat/imported/RegExTest.java | head -50
```

For each enclosing `static void <name>Test()` method:
- `serializeTest` — entire body → `failCount++; System.out.println("[skipped: pcre4j does not implement Serializable]"); report("serializeTest");` (or similar minimal change that won't break the file).

Simpler approach: do NOT modify RegExTest.java's source at all. Instead, in `RegExTestRunner` (Step 3) maintain a `SKIP` set of method names and skip them, recording each skip in the report.

- [ ] **Step 3: Write the runner**

```java
package org.pcre4j.compat.imported;

import org.junit.jupiter.api.DynamicTest;
import org.junit.jupiter.api.TestFactory;
import org.junit.jupiter.api.Test;

import java.io.ByteArrayOutputStream;
import java.io.PrintStream;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.util.*;
import java.util.stream.Stream;

class RegExTestRunner {

    private static final Set<String> SKIP = Set.of(
            "serializeTest"
    );

    @TestFactory
    Stream<DynamicTest> allRegExTests() {
        List<Method> methods = new ArrayList<>();
        for (Method m : RegExTest.class.getDeclaredMethods()) {
            if (!Modifier.isStatic(m.getModifiers())) continue;
            if (m.getParameterCount() != 0) continue;
            if (m.getReturnType() != void.class) continue;
            if (!m.getName().endsWith("Test")) continue;
            methods.add(m);
        }
        methods.sort(Comparator.comparing(Method::getName));
        return methods.stream().map(m -> DynamicTest.dynamicTest(m.getName(), () -> invoke(m)));
    }

    private void invoke(Method m) throws Exception {
        if (SKIP.contains(m.getName())) {
            org.junit.jupiter.api.Assumptions.abort("pcre4j-skip: " + m.getName());
        }
        m.setAccessible(true);
        // Capture System.out — many RegExTest methods println on internal failure even if report() doesn't throw
        PrintStream origOut = System.out;
        PrintStream origErr = System.err;
        ByteArrayOutputStream buf = new ByteArrayOutputStream();
        PrintStream tee = new PrintStream(buf, true);
        System.setOut(tee);
        System.setErr(tee);
        try {
            m.invoke(null);
        } catch (Throwable t) {
            Throwable c = t.getCause() != null ? t.getCause() : t;
            String captured = buf.toString();
            throw new AssertionError(m.getName() + " failed: " + c + "\nCaptured output:\n" + captured, c);
        } finally {
            System.setOut(origOut);
            System.setErr(origErr);
        }
    }
}
```

Note: `RegExTest.report(String)` prints `OKAY` or `FAILED` and throws `RuntimeException` only when the **whole class** ends with failures, but per-method `report()` may or may not throw. Our runner treats any exception or any `AssertionError` propagated by `report` as a failure for that DynamicTest; a method that silently increments `failCount` without throwing will appear as PASS in our run. That's a known limitation — the M5 report should note this.

- [ ] **Step 4: Run subset first to validate the runner**

```
./gradlew :compat-test:test --tests "org.pcre4j.compat.imported.RegExTestRunner" -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```

Verify the dynamic-test report names match `RegExTest`'s methods (e.g., `[1] hitEndTest`, `[2] regionTest`, ...). Expect many failures — that's the data we want.

- [ ] **Step 5: Add to imported README's modifications table**

Edit `compat-test/src/test/java/org/pcre4j/compat/imported/README.md` to list `RegExTest.java` modifications: "import switched; runner: RegExTestRunner.java skips `serializeTest`".

- [ ] **Step 6: Commit**

```
git add compat-test/src/test/java/org/pcre4j/compat/imported/RegExTest.java compat-test/src/test/java/org/pcre4j/compat/imported/RegExTestRunner.java compat-test/src/test/java/org/pcre4j/compat/imported/README.md
git commit -s -m "(test) compat-test: import RegExTest.java with reflective runner

Reflective JUnit 5 @TestFactory invokes each static xxxTest() method
of the imported RegExTest.java as a DynamicTest. Methods that
require JDK private API are listed in RegExTestRunner.SKIP.

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task M4.4: Full M4 smoke run

- [ ] **Step 1: Run the complete compat-test suite**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:test -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```

Verify: `build/reports/tests/test/index.html` opens to a tree of all tests; `compat-test/build/reports/compat/raw.jsonl` has expected line count. If any `Exception in thread "main"` or hang occurs, investigate before M5.

---

## M5 — Report renderer

### Task M5.1: `RawRecord` reader

**Files:**
- Create: `compat-test/src/main/java/org/pcre4j/compat/report/RawRecord.java`

- [ ] **Step 1: Minimal record + JSON line parser (no test dep)**

```java
package org.pcre4j.compat.report;

import java.util.*;
import java.util.regex.*;

public record RawRecord(
        String source, int caseIndex, String pattern, String input,
        String oracleCompile, Boolean oracleMatches,
        String sutCompile, String sutErr, Boolean sutMatches
) {

    // Lightweight extractor for the fields we need to classify.
    private static String extract(String json, String key) {
        Pattern p = Pattern.compile("\"" + Pattern.quote(key) + "\":\"((?:\\\\.|[^\"\\\\])*)\"");
        Matcher m = p.matcher(json);
        return m.find() ? unescape(m.group(1)) : null;
    }

    private static Boolean extractBool(String json, String key) {
        Pattern p = Pattern.compile("\"" + Pattern.quote(key) + "\":(true|false|null)");
        Matcher m = p.matcher(json);
        if (!m.find()) return null;
        String v = m.group(1);
        return v.equals("null") ? null : Boolean.valueOf(v);
    }

    private static String unescape(String s) {
        return s.replace("\\\"", "\"").replace("\\\\", "\\").replace("\\n", "\n").replace("\\t", "\t");
    }

    public static RawRecord parse(String json) {
        // Find nested "oracle":{...} and "sut":{...}
        String oracle = subObject(json, "oracle");
        String sut = subObject(json, "sut");
        return new RawRecord(
                extract(json, "source"),
                Integer.parseInt(json.replaceAll(".*\"caseIndex\":(-?\\d+).*", "$1")),
                extract(json, "pattern"),
                extract(json, "input"),
                extract(oracle, "compile"),
                extractBool(oracle, "matches"),
                extract(sut, "compile"),
                extract(sut, "err"),
                extractBool(sut, "matches")
        );
    }

    private static String subObject(String json, String key) {
        int i = json.indexOf("\"" + key + "\":{");
        if (i < 0) return "{}";
        int depth = 0;
        int start = json.indexOf('{', i);
        for (int j = start; j < json.length(); j++) {
            char c = json.charAt(j);
            if (c == '{') depth++;
            else if (c == '}') { depth--; if (depth == 0) return json.substring(start, j + 1); }
        }
        return "{}";
    }
}
```

(Pragmatic JSON-extract via regex — works because we control the writer and the values are escaped. If this proves brittle in real data, swap in `jakarta.json` or `Jackson`.)

### Task M5.2: `Classifier` — pattern-string → root cause

**Files:**
- Create: `compat-test/src/main/java/org/pcre4j/compat/report/Classifier.java`
- Test: `compat-test/src/test/java/org/pcre4j/compat/report/ClassifierTest.java`

- [ ] **Step 1: Failing test**

```java
package org.pcre4j.compat.report;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ClassifierTest {
    @Test
    void blockProperty_classifiedAsInPrefix() {
        assertEquals("block-property \\p{InXxx}", Classifier.classify("a\\p{InGreek}b"));
    }
    @Test
    void isPrefix_classifiedAsIsPrefix() {
        assertEquals("script-property \\p{IsXxx}", Classifier.classify("\\p{IsLatin}"));
    }
    @Test
    void javaPrefix_classifiedAsJavaProperty() {
        assertEquals("\\p{javaXxx}", Classifier.classify("\\p{javaWhitespace}"));
    }
    @Test
    void classIntersection_classified() {
        assertEquals("character-class intersection [...&&[...]]", Classifier.classify("[a-z&&[def]]"));
    }
    @Test
    void unicodeInlineFlag_classified() {
        assertEquals("(?U) inline UNICODE_CHARACTER_CLASS", Classifier.classify("(?U)\\w+"));
    }
    @Test
    void unmatched_returnsUnclassified() {
        assertEquals("unclassified", Classifier.classify("\\d+"));
    }
}
```

- [ ] **Step 2: Implement classifier**

```java
package org.pcre4j.compat.report;

import java.util.*;
import java.util.regex.Pattern;

public final class Classifier {

    private record Rule(Pattern p, String label) {}

    private static final List<Rule> RULES = List.of(
            new Rule(Pattern.compile("\\\\p\\{In\\w+\\}"), "block-property \\p{InXxx}"),
            new Rule(Pattern.compile("\\\\p\\{Is\\w+\\}"), "script-property \\p{IsXxx}"),
            new Rule(Pattern.compile("\\\\p\\{java\\w+\\}"), "\\p{javaXxx}"),
            new Rule(Pattern.compile("\\[[^\\]]*&&\\[[^\\]]*\\]\\]"), "character-class intersection [...&&[...]]"),
            new Rule(Pattern.compile("\\(\\?U\\)"), "(?U) inline UNICODE_CHARACTER_CLASS"),
            new Rule(Pattern.compile("\\\\R"), "\\R linebreak"),
            new Rule(Pattern.compile("\\\\X"), "\\X grapheme cluster"),
            new Rule(Pattern.compile("\\\\h|\\\\H|\\\\v|\\\\V"), "\\h \\H \\v \\V"),
            new Rule(Pattern.compile("\\(\\?<\\w+>"), "named group syntax"),
            new Rule(Pattern.compile("\\\\b\\{\\w+\\}"), "\\b{...} word boundary type")
    );

    private Classifier() {}

    public static String classify(String pattern) {
        if (pattern == null) return "unclassified";
        for (Rule r : RULES) {
            if (r.p.matcher(pattern).find()) return r.label;
        }
        return "unclassified";
    }
}
```

- [ ] **Step 3: Run tests, expect PASS**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:test --tests "org.pcre4j.compat.report.ClassifierTest" -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```

- [ ] **Step 4: Commit**

```
git add compat-test/src/main/java/org/pcre4j/compat/report/ compat-test/src/test/java/org/pcre4j/compat/report/ClassifierTest.java
git commit -s -m "(test) compat-test: add Classifier + RawRecord for report

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task M5.3: `ReportRenderer` + `compatReport` task

**Files:**
- Create: `compat-test/src/main/java/org/pcre4j/compat/report/ReportRenderer.java`
- Test: `compat-test/src/test/java/org/pcre4j/compat/report/ReportRendererTest.java`
- Modify: `compat-test/build.gradle.kts`

- [ ] **Step 1: Failing test**

```java
package org.pcre4j.compat.report;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import java.nio.file.*;
import static org.junit.jupiter.api.Assertions.*;

class ReportRendererTest {
    @Test
    void rendersSummaryAndFailures(@TempDir Path dir) throws Exception {
        Path raw = dir.resolve("raw.jsonl");
        Files.writeString(raw, String.join("\n",
                "{\"source\":\"TestCases.txt\",\"caseIndex\":0,\"flags\":0,\"pattern\":\"\\\\d+\",\"input\":\"a1\",\"oracle\":{\"compile\":\"ok\",\"matches\":false,\"lookingAt\":false,\"findAll\":[{\"start\":1,\"end\":2,\"text\":\"1\",\"groups\":[]}]},\"sut\":{\"compile\":\"ok\",\"matches\":false,\"lookingAt\":false,\"findAll\":[{\"start\":1,\"end\":2,\"text\":\"1\",\"groups\":[]}]}}",
                "{\"source\":\"TestCases.txt\",\"caseIndex\":1,\"flags\":0,\"pattern\":\"\\\\p{InGreek}\",\"input\":\"a\",\"oracle\":{\"compile\":\"ok\",\"matches\":false,\"lookingAt\":false,\"findAll\":[]},\"sut\":{\"compile\":\"err\",\"err\":\"unknown property\",\"matches\":null,\"lookingAt\":null,\"findAll\":[]}}"
        ));
        Path report = dir.resolve("report.md");
        ReportRenderer.render(raw, report);
        String body = Files.readString(report);
        assertTrue(body.contains("Total"));
        assertTrue(body.contains("block-property"));
        assertTrue(body.contains("sut-compile-error"));
    }
}
```

- [ ] **Step 2: Implement renderer**

```java
package org.pcre4j.compat.report;

import java.io.IOException;
import java.nio.file.*;
import java.util.*;
import java.util.stream.Collectors;

public final class ReportRenderer {

    private ReportRenderer() {}

    public static void render(Path rawJsonl, Path outMd) throws IOException {
        List<RawRecord> records = new ArrayList<>();
        for (String line : Files.readAllLines(rawJsonl)) {
            if (line.isBlank()) continue;
            try { records.add(RawRecord.parse(line)); } catch (RuntimeException ignored) {}
        }

        // Per-source summary
        Map<String, int[]> summary = new TreeMap<>(); // [total,pass,fail,sutCompileErr,sutRuntimeErr,behaviorDiff]
        Map<String, Map<String, Integer>> failuresByCause = new TreeMap<>();

        for (RawRecord r : records) {
            int[] s = summary.computeIfAbsent(r.source(), k -> new int[6]);
            s[0]++;
            Verdict v = classify(r);
            switch (v) {
                case PASS -> s[1]++;
                case SUT_COMPILE_ERROR -> { s[2]++; s[3]++; bump(failuresByCause, r); }
                case SUT_RUNTIME_ERROR -> { s[2]++; s[4]++; bump(failuresByCause, r); }
                case BEHAVIOR_DIFF -> { s[2]++; s[5]++; bump(failuresByCause, r); }
                case SUT_ACCEPTS_REJECTED -> { s[2]++; bump(failuresByCause, r); }
                case BOTH_REJECTED -> s[1]++;
            }
        }

        StringBuilder out = new StringBuilder();
        out.append("# pcre4j compat report vs java.util.regex\n\n");
        out.append("## Summary\n\n");
        out.append("| Source | Total | Pass | Fail | sut-compile-error | sut-runtime-error | behavior-diff |\n");
        out.append("| --- | ---: | ---: | ---: | ---: | ---: | ---: |\n");
        for (var e : summary.entrySet()) {
            int[] s = e.getValue();
            out.append("| ").append(e.getKey()).append(" | ").append(s[0]).append(" | ").append(s[1])
                    .append(" | ").append(s[2]).append(" | ").append(s[3]).append(" | ")
                    .append(s[4]).append(" | ").append(s[5]).append(" |\n");
        }

        out.append("\n## Failures by root cause\n\n");
        out.append("| Cause | Count | Sample pattern |\n| --- | ---: | --- |\n");
        // Aggregate across sources
        Map<String, Integer> totalByCause = new HashMap<>();
        Map<String, String> sampleByCause = new HashMap<>();
        for (RawRecord r : records) {
            if (classify(r) == Verdict.PASS || classify(r) == Verdict.BOTH_REJECTED) continue;
            String cause = Classifier.classify(r.pattern());
            totalByCause.merge(cause, 1, Integer::sum);
            sampleByCause.putIfAbsent(cause, r.pattern());
        }
        totalByCause.entrySet().stream()
                .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
                .forEach(e -> out.append("| ").append(e.getKey()).append(" | ").append(e.getValue())
                        .append(" | `").append(sampleByCause.get(e.getKey()).replace("|", "\\|")).append("` |\n"));

        Files.createDirectories(outMd.getParent());
        Files.writeString(outMd, out.toString());
    }

    private enum Verdict { PASS, SUT_COMPILE_ERROR, SUT_RUNTIME_ERROR, SUT_ACCEPTS_REJECTED, BEHAVIOR_DIFF, BOTH_REJECTED }

    private static Verdict classify(RawRecord r) {
        boolean oOk = "ok".equals(r.oracleCompile());
        boolean sOk = "ok".equals(r.sutCompile());
        if (!oOk && !sOk) return Verdict.BOTH_REJECTED;
        if (oOk && !sOk) {
            String err = r.sutErr() == null ? "" : r.sutErr();
            if (err.startsWith("runtime") || err.startsWith("runtime-match")) return Verdict.SUT_RUNTIME_ERROR;
            return Verdict.SUT_COMPILE_ERROR;
        }
        if (!oOk && sOk) return Verdict.SUT_ACCEPTS_REJECTED;
        if (!Objects.equals(r.oracleMatches(), r.sutMatches())) return Verdict.BEHAVIOR_DIFF;
        return Verdict.PASS;
    }

    private static void bump(Map<String, Map<String, Integer>> by, RawRecord r) {
        by.computeIfAbsent(r.source(), k -> new TreeMap<>())
                .merge(Classifier.classify(r.pattern()), 1, Integer::sum);
    }
}
```

Note: this M5 renderer compares only `matches()` for `BEHAVIOR_DIFF`. M5.5 (optional polish) can extend to compare `findAll` hits. Keep this scope minimal for M5.

- [ ] **Step 3: Run test**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:test --tests "org.pcre4j.compat.report.ReportRendererTest" -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```

- [ ] **Step 4: Add Gradle task**

Append to `compat-test/build.gradle.kts`:

```kotlin
tasks.register("compatReport") {
    group = "verification"
    description = "Render build/reports/compat/report.md from raw.jsonl"
    dependsOn("test")
    doLast {
        val raw = layout.buildDirectory.file("reports/compat/raw.jsonl").get().asFile.toPath()
        val out = layout.buildDirectory.file("reports/compat/report.md").get().asFile.toPath()
        // Invoke ReportRenderer reflectively from the test classpath
        val cl = java.net.URLClassLoader(
            sourceSets["main"].runtimeClasspath.files.map { it.toURI().toURL() }.toTypedArray(),
            ClassLoader.getSystemClassLoader()
        )
        val cls = cl.loadClass("org.pcre4j.compat.report.ReportRenderer")
        cls.getMethod("render", java.nio.file.Path::class.java, java.nio.file.Path::class.java).invoke(null, raw, out)
        println("Wrote $out")
    }
}
```

- [ ] **Step 5: Run end-to-end**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:compatReport -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
cat compat-test/build/reports/compat/report.md | head -60
```

Expected: summary table with non-zero numbers for `TestCases.txt`, `BMPTestCases.txt`, ...; failures-by-cause table with at least 3 cause categories.

- [ ] **Step 6: Commit**

```
git add compat-test/src/main/java/org/pcre4j/compat/report/ReportRenderer.java compat-test/src/test/java/org/pcre4j/compat/report/ReportRendererTest.java compat-test/build.gradle.kts
git commit -s -m "(test) compat-test: add ReportRenderer + :compatReport task

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task M5.4: README + final wiring

**Files:**
- Create: `compat-test/README.md`
- Modify: `README.md` of root (add 1 line)

- [ ] **Step 1: Write subproject README**

```markdown
# compat-test — pcre4j ↔ java.util.regex compatibility harness

> **Internal regression tool. NOT published.** See `LICENSE.NOTICE` for imported-source provenance.

## What

Runs OpenJDK 21u's own `java/util/regex` tests and `.txt` data files against `org.pcre4j.regex`, recording oracle-vs-SUT discrepancies into `build/reports/compat/raw.jsonl` and rendering a categorized summary into `build/reports/compat/report.md`.

Source of truth for the design: `docs/superpowers/specs/2026-05-28-pcre4j-jur-compat-verification-design.md`.

## Running

```bash
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 \
PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH \
./gradlew :compat-test:compatReport \
  -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```

To switch backend to JNA:

```bash
./gradlew :compat-test:compatReport -Pcompat.backend=jna ...
```

## Outputs

| Path | Purpose |
| --- | --- |
| `build/reports/compat/raw.jsonl` | One JSON line per probe (oracle + SUT MatchProbe). The fact record. |
| `build/reports/compat/report.md` | Categorized summary. Re-generable from raw.jsonl by re-running `:compatReport`. |
| `build/reports/tests/test/index.html` | Standard Gradle test report (Pass/Fail per imported JUnit test). |

## Imported sources

See `src/test/java/org/pcre4j/compat/imported/README.md` for upstream SHA + per-file modifications.

## Limitations

- Reflective driver for `RegExTest.java` only detects per-method failure when `report()` throws — methods that silently increment `failCount` without throwing may appear as PASS. The class-level final `report()` will still surface aggregate failure.
- `PatternStreamTest.java` (TestNG + JDK test-lib deps) is **not** imported; would require porting to JUnit 5.
- The `whitebox/` directory is **not** imported (tests JDK Pattern internal IR — not engine behavior).
```

- [ ] **Step 2: Optionally add one-liner to root README**

Skip unless user asks; per spec the harness is internal and not advertised.

- [ ] **Step 3: Commit**

```
git add compat-test/README.md
git commit -s -m "(docs) compat-test: add subproject README

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

### Task M5.5: Final smoke run & report inspection

- [ ] **Step 1: Clean run end-to-end**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew :compat-test:clean :compat-test:compatReport -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```

Expected: BUILD SUCCESSFUL. `report.md` exists. Open and verify:
- Summary table has rows for all 4 `.txt` files
- Failures-by-cause table has at least 5 categories
- Spec §4.2's predicted failure modes are visible (block property, javaXxx, char-class intersection, etc.)

- [ ] **Step 2: Verify root build is still untouched**

```
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH ./gradlew build -x :compat-test:test -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu
```

Expected: BUILD SUCCESSFUL, all pre-existing modules unaffected.

- [ ] **Step 3: Final commit if anything new (e.g., tweaks)**

If no diff, skip. Otherwise:

```
git status
git add -p
git commit -s -m "(test) compat-test: final tweaks after smoke run

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## Done criteria

- [ ] `./gradlew :compat-test:compatReport -Dpcre2.library.path=...` exits 0
- [ ] `compat-test/build/reports/compat/raw.jsonl` has > 2000 records (combined `.txt` + dynamic tests)
- [ ] `compat-test/build/reports/compat/report.md` shows ≥ 5 root-cause categories with non-zero counts
- [ ] Spec §4.2's 9 predicted failure categories are either present in the report or explicitly noted as "not observed"
- [ ] `./gradlew build -x :compat-test:test` still passes (no regression in main modules)
- [ ] Branch `chang/compat-test` has clean linear history with conventional commits & DCO sign-off
