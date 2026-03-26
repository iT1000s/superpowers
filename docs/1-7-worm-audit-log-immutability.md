# Story 1.7: WORM 审计日志不可篡改性验证

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **开发者**,
I want **验证 better-sqlite3 + HMAC-SHA256 链式 MAC 的 WORM 日志机制**,
So that **确认 Triage 审计日志的防篡改能力满足安全需求**.

## Acceptance Criteria

1. **Given** 创建 `audit_triage_events` 表并写入 10 条模拟审计记录
   **When** 每条记录使用前一条记录的 MAC 值作为链式输入计算 HMAC-SHA256
   **Then** 任意修改某条记录后，验证链完整性检查失败

2. **Given** 已存在有效的审计链
   **When** 删除任意一条审计记录
   **Then** 验证链完整性检查失败

3. **Given** 已存在有效的审计链
   **When** 正常追加新记录
   **Then** 验证链完整性检查通过

4. **Given** 审计写入逻辑已实现
   **When** 执行批量写入性能验证
   **Then** 记录写入性能 > 1000 条/秒

## Tasks / Subtasks

- [ ] Task 1: 建立本 Story 所需的最小 SQLite / WORM 基础设施 (AC: #1, #2, #3, #4)
  - [ ] 1.1 新建 `src/db/connection.ts`，封装 `better-sqlite3` 连接初始化；支持显式传入 `dbPath`，默认落到 `~/.lichun/db/lichun.db`
  - [ ] 1.2 新建 `src/db/migrations/001-init-schema.sql`，创建 `audit_triage_events` 表，以及本 Story 需要的最小元数据表（如 `schema_migrations`、链头快照表）
  - [ ] 1.3 新建 `src/db/migrations/migrator.ts`，实现启动自检和最小 schema version 记录；范围只覆盖 WORM POC，不提前铺开完整业务表
  - [ ] 1.4 更新 `src/db/index.ts`，导出连接、迁移和审计存储公共 API，避免继续保留空壳 `export {}`

- [ ] Task 2: 实现链式 HMAC-SHA256 审计存储 (AC: #1, #3, #4)
  - [ ] 2.1 新建 `src/db/audit-store.ts`，实现 `AuditStore` 及最小公开方法：追加审计事件、读取链头、验证链完整性
  - [ ] 2.2 每条记录至少保存 `sequence_no`、稳定序列化后的事件载荷、`prev_mac`、`mac`、时间戳等字段，并以 `HMAC-SHA256(prevMac + canonicalPayload)` 生成链式 MAC
  - [ ] 2.3 HMAC 密钥通过构造参数或配置注入，测试使用固定 fixture key；禁止在源码中硬编码真实密钥
  - [ ] 2.4 写入路径使用 prepared statement + `db.transaction()` 或等效批量写入机制，避免逐条裸写导致性能不稳定

- [ ] Task 3: 实现完整性校验并覆盖删除/篡改边界 (AC: #1, #2, #3)
  - [ ] 3.1 `verifyChainIntegrity()` 必须按 `sequence_no` 顺序重算整条链，校验 `prev_mac`、`mac` 和序号连续性
  - [ ] 3.2 设计删除检测时必须覆盖“删除最后一条记录”场景；不能只对当前剩余行重算链，否则会漏掉尾部截断
  - [ ] 3.3 通过独立的链头快照/校验元数据实现尾删检测，并在实现说明中明确该 POC 的信任边界：HMAC 密钥与链头快照属于可信输入
  - [ ] 3.4 正常追加新记录后，链头状态同步更新，`verifyChainIntegrity()` 返回通过

- [ ] Task 4: 补齐单元测试与性能验证 (AC: #1, #2, #3, #4)
  - [ ] 4.1 新建 `src/db/audit-store.test.ts`，使用真实 SQLite 临时目录验证写入 10 条模拟审计记录后链校验通过
  - [ ] 4.2 增加篡改用例：直接 UPDATE 某条记录内容或 MAC，校验失败
  - [ ] 4.3 增加删除用例：至少覆盖删除中间记录和删除最后一条记录，校验均失败
  - [ ] 4.4 增加正常追加用例：已有链上继续 append，新链校验通过
  - [ ] 4.5 增加热路径性能验证，确认批量写入吞吐 > 1000 条/秒；测试或脚本需避免把冷启动/建表耗时混入核心指标

- [ ] Task 5: 保持模块边界并为后续 Story 留出扩展位 (AC: #1, #2, #3)
  - [ ] 5.1 审计输入类型保持最小可复用，不提前实现完整 `triage-checker`、`intent-detector` 或 `alert-gateway`
  - [ ] 5.2 `src/db/` 仅依赖 Node 内建、`src/infra/` 和自身模块；不要反向导入 `src/agent/` 或侵入尚未落地的业务层
  - [ ] 5.3 设计接口时兼容 Story 2.6 的真实 Triage 事件写入，但不在本 Story 中接入运行时 hook / agent 链路
  - [ ] 5.4 保持实现轻量，不引入 ORM、异步 DB 封装或新的存储引擎

- [ ] Task 6: 验证与记录 (AC: #1, #2, #3, #4)
  - [ ] 6.1 `pnpm build`
  - [ ] 6.2 `pnpm test`
  - [ ] 6.3 执行一次定向 WORM 性能验证，记录实际吞吐结果与测试环境
  - [ ] 6.4 在 Story 完成说明中记录：删除检测方案、吞吐结果、与 Story 2.6 / 4.1 的承接关系

## Dev Notes

### 当前代码与本 Story 的真实起点

- `src/db/index.ts` 目前仍是空占位文件，还没有连接管理、迁移器或 `audit-store`。
- `src/triage/index.ts` 和 `src/triage/triage.types.ts` 仅提供风险等级类型，没有真实 Triage 处理链路。
- `package.json` 已经包含 `better-sqlite3` 与 `@types/better-sqlite3`，本 Story 不需要再引入新的关系型数据库依赖。
- Story 1.6 已完成 OpenClaw 宿主契约对齐，本 Story 不应再改 `src/hooks/`、`index.ts` 或 Agent 生命周期接入逻辑。

### 必须遵循的架构约束

- **D1 关系型存储:** 必须使用 `better-sqlite3`，不要切换成 Prisma、Drizzle、sqlite3 async binding 或远程 DB。  
- **D4 数据迁移:** 采用“版本号 + 启动自检”的最小迁移机制，本 Story 只为 WORM POC 建最小闭环。  
- **D6 WORM 完整性:** 使用 `HMAC-SHA256` 链式 MAC；每次 append 都基于上一条记录 MAC 计算。  
- **NFR-S1 数据全本地:** 默认运行时数据放在 `~/.lichun/` 下，DB 文件应位于 `~/.lichun/db/lichun.db`。  
- **命名约束:** WORM 表必须以 `audit_` 为前缀，当前故事目标表就是 `audit_triage_events`。  
- **模块边界:** `src/triage/` 只允许导入 `src/infra/` 和 `src/db/`；`src/db/` 不要导入 `src/agent/`。  
- **技术基线:** TypeScript ESM、Node.js 22+、Vitest、Oxlint/Oxfmt，保持与当前仓库一致。  

### 关键实现防线

- 不要直接对 JS 对象做“隐式哈希”。必须先生成稳定的、可重算的 canonical payload，再参与 HMAC 计算；否则对象键顺序变化会导致伪篡改或误判。
- 不要把“重算当前表内所有记录都通过”误当成完整删除检测。若删除最后一条记录，朴素链重算会把更短的前缀当成合法链。
- 为了满足“删除任意记录都失败”，需要额外保存可信的链头快照/校验元数据，并在 `verifyChainIntegrity()` 中同时校验当前尾部与快照是否一致。
- HMAC 密钥必须由调用方提供，测试中使用固定测试密钥；不要写入数据库，不要硬编码到源码。
- 性能验证要走热路径：连接和迁移完成后，再测批量 append 吞吐；实现上优先使用 prepared statement + transaction。

### 推荐实现落点

| 文件/目录 | 建议动作 | 原因 |
| --- | --- | --- |
| `src/db/connection.ts` | 新建 | 封装 DB 路径解析、目录创建、SQLite 连接和基础 pragma |
| `src/db/migrations/001-init-schema.sql` | 新建 | 定义 `audit_triage_events` 与最小元数据表结构 |
| `src/db/migrations/migrator.ts` | 新建 | 实现最小 schema version 自检，避免把建表 SQL 散落在业务代码里 |
| `src/db/audit-store.ts` | 新建 | 集中封装 append / verify / benchmark 逻辑 |
| `src/db/audit-store.test.ts` | 新建 | 用真实 SQLite 临时目录覆盖篡改、删除、追加、性能 |
| `src/db/index.ts` | 修改 | 暴露 `AuditStore` 与 DB 公共入口，符合 Story 1.1 的导出约束 |

### 与后续 Story 的边界

- **对 Story 2.6:** 本 Story 只验证 WORM 机制本身，不接真实 `triage-checker` 运行时，不负责告警推送或消息上下文打捞。
- **对 Story 4.1:** 这里允许建立最小 `connection + migrator` 骨架，但不要提前扩展成全项目通用数据层框架；4.1 再做统一化和多表迁移治理。
- **对当前 Epic 1:** 这是技术 POC，目标是证明机制可行并产出可复用的 `audit-store`，不是一次性 demo 脚本。

### 可复用资产与前序 Story 学习成果

**来自 Story 1.1:**

- 所有 `index.ts` 都必须保留有效导出；空壳导出在本 Story 结束后不应继续存在于 `src/db/index.ts`。
- 模块边界已明确：`src/triage/` 未来只应通过 `src/db/` 调用审计能力。

**来自 Story 1.5:**

- 现有代码习惯使用 `performance.now()` 记录耗时、`randomUUID()` 生成 `traceId`，可直接复用到 `AuditStore`。
- 真实资源型测试使用临时目录并在 `afterEach` 清理，这一模式可直接参考 `src/memory/vector-store.test.ts`。
- 近期 review 强调类型安全和资源释放，不能用 `any` 或跳过关闭连接来换取“测试通过”。

**来自 Story 1.6:**

- OpenClaw hook 契约已经收口，本 Story 应只落在 `src/db/` 和相关测试，不要回头修改 SDK/hook 对齐成果。
- 当前仓库已经将本地 OpenClaw 开发依赖切到 `/Users/sunny/Documents/github/openclaw`，本 Story 不需要再处理 SDK 层变更。

### Git Intelligence

- `1652810 Align planning docs with latest OpenClaw host contract`  
  说明规划文档已经与最新宿主契约对齐；1.7 应聚焦数据层 POC，而不是再做消息入口调整。
- `4aa0ce7 review(1-5): 修复 6 项代码审查发现 — batch推理/退出崩溃/类型扩展`  
  提醒本 Story 要把批量路径、资源关闭和类型约束当成显式验收点。
- `b417fa3 Memory: implement BGE-small-zh local fallback provider`  
  说明当前仓库已经接受“真实依赖 + 真实临时资源测试”的实现策略，适合沿用到 SQLite/WORM 测试。

### Testing Standards

- 单元测试必须使用真实 SQLite 临时目录，而不是把 `better-sqlite3` 全量 mock 掉；否则无法验证 DELETE / UPDATE 篡改行为。
- 至少覆盖四类断言：初始 10 条链通过、单条篡改失败、单条删除失败、正常追加通过。
- 删除用例必须包含“删除最后一条记录”，这是最容易被漏掉的边界。
- 性能验证若放在 Vitest 中，应避免冷启动噪音；若单测过于脆弱，可增加定向 benchmark 辅助，但最终必须在 Story 完成记录里写明实际 rows/sec。
- `pnpm build` 与 `pnpm test` 必须全绿；本 Story 不要求触达 OpenClaw runtime 集成测试。

### Project Structure Notes

- 本 Story 的新增代码应局限在 `src/db/` 与必要的相邻测试文件，避免把 POC 实现散落到 `src/triage/`、`src/agent/` 或 `tests/integration/`。
- 如果需要审计输入类型，优先放在 `src/db/audit-store.ts` 邻近位置或 `src/db/` 内部类型文件，不要提前把未成熟模型塞进 `src/triage/triage.types.ts`。
- 若实现默认 DB 路径解析，注意 Node.js ESM 环境下不要引入 CommonJS-only 写法。

### References

- [Source: _bmad-output/planning-artifacts/epics.md#Story 1.7]
- [Source: _bmad-output/planning-artifacts/epics.md#Story 2.6]
- [Source: _bmad-output/planning-artifacts/architecture.md#Data Architecture]
- [Source: _bmad-output/planning-artifacts/architecture.md#Security Architecture]
- [Source: _bmad-output/planning-artifacts/architecture.md#Implementation Patterns & Consistency Rules]
- [Source: _bmad-output/planning-artifacts/architecture.md#Project Structure]
- [Source: _bmad-output/planning-artifacts/prd.md#2. 核心技术约束 (Technical Constraints)]
- [Source: _bmad-output/implementation-artifacts/1-5-bge-small-zh-local-lancedb-perf.md]
- [Source: _bmad-output/implementation-artifacts/1-6-openclaw-sdk-contract-alignment.md]
- [Source: src/db/index.ts]
- [Source: src/triage/index.ts]
- [Source: src/triage/triage.types.ts]
- [Source: src/infra/types.ts]
- [Source: package.json]

## Dev Agent Record

### Agent Model Used

GPT-5 Codex

### Debug Log References

- 2026-03-25: 按 BMAD create-story 流程重读 `epics.md`、`architecture.md`、`prd.md`、上一条 Story、当前代码结构与最近提交，生成 Story 1.7 实现上下文。

### Completion Notes List

- 已将 Story 1.6 状态从 `review` 更新为 `done`，并同步 `sprint-status.yaml`。
- 已创建 Story 1.7 的完整实现文档，并将冲刺状态更新为 `ready-for-dev`。
- 已显式补充“尾删检测不能只重算当前链”这一关键实现约束，避免开发时做出无法满足 AC 的朴素实现。

### File List

- `_bmad-output/implementation-artifacts/1-7-worm-audit-log-immutability.md`

## Change Log

- 2026-03-25: 创建 Story 1.7，实现上下文聚焦于 `better-sqlite3` + `HMAC-SHA256` 链式 WORM POC，并同步关闭 Story 1.6。
