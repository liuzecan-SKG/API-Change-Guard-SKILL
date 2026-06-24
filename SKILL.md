---
name: api-change-guard
description: 分析 Java Spring 后端代码变更的影响范围、回归测试范围与 API 兼容性风险。支持未提交变更、功能分支合并前相对 master 的累计 diff、最近 N 个提交，或当前作者本人的提交。当评审涉及 Controller、DTO、Mapper、Convert、Feign、Logic、校验或 Swagger 注解，或数据上报协议的 Git diff 时使用。
---

# API Change Guard

本 skill 用于在提测、评审或客户端联调之前，分析后端代码变更的影响范围。它的主要目标是判断新需求、bug 修复、字段变更、方法逻辑变更或新增方法是否会影响已上线功能，以及需要回归测试哪些范围。代码质量评审是次要内容。

## 调用方式

任意 AI Agent 都可以通过阅读本文件并遵循下面的工作流程来使用本 skill。

建议的提示语：

- `按照 API Change Guard 的 SKILL.md，分析当前 git diff`
- `按照 API Change Guard 的 SKILL.md，分析当前分支相对 master 的累计变更`
- `按照 API Change Guard 的 SKILL.md，分析最近 3 个提交`
- `按照 API Change Guard 的 SKILL.md，只分析我本人在当前分支的提交`

命令：

- `/api-change-guard analyze diff` —— 未提交变更（默认）
- `/api-change-guard analyze branch` —— 分支相对 base 的累计变更，用于合并前评审
- `/api-change-guard analyze recent <N>` —— 最近 N 个提交
- `/api-change-guard analyze mine` —— 当前作者在本分支的提交（commit 口径）
- `/api-change-guard analyze controller <ControllerPath>`
- `/api-change-guard analyze endpoint <ControllerPath> <ApiPath>`
- `/api-change-guard generate-test <ControllerPath> <ApiPath>`

## 工作流程

1. 确定分析模式（默认 `diff`）或命令类型 / 用户意图。
2. 按所选模式收集 Git 证据（见 `## 分析模式`）。同时收集报告元数据：`git rev-parse --short HEAD` 和当前时间戳。
3. 排除 `src/test/**`、`*Test.java` 和仅测试的 diff。
4. 按下面的优先级和阈值规则对变更的 Java 文件分组。
5. 按大 diff 上限和 API 关键词门槛，读取相关的 Controller / DTO / Mapper / Convert / Feign / Logic 文件。
6. 直接以 Git diff 和文件内容作为静态证据分析影响。
7. 在 `tools/api-change-guard/reports/` 下手动生成 Markdown 报告，命名风格：`api-change-guard-<mode>-<shortSha>-<timestamp>.md`。
8. 先回复生成的报告链接，再在其下粘贴完整报告正文。
9. 如需更深入分析，读取 `tools/api-change-guard/prompts/` 下对应的 prompt，并应用到收集到的 Git/文件证据上。
10. 用下面的稳定结构返回中文 Markdown 报告。

## 分析模式

四种 diff 收集模式。默认是 `diff`。每种模式对应一个命令。每种模式都仍然排除 `src/test/**`、`*Test.java` 和仅测试的 diff，并仍然遵循 `大 Diff 规则`、`报告结构（必备）` 和 `回复约定`。

在 Windows PowerShell 下，命令逐行执行；不要用 `&&` 连接。

### base 分支探测（branch / recent / mine 模式）

1. 若 `origin/master` 存在，用 `master`。
2. 否则若 `origin/main` 存在，用 `main`。
3. 否则用 `git symbolic-ref --short refs/remotes/origin/HEAD` 探测。
4. 否则让用户指定 base 分支。

一律先 `git fetch origin`，并一律对比 `origin/<base>`（不要用可能过期的本地分支）。

```bash
git fetch origin
git rev-parse --verify origin/master
# 不存在则尝试：git rev-parse --verify origin/main
# 仍不存在则：git symbolic-ref --short refs/remotes/origin/HEAD
```

### `diff` 模式（默认）—— 未提交变更

```bash
git diff -- '*.java'
git diff --cached -- '*.java'
git diff --name-only -- '*.java'
git diff --cached --name-only -- '*.java'
```

### `branch` 模式 —— 分支相对 base 的累计变更（合并前评审）

diff 用三点 `...`（自 merge-base 以来本分支引入的改动），log 用两点 `..`：

```bash
git fetch origin
git diff origin/master...HEAD -- '*.java'
git diff --name-only origin/master...HEAD -- '*.java'
git log origin/master..HEAD --oneline --no-merges
```

### `recent <N>` 模式 —— 最近 N 个提交

```bash
git diff HEAD~<N>...HEAD -- '*.java'
git diff --name-only HEAD~<N>...HEAD -- '*.java'
git log HEAD~<N>..HEAD --oneline
```

### `mine` 模式 —— 当前作者在本分支的提交（仅 commit 口径）

按作者切分累计 diff 在理论上无法逐行精确实现（补丁不可交换；重叠编辑和删除无法干净拆分）。请使用 commit 口径；绝不要编造一份「本人净改动 diff」。

```bash
git fetch origin
git config user.email
git log -p --no-merges --reverse --author="<email>" origin/master..HEAD -- '*.java'
git log --no-merges --author="<email>" origin/master..HEAD --name-only --pretty=format: -- '*.java'
```

使用 `mine` 时，报告必须声明以下限制（用中文）：

- 这是 commit 口径过滤，不是“本人净改动 diff”；多人改同一文件或同一行时无法精确切分。
- commit 口径可能包含中间态，以及之后被他人覆盖的改动。
- 影响分析与回归判断仍以分支累计 diff（`branch` 模式）为权威基准，`mine` 仅用于缩小关注范围。

## 回复约定

最终回复必须同时包含报告链接和报告正文。不要只返回链接。

所有面向用户的输出必须使用中文，包括进度说明、报告正文、分析依据、推理摘要、待人工确认项和后续建议。除非是代码标识符、命令、文件路径、类名、方法名、注解或源码引用，否则不要输出英文的章节说明。

严格按以下顺序：

1. 第一行：`报告文件：[<filename>](<relative-path>)`
2. 空行
3. 完整的 Markdown 报告正文，以 `# API Change Guard 变更影响分析报告` 开头

不要用摘要代替粘贴报告。如果报告很长，仍要粘贴 `报告结构（必备）` 要求的章节；必要时只压缩较长的代码块或请求样例。

## 分析优先级

按以下顺序分析：

1. 影响范围：本次变更影响到的已有接口、已上线功能、客户端页面、后端链路、数据流、协议流。
2. 回归测试：需要重新测试的已上线功能和场景。
3. 兼容性风险：请求字段、返回结构、枚举/状态值、校验规则、持久化数据，以及老客户端兼容性。
4. 变更事实：新增或修改的接口、DTO 字段、方法、逻辑分支、Mapper/Convert/Feign 变化。
5. 待人工确认：无法从代码证明的调用方、配置、协议、历史数据和跨服务影响。
6. 次要代码质量提示：只提可能影响功能正确性的问题。不做完整代码评审。

## 大 Diff 规则

当变更的 Java 文件很多时，不要一次性分析整个 diff。采用「先分流、再限界、再抽样、最后明确未覆盖范围」。

### 文件优先级

按以下顺序分析：

1. Controller 文件
2. DTO / VO / BO / Request / Response 文件
3. Feign / Remote Client 文件
4. Mapper / Convert / Service / Logic 文件
5. 其他 Java 文件

始终排除：

- `src/test/**`
- `*Test.java`
- 仅测试的 diff

### 阈值

- `<= 10` 个 Java 文件：全量分析。
- `11-30` 个 Java 文件：全量分析 Controller、DTO、Feign 文件；Mapper/Logic 文件做摘要；其他文件仅列出。
- `> 30` 个 Java 文件：进入**分批扫描模式**（见 `### 大 diff 分批扫描`）。必须分批覆盖全部相关文件，**不得**因文件多就截断、把未读文件直接丢进未覆盖。

### 各类型读取上限

- Controller：最多 10 个文件
- DTO / VO / BO / Request / Response：最多 20 个文件
- Feign / Remote Client：最多 10 个文件
- Mapper / Convert / Service / Logic：最多 10 个文件
- 其他 Java 文件：默认不读取

### API 关键词门槛

只有当非 Controller 文件的 diff 命中以下任一项时，才深度分析：

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

### 大 diff 分批扫描（必须执行，不得截断）

`branch` 和 `recent` 的累计 diff 通常很大，容易触发上面的阈值。**绝不允许**因为"文件太多"就只分析一部分、把其余直接列入未覆盖。文件多时必须执行分批扫描，目标是全覆盖：

1. 估规模：`git diff --stat origin/<base>...HEAD`，拿到变更文件总数与完整列表。
2. 分批：把变更文件切成多批，每批控制在深度分析上限内（约 ≤ 20-30 个文件）。分批方式优先级：
   - 优先**按模块/包**（同一业务域的 Controller / DTO / Mapper / Logic 放一批，便于跨文件推理）；
   - 或**按 commit**（`git log origin/<base>..HEAD --oneline`，每批一个或几个 commit）。
3. 逐批分析：对每一批独立按 `分析优先级` 分析，产出该批的局部结论。
   - 若当前 Agent 支持子代理，**推荐每批派一个子代理并行分析**（上下文隔离、互不挤占）；
   - 不支持则顺序多轮，一批做完再做下一批。
4. 汇总：把各批结论合并成**一份**报告，按 `报告结构（必备）` 输出，跨批去重 / 合并影响范围、回归建议、兼容性风险。
5. 覆盖范围：`## 分析覆盖范围` 必须写明变更文件总数、批次数、每批覆盖的文件数，使覆盖率逼近 100%。`## 未覆盖风险` **只**用于真正无法解析的复杂类型 / 跨仓库调用方等，**不得**用来装"因体量没读的文件"。

### 必备「分析覆盖范围」小节

每份报告必须包含 `## 分析覆盖范围`，内容含：

- 分析模式与范围（mode、base 分支、commit 范围、实际使用的 diff 命令）
- Java 变更文件数
- 已展开分析
- 摘要分析
- 未展开分析
- 已排除测试文件

### 必备「未覆盖风险」小节

如果有文件未被深度分析，必须包含 `## 未覆盖风险`，并列出可能仍含接口影响的文件。同时提醒评审者确认返回结构变化、DTO 转换遗漏、Feign 调用方变化和隐藏的逻辑分支变化。

## 报告结构（必备）

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

## 分析规则

- 把 Git diff 和文件内容当作事实依据。
- 评审范围排除测试源码：忽略 `src/test/**`、`*Test.java` 和仅测试的 diff。
- 区分事实、推测和待人工确认项。
- 不要编造调用方。影响范围必须给出依据和概率。
- 重点提示新增必填字段、删除字段、类型变化、校验变化、枚举变化和返回结构变化。
- 对无法解析的复杂类型，输出待人工确认项，而不是静默忽略。
- 不要把注释掉的 Controller 代码当作真实接口。
- 以后端可发布性为先：区分阻塞项、必测项和非阻塞确认项。
- 对 `DataCollectDto<T>`，同时分析外层采集契约和内层 DTO。指出外层必填字段、`data` 空列表行为和嵌套协议字段。
- 对手表/4G/蓝牙上报 DTO，在有证据时指出 `@Watch4gDataLength` 的字段顺序、长度、count 字段和端侧协议兼容性。
- 生成的 MockMvc 代码只是骨架。提示开发补充 service mock 和项目特定的错误码断言。

## 公司后端规范检查

在有证据时检查以下项：

- 接口返回统一的 `CommonResult<T>` 结构。
- 必要时请求参数带校验注解。
- 存在 Swagger 注解：`@Operation`、`@Schema` 或相关 OpenAPI 注解。
- 当请求字段变为必填时，提示兼容性风险。
- 对 APP、小程序、管理后台、后端服务和设备上报链路评估客户端影响。
