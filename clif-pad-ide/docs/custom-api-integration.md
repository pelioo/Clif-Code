# ClifPad IDE 自定义 API 接入方案

> 本文档描述 ClifPad IDE 增加自定义 API 接入的技术方案。

**文档状态**: 设计完成，等待实现  
**创建日期**: 2026-05-14  
**最后更新**: 2026-05-14

---

## 一、背景与目标

### 1.1 问题描述

当前 ClifPad IDE 的 AI 接入仅支持两个硬编码的 Provider：
- **OpenRouter** - 云端 API 网关，支持 100+ 模型
- **Ollama** - 本地模型服务

用户无法接入其他 OpenAI 兼容的 API 服务，如：
- Groq
- Together AI
- Fireworks AI
- LocalAI / LM Studio
- Azure OpenAI
- Anthropic Direct

### 1.2 目标

在保持现有 OpenRouter 和 Ollama 支持的前提下，增加**自定义 API 端点**功能，允许用户配置任意 OpenAI 兼容的 API 服务商。

### 1.3 设计原则

1. **最小改动原则** - 复用现有架构，最小化代码变更
2. **一致性原则** - 复用 IDE 现有配置存储（`~/.clif/settings.json`）
3. **向后兼容** - 不破坏现有功能和配置
4. **渐进增强** - Phase 1 实现基础功能，Phase 2 支持特殊 API

---

## 二、技术架构

### 2.1 现有架构分析

```
AI 接入层 (ClifPad IDE)
├── 前端配置层
│   ├── src/components/agent/constants.ts     ← 定义 PROVIDERS 和 POPULAR_MODELS
│   ├── src/components/agent/SettingsPanel.tsx ← 设置面板 UI
│   └── src/components/agent/ProviderModelSelector.tsx
│
├── 前端状态层
│   ├── src/stores/agentStore.ts             ← AI 消息和会话管理
│   └── src/stores/settingsStore.ts          ← AI 配置持久化
│
├── 类型定义层
│   ├── src/types/ai.ts                      ← AiConfig, ModelInfo
│   └── src/types/agent.ts                   ← AgentMessage, ToolCall
│
└── 后端实现层
    ├── src-tauri/src/commands/ai.rs          ← 核心 AI 命令 (689 行)
    ├── src-tauri/src/services/ai_provider.rs ← Provider 枚举定义
    └── src-tauri/src/commands/claude_code.rs ← Claude Code CLI 集成
```

### 2.2 配置存储架构

```
~/.clif/                          ← ClifPad IDE 配置目录
├── settings.json                 ← 主编辑器设置 + AI 配置
├── api_keys.json                 ← 各 provider 的 API keys
└── agent_sessions/               ← Agent 会话历史
    └── {hash}.json

~/.clifcode/                      ← ClifCode TUI 配置目录（独立）
└── config.json
```

### 2.3 新增配置字段

在 `settings.json` 中新增以下字段：

```json
{
  "aiProvider": "custom",
  "aiModel": "gpt-4o",
  "customApiUrl": "https://api.groq.com/openai/v1",
  
  "theme": "midnight",
  "fontSize": 14
}
```

---

## 三、详细设计方案

### 3.1 文件修改清单

| 序号 | 文件路径 | 修改内容 | 优先级 |
|------|----------|----------|--------|
| 1 | `src/components/agent/constants.ts` | 添加 `custom` provider 定义 | P0 |
| 2 | `src/stores/settingsStore.ts` | 添加 `customApiUrl` 字段 | P0 |
| 3 | `src/components/agent/SettingsPanel.tsx` | 添加自定义 URL 输入 UI | P0 |
| 4 | `src/lib/tauri.ts` | 透传 `customApiUrl` 参数 | P0 |
| 5 | `src-tauri/src/commands/ai.rs` | 修改 `get_provider_url()` | P0 |
| 6 | `src-tauri/src/services/ai_provider.rs` | 添加 `Custom` variant | P1 |
| 7 | `src/components/agent/ModelBrowser.tsx` | 处理 custom 模型获取 | P2 |

### 3.2 前端修改规格

#### 3.2.1 constants.ts 修改

```typescript
// 新增 provider 选项
export const PROVIDERS = [
  { value: "openrouter", label: "OpenRouter", hint: "openrouter.ai — access 100+ models" },
  { value: "ollama", label: "Ollama", hint: "Local models — no API key needed" },
  { value: "custom", label: "Custom API", hint: "Enter your own OpenAI-compatible endpoint" },
];

// Provider 元数据配置
export const PROVIDER_META: Record<string, {
  needsApiKey: boolean;
  defaultModel: string;
  modelFetchEndpoint: string | null;
  description: string;
}> = {
  openrouter: {
    needsApiKey: true,
    defaultModel: "anthropic/claude-sonnet-4",
    modelFetchEndpoint: "https://openrouter.ai/api/v1/models",
    description: "Access 100+ models from various providers",
  },
  ollama: {
    needsApiKey: false,
    defaultModel: "llama3.1",
    modelFetchEndpoint: "http://localhost:11434/api/tags",
    description: "Local models — no API key needed",
  },
  custom: {
    needsApiKey: true,
    defaultModel: "",
    modelFetchEndpoint: null,
    description: "Connect to any OpenAI-compatible API",
  },
};
```

#### 3.2.2 settingsStore.ts 修改

```typescript
interface Settings {
  // ... 现有字段
  aiProvider: string;
  aiModel: string;
  
  // 新增字段
  customApiUrl: string;  // 自定义 API 端点
}

const defaultSettings: Settings = {
  // ...
  aiProvider: "openrouter",
  aiModel: "anthropic/claude-sonnet-4",
  customApiUrl: "",
};
```

#### 3.2.3 SettingsPanel.tsx 修改

```tsx
// 新增：自定义 URL 输入区域
<Show when={settings().aiProvider === "custom"}>
  <div class="mb-3">
    <label style={labelStyle}>API Endpoint URL</label>
    <input
      type="url"
      placeholder="https://api.example.com/v1"
      value={settings().customApiUrl}
      onInput={(e) => updateSettings({ customApiUrl: e.currentTarget.value })}
      style={inputStyle}
    />
    <span style={hintStyle}>
      Enter the base URL of your OpenAI-compatible API
    </span>
  </div>
</Show>

// Model 输入：custom provider 改为手动输入
<Show when={settings().aiProvider !== "custom"}>
  {/* 现有下拉选择器 */}
</Show>
<Show when={settings().aiProvider === "custom"}>
  <input
    type="text"
    placeholder="e.g. gpt-4o, llama-3.1-70b"
    value={settings().aiModel}
    onInput={(e) => updateSettings({ aiModel: e.currentTarget.value })}
  />
</Show>
```

### 3.3 后端修改规格

#### 3.3.1 ai.rs 修改

```rust
/// 获取 provider URL，支持自定义端点
fn get_provider_url(provider: &str, custom_url: Option<&str>) -> String {
    match provider {
        "ollama" => "http://localhost:11434/v1/chat/completions".to_string(),
        "custom" => {
            custom_url
                .filter(|u| !u.is_empty())
                .map(|u| format!("{}/chat/completions", u.trim_end_matches('/')))
                .unwrap_or_else(|| "".to_string())
        }
        "openrouter" | _ => "https://openrouter.ai/api/v1/chat/completions".to_string(),
    }
}

// 修改所有 AI 命令签名，添加 custom_url 参数
#[tauri::command]
pub async fn ai_chat(
    window: tauri::Window,
    messages: Vec<ChatMessage>,
    model: String,
    api_key: Option<String>,
    provider: String,
    custom_url: Option<String>,  // 新增
) -> Result<(), String> {
    let url = get_provider_url(&provider, custom_url.as_deref());
    // ...
}
```

#### 3.3.2 ai_provider.rs 修改

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum Provider {
    OpenRouter,
    Ollama,
    Custom,  // 新增
}

impl Provider {
    pub fn from_str(s: &str) -> Self {
        match s.to_lowercase().as_str() {
            "ollama" => Provider::Ollama,
            "custom" => Provider::Custom,
            _ => Provider::OpenRouter,
        }
    }

    pub fn base_url(&self) -> &str {
        match self {
            Provider::OpenRouter => "https://openrouter.ai/api/v1",
            Provider::Ollama => "http://localhost:11434/v1",
            Provider::Custom => "",  // 由 custom_url 参数提供
        }
    }

    pub fn name(&self) -> &str {
        match self {
            Provider::OpenRouter => "openrouter",
            Provider::Ollama => "ollama",
            Provider::Custom => "custom",
        }
    }
}
```

---

## 四、支持的 API 服务商

### 4.1 标准 OpenAI 兼容（Phase 1 支持）

| 服务商 | Base URL | 说明 |
|--------|----------|------|
| Groq | `https://api.groq.com/openai/v1` | 高速推理 |
| Together AI | `https://api.together.xyz/v1` | 开源模型 |
| Fireworks AI | `https://api.fireworks.ai/v1` | 高性能 |
| Cloudflare Workers AI | `https://api.cloudflare.com/client/v4` | Edge AI |
| LocalAI | `http://localhost:8080/v1` | 本地部署 |
| LM Studio | `http://localhost:1234/v1` | 本地模型 |
| Ollama API | `http://localhost:11434/v1` | 已支持 |

### 4.2 需要特殊处理（Phase 2）

#### Azure OpenAI
```rust
// Azure 需要不同的认证方式
if is_azure {
    req_builder = req_builder
        .header("api-key", api_key)  // 而非 Bearer
        .header("Content-Type", "application/json");
    url = "https://{resource}.openai.azure.com/openai/deployments/{deployment}/chat/completions?api-version=2024-02-15-preview";
}
```

#### Anthropic Direct
```rust
// Anthropic 使用不同端点和认证
req_builder = req_builder
    .header("x-api-key", api_key)
    .header("anthropic-version", "2023-06-01");
url = "https://api.anthropic.com/v1/messages";  // 非 /chat/completions
```

---

## 五、数据流设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户操作流程                                 │
│                                                                     │
│  1. 用户点击 Provider 选择器                                         │
│  2. 选择 "Custom API" 选项                                          │
│  3. SettingsPanel 显示自定义 URL 输入框                              │
│  4. 用户输入 API 端点 URL 和 Model 名称                              │
│  5. 用户输入/保存 API Key                                           │
│  6. 配置保存到 ~/.clif/settings.json                                 │
│  7. API Key 保存到 ~/.clif/api_keys.json                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        API 调用流程                                 │
│                                                                     │
│  Provider Selector                                                  │
│        │                                                           │
│        ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ if (provider === "custom") {                                │   │
│  │   url = settings.customApiUrl + "/chat/completions"        │   │
│  │ } else {                                                    │   │
│  │   url = PROVIDER_URLS[provider]                            │   │
│  │ }                                                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│        │                                                           │
│        ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Tauri Invoke: ai_chat({                                      │   │
│  │   provider,                                                 │   │
│  │   custom_url: settings.customApiUrl,                        │   │
│  │   model,                                                    │   │
│  │   api_key: api_keys[provider]                               │   │
│  │ })                                                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│        │                                                           │
│        ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Rust Backend: get_provider_url(provider, custom_url)       │   │
│  │                                                            │   │
│  │ match provider {                                           │   │
│  │   "ollama"    → localhost:11434/v1                        │   │
│  │   "custom"    → {custom_url}/chat/completions            │   │
│  │   "openrouter" → openrouter.ai/api/v1                    │   │
│  │ }                                                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│        │                                                           │
│        ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ HTTP POST {                                                 │   │
│  │   url: constructed_url,                                     │   │
│  │   headers: { Authorization: Bearer {key} },                │   │
│  │   body: { model, messages, stream: true }                  │   │
│  │ }                                                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│        │                                                           │
│        ▼                                                           │
│  SSE Stream → Frontend Event                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、错误处理

### 6.1 自定义 URL 验证

| 场景 | 处理方式 |
|------|----------|
| URL 为空 | 提示 "请输入 API 端点 URL" |
| URL 格式错误 | 提示 "请输入有效的 URL" |
| 无法连接 | 显示 "无法连接到 API，请检查 URL" |
| 认证失败 | 显示 "API 认证失败，请检查 API Key" |

### 6.2 URL 格式化

```rust
// 自动处理尾斜杠
let url = format!("{}/chat/completions", 
    custom_url.trim_end_matches('/')
);

// 确保协议前缀
if !url.starts_with("http://") && !url.starts_with("https://") {
    return Err("URL must start with http:// or https://");
}
```

---

## 七、测试用例

### 7.1 功能测试

| ID | 测试场景 | 预期结果 |
|----|----------|----------|
| T1 | 选择 Custom Provider | SettingsPanel 显示 URL 输入框 |
| T2 | 输入有效 URL | URL 正确保存到 settings.json |
| T3 | 输入无效 URL | 显示错误提示 |
| T4 | 保存 Custom 配置 | API Key 保存到 api_keys.json |
| T5 | 使用 Custom API 发送消息 | 消息正确发送到自定义端点 |
| T6 | 切换回 OpenRouter | 使用 OpenRouter 端点 |
| T7 | 空 URL 时发送请求 | 提示 "请配置自定义 API URL" |

### 7.2 兼容性测试

| ID | 服务商 | Base URL | 预期结果 |
|----|--------|----------|----------|
| C1 | Groq | `https://api.groq.com/openai/v1` | ✅ |
| C2 | Together AI | `https://api.together.xyz/v1` | ✅ |
| C3 | Fireworks | `https://api.fireworks.ai/v1` | ✅ |
| C4 | LocalAI | `http://localhost:8080/v1` | ✅ |
| C5 | LM Studio | `http://localhost:1234/v1` | ✅ |

---

## 八、实现检查清单

### Phase 1 - 基础自定义 API

#### 前端修改
- [ ] `constants.ts` - 添加 `custom` provider 和 `PROVIDER_META`
- [ ] `settingsStore.ts` - 添加 `customApiUrl` 字段
- [ ] `SettingsPanel.tsx` - 添加自定义 URL 输入框
- [ ] `ProviderModelSelector.tsx` - 处理 custom provider 样式
- [ ] `tauri.ts` - 透传 `customApiUrl` 参数

#### 后端修改
- [ ] `ai.rs` - 修改 `get_provider_url()` 函数
- [ ] `ai.rs` - 更新所有 AI 命令签名（ai_chat, ai_complete, ai_review_code, generate_commit_message, get_models）
- [ ] `ai_provider.rs` - 添加 `Custom` variant

#### 测试验证
- [ ] 手动输入 URL 连接测试
- [ ] LocalAI 连接测试
- [ ] LM Studio 连接测试
- [ ] Groq 连接测试
- [ ] URL 格式化测试（尾斜杠处理）
- [ ] 错误处理测试

---

## 九、附录

### A. 参考实现

**ClifCode TUI 已有自定义端点实现**，可参考：
- `clif-code-tui/src/config.rs` - Provider 定义和交互式设置
- `clif-code-tui/src/backend.rs` - API 调用逻辑

### B. OpenAI 兼容 API 列表

| 服务商 | 官方网站 | 特点 |
|--------|----------|------|
| Groq | groq.com | 极低延迟 |
| Together AI | together.ai | 开源模型丰富 |
| Fireworks AI | fireworks.ai | 高吞吐量 |
| Cloudflare Workers | cloudflare.com/ai | Edge 部署 |
| AWS Bedrock | aws.amazon.com/bedrock | 企业级 |
| Azure OpenAI | azure.microsoft.com | 企业合规 |

### C. 相关文件路径

```
clif-pad-ide/
├── src/
│   ├── components/agent/
│   │   ├── constants.ts         ← 修改
│   │   ├── SettingsPanel.tsx   ← 修改
│   │   └── ProviderModelSelector.tsx
│   ├── stores/
│   │   └── settingsStore.ts     ← 修改
│   └── lib/
│       └── tauri.ts            ← 修改
└── src-tauri/
    └── src/
        ├── commands/
        │   └── ai.rs            ← 修改
        └── services/
            └── ai_provider.rs   ← 修改
```