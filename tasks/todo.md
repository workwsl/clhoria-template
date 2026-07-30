## 修复 Skill 表声明示例与数据库注释同步

### 计划

- [x] 对照提交 `6c1219e` 与当前 Schema，找出所有仍使用 `pgTable(...)` 的 Skill 示例。
- [x] 复现并定位 `scripts/sync-comments.ts` 对 `snakeCase.table(...)` 的识别失败。
- [x] 将数据库 Schema/CRUD Skill 示例统一为 `snakeCase.table(...)`，同步修正导入与说明。
- [x] 让注释同步脚本同时识别当前 `snakeCase.table(...)` 与兼容的 `pgTable(...)` 调用。
- [x] 增加源码/运行时表覆盖校验，缺失声明时中止，避免生成清空注释的 SQL。
- [x] 添加回归测试，覆盖两种表工厂写法、无关调用与 fail-safe。
- [x] 运行注释同步 dry-run、Skill 校验、typecheck、lint 和测试。
- [x] 复核最终 diff，记录验证结果与残留风险。

### 计划自审

- 修改范围限于 Skill 示例、注释同步脚本及其回归测试，不改业务 Schema 或数据库。
- 所有数据库注释验证只使用 `--dry-run`，不会连接或修改数据库。
- 不修改无关部署配置。

### 结果回顾

- 根因：OXC 将 `snakeCase.table(...)` 解析为 `MemberExpression`，旧脚本只接受
  `pgTable(...)` 的 `Identifier`，会把全部 7 张表和 89 个字段视为无源码注释。
- 修复：更新 11 处 Skill 示例；同步器兼容两种工厂写法，并在源码扫描缺表时拒绝
  生成清空注释的 SQL。
- 验证：dry-run 识别 7 张表，生成 2 个表注释、77 个列注释和 17 个预期 NULL；
  `migrations/comments.sql` 无差异；typecheck、全量 lint、23 个测试文件共 658
  个测试全部通过。
- Skill 官方快速校验器因本机 Python 缺少 PyYAML 无法启动；已用仓库 YAML CLI
  验证 `db-schema` 与 `crud` frontmatter 语法。

---

## 同步 stoker 上游源码

### 计划

- [x] 确认 stoker 的集成方式、当前工作区状态与测试基线。
- [x] 拉取官方仓库并锁定最新提交、版本和源码清单。
- [x] 逐文件对比上游，区分上游变化与本项目中文化/响应结构/PostgreSQL 错误处理定制。
- [x] 同步必要源码与测试，并记录可追溯的上游版本。
- [x] 运行 stoker 聚焦测试、typecheck、lint 和完整测试。
- [x] 复核最终 diff，记录验证结果与残留风险。

### 计划自审

- 上游基线只采用 `w3cj/stoker` 官方仓库的 `main`，不引入第三方 fork。
- 保留本项目现有中文状态短语、`Resp.fail(...)` 响应契约和 PostgreSQL 错误映射。
- 不改动工作区中现有的 Skill、数据库注释同步、EAS 等无关变更。

### 结果回顾

- 官方 `main` 最新仍为 `v2.0.1`（`f623a4e13f579111d938fbc1dbd526f564abda5d`）；本项目运行时代码已经包含该版本逻辑，因此保留中文状态短语、统一响应结构、PostgreSQL 错误映射和 Zod v4 写法。
- 在 vendored 入口记录上游版本与提交，恢复上游对 OpenAPI error example 的两条严格回归测试。
- 原样上游测试会因遍历私有 `zodToOpenAPIRegistry._map` 而在当前类型定义下失败；改用公开 `get(...)` API 后，测试意图不变且 typecheck 通过。
- 验证：stoker 聚焦测试 `5 files / 22 tests`、全量测试 `23 files / 658 tests`、`pnpm typecheck`、`pnpm lint`、`git diff --check` 全部通过。
- 未改动现有 Skill、数据库注释同步等无关工作区变更。

---

## 修复依赖升级后的 Stoker 测试

### 计划

- [x] 对照依赖 diff 复现升级后的失败测试与类型检查。
- [x] 核对 `@asteasolutions/zod-to-openapi` v9 的 metadata/生成器公开 API 变化。
- [x] 将直依赖与 `@hono/zod-openapi` 的 v8 metadata 实现对齐，消除双注册表。
- [x] 复跑现有 Stoker 回归测试，确认无需掩盖运行时兼容问题。
- [x] 运行 Stoker 聚焦测试、typecheck、lint 和全量测试。
- [x] 复核依赖升级后的剩余风险并记录结果。

### 计划自审

- 保留其他依赖升级，只把与 Hono 当前实现不兼容的 `zod-to-openapi` 主版本恢复为 v8。
- 不通过改断言掩盖 `oneOf` 运行时生成空数组的问题。
- 不改动 Skill、数据库注释同步、部署配置等其他未提交内容。

### 结果回顾

- 根因：项目直依赖升级到 `@asteasolutions/zod-to-openapi@9.1.0`，但
  `@hono/zod-openapi@1.5.1` 仍使用 `8.5.0`；Hono 的 `.openapi(...)`
  元数据写入 v8 registry，Stoker 从 v9 registry 读取，导致 `oneOf`
  生成空数组且 example metadata 为 `undefined`。
- 修复：仅将直依赖恢复为兼容范围 `^8.5.0` 并重新生成 lockfile；
  依赖图中 Hono 与 Stoker 现已共用同一个 `8.5.0` 实例，其余升级保留。
- 验证：Stoker `5 files / 22 tests`、全量 `23 files / 658 tests`、
  `pnpm typecheck`、`pnpm lint`、`pnpm build` 全部通过。
- 残留：`@hono/node-ws@1.3.1` 声明的 node-server peer 范围仍是
  `^1.19.11`，而项目从升级前就已使用 node-server 2.x；当前代码未导入
  `@hono/node-ws`，因此未在本次 Stoker 修复中扩大范围处理。
