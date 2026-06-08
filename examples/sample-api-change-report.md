# **API Change Guard 变更影响分析报告**

## **结论先行**

- 是否影响已上线功能: 可能影响老版本 APP/设备端上报。
- 是否需要回归测试: 需要，重点回归保存接口和老版本请求体。
- 高风险影响点: 新增必填字段。
- 重点回归范围: `POST /wearingStatus/save`、采集上报链路、字段校验链路。
- 是否需要客户端/其他后端服务配合: 需要确认 APP/设备端是否补传新增字段。

## **变更影响范围**

### **已有功能影响**

- 新增字段 `deviceType`（String，必填，含义：设备类型）。依据：新增字段；校验注解 @NotEmpty
- 新增字段 `wearingStatus`（Integer，必填，含义：佩戴状态）。依据：新增字段；校验注解 @NotNull, @Min(0), @Max(2)

### **新增功能影响**

- 未识别新增功能入口，主要是已有保存接口请求契约变化。

### **端侧影响**

- APP: 高概率。依据：接口路径和 DTO 命中 wearingStatus、health、collect 等穿戴健康数据关键词
- 设备端上报: 中概率。依据：接口属于采集上报链路，需要确认设备协议是否直接传递该字段
- 管理后台: 低概率。依据：该接口为保存上报数据，后台可能只查询展示结果

### **后端链路影响**

- Controller 入参、DTO 校验、Logic 保存链路都需要回归。

### **数据影响**

- 新增字段可能影响请求数据校验和后续持久化/展示逻辑。

## **回归测试建议**

### **必须回归**

- `save_shouldSuccess_whenRequestValid`: 传入完整合法参数，预期返回成功响应
- `save_shouldFail_whenDeviceTypeMissing`: 缺少必填字段 `deviceType`，预期参数校验失败或返回业务错误码
- `save_shouldFail_whenWearingStatusMissing`: 缺少必填字段 `wearingStatus`，预期参数校验失败或返回业务错误码

### **建议回归**

- `save_shouldFail_whenWearingStatusOutOfRange`: `wearingStatus` 传入 -1 或 3，预期参数校验失败或返回业务错误码

### **可不回归**

- 与该保存接口无调用关系的后台纯查询页面可不作为重点回归。

## **兼容性风险**

### **请求参数兼容性**

- 新增必填字段 `deviceType`，老版本 APP 或调用方未传该字段时可能请求失败。依据：新增 @NotEmpty 字段
- 新增必填字段 `wearingStatus`，历史请求体为空或字段缺失时会触发参数校验。依据：新增 @NotNull 字段

### **返回结构兼容性**

- 返回类型未变化。

### **枚举/状态值兼容性**

- `wearingStatus` 状态值含义需确认。

### **校验规则兼容性**

- 新增必填校验，需要确认老版本兼容策略。

### **数据兼容性**

- 新字段可能影响历史数据补录或空值处理。

## **变更事实摘要**

### **新增接口**

- 未识别新增接口。

### **修改接口**

- `POST /wearingStatus/save` 请求字段契约变化。

### **新增/修改 DTO 字段**

- 新增 `deviceType`、`wearingStatus`。

### **新增/修改方法逻辑**

- 未识别方法逻辑变化。

### **Mapper / Convert / Feign 变化**

- 未识别相关变化。

## **关键链路分析**

### **调用入口**

- `POST /wearingStatus/save`

### **核心处理逻辑**

- Controller 接收入参后进入保存逻辑。

### **数据转换链路**

- 需确认 DTO 字段是否被完整使用。

### **持久化/缓存/MQ/远程调用影响**

- 需确认新增字段是否持久化或仅用于校验。

## **测试点清单**

### **正常场景**

- 完整合法参数保存成功。

### **异常场景**

- 缺少 `deviceType`、`wearingStatus`。

### **边界场景**

- `wearingStatus` 非法状态值。

### **历史兼容场景**

- 老版本请求体不传新增字段。

### **回归场景**

- 回归 APP/设备端上报链路。

## **分析覆盖范围**

- Java 变更文件数：示例未统计
- 已展开分析：保存接口和 DTO 字段
- 摘要分析：无
- 未展开分析：无
- 已排除测试文件：`src/test/`**、`*Test.java`

## **未覆盖风险**

- 示例报告未包含真实调用方扫描，需人工确认端侧调用范围。

## **JUnit / MockMvc 测试骨架**

```
@SpringBootTest
@AutoConfigureMockMvc
class WearingStatusControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void save_shouldSuccess_whenRequestValid() throws Exception {
        String body = """
                {
                  "deviceType": "test-deviceType",
                  "wearingStatus": 1
                }
                """;

        mockMvc.perform(post("/wearingStatus/save")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(body))
                .andExpect(status().isOk());
    }
}

```

## **Apifox / Postman 请求样例**

```
curl -X POST "http://localhost:8080/wearingStatus/save" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "tenantId: <tenantId>" \
  -H "userId: <userId>" \
  -d '{
  "deviceType": "test-deviceType",
  "wearingStatus": 1
}'

```

## **展示讲解词**

这个报告不是简单复述 diff，而是把代码事实转成接口协作语言：客户端能看到是否需要改请求体，测试能直接拿到边界用例，后端 reviewer 能快速定位兼容性风险。对于新增必填字段这种高频问题，工具能在提测前提醒老版本兼容风险。

## **待人工确认项**

- 是否允许老版本 APP 不传 `deviceType`，需要产品和接口负责人确认。
- `wearingStatus` 的枚举含义需要和客户端、设备协议保持一致。
- MockMvc 骨架需要开发根据 Controller 依赖补充 Service Mock 和业务错误码断言。

## **次要代码质量提示**

- 未输出完整代码质量评审；仅提示可能影响功能正确性的事项。

