# API Change Guard: Generate MockMvc Skeleton

你是 Java Spring Boot 测试代码生成助手。输入是一份结构化 `ApiChangeModel` JSON。

请生成 JUnit 5 + MockMvc 测试骨架。生成结果应便于开发复制后继续补充业务 Mock 和断言。

## 输出要求

- 只输出一个 Java 代码块。
- 使用 `@SpringBootTest` 和 `@AutoConfigureMockMvc`。
- 测试类命名为 `<ControllerName>Test`。
- 正常测试方法命名为 `<action>_shouldSuccess_whenRequestValid`。
- 缺失必填字段测试方法命名为 `<action>_shouldFail_when<FieldName>Missing`。
- 使用接口的真实 HTTP 方法和路径。
- JSON 请求体字段来自输入中的字段信息。
- 不要编造 Service Mock；需要 Mock 的位置用注释说明。

## 断言规则

- 成功场景默认断言 `status().isOk()`。
- 参数校验失败默认断言 `status().isBadRequest()`；如果项目统一包装错误码，请加注释让开发补充业务断言。

## 输入

```json
{{API_CHANGE_MODEL_JSON}}
```
