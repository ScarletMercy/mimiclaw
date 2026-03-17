# Longcat 模型提供商支持 - 实现总结

## 概述
为 MimiClaw 添加了 Longcat 模型提供商支持。Longcat 使用 OpenAI 兼容的 API 格式，可以无缝集成到现有系统中。

## 修改的文件

### 1. main/mimi_config.h
**更改**: 添加了 Longcat API URL 配置
```c
#define MIMI_LONGCAT_API_URL         "https://api.longcat.chat/openai/v1/chat/completions"
```

### 2. main/llm/llm_proxy.c
**更改**: 实现了 Longcat 提供商支持

#### 添加的函数
- `provider_is_longcat()`: 检查当前提供商是否为 longcat
- `provider_is_openai_compatible()`: 检查是否为 OpenAI 兼容的提供商 (openai 或 longcat)

#### 修改的函数
1. **llm_api_url()**: 根据 provider 返回正确的 API URL
   - longcat: `https://api.longcat.chat/openai/v1/chat/completions`
   - openai: `https://api.openai.com/v1/chat/completions`
   - anthropic: `https://api.anthropic.com/v1/messages`

2. **llm_api_host()**: 返回正确的 API 主机
   - longcat: `api.longcat.chat`
   - openai: `api.openai.com`
   - anthropic: `api.anthropic.com`

3. **llm_api_path()**: 返回正确的 API 路径
   - longcat: `/openai/v1/chat/completions`
   - openai: `/v1/chat/completions`
   - anthropic: `/v1/messages`

4. **llm_http_direct()**: 修改 HTTP 请求头
   - OpenAI 兼容提供商使用 `Authorization: Bearer <token>`
   - Anthropic 使用 `x-api-key: <token>` 和 `anthropic-version`

5. **llm_http_via_proxy()**: 修改代理隧道请求头
   - 与 direct 路径使用相同的认证方式

6. **llm_chat_tools()**: 修改请求体构建
   - OpenAI 兼容提供商使用 `max_completion_tokens`
   - Anthropic 使用 `max_tokens`
   - OpenAI 兼容提供商的消息格式使用 `convert_messages_openai()`

7. **响应解析**: 使用 OpenAI 兼容的解析逻辑
   - 解析 `choices[0].message.content` 和 `tool_calls`
   - 检查 `finish_reason` 是否为 `tool_calls`

### 3. README.md
**更改**: 更新文档以说明 Longcat 支持
- 概述部分提到支持 Anthropic、OpenAI 和 Longcat
- 快速开始部分添加 Longcat API key 获取说明
- 配置示例更新，provider 选项增加 longcat
- CLI 命令示例更新

### 4. main/mimi_secrets.h.example
**更改**: 更新配置文件模板
- 注释说明支持 "anthropic", "openai", 或 "longcat"

## 使用方法

### 构建时配置 (mimi_secrets.h)
```c
#define MIMI_SECRET_API_KEY         "your-longcat-api-key"
#define MIMI_SECRET_MODEL           "your-model-name"
#define MIMI_SECRET_MODEL_PROVIDER  "longcat"
```

### 运行时配置 (CLI)
```
mimi> set_api_key your-longcat-api-key
mimi> set_model_provider longcat
mimi> set_model your-model-name
```

## 技术细节

### API 兼容性
- Longcat 使用 OpenAI 兼容的 API 格式
- 请求格式与 OpenAI 相同
- 响应格式与 OpenAI 相同
- 认证方式使用 Bearer Token

### 消息格式转换
Longcat (OpenAI 兼容) 和 Anthropic 使用不同的消息格式：
- **Anthropic**: 使用 `system` 顶级字段，content 是数组，支持 `tool_use` 和 `tool_result` 块
- **OpenAI/Longcat**: 使用 `messages` 数组，system 消息作为数组元素，content 是字符串，支持 `tool_calls` 数组

代码中的 `convert_messages_openai()` 函数负责将 Anthropic 格式转换为 OpenAI 兼容格式，使得同一套 Agent 逻辑可以无缝支持不同提供商。

### 工具调用支持
- Anthropic: 使用 `tool_use` 和 `tool_result` 块
- OpenAI/Longcat: 使用 `tool_calls` 数组和 `tool` 角色

响应解析会自动适配不同格式，提取文本内容和工具调用信息。

## 测试建议

1. **基本测试**
```bash
# 配置 Longcat
mimi> set_model_provider longcat
mimi> set_api_key sk-xxx...
mimi> set_model gpt-4o

# 通过 Telegram 发送简单测试消息
```

2. **工具调用测试**
```
# 发送需要搜索的消息
What's the weather today?
```

3. **多轮对话测试**
```
# 测试会话记忆
Tell me about yourself
Remember my name is Alice
What's my name?
```

4. **代理测试** (如需要)
```bash
mimi> set_proxy 127.0.0.1 7897
```

## 注意事项

1. Longcat API endpoint 使用的是 OpenAI 兼容格式
2. 所有 OpenAI 兼容的提供商 (OpenAI, Longcat 等) 共享相同的请求/响应处理逻辑
3. 可以通过 CLI 在运行时切换提供商，无需重新编译
4. 模型名称需要根据 Longcat 支持的模型进行配置

## 相关文件
- `main/mimi_config.h` - 配置常量
- `main/llm/llm_proxy.c` - LLM API 代理实现
- `main/llm/llm_proxy.h` - LLM API 代理接口
- `main/cli/serial_cli.c` - CLI 命令实现
- `README.md` - 用户文档
- `main/mimi_secrets.h.example` - 配置模板
