# Java（Maven）代码规范落地方案

> 从工具选型到团队推广，一份可操作的完整指南。

---

## 一、为什么需要代码规范

### 1.1 没有规范的痛点

代码规范不是"加分项"，而是工程效率的底线保障：

- **Code Review 内耗**：评审者在格式、命名、import 顺序上反复纠正，没精力关注逻辑正确性
- **新人上手慢**：每个文件风格不同，读代码时在"适应风格"上浪费脑力
- **合并冲突多**：不同 IDE 的自动格式化互相覆盖，一个换行就能引发冲突
- **技术债隐蔽积累**：空指针风险、死代码、无用 import 散落各处，无人主动清理
- **线上事故可追溯性差**：缺乏一致的 Javadoc 和命名，排查问题时理解成本成倍增加

### 1.2 三层目标体系

规范落地不是"把所有规则都打开"就完事了。好的方案分层推进，每层解决不同的问题：

| 层级 | 目标 | 谁来做 | 反馈速度 |
|------|------|--------|----------|
| **第一层：自动格式化** | 缩进、换行、空格、import 排序——机器全权负责，零争议 | Spotless + google-java-format | 亚秒级（IDE 保存时） |
| **第二层：风格规范** | 命名、Javadoc、文件结构——有明确标准，但需要人遵守 | Checkstyle | 实时（IDE 红线） |
| **第三层：代码质量** | 潜在 bug、空指针、性能隐患——机器辅助发现，人判断 | PMD | 分钟级（CI 阶段） |

三层递进关系：**第一层不过，根本没资格谈第二层；第二层不达标，第三层的告警会被淹没在噪音里。**

### 1.3 本文档的定位

这是一份**可落地的实施方案**，不是工具大全。每个选型有明确理由，每段配置有解释。读完你可以直接在你的 Maven 项目里把它跑起来。

---

## 二、工具选型与对比

### 2.1 格式化工具：为什么选 google-java-format

| 候选 | Maven 插件成熟度 | IDE 支持 | 可配置性 | 输出确定性 |
|------|:---:|:---:|:---:|:---:|
| **google-java-format** | ⭐⭐⭐ | IntelliJ / VS Code / Eclipse | 零配置（只有 Google / AOSP 两档） | ✅ 同一输入永远同一输出 |
| Eclipse Formatter | ⭐⭐ | Eclipse 原生，IntelliJ 需导入 | 极高（XML 配置上百项） | ❌ 不同版本可能不同 |
| Prettier-Java | ⭐ | VS Code 为主 | 低（和 Prettier 理念一致） | ✅ |
| Spring Java Format | ⭐⭐ | IntelliJ / Eclipse | 中 | ✅ |

**选择 google-java-format 的理由**：

1. **零配置**：不需要团队争论"缩进 2 格还是 4 格"，Google 已经替你定了
2. **输出确定性**：同一份代码，任何人、任何时间跑出来的结果一模一样，消除 CI vs 本地差异
3. **行业事实标准**：大量开源项目（Guava、gRPC、Protobuf）在用，新人很可能已经熟悉
4. **AOSP 风格可选**：如果团队偏好 4 空格缩进（Android 开源项目风格），一个配置项切换

### 2.2 编排器：为什么选 Spotless

| 候选 | ratchetFrom（增量） | 统一 apply/check 语义 | 多语言支持 | 
|------|:---:|:---:|:---:|
| **Spotless** | ✅ | ✅ | Java / Kotlin / XML / JSON / Markdown... |
| Maven Formatter Plugin | ❌ | ❌ | 仅 Java |
| editorconfig + Maven 插件 | ❌ | ❌ | 仅空白字符层面 |

**选择 Spotless 的关键理由**：

- **`ratchetFrom`**：只检查与 `origin/main` 有差异的文件。这是存量项目落地的核心能力——老代码不动，新代码必须干净
- **统一入口**：`mvn spotless:apply` 一键搞定格式化和 import 整理，不用记三个命令
- **可扩展**：日后加 XML/POM/Markdown 格式化，同一个插件搞定

### 2.3 风格检查：为什么选 Checkstyle

| 候选 | 规则丰富度 | Maven 集成 | 无需服务端 | IDE 实时提示 |
|------|:---:|:---:|:---:|:---:|
| **Checkstyle** | ⭐⭐⭐（200+ 规则） | ✅ | ✅ | ✅ |
| SonarQube | ⭐⭐⭐ | 需 SonarQube Server | ❌ | 需 SonarLint 连接 |
| Error Prone | ⭐ | 编译期检查 | ✅ | ✅ |

**选择 Checkstyle 的理由**：零基础设施依赖，规则库成熟（Sun/Google 官方风格直接可用），Maven 原生绑定 `verify` 阶段。

### 2.4 质量检查：为什么选 PMD

Java 静态分析有三个技术路线：

| 工具 | 分析方式 | 优势 | 劣势 |
|------|---------|------|------|
| **PMD** | AST 源码分析 | 不需编译，速度快，规则易懂 | 跨文件分析弱 |
| SpotBugs | 字节码分析 | 能发现深层 bug（如空指针路径） | 必须先编译，CI 慢 |
| Error Prone | 编译器插件 | 编译期拦截，零运行时开销 | 侵入性强，规则较少 |

**选择 PMD 的理由**：源码级分析不需要编译，适合放在 `verify` 阶段跑；规则按类别清晰分组（`errorprone` / `bestpractices` / `performance` / `security`），团队可以按需逐步开启。**如果需要更深层的 bug 检测，可以后续补充 SpotBugs，两者不冲突。**

### 2.5 明确不用的工具及其原因

| 工具 | 不选的理由 |
|------|-----------|
| **SonarQube Server** | 需要独立服务端，小团队运维成本高；反馈链路过长（推代码 → 等扫描 → 看 Web 控制台）。SonarLint 独立使用可以考虑 |
| **SpotBugs（当前方案默认不含）** | 需要 `mvn compile` 先跑完，比 PMD 慢；PMD 的 `errorprone` 类别已覆盖大部分常见 bug。如果需要，可在 `verify` 阶段追加 |
| **EditorConfig 单独使用** | 只管空白字符（缩进风格、行尾空格、文件末尾空行），完全不涉及 Java 语法。作为补充可以，作为主力不够 |

### 2.6 职责边界总则

每个规则应该**只在唯一一个工具中配置**。如果两个工具同时管同一件事，不仅浪费 CI 时间，更重要的是可能产生矛盾——A 工具改完，B 工具又报错。

具体分工见 [第十一章](#十一工具关系深度解析)。

---

## 三、架构总览

### 3.1 整体架构图

```
mvn spotless:apply  ← 一键搞定格式化 + import
         │
    Spotless（编排器）
      ├── google-java-format  → 缩进、换行、空格、花括号
      ├── importOrder         → import 分组排序
      ├── removeUnusedImports → 清理无用 import
      ├── trimTrailingWhitespace → 行尾空格
      └── endWithNewline      → 文件末尾空行

mvn verify  ← CI 标准检查（依次执行）
      ├── spotless:check      → 格式是否合规
      ├── checkstyle:check    → 命名、Javadoc、结构
      └── pmd:check           → 空指针、死代码、性能
```

### 3.2 三层反馈循环

```
你的 IDE（写代码时）          ← 亚秒级，实时反馈
    │
    ├── 格式化快捷键 → google-java-format 即改
    ├── Checkstyle 插件 → 红线即时提示
    └── PMD 插件 → 黄线即时提示
    │
    ▼
你的 Git（push 时）            ← 秒级，轻量拦截
    │
    ├── pretty-format-java  → 快速格式化（毫秒级）
    ├── trailing-whitespace → 清理行尾空格
    └── spotless:check      → 格式最终确认（~3秒，有 JVM 启动开销）
    │
    ▼
你的 CI（PR/合并时）           ← 分钟级，最终防线
    │
    ├── mvn spotless:check      → 格式化回归检查
    ├── mvn checkstyle:check    → 风格规范检查
    └── mvn pmd:check           → 代码质量检查
    │
    ▼
全部通过 → ✅ 合并 / 部署
任何失败 → ❌ 拦截 + 开发者修复
```

**设计原则**：每一层只做它最适合做的事。IDE 做最快的反馈，Git Hook 做最轻的拦截，CI 做最全的检查。不在 Git Hook 里跑重任务（Checkstyle/PMD），避免把 push 变成"去喝杯咖啡"。

---

## 四、Maven 配置详解

### 4.1 pom.xml — Spotless 核心配置

> **设计意图**：Spotless 负责一切"机器可以自动决定"的事情——格式化、import 排序、空白清理。这些事不应该让人来操心，也不应该在 Code Review 中讨论。

#### 完整配置

```xml
<project>
  <!-- ... 已有内容 ... -->

  <build>
    <plugins>
      <!-- ===== Spotless：代码格式化 + import 整理 ===== -->
      <plugin>
        <groupId>com.diffplug.spotless</groupId>
        <artifactId>spotless-maven-plugin</artifactId>
        <version>2.44.3</version>
        <configuration>
          <!-- 增量模式：只扫与 main 分支不同的文件 -->
          <ratchetFrom>origin/main</ratchetFrom>

          <java>
            <!-- 格式化引擎：Google Java Style -->
            <googleJavaFormat>
              <!-- 可选：AOSP 风格（4空格缩进） -->
              <!-- <style>AOSP</style> -->
            </googleJavaFormat>

            <!-- import 排序规则 -->
            <importOrder>
              <order>java,javax,org,com,</order>
              <!-- 最后一个逗号表示「其他所有」 -->
            </importOrder>

            <!-- 自动清理无用 import -->
            <removeUnusedImports />

            <!-- 通用规范 -->
            <trimTrailingWhitespace />
            <endWithNewline />
          </java>

          <!-- 排除自动生成代码 -->
          <excludes>
            <exclude>build/**</exclude>
            <exclude>generated/**</exclude>
            <exclude>target/generated-sources/**</exclude>
          </excludes>
        </configuration>
      </plugin>

      <!-- ===== Checkstyle：命名规范 + 风格检查 ===== -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-checkstyle-plugin</artifactId>
        <version>3.6.0</version>
        <configuration>
          <configLocation>checkstyle.xml</configLocation>
          <failOnViolation>true</failOnViolation>
          <violationSeverity>warning</violationSeverity>
          <excludes>**/generated/**/*</excludes>
        </configuration>
        <executions>
          <execution>
            <phase>verify</phase>  <!-- CI 打包阶段才跑 -->
            <goals>
              <goal>check</goal>
            </goals>
          </execution>
        </executions>
      </plugin>

      <!-- ===== PMD：代码质量检查 ===== -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-pmd-plugin</artifactId>
        <version>3.26.0</version>
        <configuration>
          <failOnViolation>true</failOnViolation>
          <rulesets>
            <ruleset>pmd-rules.xml</ruleset>
          </rulesets>
          <excludes>
            <exclude>**/generated/**</exclude>
          </excludes>
        </configuration>
        <executions>
          <execution>
            <phase>verify</phase>
            <goals>
              <goal>check</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

#### 关键参数解释

**`ratchetFrom: origin/main`**（Spotless 独有能力）

指定一个 Git 引用作为"基准"，Spotless 只检查**相对于该基准有变更的文件**。这带来的好处：
- 存量项目不慌——老代码不被触碰，新代码和修改的代码必须干净
- CI 速度快——只扫改动的文件，不和全量代码较劲
- 自然演进——随着正常开发迭代，越来越多的文件被"洗白"

限制：CI 必须做完整 clone（`fetch-depth: 0`），否则 Git 历史不完整，ratchet 无法工作。

**`googleJavaFormat` 风格选择**

- **Google 风格**（默认）：2 空格缩进，100 字符行宽上限
- **AOSP 风格**：4 空格缩进，其余和 Google 基本一致

选择建议：新项目直接用 Google 默认。如果团队绝大多数人来自 Android 开发背景，可考虑 AOSP。**一旦选定就不要改**——全量重格式化会污染 git blame。

**`importOrder`**

`java,javax,org,com,` 的含义：
- `java.*` 在最前（JDK 标准库）
- `javax.*` 紧随其后（JDK 扩展）
- `org.*` 第三方库（如 `org.springframework`）
- `com.*` 第三方库 + 业务代码（如 `com.google.common`、`com.example`）
- 末尾的裸逗号 `,` 代表"其他所有"（如 `lombok.*`）

如果你的业务代码统一在某包下（如 `com.mycompany`），可以改写成：
```xml
<order>java,javax,org,com.mycompany,com,</order>
```
这样你的业务 import 会和第三方 `com.*` 分开。

**`removeUnusedImports`**

Spotless 用编译器级别的分析来判断 import 是否使用，比 Checkstyle 的 `UnusedImports` 规则准确得多。因此 Checkstyle 里应该**关闭** `UnusedImports` 规则——这个活完全交给 Spotless。

**`trimTrailingWhitespace` 和 `endWithNewline`**

google-java-format 不管行尾空格和文件末尾空行，这两项作为兜底补充。

**`excludes` 排除规则**

以下目录永远不该被格式化工具碰：
- `build/**`、`target/**`：构建产物
- `generated/**`、`target/generated-sources/**`：自动生成的 Java 文件（如 MapStruct Mapper 实现、gRPC stub）

### 4.2 checkstyle.xml — Checkstyle 规则配置

> **设计意图**：Checkstyle 负责"人能遵守但机器不识别的规范"——命名、Javadoc、代码结构。规则选择原则：**不和 Spotless 管的事重叠**，每条规则有明确的"抓到什么问题"。

```xml
<!-- checkstyle.xml -->
<!DOCTYPE module PUBLIC
  "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
  "https://checkstyle.org/dtds/configuration_1_3.dtd">
<module name="Checker">
  <!-- 文件级别 -->
  <property name="charset" value="UTF-8"/>
  <property name="fileExtensions" value="java"/>

  <!-- 文件尾部空行 -->
  <module name="NewlineAtEndOfFile"/>

  <module name="TreeWalker">
    <!-- ===== 命名规范 ===== -->
    <module name="PackageName">
      <property name="format" value="^[a-z]+(\.[a-z][a-z0-9]*)*$"/>
    </module>
    <module name="TypeName"/>          <!-- 类名：大驼峰 -->
    <module name="MethodName"/>        <!-- 方法名：小驼峰 -->
    <module name="ParameterName"/>     <!-- 参数名：小驼峰 -->
    <module name="LocalVariableName"/> <!-- 局部变量：小驼峰 -->
    <module name="ConstantName"/>      <!-- 常量：UPPER_CASE -->
    <module name="StaticVariableName"/> <!-- 静态变量：小驼峰 -->
    <module name="MemberName"/>        <!-- 成员变量：小驼峰 -->

    <!-- ===== Javadoc 规范 ===== -->
    <module name="JavadocMethod"/>
    <module name="JavadocType"/>
    <module name="JavadocStyle"/>

    <!-- ===== 代码规范 ===== -->
    <module name="EqualsHashCode"/>    <!-- 重写 equals 必须重写 hashCode -->
    <module name="IllegalInstantiation"/> <!-- 禁止实例化某些类 -->
    <module name="SimplifyBooleanExpression"/>
    <module name="SimplifyBooleanReturn"/>
    <module name="StringLiteralEquality"/> <!-- 字符串用 equals 不用 == -->
    <module name="UnusedImports"/>     <!-- 无用 import（Spotless 已管，可不要） -->
  </module>
</module>
```

#### 核心规则解析

**命名规范组**

| 规则 | 抓什么 | 示例违规 |
|------|--------|---------|
| `TypeName` | 类/接口/枚举不是大驼峰 | `class user_service` |
| `MethodName` | 方法不是小驼峰 | `void GetUser()` |
| `ConstantName` | 常量（`static final`）不是全大写+下划线 | `static final int maxRetry = 3` |
| `MemberName` | 成员变量不是小驼峰 | `private String UserName;` |

**Javadoc 组 — 何时开启、何时放宽**

- **库项目 / 对外 API**：必须开启，这是使用者的文档
- **内部微服务**：可以只在 `public` 方法上要求（配置 `scope="public"`）
- **Spring Boot Controller**：`@Override` 方法和 REST 端点可以考虑排除，减少噪音

**关键 bug 预防规则**

- **`EqualsHashCode`**：重写 `equals()` 不重写 `hashCode()`，导致对象放进 `HashMap`/`HashSet` 后找不到。这是真实 bug，不是风格问题
- **`StringLiteralEquality`**：用 `==` 比较字符串。在 Java 里 `==` 比较引用地址，"hello" == new String("hello") 是 `false`
- **`IllegalInstantiation`**：禁止用 `new Boolean("true")`（Java 9 已废弃），应该用 `Boolean.valueOf()`

### 4.3 pmd-rules.xml — PMD 规则配置

> **设计意图**：PMD 负责"机器能分析出的潜在问题"——空指针路径、未使用变量、性能隐患。规则选择原则：**优先开启高信号低噪音的类别（errorprone、bestpractices），按团队承受能力逐步增加。**

```xml
<!-- pmd-rules.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<ruleset name="Custom Rules"
  xmlns="http://pmd.sourceforge.net/ruleset/2.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0 https://pmd.sourceforge.net/ruleset_2_0_0.xsd">
  <description>项目代码质量规则</description>

  <!-- 最佳实践 — 始终开启，噪音极低 -->
  <rule ref="category/java/bestpractices.xml">
    <exclude name="JUnitAssertionsShouldIncludeMessage"/>
    <exclude name="JUnitTestContainsTooManyAsserts"/>
  </rule>

  <!-- 潜在 bug — 始终开启，抓到大概率是真问题 -->
  <rule ref="category/java/errorprone.xml">
    <exclude name="BeanMembersShouldSerialize"/>
  </rule>

  <!-- 性能 — 影响可控时开启 -->
  <rule ref="category/java/performance.xml"/>

  <!-- 代码风格 — 选择性开启，不和 Spotless/Checkstyle 重叠 -->
  <rule ref="category/java/codestyle.xml/EmptyMethodInAbstractClassShouldBeAbstract"/>
  <rule ref="category/java/codestyle.xml/ExtendsObject"/>
  <rule ref="category/java/codestyle.xml/ForLoopShouldBeWhileLoop"/>
  <rule ref="category/java/codestyle.xml/UnnecessaryImport"/>
  <rule ref="category/java/codestyle.xml/UnnecessaryModifier"/>
  <rule ref="category/java/codestyle.xml/UnnecessaryReturn"/>
</ruleset>
```

#### PMD 规则类别分级

| 类别 | 信号质量 | 存量项目建议 | 说明 |
|------|:---:|------|------|
| `errorprone` | ⭐⭐⭐ 高 | **第一阶段就开** | 空指针、资源泄漏、equals 误用——抓到大概率是真 bug |
| `bestpractices` | ⭐⭐⭐ 高 | **第一阶段就开** | 用了废弃 API、switch 缺 default、异常被吞——成熟且噪音低 |
| `security` | ⭐⭐⭐ 高 | **立即开启** | 硬编码密码、SQL 注入风险——不管项目多老都得查 |
| `performance` | ⭐⭐ 中 | 第二阶段 | 字符串拼接、无脑装箱、循环里创建对象——看团队对性能的敏感度 |
| `codestyle` | ⭐ 低~中 | 第三阶段 | 很多和 Checkstyle 重叠，需要精选子规则开启 |
| `design` | ⭐ 低 | 第四阶段（可选） | 类过长、方法参数过多——存量项目噪音极大，建议新项目用 |
| `multithreading` | ⭐⭐ 中 | 有并发代码时开 | 线程安全相关——项目没有多线程就别开，开了全是噪音 |

### 4.4 Maven 生命周期与代码检查的绑定

Checkstyle 和 PMD 绑定在 `verify` 阶段，而不是 `compile` 或 `test`：

```
compile → test → package → verify → install → deploy
                              ↑
                     Checkstyle + PMD 在这里
```

**为什么不绑在更早的阶段？**

- `compile` 阶段：开发时频繁跑，如果格式问题打断编译流程，太影响体验
- `test` 阶段：测试阶段的职责是验证逻辑，混入格式检查混淆了关注点
- `verify` 阶段：语义是"验证制品质量"，是代码检查的天然位置。CI 跑 `mvn verify` 时一并执行

你仍然可以手动提前检查：
```bash
mvn checkstyle:check   # 不跑测试，只跑 Checkstyle
mvn pmd:check          # 不跑测试，只跑 PMD
```

---

## 五、渐进式落地策略

### 5.1 新项目（Greenfield）

新项目没有历史包袱，**第一天全部启用**：

1. 配好 pom.xml、checkstyle.xml、pmd-rules.xml
2. Spotless `ratchetFrom` 设为 `origin/main`
3. Checkstyle `violationSeverity` 前两周设为 `warning`，团队适应后改为 `error`
4. CI 从第一个 PR 开始拦截

**第一周可能的情况**：Checkstyle 和 PMD 报大量 warning。正常——团队在摸索规则的边界。两周后再收紧。

### 5.2 存量项目（Brownfield）— 五阶段渐进

存量项目如果一刀切全上，结果一定是"CI 红了也没人管"。必须分阶段：

#### 第一阶段（第 1 周）：格式化，只扫新代码

```xml
<!-- Spotless 配置 -->
<ratchetFrom>origin/main</ratchetFrom>
```

- 只开启 Spotless，**不开启 Checkstyle、不开启 PMD**
- `ratchetFrom` 确保只检查改动的文件
- 团队成员习惯 IDE 保存时自动格式化
- 验证标准：一周内无人抱怨格式化打断工作

#### 第二阶段（第 2-4 周）：引入 Checkstyle，宽松模式

```xml
<failOnViolation>false</failOnViolation>
<violationSeverity>warning</violationSeverity>
```

- 生成基线报告：`mvn checkstyle:check`
- 找出 Top-10 高频违规，集中修复
- **不阻断 CI**——红了只是提醒，不阻塞合并
- 在 `suppressions.xml` 中批量豁免已知的老文件（详见 [第六章](#六抑制与排除机制)）

#### 第三阶段（第 2 个月）：引入 PMD，先抓 bug

```xml
<!-- pmd-rules.xml：最小规则集 -->
<rule ref="category/java/errorprone.xml"/>
<rule ref="category/java/bestpractices.xml"/>
```

- 只开 `errorprone` 和 `bestpractices`，不开 `design` 和 `codestyle`
- 修复 PMD 找出的真实 bug——这个阶段**通常能发现几个线上隐患**
- `failOnViolation=false`，只报告不阻断

#### 第四阶段（第 3 个月）：锁死

```xml
<failOnViolation>true</failOnViolation>
```

- 此时团队已适应规则，CI 拦截是合理的
- 逐步追加 PMD 规则类别（`performance` → `codestyle` → `design`）
- 在 GitHub/GitLab 配置分支保护，`format` 和 `lint` 检查必须通过才能合并

#### 第五阶段（时机成熟时）：全量格式化

```bash
# 在一个专门的 PR 里
mvn spotless:apply
git add .
git commit -m "style: 全量应用 google-java-format"
```

**全量格式化前必须做的事**：配置 `.git-blame-ignore-revs`，把格式化 commit 加进去，这样 `git blame` 不会追溯到格式化提交：

```bash
# .git-blame-ignore-revs
# 全量格式化提交，git blame 时跳过
<格式化commit的SHA>
```

```bash
# 团队成员执行
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

GitHub 会自动识别 repo 根目录的 `.git-blame-ignore-revs` 文件。

### 5.3 `ratchetFrom` 的实现原理与限制

`ratchetFrom` 本质上是 `git diff origin/main --name-only` + Spotless 检查。它只作用于 Spotless（格式化），**对 Checkstyle 和 PMD 无效**。

如果需要对 Checkstyle/PMD 也做到"只查增量"，可以用 Maven 的 `-Dcheckstyle.suppressionsLocation` 配合文件清单，或者借助 CI 平台的能力（如 GitHub Actions 的 `changed-files` action）只对变更文件跑检查。但这会增加复杂度，建议在"渐进式五阶段"里用 `failOnViolation=false` + 手动修复 Top-N 的方式消化，而不是引入更多工具。

---

## 六、抑制与排除机制

任何规范都有例外。**关键不是杜绝例外，而是让每个例外都可追溯、可审计。**

### 6.1 抑制级别总览

| 级别 | 范围 | 写法 | 何时用 |
|------|------|------|--------|
| **行级** | 一行代码 | `// NOPMD` 或 `// spotless:off` | 孤例异常 |
| **方法/类级** | 一个方法或类 | `@SuppressWarnings("PMD.RuleName")` | 该规则对此方法不适用 |
| **文件级** | 整个文件 | `// CHECKSTYLE:OFF` 或 suppressions.xml | 自动生成或第三方代码 |
| **规则级** | 整个项目 | 从规则文件删除该规则 | 团队一致认为规则不合理 |

### 6.2 Spotless 抑制

Spotless 的抑制最简单——因为它的规则全是"自动修复"型，几乎不存在"故意不格式化"的场景。唯一可能用的：

```java
// spotless:off
public class SpecialFormatting {
    // 这段代码的特殊格式我需要保留
}
// spotless:on
```

**什么时候不该用**：如果一段代码"必须"保持某种格式，先问自己两个问题：
1. 能不能调整 Spotless 配置让它接受这种格式？
2. 如果确实不能，这段代码是不是应该放在被 `excludes` 排除的目录里？

99% 的情况下不需要 `spotless:off`。

### 6.3 Checkstyle 抑制

#### 方式一：suppressions.xml（推荐）

```xml
<!-- suppressions.xml -->
<!DOCTYPE suppressions PUBLIC
  "-//Checkstyle//DTD SuppressionXpathFilter Experimental Configuration 1.2//EN"
  "https://checkstyle.org/dtds/suppressions_1_2_xpath_experimental.dtd">
<suppressions>
  <!-- 排除整个 package -->
  <suppress files="com[\\/]example[\\/]generated[\\/].*" checks=".*"/>

  <!-- 排除特定文件的特定规则 -->
  <suppress files="LegacyService\.java" checks="JavadocMethod"/>

  <!-- 排除 Lombok 生成的代码 -->
  <suppress files=".*" checks="MagicNumber" lines="1-9999"/>
</suppressions>
```

在 `pom.xml` 中引用：

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-checkstyle-plugin</artifactId>
  <configuration>
    <configLocation>checkstyle.xml</configLocation>
    <suppressionsLocation>suppressions.xml</suppressionsLocation>
    <suppressionsFileExpression>checkstyle.suppressions.file</suppressionsFileExpression>
  </configuration>
</plugin>
```

#### 方式二：注解抑制

```java
@SuppressWarnings("checkstyle:JavadocMethod")
public void legacyMethod() {
    // 这个老方法暂时不补 Javadoc
}
```

**要求**：旁边必须有注释解释**为什么**抑制，以及**什么时候**可以去掉：

```java
// TODO(PROJ-1234): 等 Q3 重构后补上 Javadoc，届时移除 SuppressWarnings
@SuppressWarnings("checkstyle:JavadocMethod")
public void legacyMethod() {
```

#### 方式三：注释包裹

```java
// CHECKSTYLE:OFF
public class GeneratedStub {
    // 自动生成的代码
}
// CHECKSTYLE:ON
```

**仅在自动生成代码或第三方源码上使用**。手写代码应该用 `suppressions.xml` 或注解。

### 6.4 PMD 抑制

#### 方式一：注解（推荐）

```java
@SuppressWarnings("PMD.AvoidInstantiatingObjectsInLoops")
public void processBatch() {
    for (BatchItem item : items) {
        // 这里创建对象是合理的——item 数量很小（≤20）
        Result r = new Result(item);
    }
}
```

#### 方式二：行级注释

```java
public void legacy() {
    System.exit(1); // NOPMD — 这是命令行工具，退出是正常行为
}
```

#### 方式三：规则文件中排除

```xml
<!-- pmd-rules.xml 中排除特定文件或类 -->
<rule ref="category/java/errorprone.xml">
    <exclude name="DataflowAnomalyAnalysis"/>
</rule>
```

### 6.5 抑制的治理

**抑制不是"设了就不用管"**。建立以下习惯：

1. **每个抑制必须有注释说明原因**（`// NOPMD — 原因` 或 `@SuppressWarnings` 旁加 `// TODO`）
2. **PR Review 时检查抑制**：新增的抑制标记和新增的代码一样需要 Review
3. **每季度审计一次** `suppressions.xml`，清理已经不再需要的抑制项
4. **CI 中可以加一步**：统计抑制标记数量，如果比上次多了就报警（说明有人在滥用）

---

## 七、IDE 配置

### 7.1 IntelliJ IDEA

#### google-java-format 插件

1. `Settings → Plugins → 搜索 "google-java-format" → 安装`
2. `Settings → Editor → Code Style → Java → 齿轮图标 → "google-java-format" 设为默认`
3. 启用 `Reformat on save`：`Settings → Tools → Actions on Save → 勾选 "Reformat code"`

#### Checkstyle 插件

1. 安装 Checkstyle-IDEA 插件
2. `Settings → Tools → Checkstyle → 添加配置 → 指向项目中的 checkstyle.xml`
3. 勾选 `Active`——违反规则的代码行会实时标红

#### PMD 插件

1. 安装 PMD-IDEA 插件
2. `Settings → Tools → PMD → 添加规则集 → 指向 pmd-rules.xml`
3. 右键项目 → `Run PMD` → 扫描结果展现在 PMD 面板

#### 团队共享配置

将 IDE 配置放入版本控制，新人 clone 即用：

```
.idea/
  codeStyles/
    Project.xml          ← 代码风格设置（要和 google-java-format 一致）
  checkstyle-idea.xml    ← Checkstyle 插件配置
```

### 7.2 VS Code

#### 前提：Java 扩展包

安装 Extension Pack for Java（包含 Language Support for Java by Red Hat）。

#### 格式化配置

`.vscode/settings.json`：

```json
{
  "java.format.settings.profile": "GoogleStyle",
  "java.format.settings.url": "https://raw.githubusercontent.com/google/styleguide/gh-pages/eclipse-java-google-style.xml",
  "editor.formatOnSave": true,
  "java.saveActions.organizeImports": true,
  "[java]": {
    "editor.defaultFormatter": "redhat.java"
  }
}
```

#### Checkstyle 集成

安装 `shengchen.vscode-checkstyle` 扩展，在 `.vscode/settings.json` 中配置：

```json
{
  "checkstyle.configurationFile": "${workspaceFolder}/checkstyle.xml",
  "checkstyle.version": "10.21.1"
}
```

#### PMD 集成

安装 `josevseb.pmd` 扩展或使用命令行手动触发。

> **VS Code 的限制**：PMD 扩展的成熟度不如 IntelliJ。如果团队重度使用 VS Code，建议将 PMD 完全交给 CI，IDE 层只做 Spotless + Checkstyle。

### 7.3 配置即代码原则

- `.vscode/settings.json`、`.idea/codeStyles/`、`.editorconfig` 统一放入版本控制
- 新人 clone 项目后，IDE 自动读取这些配置，**零人工设置**
- 如果团队使用 Eclipse，可额外提交 Eclipse 格式化配置文件（从 google-java-format 导出）

---

## 八、Pre-commit / Pre-push 集成

### 8.1 延迟预算：commit 还是 push？

一个关键设计决策：**把耗时操作放在哪个 Git 阶段？**

| 操作 | 耗时 | 放 commit | 放 push | 理由 |
|------|:---:|:---:|:---:|------|
| `pretty-format-java` | ~200ms | ✅ | - | 轻量，每次提交跑不卡 |
| `trailing-whitespace` | ~50ms | ✅ | - | 轻量 |
| `spotless:check` | ~3-5s（JVM 冷启动） | ❌ | ✅ | 每次 `git commit` 等 5 秒不可接受 |
| `checkstyle:check` | ~10-30s | ❌ | ❌ | 太重，只放 CI |
| `pmd:check` | ~20-120s | ❌ | ❌ | 太重，只放 CI |

**结论**：
- **pre-commit**：只放毫秒级操作（轻量格式化、通用清理）
- **pre-push**：放 Spotless（JVM 冷启动 3 秒，每次 push 等一等可以接受）
- **CI**：放 Checkstyle + PMD（几十秒到几分钟，异步跑不阻塞开发者）

### 8.2 `.pre-commit-config.yaml`

```yaml
# .pre-commit-config.yaml
repos:
  # Java 轻量格式化
  - repo: https://github.com/macisamuele/language-formatters-pre-commit-hooks
    rev: v2.14.0
    hooks:
      - id: pretty-format-java
        args: [--autofix, --google-java-formatter-version=1.25.2]

  # Maven/XML 文件格式化
  - repo: local
    hooks:
      - id: spotless-check
        name: Spotless (Java)
        entry: ./mvnw spotless:check
        language: system
        pass_filenames: false
        types: [java]
        stages: [push]  # 只在 git push 时跑

  # 通用清理
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-added-large-files
        args: ['--maxkb=500']
```

### 8.3 安装与验证

```bash
# 安装 pre-commit（需 Python/pip）
pip install pre-commit
# 或 macOS: brew install pre-commit

# 注册 hooks
pre-commit install              # 注册 pre-commit hooks
pre-commit install --hook-type pre-push  # 注册 pre-push hooks（关键！很多人漏掉）

# 首次全量运行验证
pre-commit run --all-files

# 跳过 hook（紧急情况，需要审计）
git push --no-verify  # ⚠️ 需要后续补上检查
```

> **注意**：`git push --no-verify` 应该只在紧急热修复时使用，且需要 Team Lead 批准。CI 层会作为第二道防线拦截。

---

## 九、CI 集成

### 9.1 GitHub Actions — 基础版

```yaml
# .github/workflows/lint.yml
name: Code Quality
on: [push, pull_request]

jobs:
  format:
    name: Format Check (Spotless)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # ratchetFrom 需要完整 git 历史
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
      - run: mvn spotless:check

  lint:
    name: Lint (Checkstyle + PMD)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
      - run: mvn verify
```

**为什么 `format` 和 `lint` 要分成两个 job？**

- 失败时一眼就知道是格式问题还是质量问题
- 并行执行，不互相等待
- 可以在分支保护规则中分别设置（比如 `format` 必须过，`lint` 初期只是 warning）

### 9.2 GitHub Actions — 进阶：PR 内联评论

使用 `reviewdog` 将 Checkstyle/PMD 的报告注入 PR 的 Review Comments：

```yaml
  lint-report:
    name: Lint Report
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
      - run: mvn checkstyle:checkstyle pmd:pmd
      - uses: reviewdog/action-setup@v1
      - name: Run reviewdog (Checkstyle)
        env:
          REVIEWDOG_GITHUB_API_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          cat target/checkstyle-result.xml | reviewdog -f=checkstyle -reporter=github-pr-review
```

这样，违规会直接出现在 PR 的对应代码行上，开发者在 PR 页面就能看到，不用去翻 CI 日志。

### 9.3 GitLab CI

```yaml
# .gitlab-ci.yml
code-quality:
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn spotless:check
    - mvn verify
  artifacts:
    reports:
      codequality: target/pmd.xml  # GitLab 原生支持 PMD 报告格式
```

### 9.4 通用 CI 服务器（Jenkins 等）

```groovy
// Jenkinsfile
pipeline {
    agent any
    stages {
        stage('Code Quality') {
            steps {
                sh 'mvn spotless:check'
                sh 'mvn verify'
            }
        }
    }
    post {
        always {
            // 发布 Checkstyle/PMD 报告为 JUnit 格式，可集成到 Jenkins UI
            junit 'target/checkstyle-result.xml'
            junit 'target/pmd.xml'
        }
    }
}
```

### 9.5 CI 缓存策略

```yaml
# GitHub Actions
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      ${{ runner.os }}-maven-
```

初次运行约需 2-3 分钟（下载依赖），缓存命中后约 30-60 秒。

---

## 十、规则定制指南

### 10.1 何时定制 vs 接受默认

**默认规则 = 社区共识 = 少争论。** 只在以下情况定制：

1. 默认规则**主动阻碍**了项目合理的代码模式（如 Lombok 生成的代码被报违规）
2. 团队有**统一的、文档化的**编码约定，且与默认规则冲突
3. 项目类型特殊（如命令行工具、SDK），部分规则天然不适用

**不要定制的情况**：个人偏好、临时不爽、懒得修代码。

### 10.2 常见定制场景

#### 场景一：使用 Lombok

Lombok 生成的 getter/setter/constructor 会被 Javadoc 规则误报。解决：

```xml
<!-- checkstyle.xml — 排除 Lombok 注解的类 -->
<module name="SuppressionXpathFilter">
    <property name="file" value="checkstyle-xpath-suppressions.xml"/>
</module>
```

```xml
<!-- checkstyle-xpath-suppressions.xml -->
<suppressions>
    <suppress-xpath checks="JavadocMethod"
        query="//CLASS_DEF[./MODIFIERS/ANNOTATION/IDENT[@text='Data']]"/>
    <suppress-xpath checks="JavadocMethod"
        query="//CLASS_DEF[./MODIFIERS/ANNOTATION/IDENT[@text='Value']]"/>
</suppressions>
```

#### 场景二：调整行宽

google-java-format 默认 100 字符。如果想放宽（比如到 120），需要在 Spotless 和 Checkstyle 两边同步：

```xml
<!-- pom.xml — Spotless 不支持直接改行宽（google-java-format 写死了） -->
<!-- 如果需要改行宽，改用 Eclipse Formatter 或 palantir-java-format -->
```

**注意**：google-java-format **不支持**自定义行宽。这是它的哲学——"别争论，接受标准"。如果团队确实需要 120/150 行宽，需要换格式化引擎。最接近的是 `palantir-java-format`（基于 google-java-format，但允许配置）。

#### 场景三：Spring Boot Controller 排除 Javadoc

```xml
<!-- checkstyle.xml — JavadocMethod 只检查 public 方法 -->
<module name="JavadocMethod">
    <property name="scope" value="public"/>
    <property name="allowMissingParamTags" value="true"/>
</module>
```

或者用 `suppressions.xml` 排除所有 `*Controller.java`。

#### 场景四：禁止特定方法调用（自定义 PMD 规则）

```xml
<!-- pmd-rules.xml -->
<rule name="ForbiddenSystemExit"
      message="Web 应用中禁止调用 System.exit()"
      class="net.sourceforge.pmd.lang.java.rule.bestpractices.
            AbstractJavaRule">
    <properties>
        <property name="xpath">
            <value>
<![CDATA[
//MethodCall[
  pmd-java:matchesSignature("java.lang.System", "exit")
]
]]>
            </value>
        </property>
    </properties>
</rule>
```

### 10.3 验证自定义规则

在提交新规则前：

```bash
# 1. 找一个已知违规的示例文件，跑一次
mvn checkstyle:check -Dcheckstyle.includes=**/TestExample.java

# 2. 找一个应该通过的正常文件，确认不会被误杀
mvn checkstyle:check -Dcheckstyle.includes=**/NormalClass.java

# 3. 全量检查
mvn verify
```

---

## 十一、工具关系深度解析

### 11.1 职责划分

| 关注点 | 负责工具 | 说明 |
|--------|:--------:|------|
| 缩进、换行、空格、花括号位置 | **google-java-format** (via Spotless) | 机器全权决定，不接受配置 |
| import 排序 | **Spotless** `importOrder` | |
| 清理无用 import | **Spotless** `removeUnusedImports` | 比 Checkstyle/PMD 准确 |
| 行尾空格 | **Spotless** `trimTrailingWhitespace` | |
| 文件末尾空行 | **Spotless** `endWithNewline` | |
| 命名规范（类名、方法名、常量等） | **Checkstyle** | |
| Javadoc | **Checkstyle** | |
| import 冗余 **检查** | **Checkstyle** `UnusedImports` | 建议关闭——交给 Spotless |
| 潜在 bug（空指针、资源泄漏） | **PMD** `errorprone` | |
| 最佳实践（switch 缺 default 等） | **PMD** `bestpractices` | |
| 性能问题（无脑装箱、循环创建对象） | **PMD** `performance` | |
| 安全漏洞（硬编码密码、SQL 注入） | **PMD** `security` | |

### 11.2 冲突预防清单

以下规则**应该从 Checkstyle 中关闭**，因为它们已经被 Spotless/google-java-format 处理了：

| Checkstyle 规则 | 原因 |
|-----------------|------|
| `UnusedImports` | Spotless `removeUnusedImports` 更准确 |
| `LineLength` | google-java-format 自己控制行宽 |
| `Indentation` | google-java-format 自己控制缩进 |
| `LeftCurly` / `RightCurly` | google-java-format 自己控制花括号位置 |
| `WhitespaceAround` | google-java-format 自己控制空格 |
| `ImportOrder` | Spotless `importOrder` 更灵活 |
| `FileTabCharacter` | google-java-format 只用空格 |

以下 PMD codestyle 规则也应该考虑关闭：

| PMD 规则 | 原因 |
|----------|------|
| `UnnecessaryImport` | Spotless 已管 |
| `ControlStatementBraces` | google-java-format 已管 |

### 11.3 各工具的盲区

| 盲区 | 补充工具（如果需要） |
|------|-------------------|
| 包依赖循环检测 | **ArchUnit** |
| 测试覆盖率 | **JaCoCo** |
| 第三方依赖漏洞 | **OWASP Dependency Check** |
| 许可证头检查 | Spotless `licenseHeader` 步骤 |
| 非 Java 文件格式 | Spotless 支持 XML/POM/JSON/Markdown 等 |
| Cookie/敏感信息检测 | **detect-secrets** 或 **truffleHog** |

---

## 十二、团队推广策略

技术方案只占成功落地的 30%。剩下 70% 是人的工作。

### 12.1 推广前：争取认同

1. **先分享问题，再提方案**：在团队会议上展示最近 3 个月因格式和风格不一致导致的 Code Review 内耗实例
2. **让团队参与决策**：把工具选型对比（第二章的内容）分享给团队，让大家有参与感，而不是"TL 拍板了你们执行"
3. **指定 Owner**：至少有一个"代码规范负责人"负责配置维护和答疑，不要期望所有人自觉
4. **提前说明 escape hatch**：在宣布时就讲清楚"有些合理例外可以抑制"（第六章），消除"一刀切"的恐惧

### 12.2 推广中：降低摩擦

**第一周的沟通模板**（Slack/飞书/企业微信）：

> 大家好，这周我们开始启用代码规范自动化。
>
> **你现在需要做的事**：
> 1. 装 IDE 插件（文档第七章，2 分钟搞定）
> 2. 保存时自动格式化就 OK 了
>
> **你不需要担心的**：
> - 格式化是全自动的，不需要你手动做任何事
> - 前两周 CI 只提醒不拦截，不影响合并
> - 对某个规则有疑问？直接找 [Owner] 或者在这个 thread 里聊
>
> 文档：[链接]

**常见抵触应对**：

| 抵触说法 | 回应 |
|---------|------|
| "格式化让我代码变丑了" | 格式化风格是 Google 定的，10 万+ 项目在用。先适应两周，如果确实有具体规则不合理，我们讨论定制（不是关掉） |
| "提交变慢了" | 耗时操作全在 CI 异步跑，本地提交只有毫秒级 hook。如果你发现卡了，来找我排查 |
| "某个规则不合理" | 走定制流程。但需要有具体 case，不是"我觉得不好看" |
| "紧急情况被 CI 卡了" | `git push --no-verify` + Team Lead 批准 + 事后补修——这个是写好的应急流程 |

### 12.3 持续运营

- **月度趋势报告**：从 CI 日志中统计 violations 趋势（下降是好事）
- **季度规则审查**：去掉从未触发过的规则，新增针对近期故障的规则
- **定期轮值 Owner**：让不同团队成员轮流做"规范负责人"，避免知识单点

---

## 十三、故障排查与 FAQ

### 13.1 Spotless 相关问题

**Q1：`mvn spotless:apply` 格式化的结果和 IDE 保存时格式化的不一样？**

大概率是 IDE 插件里 google-java-format 的版本和 Maven 插件里引用的版本不一致。

排查：
```bash
# 查看 Maven 用的哪个版本
mvn dependency:resolve -DincludeArtifactIds=google-java-format | grep google-java-format

# 检查 IDE 插件版本
# IntelliJ: Settings → Plugins → google-java-format → 版本号
# VS Code: 检查 java.format.settings.url 指向的版本
```

**方案**：锁定一致版本。在 pom.xml 中显式声明 google-java-format 版本。如果不确定，以 **Maven 插件打印的结果为准**（那是 CI 用的）。

**Q2：CI 上 `ratchetFrom` 不生效，格式检查了全量文件？**

CI 的 checkout 是 shallow clone（`fetch-depth: 1`），没有 `origin/main` 的历史。修改 CI 配置：

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # 必须有完整历史
```

**Q3：Spotless 在 `apply` 和 `check` 之间反复横跳？**

某个文件 A 规则说"这里要有空格"，B 规则说"这里不能有空格"——format 完 A 报错，再 format 完 B 报错。

常见原因：`importOrder` 和 `googleJavaFormat` 冲突（比如 import 分组之间的空行数）。调整 importOrder 配置：

```xml
<importOrder>
  <order>java,javax,org,com,</order>
  <wildcardsLast>true</wildcardsLast>
</importOrder>
```

**Q4：`mvn spotless:apply` 报 `Unable to locate executable`？**

google-java-format 需要通过 Maven 下载。可能原因：
- 公司内网 Maven 代理没有同步该版本
- 版本号写错

先检查 Maven 本地仓库有没有这个 jar：`~/.m2/repository/com/google/googlejavaformat/`。如果没有，手动 `mvn dependency:resolve` 触发下载。

### 13.2 Checkstyle 相关问题

**Q5：Checkstyle 报告几千个 violations，无从下手？**

存量项目正常现象。步骤：
1. 先生成报告：`mvn checkstyle:check`
2. 按规则分组统计 violations，找 Top-10 最高频的
3. 对 Top-10 逐条评估：是接受规则（修代码）还是关闭规则（配 suppression）
4. 批量加 `suppressions.xml` 豁免已知的老文件
5. 新代码不豁免

**Q6：Checkstyle 找不到 `checkstyle.xml`？**

`configLocation` 是相对于项目根目录的路径。多模块 Maven 项目，每个子模块需要能访问到同一个文件。推荐做法：

```xml
<configLocation>${project.basedir}/../checkstyle.xml</configLocation>
```

或者用 Maven 属性统一管理：

```xml
<properties>
  <checkstyle.config.location>${maven.multiModuleProjectDirectory}/checkstyle.xml</checkstyle.config.location>
</properties>
```

**Q7：JavadocMethod 报 Lombok 生成的 getter/setter？**

参见 [第十章场景一](#场景一使用-lombok)，用 `SuppressionXpathFilter` 排除 `@Data`、`@Value` 注解的类。

### 13.3 PMD 相关问题

**Q8：PMD 扫描超过 2 分钟，太慢了？**

优化顺序：
1. 检查是否排除了 `target/`、`generated/` 等不需要扫描的目录
2. 减少规则类别：先只保留 `errorprone` + `bestpractices`，逐步加
3. 增加 PMD 的 JVM 内存：`MAVEN_OPTS="-Xmx1024m" mvn pmd:check`
4. 考虑在 CI 中使用 PMD 的增量分析模式

**Q9：`DataflowAnomalyAnalysis` 规则总是误报？**

这是 PMD 中已知的高误报规则。**建议直接在 pmd-rules.xml 中关闭**：

```xml
<rule ref="category/java/errorprone.xml">
    <exclude name="DataflowAnomalyAnalysis"/>
</rule>
```

**Q10：PMD 报 `CloseResource`，但我的代码在 finally 里关了？**

PMD 可能不识别某些非标准资源关闭模式。如果确认代码正确，可以加抑制：

```java
@SuppressWarnings("PMD.CloseResource")
public void readFile(String path) {
    // 资源在 helper 方法里正确关闭了
}
```

### 13.4 集成相关问题

**Q11：本地 pre-commit hook 通过，CI 还是失败了？**

常见原因：

| 可能原因 | 排查方法 |
|---------|---------|
| Java 版本不一致 | 本地 `java -version` vs CI 配置的 `java-version` |
| Maven 版本不一致 | `mvn --version` vs CI 中的 Maven 版本 |
| OS 换行符差异 | `git config core.autocrlf`。Windows 本地可能自动转换换行符 |
| pre-commit 跳过了 | 确认 `pre-commit install --hook-type pre-push` 执行过 |
| Spotless 未在 pre-push 中跑 | 检查 `.pre-commit-config.yaml` 中 `stages: [push]` |

**Q12：多模块 Maven 项目，检查跑了两次（父子模块各一次）？**

父 POM 中用 `<inherited>false</inherited>` 或子模块 `<skip>true</skip>`：

```xml
<!-- 只在父 POM 中执行 -->
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-checkstyle-plugin</artifactId>
  <inherited>false</inherited>
</plugin>
```

**Q13：新增了一条 PMD 规则，CI 没报任何新违规？**

可能原因：
1. Maven 缓存了旧的规则文件——执行 `mvn clean pmd:check`
2. 该规则被已存在的排除规则覆盖了
3. 代码中确实没有触发该规则的用例

验证：故意写一段违规代码，跑一次看是否被拦截。

**Q14：想把 CI 中的 Spotless/Checkstyle/PMD 报告可视化？**

- GitHub Actions + reviewdog（见 [9.2 节](#92-github-actions--进阶pr-内联评论)）
- GitLab 原生支持 `artifacts:reports:codequality`
- Jenkins 用 `junit` publisher 发布 XML 报告

### 13.5 应急程序

**Q15：CI 拦截了一个紧急热修复部署，怎么处理？**

应急流程（**必须有审计记录**）：

1. Developer 发起请求：说明哪个规则被拦截、为什么来不及修
2. Team Lead（或两位 Senior）批准
3. Developer 临时加抑制注释，附带 follow-up ticket（如 `TODO(PROJ-999): 热修复后修`）
4. 部署完成后，**同一周内**提交修复 PR，移除抑制

不要用 `git push --no-verify` 绕过这个流程——CI 层仍然会拦截。

**Q16：想从一个已有的 Checkstyle 配置迁移到这套方案？**

迁移步骤：
1. 对比现有 `checkstyle.xml` 和本方案的 `checkstyle.xml`，找出差异规则
2. 新增 Spotless 配置（`ratchetFrom` 确保不影响存量文件）
3. 逐步将 Checkstyle 规则替换为本方案版本，每次替换一个类别
4. 新增 PMD 配置，按五阶段渐进式开启
5. 最终移除旧配置

---

## 十四、落地清单

### 14.1 新项目路径 (~25 分钟)

| 步骤 | 操作 | 时间 | 验证方法 |
|------|------|:---:|------|
| ① | **配 pom.xml** — 加 Spotless、Checkstyle、PMD 三个 plugin | 5min | `mvn validate` 不报错 |
| ② | **写 checkstyle.xml** — 复制第四章的规则 | 2min | `mvn checkstyle:check` 能跑 |
| ③ | **写 pmd-rules.xml** — 复制第四章的规则 | 2min | `mvn pmd:check` 能跑 |
| ④ | **首次全量格式化** — `mvn spotless:apply` | 1-3min | `mvn spotless:check` 返回 BUILD SUCCESS |
| ⑤ | **配 pre-commit** — 配置 `.pre-commit-config.yaml` 并安装 | 3min | `pre-commit run --all-files` 无报错 |
| ⑥ | **配 CI** — `.github/workflows/lint.yml` | 3min | 推送后 CI 绿色 |
| ⑦ | **装 IDE 插件** — google-java-format + Checkstyle + PMD | 3min | 写一行格式错误代码，IDE 实时提示 |
| ⑧ | **配置 .git-blame-ignore-revs** — 预置文件 | 1min | 文件存在 |
| | **合计** | **~22min** | |

### 14.2 存量项目路径 (~3 个月)

| 阶段 | 时间 | 操作 | 验收标准 |
|------|:---:|------|------|
| **第一阶段** | 第 1 周 | Spotless + `ratchetFrom`，不开启 Checkstyle/PMD | 一周内无人投诉格式化打断工作 |
| **第二阶段** | 第 2-4 周 | 开启 Checkstyle，`failOnViolation=false`，修复 Top-10 违规 | violations 趋势下降 |
| **第三阶段** | 第 2 个月 | 开启 PMD `errorprone` + `bestpractices`，修复真实 bug | 至少发现并修复 3 个真实隐患 |
| **第四阶段** | 第 3 个月 | `failOnViolation=true`，CI 锁死，追加 PMD 规则类别 | 所有 PR 必须通过 CI 检查 |
| **第五阶段** | 团队准备好时 | 全量格式化 + `.git-blame-ignore-revs` | `git blame` 不受全量格式化 commit 干扰 |

---

## 十五、附录：速查手册

### 15.1 命令速查

```bash
# === 格式化 ===
mvn spotless:apply           # 一键格式化 + import 整理
mvn spotless:check           # 仅检查，不改代码

# === 检查 ===
mvn checkstyle:check         # 仅 Checkstyle
mvn pmd:check                # 仅 PMD
mvn verify                   # 完整检查（spotless:check → checkstyle:check → pmd:check）

# === 报告 ===
mvn checkstyle:checkstyle    # 生成 Checkstyle 报告（XML/HTML）
mvn pmd:pmd                  # 生成 PMD 报告（XML/HTML）

# === pre-commit ===
pre-commit run --all-files   # 全量运行所有 hook
pre-commit run spotless-check  # 只运行 Spotless hook
git push --no-verify         # 跳过 pre-push hook（紧急情况才用）
```

### 15.2 需要创建的文件清单

| 文件 | 作用 | 章节 |
|------|------|:---:|
| `pom.xml`（修改） | 添加 Spotless / Checkstyle / PMD 插件 | [第四章](#四maven-配置详解) |
| `checkstyle.xml` | Checkstyle 规则定义 | [4.2](#42-checkstylexml--checkstyle-规则配置) |
| `pmd-rules.xml` | PMD 规则定义 | [4.3](#43-pmd-rulesxml--pmd-规则配置) |
| `suppressions.xml` | Checkstyle 抑制配置 | [6.3](#63-checkstyle-抑制) |
| `.pre-commit-config.yaml` | Git hooks 配置 | [第八章](#八pre-commit--pre-push-集成) |
| `.github/workflows/lint.yml` | CI 流水线 | [第九章](#九ci-集成) |
| `.git-blame-ignore-revs` | 全量格式化 commit 忽略列表 | [5.2](#52-存量项目brownfield--五阶段渐进) |
| `.vscode/settings.json` | VS Code 格式化配置 | [7.2](#72-vs-code) |
| `.idea/codeStyles/Project.xml` | IntelliJ 团队共享代码风格 | [7.1](#71-intellij-idea) |

### 15.3 版本兼容矩阵

| 组件 | 推荐版本 | 最低 Java 版本 |
|------|---------|:---:|
| Spotless Maven Plugin | 2.44.3 | Java 8+ |
| google-java-format | 1.25.2 | Java 11+ |
| Maven Checkstyle Plugin | 3.6.0 | Java 8+ |
| Checkstyle | 10.21.1 | Java 8+ |
| Maven PMD Plugin | 3.26.0 | Java 8+ |
| PMD | 7.9.0 | Java 8+ |

> 版本会持续更新。建议定期检查是否有新版本——通常 minor 版本升级不会有破坏性变化，但规则可能会调整。
