# parallel-subagents

一个 AI agent skill（通用 SKILL.md 格式，任何支持该格式的 agent 均可使用）：把用户给定的多任务清单并行派发给最多 N 个子 Agent，每个子 Agent 独占 `.subagents/` 下一个工作副本，通过分支 + push 交付，merge 由人工（或用户明确授权的主 Agent）执行。

- **N 是并行上限**（防止开过多任务拖死机器），不是任务数；实际每波派发 `min(N, T)` 个。
- **仅显式触发**：输入 `/parallel-subagents`，或明确说"并行派发子 agent"等指令。不会自动触发。
- **只消费清单**：skill 不负责把大目标拆成任务；任务拆分请在触发前自行完成或另行处理。

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

## 执行流程

1. **前置硬检查**（任一不过即停下询问，不自动降级）：
   - 是 git 仓库，且有可推送的 remote，且有 push 权限；
   - 主工作区干净（有未提交改动会问：先提交 / 暂存 / 照旧）；
   - `.gitignore` 忽略 `.subagents/`（缺则自动补）。
2. **准备工作副本**：复用 `.subagents/` 下空闲目录，不足则从当前项目 clone（排除 `.subagents/` 与 `.git/`）；恰好占用 `min(N, T)` 个。
3. **分波派发**：每波最多 N 个子 Agent 后台并行；每个子 Agent 独占一个目录，从主仓库当前分支 HEAD 拉出 `subagent/<任务slug>-<短id>` 分支，完成后只提交并 push 自己的分支，禁止碰主分支、禁止 merge。
4. **释放判据**（三条全满足才释放目录）：
   - 子 Agent 进程已结束；
   - `git ls-remote` 核实远端分支存在；
   - 该目录无未提交改动。
   任一不满足 → 标记异常并报告，不静默释放。
5. **波次闸门**：本波分支全部 push 完成即开下一波（**不等 merge**，merge 由你穿插进行）。
6. **波间刷新**：已释放的空闲目录 `git fetch` 后切回主分支并硬同步到最新，复用不重建。

## 交付与合并

- 交付 = 已 push 的分支。**skill 从不自动 merge**。
- merge 由你手动执行，或你在当轮明确授权后由主 Agent 执行；遇冲突即停并报告，不强行解决。
- 分支确认已合并后，**不自动删除**，会问一次是否删除本地/远端分支。

## 安全约束

- 不擅自删除、覆盖或清理任何未确认的目录、改动、分支。
- 不创建超出本波需要的活跃工作目录。
- 目录占用状态（占用 / 空闲 / 异常）在会话内可追踪。

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
