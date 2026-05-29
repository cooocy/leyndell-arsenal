# leyndell-arsenal

个人代码规范与协作约定的「军械库」。集中存放跨项目通用的开发规范，供日常开发和 AI 协作（Claude Code 等）参考。

## 约定文档

| 文档 | 说明 |
| --- | --- |
| [java_convention.md](./java_convention.md) | Java 项目通用规范：DDD 分层、包结构、Entity / Repository / AppService / Controller 写法、错误处理、Liquibase、常用库选型等 |
| [frontend_convention.md](./frontend_convention.md) | 前端项目通用规范：协作流程约定 |

## 核心协作流程

> 两份规范开头都强调的最重要一条：

- **不要自动执行打包、运行命令**：`gradle` / `mvn` / `java`、`pnpm build` / `dev` / `vite` / `tsc` / `lint` 等一律不跑。
- 改完代码直接说明变更点，编译、类型检查、构建、启动由人工执行。
- 确需启动项目验证时，先告知，由人工决定是否启动。
- 允许的操作：读代码、改代码。

## 使用方式

将本仓库作为规范的单一事实来源（single source of truth）：

- 新项目开工前先通读对应技术栈的约定文档。
- 在项目里通过 `CLAUDE.md` 或类似入口引用这里的规范，保持团队/AI 行为一致。
- 规范有更新时改这里，下游引用自动同步认知。

## 维护

规范随实践演进。新增或调整约定时，直接编辑对应的 `*_convention.md`，并在上表登记新文档。
