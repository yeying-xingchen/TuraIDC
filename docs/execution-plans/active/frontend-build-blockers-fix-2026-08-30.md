---
status: 进行中
updated: 2026-08-30
owner: frontend-platform
---

# 前端构建阻断修复（2026-08-30）

## 背景

全量验证 `pnpm run verify:frontends` 与各端 `pnpm run build` 时发现构建/验证链路存在多处阻断：

| #   | 模块                                             | 级别 | 问题                                                                                                                                                                                                                                                  |
| --- | ------------------------------------------------ | ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `frontend-user-v3-www`                           | P0   | `vite.config.js` 将 `element-plus` alias 硬编码为仓库根 `../node_modules/element-plus`，pnpm 严格 node_modules 布局下该路径不存在，`bootstrap.ts` 导入 `element-plus/es/...` 加载失败，构建必然失败                                                   |
| 2   | `frontend-user-v4-console`                       | P0   | 页面复用 `shared/user-v3/components/StatusTag.vue`（模板用 `<t-tag>`），`TDesignResolver({ esm: true })` 生成 `tdesign-vue-next/esm` 导入，但 `shared/package.json` 未声明 `tdesign-vue-next` 依赖，pnpm 严格模式下 Rollup 从 shared 目录向上解析失败 |
| 3   | `shared` / `frontend-user-v3-www`                | P1   | tsconfig 声明 `types: ["node"]`，但对应 `package.json` 均未声明 `@types/node`，`tsc`/`vue-tsc` 报 TS2688                                                                                                                                              |
| 4   | `shared` / `frontend-user-v3-www`                | P1   | `eslint.config.js` 导入 `@eslint/js`、`eslint-plugin-vue`、`globals`、`typescript-eslint`，但 `package.json` 均未声明这些依赖，lint 报 `ERR_MODULE_NOT_FOUND`                                                                                         |
| 5   | CI (`.github/workflows/ci.yml`)                  | P0   | 前端 job 使用 `npm ci`，但项目是 pnpm workspace（无 `package-lock.json`），CI 必然失败                                                                                                                                                                |
| 6   | `frontend-admin-v3` / `frontend-user-v4-console` | P2   | lint 存在 prettier 格式错误（admin 3 处、v4 8 处），`--max-warnings 0` 下 lint 失败                                                                                                                                                                   |

## 范围与验收

- [ ] `pnpm run build:user-v3-www` 成功。
- [ ] `pnpm run build:user-v4-console` 成功。
- [ ] `pnpm run build:admin-v3` 成功。
- [ ] `pnpm run typecheck:frontends` 全绿。
- [ ] `pnpm run lint:frontends` 全绿。
- [ ] `pnpm run docs:check` 通过（含新增执行计划登记）。
- [ ] CI 前端 job 改为 pnpm 命令，与项目 packageManager 对齐。

## 实施步骤

1. 新建分支 `fix/frontend-build-blockers`。
2. 修复 www `vite.config.js`：删除指向根 node_modules 的 `element-plus` alias，由应用自身 node_modules 解析。
3. 修复 v4-console 构建：`shared/package.json` 声明 `tdesign-vue-next` 依赖（与 v4-console 对齐 `^1.20.6`），重新生成 lock。
4. 补全 `shared` 与 `frontend-user-v3-www` 的 devDependencies：`@types/node` 及 eslint 相关依赖。
5. 修复 CI：前端 job 改用 pnpm 安装与验证命令。
6. 修复 prettier 格式错误（admin / v4），执行 `--fix`。
7. 全量验证并记录结果。
8. 展示改动摘要，用户确认后提交。

## 风险与回滚

- `shared` 新增 `tdesign-vue-next` 依赖后，admin（1.19.1）与 v4（1.20.6）可能各自解析版本；`StatusTag.vue` 仅 v4-console 使用，admin 不引用，实际影响可控。若出现重复版本，可对齐 shared 声明到最小兼容范围。
- `pnpm-lock.yaml` 会因新增依赖被更新，属预期改动。
- CI 修改不触碰后端 job，回滚只需还原 ci.yml。

## 进度

- [x] 全量验证复现各构建/验证阻断。
- [ ] 创建执行计划文档与分支。
- [ ] 逐项修复并验证。

## 决策日志

| 日期       | 决策                                                              | 原因                                                                                                                                  |
| ---------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-08-30 | `shared` 采用补充 `tdesign-vue-next` 依赖而非迁移 `StatusTag.vue` | FRONTEND.md 明确 v4-console 优先复用 `shared/user-v3` 的 `StatusTag` 等组件，移动组件破坏设计意图；补依赖最贴合现有架构               |
| 2026-08-30 | CI 前端 job 从 `npm ci` 改为 pnpm                                 | 仓库是 pnpm workspace（`pnpm-lock.yaml`、`pnpm-workspace.yaml`、根 scripts 均为 pnpm 命令），`npm ci` 无 `package-lock.json` 必然失败 |
