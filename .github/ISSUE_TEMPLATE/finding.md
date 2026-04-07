---
name: "🔍 审计发现 (Sub-issue)"
about: 记录单条具体的审计发现
title: "[<REPO>-<ID>] [<SEVERITY>] <简短标题>"
labels: "status: needs-triage, source: ai-audit"
---

## 基本信息

- **Finding ID**: <!-- GRETH-020 / GSDK-017 / ... -->
- **严重等级**: <!-- CRITICAL / HIGH / MEDIUM / LOW / INFO -->
- **目标仓库**: <!-- gravity-reth / gravity-sdk / contracts -->
- **所属审计**: <!-- Parent Issue #<number> -->
- **代码作者**: <!-- @github_handle (via git blame) -->

## 问题描述

**文件**: `<!-- crates/xxx/src/xxx.rs:line_start-line_end -->`

<!-- 详细描述问题，包括代码片段 -->

```rust
// 相关代码
```

## 影响分析

<!-- 描述攻击场景和影响范围 -->

**可利用性**: <!-- 容易 / 中等 / 困难 -->
**影响范围**: <!-- 单节点 / 所有验证者 / 链上资金 / ... -->

## 复现步骤

<!-- 如果有 PoC，在此描述 -->

1.
2.

## 修复建议

<!-- 具体的修复方案 -->

## 参考

- 原始报告位置: <!-- audits/2026-03-05-phase2/ -->
- 相关 Finding: <!-- #<issue_number> (如有关联) -->
