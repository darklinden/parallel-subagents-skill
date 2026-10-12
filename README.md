# parallel-subagents

一个 AI agent skill（通用 SKILL.md 格式，任何支持该格式的 agent 均可使用）：把用户给定的多任务清单并行派发给最多 N 个子 Agent，每个子 Agent 独占 `.subagents/` 下一个 clone（工作副本），通过分支 + push 交付，merge 由人工（或用户明确授权的主 Agent）执行。

- **N 是并行上限**（防止开过多任务拖死机器），不是任务数；实际每波派发 `min(N, T)` 个。
- **仅显式触发**：输入 `/parallel-subagents`，或明确说"并行派发子 agent"等指令。不会自动触发。
- **只消费清单**：skill 只消费你提供的任务清单；任务拆分请在触发前自行完成或另行处理。

## 使用方式

```
/parallel-subagents
任务清单：
1. 修复登录页的表单校验
2. 给 API 层补充超时重试
3. 更新 README 的安装章节
N = 2
```

- 触发消息应包含：任务清单（每项一行）+ N。
- 缺 N 会追问一次；缺清单会追问一次。

## 行为规范

所有执行细节——前置检查、clone 复用、分支命名约定、释放判据、安全约束——**以 [`parallel-subagents/SKILL.md`](parallel-subagents/SKILL.md) 为准**；本文档不重复规则，避免同一规则在两处各自漂移。

分支名按该约定生成，形如 `subagent/fix-login-validation-3f9a2c`（示例）。

## 交付与合并

- 交付 = 已 push 的分支。**skill 从不自动 merge**。
- merge 由你手动执行，或你在当轮明确授权后由主 Agent 执行；遇冲突即停并报告，由你决定如何解决。
- 分支确认已合并后，**不自动删除**，会问一次是否删除本地/远端分支。

## 目录结构

```
parallel-subagents/
└── SKILL.md      # skill 本体（全英文，通用 SKILL.md 格式）
README.md         # 本说明
```

## 安装

通过 skills CLI 安装（推荐）：

```bash
npx skills@latest add parallel-subagents
```

或将 `parallel-subagents/` 目录手动拷贝到目标 agent 的 skills 目录，即可跨项目/跨工具复用。
