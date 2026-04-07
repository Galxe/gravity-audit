# GravityAIAudit

Gravity L1 安全审计问题统一追踪仓库。

所有审计轮次（内部 AI 审计、外部专业审计）的发现均在此集中管理，覆盖 `gravity-reth`、`gravity-sdk` 及相关合约。

---

## 目录

- [一、仓库结构](#一仓库结构)
- [二、Issue 层级规范](#二issue-层级规范)
- [三、Label 体系](#三label-体系)
- [四、认领与处理流程](#四认领与处理流程)
- [五、AI 自动化辅助](#五ai-自动化辅助)
- [六、代码子模块](#六代码子模块)

---

## 一、仓库结构

```
GravityAIAudit/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── audit-round.md        # Parent Issue 模板（审计轮次）
│   │   └── finding.md            # Sub-issue 模板（单条发现）
│   └── workflows/
│       └── auto-triage.yml       # AI 自动分拣 Workflow（进阶）
├── audits/
│   ├── 2026-02-23-gravity-reth/  # 每轮审计的原始报告存档
│   ├── 2026-02-28-gravity-sdk/
│   └── 2026-03-05-phase2/
├── gravity-reth/                  # git submodule
├── gravity-sdk/                   # git submodule
└── README.md
```

---

## 二、Issue 层级规范

### Parent Issue（审计轮次）

一轮审计对应一个 Parent Issue，用于承载宏观信息。

**命名格式：**
```
[AUDIT] <来源> <轮次> — <目标仓库> <版本/日期>
```

**示例：**
- `[AUDIT] AI Phase 1 — gravity-reth 2026-02-23`
- `[AUDIT] AI Phase 2 — gravity-reth + gravity-sdk 2026-03-05`
- `[AUDIT] Trail of Bits Round 1 — gravity-reth v1.0.0`

**必填字段（Issue body）：**
```
- **审计来源**: AI (Claude Opus 4.6) / Trail of Bits / OtterSec / ...
- **审计范围**: gravity-reth crates/... , gravity-sdk crates/...
- **代码版本**: commit hash 或 tag
- **报告链接**: 指向 audits/ 目录下的原始报告
- **统计**: CRITICAL x, HIGH x, MEDIUM x, LOW x, INFO x
```

### Sub-issue（单条发现）

每条审计发现对应一个 Sub-issue，挂载在对应的 Parent Issue 下。

**命名格式：**
```
[<REPO>-<ID>] [<SEVERITY>] <简短标题>
```

**示例：**
- `[GRETH-020] [CRITICAL] Pipeline permanent deadlock via barrier timeout gap`
- `[GSDK-017] [HIGH] Nested mutex holding in BlockBufferManager`

**必填字段见 Issue 模板。**

---

## 三、Label 体系

### 严重等级（自动从 Issue title 推断）

| Label | 颜色 | 含义 |
|-------|------|------|
| `severity: critical` | `#B60205` 深红 | 可导致链上状态分叉、资金损失、永久 DoS |
| `severity: high` | `#E11D48` 红 | 可被触发的严重安全问题 |
| `severity: medium` | `#F97316` 橙 | 防御深度问题，需计划修复 |
| `severity: low` | `#EAB308` 黄 | 低风险，建议改进 |
| `severity: info` | `#6B7280` 灰 | 信息性，无安全影响 |

### 目标仓库

| Label | 含义 |
|-------|------|
| `repo: gravity-reth` | 问题位于 gravity-reth |
| `repo: gravity-sdk` | 问题位于 gravity-sdk |
| `repo: contracts` | 问题位于链上合约 |
| `repo: cross` | 跨仓库问题 |

### 处理状态

| Label | 含义 |
|-------|------|
| `status: needs-triage` | 待确认（新建时默认） |
| `status: confirmed` | 已确认为真实问题 |
| `status: invalid` | 误报，经讨论确认无效 |
| `status: wont-fix` | 已知问题，决定不修复（需说明理由） |
| `status: in-progress` | 修复中 |
| `status: fixed` | 已修复，待验证 |
| `status: verified` | 修复已验证通过 |

### 审计来源

| Label | 含义 |
|-------|------|
| `source: ai-audit` | AI 自动化审计发现 |
| `source: manual-audit` | 人工专业审计发现 |
| `source: community` | 社区/漏洞赏金发现 |

### 开发者归属（按 git blame 自动分配）

按实际团队成员 GitHub handle 创建，例如：
- `owner: @alice`
- `owner: @bob`

---

## 四、认领与处理流程

### 4.1 新建流程（审计完成后）

```
1. 创建 Parent Issue（审计轮次）
2. 对每条发现，运行：
   git blame <file> -L <line_start>,<line_end>
   定位代码作者
3. 在 GravityAIAudit 创建 Sub-issue
   - 打上 severity: xxx / repo: xxx / source: xxx / owner: @xxx 标签
   - 在 Parent Issue body 中用 tasklist 列出所有 Sub-issue 编号
4. 通知相关开发者
```

### 4.2 修复流程（开发者视角）

1. 开发者筛选带有自己 `owner:` label 的 Issue
2. 在对应代码仓库（gravity-reth / gravity-sdk）创建修复分支
3. 提交 PR 时，在 PR description 中写：
   ```
   Fixes Galxe/GravityAIAudit#<issue_number>
   ```
4. PR 合并后，在 Issue 中 comment 修复说明并关闭

### 4.3 拒绝/无效流程

如果经讨论认为某条发现是误报或决定不修复：

1. 在 Issue 中添加 comment，说明理由
2. 将 label 改为 `status: invalid` 或 `status: wont-fix`
3. 由开发者本人或 Tech Lead 关闭 Issue

### 4.4 进度追踪

Parent Issue body 使用 GitHub Tasklist 格式：

```markdown
## Findings

### CRITICAL
- [ ] #12 [GRETH-020] Pipeline permanent deadlock
- [ ] #13 [GRETH-021] Cache eviction of unpersisted trie nodes

### HIGH
- [x] #14 [GRETH-022] BLOCKHASH opcode unimplemented ✅ fixed in gravity-reth#276
- [ ] #15 [GRETH-023] Token loss during epoch transitions

...
```

GitHub 会自动在 Parent Issue 上渲染进度条（已关闭 / 总数）。

---

## 五、AI 自动化辅助

### 5.1 当前可用（Claude Code）

```bash
# 在 GravityAIAudit 目录下，让 Claude Code 执行：

# 批量从审计报告创建 Issues
"读取 audits/2026-03-05-phase2/ 下的报告，
 为每条 HIGH+ 发现创建 Sub-issue，
 并用 git blame 定位代码作者打上 owner label"

# 每日进度汇总
"扫描所有 open issues，按严重等级和开发者归属统计进度，输出 markdown 表格"
```

### 5.2 进阶：GitHub Actions 自动化

`.github/workflows/auto-triage.yml` 中可配置：

- **新 Issue 自动分类**：读取 Issue title 解析 severity/repo，自动打 label
- **语义判定关闭**：Agent 读取 comment 语义（例如"confirmed false positive"），自动打 `status: invalid` 并关闭
- **智能 blame 分发**：Agent 解析 Issue body 中的文件路径和行号，自动运行 `git blame`（对 submodule 代码），打上对应开发者的 `owner:` label

---

## 六、代码子模块

被审计代码通过 submodule 引入，便于在 Issue 中直接对照：

```bash
# 初始化后执行（仓库已配置）
git submodule update --init --recursive

# 更新到最新
git submodule update --remote gravity-reth
git submodule update --remote gravity-sdk
```

| 子模块 | 仓库 | 说明 |
|--------|------|------|
| `gravity-reth/` | https://github.com/Galxe/gravity-reth | EVM 执行层 |
| `gravity-sdk/` | https://github.com/Galxe/gravity-sdk | AptosBFT 共识层 |

---

## 已完成审计轮次

| 日期 | 来源 | 范围 | Findings | Issue |
|------|------|------|----------|-------|
| 2026-02-23 | AI (Claude Opus 4.6) | gravity-reth | CRITICAL:0 HIGH:3 MED:8 LOW:8 | [#1](#) |
| 2026-02-28 | AI (Claude Opus 4.6) | gravity-sdk | CRITICAL:0 HIGH:2 MED:5 LOW:4 | [#2](#) |
| 2026-03-05 | AI Phase 2 (Multi-agent) | gravity-reth + gravity-sdk | CRITICAL:2 HIGH:16 MED:25 LOW:16 | [#3](#) |

---

*维护者：Galxe Engineering Team*
