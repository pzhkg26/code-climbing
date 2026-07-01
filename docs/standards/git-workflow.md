# Git 工作流规范

## 分支策略 — GitHub Flow

```
main ───●────●────●────●
         \        /
agent/xxx ●──────
```

- `main` — 主分支，始终保持可部署状态
- `agent/xxx` — 由 Agent 创建的工作分支，如 `agent/add-auth`、`agent/fix-timeout`

## 提交流程

1. Agent 从 main 创建 `agent/<description>` 分支
2. 在分支上开发并提交
3. 推送到 GitHub，等待审核
4. 审核通过后由用户合并到 main

## Commit 消息规范 — Conventional Commits

格式：

```
<type>: <简短描述>

<可选详细说明>
```

### 类型说明

| 类型 | 适用场景 |
|------|---------|
| `feat` | 新功能 |
| `fix` | 修复 |
| `chore` | 构建、依赖、配置等杂项 |
| `docs` | 文档 |
| `refactor` | 重构 |
| `test` | 测试 |
| `style` | 代码格式（不影响逻辑） |
| `perf` | 性能优化 |

### 示例

```
feat: 添加用户登录接口
fix: 修复文件上传路径编码问题
chore: 升级 Spring Boot 版本至 3.2.0
docs: 补充 API 接口文档
refactor: 拆分 PaymentService 为多个子服务
test: 增加 Kafka 消费端单元测试
```
