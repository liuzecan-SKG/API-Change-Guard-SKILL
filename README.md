# **API Change Guard**

API Change Guard 用于分析 Java Spring 后端代码变更的影响范围，并生成面向后端、客户端和测试同学的稳定报告。它的主要目标是判断代码变更是否影响已上线功能，以及需要回归测试哪些范围。

## **推荐仓库结构**

如果将本工具作为独立仓库共享，建议保持以下结构：

```
api-change-guard/
  README.md
  SKILL.md
  prompts/
    analyze-api-change.md
    generate-test-cases.md
    generate-mockmvc.md
  examples/
    sample-api-change-report.md

```

其中：

- `SKILL.md` 是通用 Agent 版本，适合 Cursor、Claude Code、Codex CLI 或其他能读取文件的 AI Agent。
- `README.md` 是安装和使用说明。
- `prompts/` 是可选增强 Prompt。
- `examples/` 是示例报告。

## **支持的输入**

- 当前 Java Git diff：`/api-change-guard analyze diff`
- 指定 Controller 文件：`/api-change-guard analyze controller <ControllerPath>`
- 指定 Controller 接口：`/api-change-guard analyze endpoint <ControllerPath> <ApiPath>`

## **输出内容**

- 生成到 `tools/api-change-guard/reports/` 下的 Markdown 报告文件
- 已上线功能影响范围总结
- 回归测试范围
- API 和业务链路变更摘要
- 采集 DTO、手表上报 DTO 的字段和协议细节
- 对 APP、小程序、管理后台、后端服务、设备上报链路的影响范围推测
- 带证据的兼容性风险
- 边界场景和异常场景测试清单
- JUnit / MockMvc 测试骨架
- 可复制到 Apifox / Postman 的 curl 请求样例

## **安装和使用**

### **通用 Agent 使用方式**

适用于任意能读取项目文件、执行 Git 命令、创建 Markdown 文件的 AI Agent。

1. 将 `SKILL.md` 放到目标项目中，例如：

```
tools/api-change-guard/SKILL.md

```

1. 对 Agent 说：

```
请读取 tools/api-change-guard/SKILL.md，并按照 API Change Guard 流程分析当前 git diff。

```

或者：

```
按照 API Change Guard 的 SKILL.md，分析当前 Java 代码变更对已上线功能的影响范围。

```

### **Cursor 使用方式**

项目级安装：

```
.cursor/skills/api-change-guard/SKILL.md

```

可以直接复制通用版 `SKILL.md` 到上述路径。

然后在 Cursor 中输入：

```
/api-change-guard analyze diff

```

如果 slash command 未被自动识别，也可以直接输入：

```
请读取 .cursor/skills/api-change-guard/SKILL.md，并按照 API Change Guard 流程分析当前 git diff。

```

### **Claude Code / Codex / 其他 Agent 使用方式**

将 `SKILL.md` 放到项目根目录或工具目录，然后输入：

```
读取 SKILL.md，并按 API Change Guard 流程分析当前 git diff。

```

如果项目里有多个 `SKILL.md`，建议明确路径：

```
读取 tools/api-change-guard/SKILL.md，并按其中规则分析当前 git diff。

```

## **执行流程**

该 MVP 采用“通用 Agent 规则 + Git 命令分析”方案，不依赖 Python，也不需要额外本地运行时。只需要 Git 和一个能读取文件、执行命令、创建 Markdown 文件的 AI Agent 即可使用。

AI Agent 会通过以下 Git 命令收集证据：

```
git diff -- '*.java'
git diff --cached -- '*.java'
git diff --name-only -- '*.java'
git diff --cached --name-only -- '*.java'
git rev-parse --short HEAD

```

然后 AI Agent 会读取相关的 Controller / DTO / Mapper / Convert / Feign / Logic 文件，基于 Git diff 和文件内容分析影响范围，在 `tools/api-change-guard/reports/` 下生成 Markdown 报告，并返回报告链接和报告正文。

报告文件名包含分析目标、当前 commit 短 SHA 和时间戳，例如：

```
tools/api-change-guard/reports/api-change-guard-diff-a1b2c3d-20260605-182900.md

```

## **回复格式要求**

最终回复必须同时包含报告链接和报告正文，不能只返回链接。

固定顺序：

```
报告文件：[<filename>](<relative-path>)

# API Change Guard 变更影响分析报告

...完整报告正文...

```

如果报告很长，也不能只总结报告；必须保留报告结构中的核心章节。必要时只能压缩较长的代码块或请求样例。

## **报告结构**

```
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

## **大 Diff 分析规则**

- `<= 10` 个 Java 文件：全量分析。
- `11-30` 个 Java 文件：全量分析 Controller、DTO、Feign；Mapper / Logic 做摘要分析；其他文件只列出。
- `> 30` 个 Java 文件：进入大 diff 模式，只深度分析接口相关文件，其余列入未覆盖范围。
- 文件优先级：Controller、DTO / VO / BO / Request / Response、Feign / Remote Client、Mapper / Convert / Service / Logic、其他 Java 文件。
- 非 Controller 文件只有命中 API 关键词时才深度分析，例如 mapping 注解、`@RequestBody`、`@Schema`、校验注解、`CommonResult`、`DataCollectDto`、`FeignClient`。

## **可靠性规则**

- Git diff 和文件内容是事实依据。
- 测试源码不纳入评审范围，忽略 `src/test/`**、`*Test.java` 和测试专用 diff。
- 注释掉的 Controller 代码不视为真实接口。
- 报告应以影响范围为中心，先判断是否影响已上线功能，再补充代码质量提示。
- 必须明确回归测试范围。
- 对大 diff 或部分分析的 diff，必须输出分析覆盖范围和未覆盖风险。
- `DataCollectDto<T>` 需要同时分析外层采集字段和内层 DTO。
- 带 `@Watch4gDataLength` 的手表 / 4G / 蓝牙 DTO，需要关注协议字段顺序、长度和 countField 兼容性。
- AI 结论必须区分事实、推测和待人工确认项。
- 无法解析的复杂类型必须写入待人工确认项，不能静默忽略。
- 生成的 MockMvc 代码只是骨架，开发仍需补充 Service Mock 和精确断言。
- 影响范围不是最终事实，报告必须给出概率和依据。
- 如果收集到的证据有限，仍需输出可读报告，并明确待人工确认项。

## **已知限制**

- MVP 依赖 AI Agent 直接理解 Git diff 和 Java 文件，因此分析质量取决于可获得的 diff 和文件证据。
- 多行方法签名、深层泛型 DTO 可能需要人工复核。
- Feign、Mapper 和前端调用方分析在 MVP 阶段可能会被标记为待人工确认。
- 生成的测试代码默认基于 JUnit 5、Spring Boot Test 和 MockMvc，项目特定的鉴权、租户 Header、Service Mock 需要开发自行补充。

## **误判记录**

当工具输出错误或不够准确的结论时，建议按以下格式记录，便于后续优化 Skill：

```
### Case: <short title>

- Input: Controller 路径、接口路径或 diff 摘要
- Wrong output: 工具输出了什么错误结论
- Expected output: 评审期望的正确结论
- Fix rule: 需要更新的 Skill 规则、Prompt 规则或人工确认规则
```

