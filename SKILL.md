---
name: api-change-guard
description: Analyze Java Spring backend code changes for impact scope, regression testing needs, and API compatibility risks. Works on uncommitted changes, a feature branch's cumulative diff vs master before merge, the last N commits, or the current author's commits. Use when reviewing Git diffs involving Controller, DTO, Mapper, Convert, Feign, Logic, validation or Swagger annotations, or data upload protocols.
---

# API Change Guard

Use this skill to analyze the impact scope of backend code changes before testing, review, or client synchronization. Its primary goal is to identify whether new requirements, bug fixes, field changes, method logic changes, or new methods affect existing online functionality and what regression testing is needed. Code quality review is secondary.

## How To Invoke

Any AI agent can use this skill by reading this file and following the workflow.

Suggested prompts:

- `按照 API Change Guard 的 SKILL.md，分析当前 git diff`
- `按照 API Change Guard 的 SKILL.md，分析当前分支相对 master 的累计变更`
- `按照 API Change Guard 的 SKILL.md，分析最近 3 个提交`
- `按照 API Change Guard 的 SKILL.md，只分析我本人在当前分支的提交`

Commands:

- `/api-change-guard analyze diff` — uncommitted changes (default)
- `/api-change-guard analyze branch` — branch cumulative changes vs base, for pre-merge review
- `/api-change-guard analyze recent <N>` — last N commits
- `/api-change-guard analyze mine` — current author's commits on this branch (commit scope)
- `/api-change-guard analyze controller <ControllerPath>`
- `/api-change-guard analyze endpoint <ControllerPath> <ApiPath>`
- `/api-change-guard generate-test <ControllerPath> <ApiPath>`

## Workflow

1. Determine the analysis mode (default `diff`) or command type / user intent.
2. Collect Git evidence according to the chosen mode (see `## Analysis Modes`). Also collect report metadata: `git rev-parse --short HEAD` and current timestamp.
3. Exclude `src/test/**`, `*Test.java`, and test-only diffs.
4. Group changed Java files by the priority and threshold rules below.
5. Read relevant Controller / DTO / Mapper / Convert / Feign / Logic files according to the large diff limits and API keyword gate.
6. Analyze impact directly using Git diff and file contents as static evidence.
7. Create a Markdown report manually under `tools/api-change-guard/reports/` using the naming style: `api-change-guard-<mode>-<shortSha>-<timestamp>.md`.
8. Reply with the generated report link first, then paste the complete report content below it.
9. If deeper analysis is needed, read the matching prompt in `tools/api-change-guard/prompts/` and apply it to the collected Git/file evidence.
10. Return a Chinese Markdown report using the stable structure below.

## Analysis Modes

Four diff-collection modes. Default is `diff`. Each mode maps to one command. Every mode still excludes `src/test/**`, `*Test.java`, and test-only diffs, and still follows `Large Diff Rules`, `Required Report Structure`, and `Response Contract`.

On Windows PowerShell, run commands one per line; do not join them with `&&`.

### Base Branch Resolution (branch / recent / mine modes)

1. Use `master` if `origin/master` exists.
2. Else use `main` if `origin/main` exists.
3. Else detect with `git symbolic-ref --short refs/remotes/origin/HEAD`.
4. Else ask the user to specify the base branch.

Always `git fetch origin` first, and always compare against `origin/<base>` (never the local branch, which may be stale).

```bash
git fetch origin
git rev-parse --verify origin/master
# if missing, try: git rev-parse --verify origin/main
# if still missing: git symbolic-ref --short refs/remotes/origin/HEAD
```

### Mode `diff` (default) — uncommitted changes

```bash
git diff -- '*.java'
git diff --cached -- '*.java'
git diff --name-only -- '*.java'
git diff --cached --name-only -- '*.java'
```

### Mode `branch` — branch cumulative changes vs base (pre-merge review)

Use three-dot `...` for diff (changes this branch introduced since the merge-base) and two-dot `..` for log:

```bash
git fetch origin
git diff origin/master...HEAD -- '*.java'
git diff --name-only origin/master...HEAD -- '*.java'
git log origin/master..HEAD --oneline --no-merges
```

### Mode `recent <N>` — last N commits

```bash
git diff HEAD~<N>...HEAD -- '*.java'
git diff --name-only HEAD~<N>...HEAD -- '*.java'
git log HEAD~<N>..HEAD --oneline
```

### Mode `mine` — current author's commits on this branch (commit scope only)

Per-author attribution of a cumulative diff is theoretically NOT solvable line-by-line (patches do not commute; overlapping edits and deletions cannot be split cleanly). Use commit scope; never fabricate a clean per-author net diff.

```bash
git fetch origin
git config user.email
git log -p --no-merges --reverse --author="<email>" origin/master..HEAD -- '*.java'
git log --no-merges --author="<email>" origin/master..HEAD --name-only --pretty=format: -- '*.java'
```

When using `mine`, the report MUST state these limits (in Chinese):

- 这是 commit 口径过滤，不是“本人净改动 diff”；多人改同一文件或同一行时无法精确切分。
- commit 口径可能包含中间态，以及之后被他人覆盖的改动。
- 影响分析与回归判断仍以分支累计 diff（`branch` 模式）为权威基准，`mine` 仅用于缩小关注范围。

## Response Contract

The final answer must include both the report link and the report body. Do not return only the link.

All user-facing output must be in Chinese, including progress updates, report content, analysis basis, reasoning summaries, manual confirmation notes, and follow-up suggestions. Do not output English section explanations unless they are code identifiers, commands, file paths, class names, method names, annotations, or quoted source text.

Use this exact order:

1. First line: `报告文件：[<filename>](<relative-path>)`
2. Blank line
3. Full Markdown report content, starting with `# API Change Guard 变更影响分析报告`

Never summarize the report instead of pasting it. If the report is very long, still paste the sections required by `Required Report Structure`; only compress long code blocks or request samples when necessary.

## Analysis Priority

Analyze in this order:

1. Impact scope: existing APIs, online features, client pages, backend flows, data flows, and protocol flows affected by the change.
2. Regression testing: already-online features and scenarios that need retesting.
3. Compatibility risk: request fields, response shape, enum/status values, validation rules, persisted data, and old client compatibility.
4. Change facts: added or changed APIs, DTO fields, methods, logic branches, Mapper/Convert/Feign changes.
5. Manual confirmations: callers, config, protocol, historical data, and cross-service effects that cannot be proven from code.
6. Secondary code quality hints: only mention issues that may affect functional correctness. Do not perform a full code review.

## Large Diff Rules

When Java changed files are many, do not analyze the entire diff in one pass. Use "先分流、再限界、再抽样、最后明确未覆盖范围".

### File Priority

Analyze in this order:

1. Controller files
2. DTO / VO / BO / Request / Response files
3. Feign / Remote Client files
4. Mapper / Convert / Service / Logic files
5. Other Java files

Always exclude:

- `src/test/**`
- `*Test.java`
- test-only diffs

### Thresholds

- `<= 10` Java files: full analysis.
- `11-30` Java files: fully analyze Controller, DTO, and Feign files; summarize Mapper/Logic files; list other files.
- `> 30` Java files: large diff mode. Only deep analyze API-related files; list the rest as uncovered.

### Per-Type Read Limits

- Controller: max 10 files
- DTO / VO / BO / Request / Response: max 20 files
- Feign / Remote Client: max 10 files
- Mapper / Convert / Service / Logic: max 10 files
- Other Java files: do not read by default

### API Keyword Gate

Only deep analyze a non-Controller file if its diff contains one of:

- `@RequestMapping`
- `@GetMapping`
- `@PostMapping`
- `@PutMapping`
- `@DeleteMapping`
- `@RequestBody`
- `@RequestParam`
- `@PathVariable`
- `@ParameterObject`
- `@Schema`
- `@NotNull`
- `@NotEmpty`
- `@NotBlank`
- `CommonResult`
- `DataCollectDto`
- `FeignClient`

### Branch / Recent Mode Sizing

`branch` and `recent` cumulative diffs are usually larger than a single uncommitted diff, so they hit the thresholds above more often. Do not silently truncate non-Controller changes that matter for a pre-merge review:

1. Size first: `git diff --stat origin/master...HEAD`.
2. If file count exceeds the deep-analysis limits, prefer batching over truncation: analyze by commit (`git log origin/master..HEAD`) or by module/package, so core Logic/Service changes are not skipped.
3. Record commit count and analysis batches in `## 分析覆盖范围`; list any skipped files under `## 未覆盖风险`.

### Required Coverage Section

Every report must include `## 分析覆盖范围` with:

- 分析模式与范围（mode、base 分支、commit 范围、实际使用的 diff 命令）
- Java 变更文件数
- 已展开分析
- 摘要分析
- 未展开分析
- 已排除测试文件

### Required Uncovered Risk Section

If any file is not deeply analyzed, include `## 未覆盖风险` and list files that may still contain API impact. Also ask reviewers to confirm response shape changes, DTO conversion omissions, Feign caller changes, and hidden logic branch changes.

## Required Report Structure

```markdown
## 结论先行
## 变更影响范围
### 已有功能影响
### 新增功能影响
### 端侧影响
### 后端链路影响
### 数据影响
## 回归测试建议
### 必须回归
### 建议回归
### 可不回归
## 兼容性风险
### 请求参数兼容性
### 返回结构兼容性
### 枚举/状态值兼容性
### 校验规则兼容性
### 数据兼容性
## 变更事实摘要
### 新增接口
### 修改接口
### 新增/修改 DTO 字段
### 新增/修改方法逻辑
### Mapper / Convert / Feign 变化
## 关键链路分析
### 调用入口
### 核心处理逻辑
### 数据转换链路
### 持久化/缓存/MQ/远程调用影响
## 测试点清单
### 正常场景
### 异常场景
### 边界场景
### 历史兼容场景
### 回归场景
## 分析覆盖范围
## 未覆盖风险
## JUnit / MockMvc 测试骨架
## Apifox / Postman 请求样例
## 待人工确认项
## 次要代码质量提示
```

## Analysis Rules

- Treat Git diff and file contents as facts.
- Exclude test sources from review scope: ignore `src/test/**`, `*Test.java`, and test-only diffs.
- Separate facts, estimates, and manual confirmations.
- Do not invent client callers. Impact range must include evidence and probability.
- Highlight added required fields, removed fields, type changes, validation changes, enum changes, and response structure changes.
- For unknown complex types, output a manual confirmation item instead of silently ignoring them.
- Do not treat commented-out Controller code as a real API endpoint.
- Put backend release readiness first: distinguish blocking items, must-test items, and non-blocking confirmations.
- For `DataCollectDto<T>`, analyze both the outer collection contract and the nested DTO. Call out required outer fields, `data` empty-list behavior, and nested protocol fields.
- For watch/4G/Bluetooth upload DTOs, call out `@Watch4gDataLength` field order, length, count-field, and client protocol compatibility when evidence is available.
- Generated MockMvc code is a skeleton. Tell developers to fill service mocks and project-specific error-code assertions.

## Company Backend Checks

Check these items when evidence is available:

- API returns the unified `CommonResult<T>` shape.
- Request parameters have validation annotations when required.
- Swagger annotations are present: `@Operation`, `@Schema`, or related OpenAPI annotations.
- Compatibility risk is called out when request fields become required.
- Client impact is estimated for APP, mini program, admin console, backend services, and device upload flows.
