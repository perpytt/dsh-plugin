# dsh-llm-glm-gateway

GLM-5.2（自建 OpenAI 兼容网关，地址通过 `baseURL` 配置）的 DeepSeek Harness LLM
provider 适配插件。

基于 `@deepseek-ai/dsh-llm-deepseek`（MIT）fork，协议形态与其一致（OpenAI 兼容
chat-completions + SSE + `reasoning_content` + `stream_options.include_usage`），
仅针对本网关的两处工具调用差异做了修复：

- **流式分片里的空 `function.name` 不再覆盖首分片写入的真实函数名**（网关在后续参数增量分片里
  重复发送 `"name":""`）。
- **`arguments` 的前导 `{}` 脏前缀会被清洗**（非流式响应曾稳定出现
  `"{}{\"city\":\"北京\"}"`），保证存入 harness 历史、用于工具分发的参数始终是干净 JSON。

## 配置

```yaml
- id: llm-glm-gateway
  name: dsh-llm-glm-gateway
  config:
    apiKeyEnv: TEST_KEY        # default; resolved per request via ctx.credentials, then env
    baseURL: http://your-gateway-host:port/v1   # required; $GLM_GATEWAY_BASE_URL overrides
    thinking: enabled          # optional
    reasoningEffort: high      # off | low | high | max
    maxTokens: 131072          # optional; gateway上限 [1,131072]，默认 131072
    models:                    # optional; defaults to glm-5.2
      - id: glm-5.2
        name: GLM-5.2
    retryPolicy:               # optional
      mode: normal
```

provider 路由：`glm-gateway`（Web UI 显示 “GLM Gateway”）。

## 结构

- `lib/index.js` — 单文件 ESM 实现（serialize / SSE parse / translate / adapter / 注册）。
- 运行期只需 `lib/index.js`；peer 依赖全部来自 harness 运行树
  （`@deepseek-ai/dsh-llm`、`dsh-settings`、`dsh-credentials` 等）。

## 与官方适配器的差异清单

1. `translate()` 工具分片：`id`/`name` 仅在非空字符串时写入。
2. 新增 `sanitizeToolArguments()`，`closeBlock()` 对 tool-call 应用。
3. 标识与默认值：route `glm-gateway`、plugin name `llm-glm-gateway`、默认模型 `glm-5.2`、
   baseURL 指向本网关、`apiKeyEnv` 默认 `TEST_KEY`、`BASE_URL_ENV=GLM_GATEWAY_BASE_URL`。
4. 纯文本模型目录（不宣传图片输入），其余图片/Files 代码保留但不会触发。

## 验证

- `node --check lib/index.js` 通过。
- 已用 curl 实测：本网关接受 dsh-llm-deepseek 同款完整请求（system 提示 + 历史含
  `reasoning_content` 回传 + 干净 `tool_calls` 回传 + tools + stream/stream_options），
  工具结果正常被模型使用。
