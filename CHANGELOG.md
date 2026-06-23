# Changelog

本文件记录 API Change Guard 的版本变更，版本遵循语义化版本（SemVer）。

## [1.1.0] - 2026-06-23

### 新增
- 新增 4 种分析范围模式，每种独立命令，`analyze diff` 仍为默认：
  - `analyze diff`：未提交变更（默认）
  - `analyze branch`：功能分支相对主分支的累计变更，用于合并前评估
  - `analyze recent <N>`：最近 N 个提交
  - `analyze mine`：仅本人在本分支的提交（commit 口径）
- base 分支探测：优先 `master`，兜底 `main`，再用 `git symbolic-ref` 探测或由用户指定。
- branch / recent / mine 模式强制先 `git fetch origin`，并一律对比 `origin/<base>`，避免本地分支过期。
- 大 Diff 规则新增「分支 / 最近模式分批」：先 `git diff --stat` 估规模，超限按 commit 或模块分批分析。

### 变更
- 报告「分析覆盖范围」需注明分析模式、base 分支、commit 范围、实际使用的 diff 命令。
- 报告文件命名调整为 `api-change-guard-<mode>-<shortSha>-<timestamp>.md`。

### 说明
- `analyze mine` 是 commit 口径过滤，不是「本人净改动 diff」；多人改同一文件或同一行无法精确切分，影响分析与回归判断仍以 `branch` 模式为权威基准。

## [1.0.0] - 2026-06-05

### 新增
- 基于 Git diff 的 Java Spring 后端接口变更影响分析，纯 Cursor Skill + Git 命令方案，不依赖 Python。
- 影响范围优先的固定报告结构、大 Diff 分流规则、测试源码排除、中文输出约束、多 Agent 通用化。
