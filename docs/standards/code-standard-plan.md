# Java（Maven）代码规范落地方案

> 纯 Maven Java 项目，一整套从 IDE 到 CI 的完整方案

---

## 一、架构总览

| 概念 | Java 生态的对应工具 | 说明 |
|------|-------------------|------|
| **Orchestrator（编排器）** | **Spotless** | 统一入口，一个命令跑所有 |
| **Formatter（格式化）** | **google-java-format** | 自动排版：缩进、换行、空格 |
| **Linter（规范检查）** | **Checkstyle** | 命名规范、Javadoc、import 顺序等 |
| **Linter（代码质量）** | **PMD** | 潜在 bug、空指针、死代码 |
| **Import 整理** | **Spotless (removeUnusedImports + importOrder)** | 自动清理排序 |

```
mvn spotless:apply  ← 一键搞定所有
         │
    Spotless（编排）
      ├── google-java-format  → 格式化代码
      ├── importOrder         → import 排序
      ├── removeUnusedImports → 清理无用 import
      │
mvn spotless:check  ← 可以拆出来配 Checkstyle + PMD
                      （但推荐放 CI，不放 pre-commit）
```

---

## 二、Maven 配置

### 2.1 pom.xml — Spotless 核心配置

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
              <!-- 可选：AOSP 风格（4空格缩进，和 Google 有点区别） -->
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

---

### 2.2 Checkstyle 规则文件

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
    <!-- 命名规范 -->
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

    <!-- Javadoc 规范 -->
    <module name="JavadocMethod"/>
    <module name="JavadocType"/>
    <module name="JavadocStyle"/>

    <!-- 代码规范 -->
    <module name="EqualsHashCode"/>    <!-- 重写 equals 必须重写 hashCode -->
    <module name="IllegalInstantiation"/> <!-- 禁止实例化某些类 -->
    <module name="SimplifyBooleanExpression"/>
    <module name="SimplifyBooleanReturn"/>
    <module name="StringLiteralEquality"/> <!-- 字符串用 equals 不用 == -->
    <module name="UnusedImports"/>     <!-- 无用 import（Spotless 已管，可不要） -->
  </module>
</module>
```

---

### 2.3 PMD 规则文件

```xml
<!-- pmd-rules.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<ruleset name="Custom Rules"
  xmlns="http://pmd.sourceforge.net/ruleset/2.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0 https://pmd.sourceforge.net/ruleset_2_0_0.xsd">
  <description>项目代码质量规则</description>

  <!-- 最佳实践 -->
  <rule ref="category/java/bestpractices.xml">
    <exclude name="JUnitAssertionsShouldIncludeMessage"/>
    <exclude name="JUnitTestContainsTooManyAsserts"/>
  </rule>

  <!-- 潜在 bug -->
  <rule ref="category/java/errorprone.xml">
    <exclude name="BeanMembersShouldSerialize"/>
  </rule>

  <!-- 性能 -->
  <rule ref="category/java/performance.xml"/>

  <!-- 代码风格（PMD 管的和 Spotless 互补，不冲突） -->
  <rule ref="category/java/codestyle.xml/EmptyMethodInAbstractClassShouldBeAbstract"/>
  <rule ref="category/java/codestyle.xml/ExtendsObject"/>
  <rule ref="category/java/codestyle.xml/ForLoopShouldBeWhileLoop"/>
  <rule ref="category/java/codestyle.xml/UnnecessaryImport"/>
  <rule ref="category/java/codestyle.xml/UnnecessaryModifier"/>
  <rule ref="category/java/codestyle.xml/UnnecessaryReturn"/>
</ruleset>
```

---

## 三、本地使用

```bash
# === 一键格式化 + import 整理 ===
mvn spotless:apply

# === 检查（只报错不改） ===
mvn spotless:check

# === 如有 Checkstyle（手动跑） ===
mvn checkstyle:check

# === 如有 PMD（手动跑） ===
mvn pmd:check

# === 三合一检查（CI 标准） ===
mvn verify    # 会依次跑 spotless:check → checkstyle:check → pmd:check
```

---

## 四、Pre-commit 集成

```yaml
# .pre-commit-config.yaml
repos:
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
        stages: [push]  # 只在 git push 时跑，不提提交时卡住

  # 通用检查
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-added-large-files
        args: ['--maxkb=500']
```

> **重要**：Spotless 启动 JVM 需要几秒，放 `git push` hook 比 `git commit` hook 更合理，避免每次提交都等。本地开发靠 IDE 插件实时反馈。

---

## 五、CI 集成

### GitHub Actions

```yaml
# .github/workflows/lint.yml
name: Code Quality
on: [push, pull_request]

jobs:
  format:
    name: Format Check
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
    name: Lint Checkstyle + PMD
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

### GitLab CI / Jenkins / 通用

```bash
# 任何 CI 平台通用
mvn spotless:check && mvn verify
```

---

## 六、IDE 配置（最佳体验层）

### IntelliJ IDEA

| 工具 | 做法 | 效果 |
|------|------|------|
| **google-java-format 插件** | 安装插件 → Settings → 设为默认格式化工具 | 保存自动格式化 |
| **Save Actions 插件** | 配置：保存时自动优化 import + 格式化 | 边写边顺 |
| **Checkstyle 插件** | 安装 → 指向 checkstyle.xml | 实时红线提示 |

**一键搞定**：插件装完，写代码时**零延迟实时提示**，根本不用等到 pre-commit 或 CI。

---

## 七、完整落地清单

| 步骤 | 操作 | 时间 |
|------|------|------|
| ① | **配 pom.xml** — 加 spotless、checkstyle、pmd 三个 plugin | 5min |
| ② | **写 checkstyle.xml** — 复制上面的规则 | 2min |
| ③ | **写 pmd-rules.xml** — 复制上面的规则 | 2min |
| ④ | **首次全量修复** — `mvn spotless:apply` | 1~3min |
| ⑤ | **配 pre-commit** — `.pre-commit-config.yaml` | 2min |
| ⑥ | **配 CI** — `.github/workflows/lint.yml` | 3min |
| ⑦ | **装 IDE 插件** — google-java-format + Checkstyle | 2min |
| ⑧ | **首次提交** — 验证所有环节通过 | 5min |
| | **合计** | **~20分钟** |

---

## 八、各工具关系图解

```
你的 IDE（写代码时）
    │
    ├── IntelliJ google-java-format 插件 → 保存即格式化 ❮❮ 最快的反馈
    ├── IntelliJ Checkstyle 插件       → 边写边标红
    └── IntelliJ PMD 插件              → 边写边标红
    │
    ▼
你的 Git（提交时）
    │
    ├── pre-commit hook
    │   ├── pretty-format-java  → 格式化（轻量，毫秒级）
    │   └── trailing-whitespace / EOF fixer → 通用清理
    │   ⚠️ Spotless 太重放 CI，不放 pre-commit
    │
    ▼
你的 CI（推送时）
    │
    ├── mvn spotless:check      → 格式化 + import 检查
    ├── mvn checkstyle:check    → 命名 + Javadoc 规范
    └── mvn pmd:check           → 代码质量（空指针、死代码等）
    │
    ▼
    全部通过 → ✅ 合并 PR / 部署
    任何失败 → ❌ 拦截，开发者看 CI 日志修
```

---

## 九、常见问题

**Q：Spotless 和 Checkstyle 会不会打架？**
A：会。比如 google-java-format 改完换行，Checkstyle 可能报 `LineLength`。解法：**Checkstyle 只检查 formatter 管不到的规则**（命名、Javadoc、结构），不管缩进和行长。

**Q：PMD 和 Checkstyle 重复了？**
A：互补不重叠——Checkstyle 管**风格**（命名、Javadoc），PMD 管**质量**（空指针、未用变量、性能）。

**Q：ruff 还用吗？**
A：如果你项目里有 Python 脚本（测试脚本、工具脚本），可以配上 Ruff。纯 Java 项目不需要。要不要我加一段项目里如果有 Python 辅助脚本时的 Ruff 配置？
