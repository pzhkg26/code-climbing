# Git 工作流规范参考

## 分支策略 — GitHub Flow

保持简单，`main` 始终可部署，工作在特性分支上。

```
main ──────●──────────●──────────●
            \        /          /
feature/xxx ●───────          /
                              /
hotfix/xxx  ●────────────────
```

### 分支约定

| 分支 | 用途 | 说明 |
|------|------|------|
| `main` | 主分支 | 始终可部署、可发布 |
| `feature/<描述>` | 新功能开发 | 如 `feature/add-login`, `feature/user-avatar` |
| `fix/<描述>` | Bug 修复 | 如 `fix/null-pointer-exception` |
| `refactor/<描述>` | 重构 | 如 `refactor/extract-payment-service` |
| `docs/<描述>` | 文档改动 | 如 `docs/update-api-guide` |
| `chore/<描述>` | 杂项（依赖、配置等） | 如 `chore/upgrade-spring-boot` |

### 基本原则

- **分支前缀表明意图**：`feature/`、`fix/`、`refactor/`、`docs/`、`chore/`
- **分支名用 kebab-case**（小写字母 + 连字符），简明扼要
- **用完即删**：合并到 main 后删除特性分支，保持仓库整洁
- **不要在 main 上直接提交**，任何改动都通过分支 + 合并

> 参考：[GitHub Flow 官方文档](https://docs.github.com/en/get-started/using-github/github-flow)

---

## 提交流程

1. 从 main 创建特性分支
2. 在分支上开发，原子化提交
3. 推送到远程
4. 创建 Pull Request
5. Review 通过后合并到 main
6. 删除特性分支

单人仓库的简化流程：建分支 → 开发提交 → 合并回 main → 删分支。

---

## Commit 规范 — Conventional Commits

> 参考：[conventionalcommits.org](https://www.conventionalcommits.org/)（Angular 团队率先推广）

### 格式

```
<type>: <简短描述>

<可选详细说明>
```

### 类型

| 类型 | 适用场景 | 示例 |
|------|---------|------|
| `feat` | 新功能 | `feat: 添加用户登录接口` |
| `fix` | 修复 | `fix: 修复文件上传路径编码问题` |
| `chore` | 构建、依赖、配置 | `chore: 升级 Spring Boot 至 3.2.0` |
| `docs` | 文档 | `docs: 补充 API 接口文档` |
| `refactor` | 重构 | `refactor: 拆分 PaymentService` |
| `test` | 测试 | `test: 增加 Kafka 消费端单元测试` |
| `style` | 代码格式（不影响逻辑） | `style: 格式化代码，统一缩进` |
| `perf` | 性能优化 | `perf: 缓存热点数据查询结果` |

### 原则

- 首字母 **小写**，句末 **不加句号**
- 描述使用**祈使句**（"添加"、"修复"、"拆分"），不要用过去式（"添加了"、"修复了"）
- 简短描述控制在 50 字符以内，必要时换行写详细说明

```bash
# 好的示例
feat: 添加 OAuth2 登录支持
fix: 修复并发场景下用户余额扣减异常
refactor: 抽取公共校验逻辑到 Validator 类

# 不推荐的示例
feat: 添加了 OAuth2 登录支持。（句尾句号，且用了过去式）
fix: bug fix。（描述太模糊，看不出改了什么）
```

---

## 合并策略

### 推荐：Squash Merge

特性分支上的多次提交合并为一次，保持 main 历史线性清晰。

```
feature/add-login 的提交记录：
  feat: 添加登录接口
  fix: 修复参数校验
  style: 格式化代码
  fix: 修改错误提示文案

合并到 main 后变为：
  feat: 添加登录接口（feature/add-login）   ← 一条提交
```

**适合场景**：特性分支上的"修复手误"、"临时调试"等中间提交，其他人不需要看到。

### 备选：Merge Commit

保留完整的开发历史，适合多人协作的大特性。

```
main ───────●─────────────────●
             \               /
feature       ● → ● → ● → ●
```

**适合场景**：需要完整追溯开发过程，或者多人协作同一个特性分支。

### 原则

- 同一项目统一用一种策略，不要混用
- 如果 squash 合并，commit 描述写清楚功能，不要写"squash merge"

> 参考：[GitHub 合并策略文档](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/about-merge-methods-on-github)

---

## 常用命令速查

```bash
# 创建分支并切换
git checkout -b feature/add-login

# 原子化提交
git add src/com/example/LoginService.java
git commit -m "feat: 添加登录接口"

# 推送到远程
git push origin feature/add-login

# 同步 main 的最新改动到当前分支
git fetch origin
git rebase origin/main       # 推荐，历史线性
# 或
git merge origin/main        # 保留 merge commit

# 切换到 main 并拉取最新
git checkout main
git pull origin main

# 删除本地和远程分支
git branch -d feature/add-login
git push origin --delete feature/add-login
```

---

## Commit 颗粒度原则

**原子化提交**：一个提交只做一件事。

| ✅ 好的拆分 | ❌ 差的拆分 |
|------------|------------|
| `提交 A`：添加订单查询接口 | `提交 X`：「开发了订单模块」—— 一个提交包含了接口、数据库、前端改动 |
| `提交 B`：添加订单数据库表 | |

**如何判断粒度合适？**

一个提交应该包含：

- 如果代码回退这个提交，不会留下半成品状态
- Commit message 能用一句话说清楚做了什么

---

## 需避免的实践

| 实践 | 为什么避免 | 更好的做法 |
|------|-----------|-----------|
| 直接提交到 main | 无法 review，容易误推 | 建特性分支 |
| 大提交（几百个文件） | 无法 review，回退困难 | 原子化提交 |
| 模糊描述（"update"、"fix bug"） | 无法追溯变更意图 | 用 Conventional Commits |
| 提交中混入无关改动 | review 困难，回退连带 | 用 `git add -p` 分段暂存 |
| 把 .idea / target / node_modules 提交 | 污染仓库，不同机器冲突 | 配置好 `.gitignore` |
| 用 `git reset` 丢失了未推送的提交 | 不可恢复 | 用 `git revert` 替代 |

---

## 参考资料

| 资源 | 链接 |
|------|------|
| GitHub Flow | https://docs.github.com/en/get-started/using-github/github-flow |
| Conventional Commits | https://www.conventionalcommits.org/ |
| GitHub 合并策略 | https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/about-merge-methods-on-github |
| Pro Git  Book（官方教材） | https://git-scm.com/book |
| Git 命令文档 | https://git-scm.com/docs |
| Git - 如何写好 Commit Message | https://cbea.ms/git-commit/ |
