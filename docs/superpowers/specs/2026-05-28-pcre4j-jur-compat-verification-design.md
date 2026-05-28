# pcre4j ↔ `java.util.regex` 兼容性实测验证

**Date**: 2026-05-28
**Branch**: `chang/compat-test`
**Status**: Design approved, ready for implementation plan
**Driver doc**: `~/chang/OneDrive - Microsoft/My/ai-task/regex/pcre2-replace-jur-feasibility.md`

## 1. 目标与非目标

### 目标

依据可行性报告 §8 验证清单，**实际**测量 pcre4j 当前 `main` HEAD 与 JDK 21u `java.util.regex` 的兼容性差距，使后续"是否把 `java.util.regex` 迁到 pcre4j、走哪条形态（A/B/C）"的决策有量化依据。

具体可交付：
- 一个本地运行的 `:compat-test` Gradle 子项目，能让 oracle (`java.util.regex`) 与 SUT (`org.pcre4j.regex`) 在同一批输入上跑出可对比的结果记录
- 每条用例一行的原始记录 `build/reports/compat/raw.jsonl`（永久制品，可重复分析）
- 按失败原因分类聚合的 `build/reports/compat/report.md`

### 非目标

- 不修复 pcre4j 的任何兼容性问题（只测量、不改 pcre4j 源码）
- 不构建可行性报告 §6.3 的模式转译层
- 不做性能 / JIT / 内存对比
- 不做 `Pattern.serialize` / Serializable 对比（pcre4j 不实现，预期失败）
- 不向 pcre4j 上游提交 PR；产物只活在本地 worktree（`/root/SourceCode/pcre4j-compat-test`，分支 `chang/compat-test`）

## 2. 总体架构

新增 `:compat-test` Gradle 子项目，挂在 pcre4j 仓库下、依赖 `:regex` 和 `:ffm`（后端默认 FFM，理由见 §4）。

数据流：

```
JDK 21u test 源码/数据 ──(复制 + 适配层)──► :compat-test
                                                │
                          ┌─────────────────────┼─────────────────────┐
                          ▼                                           ▼
                   Oracle: java.util.regex                    SUT: org.pcre4j.regex
                          │                                           │
                          └────────── ComparisonRecorder ─────────────┘
                                                │
                          ┌─────────────────────┴─────────────────────┐
                          ▼                                           ▼
                   build/reports/compat/raw.jsonl          build/reports/compat/report.md
                   (每条用例一行)                              (按原因分类聚合)
```

- `raw.jsonl` 是事实记录，永久制品
- `report.md` 是分类视图，可从 raw 重生成

JDK 测试源码与数据文件来源：[`openjdk/jdk21u`](https://github.com/openjdk/jdk21u/tree/master/test/jdk/java/util/regex)，license 为 GPLv2+CE。子目录 `compat-test/LICENSE.NOTICE` 单独声明此 license 来源，与 pcre4j 主体 LGPL-3.0 隔离。**产物不发布上游**，由 ROADMAP 注明"内部回归工具"。

## 3. 后端选择

**默认 FFM**，依据：
- pcre4j 的 JNA / FFM 两个后端实现同一 `IPcre2` 契约，共享 `testFixtures` 下的 contract tests，功能等价（无超集差异）
- FFM 直接调用 native，无 JNA marshalling 开销，性能更优；亦为 pcre4j 项目自身的推荐
- 用户判定标准："功能等价则选性能好的"

Gradle property `-Pcompat.backend=jna` 切换到 JNA，仅在 FFM 上出现可疑 native crash 时做交叉验证。

注意：Java 21 上 FFM 仍是 preview，构建必须带 `--enable-preview`；`:compat-test` 复用 `:ffm` 模块已配好的 JVM args。

## 4. 测试驱动：两类用例

### 4.1 数据驱动（`.txt`，4 个文件）

| 文件 | 行数级别 | 内容 |
|---|---|---|
| `TestCases.txt` | ~300 行 | 通用 pattern/input/期望 |
| `BMPTestCases.txt` | ~600 行 | BMP 字符 |
| `SupplementaryTestCases.txt` | ~1200 行 | 增补平面字符 |
| `GraphemeTestCases.txt` | ~50 行 | 字素簇 |

格式：每条用例三行（pattern / input / `<true|false> [matched-text] [group-index]`）。

实现：`TxtCaseRunner` 解析、`TxtCompatTest` 用 JUnit 5 `@ParameterizedTest` 跑全量。每条用例：
- 用 oracle 探测一次 → `MatchProbe` (oracle)
- 用 SUT 探测一次 → `MatchProbe` (sut)
- 三元 (oracle, sut, expected-from-file) 写入 `raw.jsonl`

### 4.2 Java 测试驱动（9 个 `.java` 文件）

| 文件 | 主要覆盖 |
|---|---|
| `RegExTest.java` (190 KB, 121 个 test 方法) | 几乎全部 Pattern/Matcher API，含 `hitEndTest` / `regionTest` / `boundsTest` / `replaceFirstTest` / `globalSubstitute` / `appendTest` / 30+ `stringBuffer*` `stringBuilder*` 替换测试 |
| `NamedGroupsTests.java` | 命名组 |
| `PatternStreamTest.java` | `splitAsStream` |
| `SplitWithDelimitersTest.java` | Java 21 新 `splitWithDelimiters` |
| `POSIX_ASCII.java` / `POSIX_Unicode.java` | POSIX 字符类（含 18 个 `\p{javaXxx}`） |
| `ImmutableMatchResultTest.java` | `MatchResult` 不可变性 |
| `NegativeArraySize.java` | 边界回归 |

实现：复制源码到 `src/test/java/org/pcre4j/compat/imported/`，仅改 import (`java.util.regex` → `org.pcre4j.regex`)，把旧风格 main + 反射跑 testXxx 改写成 JUnit 5 `@Test` 方法。SUT 跑出的任何断言失败 = 一次"行为不一致"，记录到 `raw.jsonl`，`cause` 字段标 `assertion-failure: <断言信息>`。

### 4.3 显式跳过（with 标记）

整目录跳过：
- `whitebox/` — 测 JDK Pattern 内部 IR 节点，反射 JDK 私有字段，与引擎语义无关

单方法跳过（在 imported 文件中加 `@Disabled("jdk-internal: ...")` 或 `@Disabled("pcre4j-not-implemented: ...")`）：
- 任何用 `java.util.regex.Pattern.class.getDeclaredField/Method/Constructor` 反射访问 JDK 私有 API 的方法
- `RegExTest.serializeTest()` — pcre4j 不实现 Serializable

**绝不跳过**：`hitEndTest` / `regionTest` / `boundsTest` / `surrogatePairOverlapRegion` / `replaceFirstTest` / `globalSubstitute` / `appendTest` / `literalReplacementTest` / 30+ `stringBuffer*` `stringBuilder*` 方法。即使它们大量失败，这正是要测量的差距。

所有跳过项在 `report.md` 的"显式跳过"一节列出，保持透明。

## 5. 对比协议

```java
record MatchProbe(
    Outcome compile,          // OK | SyntaxError(msg)
    Boolean matchesFull,      // matcher.matches() 结果
    Boolean lookingAt,        // matcher.lookingAt() 结果
    List<Hit> findAll         // 连续 find() 直到 false 的全部命中
) {}

record Hit(int start, int end, String text, List<Group> groups) {}
record Group(String name, int start, int end, String text) {}
```

每条数据驱动用例对 oracle / SUT 各算一个 `MatchProbe`，按下表分类：

| oracle.compile | sut.compile | 分类 |
|---|---|---|
| OK | OK | matches/lookingAt/findAll 比较，不同 → `behavior-diff` |
| OK | SyntaxError | `sut-compile-error` |
| SyntaxError | OK | `sut-accepts-rejected` |
| SyntaxError | SyntaxError | **视为兼容**（不算失败，错误消息文本不比较） |
| OK | OK，但 SUT 抛 RuntimeException | `sut-runtime-error` |

**本期 `MatchProbe` 不直接包含 `hitEnd` / `requireEnd` / `region`**（数据驱动 `.txt` 没这些维度）；但 `RegExTest.java` 的 `hitEndTest` / `regionTest` / `boundsTest` 通过断言式测试覆盖，等同验证。

## 6. 模块结构

```
compat-test/
├── build.gradle.kts                    # apply java + junit + ffm + regex
├── README.md                           # 用途、跑法、报告位置
├── LICENSE.NOTICE                      # 子目录 license: imported sources GPLv2+CE
└── src/
    ├── main/java/org/pcre4j/compat/
    │   ├── MatchProbe.java
    │   ├── Probes.java                 # oracle/SUT 通用探测器
    │   ├── ComparisonRecorder.java     # 写 raw.jsonl，线程安全
    │   ├── TxtCaseRunner.java          # 解析 .txt + 跑 oracle/SUT
    │   └── report/
    │       ├── RawRecord.java
    │       ├── Classifier.java         # 基于 pattern 字符串的正则分类
    │       └── ReportRenderer.java
    └── test/
        ├── java/org/pcre4j/compat/
        │   ├── TxtCompatTest.java      # @ParameterizedTest 跑 4 个 .txt
        │   └── imported/
        │       ├── RegExTest.java      # @<sha>，仅 import / @Disabled 改动
        │       ├── NamedGroupsTests.java
        │       ├── PatternStreamTest.java
        │       ├── SplitWithDelimitersTest.java
        │       ├── POSIX_ASCII.java
        │       ├── POSIX_Unicode.java
        │       ├── ImmutableMatchResultTest.java
        │       ├── NegativeArraySize.java
        │       └── README.md           # @<sha> + 修改清单
        └── resources/imported/
            ├── TestCases.txt
            ├── BMPTestCases.txt
            ├── SupplementaryTestCases.txt
            └── GraphemeTestCases.txt
```

### 6.1 build.gradle.kts 关键点

- 继承 root 的 java toolchain 21、checkstyle
- `dependencies { testImplementation(project(":regex")); testImplementation(project(":ffm")) }`，按 `-Pcompat.backend` 切换 FFM/JNA
- 复用 `:ffm` 的 test JVM args（`--enable-preview` 等）防止漏配
- `tasks.test { systemProperty("pcre2.library.path", System.getProperty("pcre2.library.path")) }`
- root checkstyle 排除 `compat-test/src/test/java/org/pcre4j/compat/imported/**`（外来源码不强求符合 pcre4j 风格）
- 新增 `compatReport` task：读 raw.jsonl，调 `ReportRenderer` 产 report.md
- `settings.gradle.kts` 加 `include(":compat-test")`

### 6.2 仓库 CI 隔离

- `:compat-test` 不在 root `subprojects {}` 的默认 build 链路里，`./gradlew build` 不被牵连
- 仅通过显式 `:compat-test:test` 触发
- ROADMAP 加一行"内部兼容性回归工具，不发布"

## 7. 运行入口

```bash
# 全量 compat 测试 + 生成报告（默认 FFM 后端）
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 \
PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH \
./gradlew :compat-test:test :compat-test:compatReport \
  -Dpcre2.library.path=/usr/lib/x86_64-linux-gnu

# 只跑 .txt 数据用例（快速迭代）
./gradlew :compat-test:test --tests "org.pcre4j.compat.TxtCompatTest" ...

# 切换到 JNA 后端做差异回归
./gradlew :compat-test:test -Pcompat.backend=jna ...
```

`report.md` 结构示例：

```markdown
# pcre4j compat report vs java.util.regex (oracle)
# pcre4j HEAD: <commit-sha>   FFM backend   PCRE2: <version>

## Summary
| Source | Total | Pass | Fail | sut-compile-error | sut-runtime-error | behavior-diff |
| TestCases.txt | 312 | 240 | 72 | 51 | 0 | 21 |
| BMPTestCases.txt | ... |
| RegExTest.java (assertion-driven) | 121 methods | 89 | 32 | - | - | - |
| ...

## Failures by root cause
| Cause | Count | Sample pattern |
| 块属性 \p{InXxx}        | 34 | a\p{InGreek} |
| Is 前缀 \p{IsXxx}       | 12 | \p{IsLatin} |
| Java \p{javaXxx}        | 18 | \p{javaWhitespace} |
| 字符类交集 &&           | 18 | [a-z&&[def]] |
| (?U) 内联反义           |  3 | (?U)\w+ |
| UNICODE_CASE 被忽略     |  7 | (?i) tüRkİye |
| 其他/未分类             |  9 | ... |

## Explicit skips
- `whitebox/**` — 测 JDK Pattern 内部 IR 节点
- `RegExTest.serializeTest` — pcre4j 不实现 Serializable
- `RegExTest.xxx` — 反射 JDK 私有 API
```

## 8. 里程碑

| # | 阶段 | 验证标准 |
|---|---|---|
| M1 | 子项目骨架（build.gradle.kts + settings.gradle.kts + 空目录） | `./gradlew :compat-test:compileTestJava` 通过；`./gradlew build` 不被牵连 |
| M2 | `MatchProbe` / `Probes` / `ComparisonRecorder` 实现 + sanity 自测 | 单测：`Probes.run(api, "\\d+", "abc123")` 返回正确 `MatchProbe`；jsonl schema 稳定 |
| M3 | `.txt` 数据驱动跑通（4 个文件） | `./gradlew :compat-test:test --tests TxtCompatTest` 跑完；raw.jsonl 行数 = 用例总数；目测前 20 条 diff 与可行性报告 §3 预测吻合 |
| M4 | 9 个 `.java` 测试搬运 + 适配 JUnit 5 | `./gradlew :compat-test:test` 跑完，输出每个 java 测试的 oracle-pass / SUT-fail / SUT-pass 统计；`POSIX_Unicode.java` 大面积失败（验证 `\p{javaXxx}` 推断） |
| M5 | `Classifier` + `ReportRenderer` + `compatReport` Gradle task + compat-test/README.md | `./gradlew :compat-test:compatReport` 产 `report.md`，至少识别可行性报告 §4.2 表里的 9 类失败原因 |

每个 M 结束跑一次 sanity 验证再进入下一阶段。

## 9. 风险与缓解

| 风险 | 缓解 |
|---|---|
| `RegExTest.java` 大量使用 JDK 内部包私有 helper | 改 import 后编译不过的方法 `@Disabled("jdk-internal")` + report 列出 |
| FFM preview 在某用例上 native crash | 切 JNA 后端复跑；两后端都崩 → 最小复现，发 issue 给 pcre4j 上游 |
| pcre4j 抛 `IllegalStateException` 而非 `PatternSyntaxException` | `Probes` 兜底 catch `RuntimeException`，归 `sut-runtime-error` |
| `.txt` 文件有 OpenJDK 自有转义约定 | `TxtCaseRunner` 严格参照 OpenJDK 源码 `processTestCases()` 的解析逻辑实现 |
| GPLv2+CE 与 LGPL-3.0 共存合规 | 不发布上游；compat-test/LICENSE.NOTICE 明确隔离；ROADMAP 注明 |

## 10. 显式 out-of-scope

避免范围蠕变，本期不做：

- 性能 / JIT / 内存对比
- `Pattern.serialize` / Serializable 对比（imported `serializeTest` 标 `@Disabled`）
- 可行性报告 §6.3 的模式转译层（只测量、不修复）
- JNA 后端的默认并行回归（按需手动 `-Pcompat.backend=jna`）
- 把 compat-test 推到 GitHub Actions / 上游 CI

发现需要时再单独开任务。

## 11. 决策记录

| 决策 | 理由 |
|---|---|
| 在 pcre4j 仓库内新增 `:compat-test` 子项目（vs 独立项目） | 复用现有 Gradle 构建、checkstyle、`testFixtures`；用户偏好 |
| 仅 FFM 后端（vs 双后端） | 功能等价、性能优；JNA 留作交叉验证手段 |
| 轻量 JUnit harness（vs jtreg 原生） | 避免 jtreg 环境搭建；oracle-vs-SUT diff 模式更直接 |
| 用 pcre4j HEAD（vs 1.0.1 tag） | 反映当前真实状态，可能发现报告后上游已修的项 |
| 报告分类基于 pattern 字符串的正则匹配（vs 语义解析） | 简单稳定，本期足够；后续可升级 |
| 单文件 raw.jsonl + 独立 report task | 事实/视图分离，分类规则可重生成 |
| 不发布上游、不进 root build | 隔离 GPLv2+CE 合规风险；不增加主 CI 耗时 |
