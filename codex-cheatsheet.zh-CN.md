# Codex 一页速查表（映射 `.claude` 工作流）

## 1) 启动任务（对应 `/start`）
```text
请先做任务路由：
1. 判断项目状态（新任务/进行中/已有计划）
2. 评估规模（小<3文件，中3-10文件，大>10文件或架构改动）
3. 给出推荐路径（直接实现 / TASKS后实现 / 先计划）
```

## 2) 先计划后执行（对应 `/plan`）
```text
先输出计划，不要直接改代码。
格式：
- Goal
- Files
- Steps（每步<=20分钟）
- Risks
- Verification
```

## 3) 任务清单（对应 `task-management`）
在仓库根目录维护 `TASKS.md`：
- `[ ] Task` 待办
- `[ ] **-> Task**` 进行中
- `[x] ~~Task~~` 完成

执行规则：
1. 开工前先写任务
2. 任一时刻只允许 1 个进行中
3. 每完成一步就更新状态

## 4) 实现阶段硬约束（对应 `forbidden` + `pragmatism`）
```text
约束：
- 禁止 any / enum / 空 catch / console.log
- 禁止读取 .env
- 不做超出任务边界的重构
- 先最简单可工作方案
```

## 5) 验证门禁（对应 `/verify`）
按顺序运行：
1. `npm run typecheck`（或 `npx tsc --noEmit`）
2. `npm run lint`（或 `npx eslint .`）
3. `npm test`

规则：
- 任何失败先修复再重跑
- 三项全绿才可宣称完成

## 6) 提交前自审（对应 `/review`）
```text
请先做严格自审并列出问题（按严重级）：
- 行为回归
- 类型安全与异常处理
- 安全风险
- 可维护性
- 测试缺口
每条给出 file:line + 影响 + 修复建议
```

## 7) 提交规范（对应 `/commit`）
1. `git status` + `git diff`
2. 只 add 相关文件（禁止 `git add .` / `git add -A`）
3. Conventional Commit：
   - `feat:` `fix:` `refactor:` `docs:` `test:` `chore:`

## 8) 通用高质量提示词（复制即用）
```text
请按以下流程执行：任务路由 -> 计划 -> TASKS -> 实现 -> 验证 -> 自审 -> 提交建议。
必须遵守：
- 禁止 any/enum/空catch/console.log
- 禁止读取 .env
输出：
1) 计划（Goal/Files/Steps/Risks/Verification）
2) 变更文件列表
3) 验证结果（typecheck/lint/test）
4) 风险与后续建议
```

## 9) 零兼容策略（对应 `zero-compat`）
```text
不做兼容层，不保留历史包袱。
允许破坏旧格式，优先清理废弃代码。
```
