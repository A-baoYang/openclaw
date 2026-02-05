# OpenClaw Core Module Extraction Plan

> **Purpose:** 從 openclaw 抽取 Core Agent & AI System 及 Multi-Channel Messaging 為獨立模組
> **Target:** 供 DataAgent AI Agent Platform 及其他專案使用
> **Date:** 2026-02-02

---

## 1. 模組架構規劃

### 1.1 兩個獨立套件

```
~/dev/
├── dataagent-rag-core/           # 已存在 - RAG 核心
├── dataagent-agent-core/         # 新建 - Agent & AI 核心
│   ├── src/
│   │   ├── agent/                # Agent 執行引擎
│   │   ├── tools/                # Tool 系統
│   │   ├── llm/                  # LLM Provider 整合
│   │   ├── prompt/               # System Prompt 建構
│   │   ├── streaming/            # Streaming 回應處理
│   │   ├── memory/               # Memory/Context 管理
│   │   └── session/              # Session 管理
│   ├── package.json
│   └── tsconfig.json
│
└── dataagent-channels/           # 新建 - Multi-Channel 核心
    ├── src/
    │   ├── core/                 # Channel 抽象層
    │   ├── routing/              # 訊息路由
    │   ├── adapters/             # 通用 Adapter 介面
    │   ├── plugins/              # Plugin 系統
    │   │   ├── telegram/
    │   │   ├── discord/
    │   │   ├── slack/
    │   │   ├── line/
    │   │   ├── whatsapp/
    │   │   └── web-widget/       # 新增 - Web Widget
    │   └── utils/                # 共用工具
    ├── package.json
    └── tsconfig.json
```

### 1.2 套件依賴關係

```
                    ┌─────────────────────┐
                    │  dataagent-rag-core │ (Python)
                    └─────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 DataAgent AI Agent Platform                  │
│                    (FastAPI + Next.js)                       │
└─────────────────────────────────────────────────────────────┘
           │                               │
           ▼                               ▼
┌─────────────────────┐     ┌─────────────────────────────┐
│ dataagent-agent-core│     │    dataagent-channels       │
│     (TypeScript)    │◄────│       (TypeScript)          │
└─────────────────────┘     └─────────────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    ▼                     ▼                     ▼
              ┌──────────┐         ┌──────────┐          ┌──────────┐
              │ Telegram │         │  Discord │          │   LINE   │
              │  Plugin  │         │  Plugin  │          │  Plugin  │
              └──────────┘         └──────────┘          └──────────┘
```

---

## 2. dataagent-agent-core 詳細規劃

### 2.1 從 openclaw 抽取的模組

| 來源路徑 (openclaw) | 目標路徑 (dataagent-agent-core) | 說明 |
|---------------------|----------------------------------|------|
| `src/agents/pi-embedded-runner/` | `src/agent/runner/` | Agent 執行引擎核心 |
| `src/agents/pi-tools.ts` | `src/tools/factory.ts` | Tool 工廠 |
| `src/agents/pi-tools.schema.ts` | `src/tools/schema.ts` | Schema 正規化 |
| `src/agents/pi-tools.policy.ts` | `src/tools/policy.ts` | Tool 存取控制 |
| `src/agents/tool-policy.ts` | `src/tools/policy-resolver.ts` | Policy 解析 |
| `src/agents/system-prompt.ts` | `src/prompt/builder.ts` | System Prompt 建構 |
| `src/agents/system-prompt-params.ts` | `src/prompt/params.ts` | Prompt 參數解析 |
| `src/agents/pi-embedded-subscribe.ts` | `src/streaming/subscriber.ts` | Streaming 訂閱 |
| `src/agents/pi-embedded-subscribe.handlers.ts` | `src/streaming/handlers/` | 事件處理器 |
| `src/agents/pi-embedded-block-chunker.ts` | `src/streaming/chunker.ts` | 文字切塊 |
| `src/agents/model-selection.ts` | `src/llm/model-selection.ts` | Model 解析 |
| `src/agents/model-auth.ts` | `src/llm/auth.ts` | 認證管理 |
| `src/agents/model-catalog.ts` | `src/llm/catalog.ts` | Model 目錄 |
| `src/agents/model-fallback.ts` | `src/llm/fallback.ts` | Failover 邏輯 |
| `src/memory/manager.ts` | `src/memory/manager.ts` | Memory 管理 |
| `src/memory/embeddings.ts` | `src/memory/embeddings.ts` | Embedding 計算 |
| `src/agents/compaction.ts` | `src/session/compaction.ts` | Session 壓縮 |
| `src/agents/context-window-guard.ts` | `src/session/token-guard.ts` | Token 預算 |

### 2.2 關鍵類型定義

```typescript
// src/agent/types.ts
export interface AgentRunParams {
  agentId: string;
  sessionKey: string;
  message: UserMessage;
  tools?: AgentTool[];
  systemPrompt?: string;
  modelConfig?: ModelConfig;
  streamingOptions?: StreamingOptions;
}

export interface AgentRunResult {
  payloads: ResponsePayload[];
  meta: RunMeta;
  toolCalls?: ToolCallResult[];
}

export interface ResponsePayload {
  text?: string;
  mediaUrl?: string;
  mediaUrls?: string[];
  isError?: boolean;
}

// src/tools/types.ts
export interface AgentTool<TInput = unknown, TOutput = unknown> {
  name: string;
  description: string;
  schema: TSchema;
  execute: (input: TInput, context: ToolContext) => Promise<TOutput>;
  policy?: ToolPolicy;
}

export interface ToolContext {
  agentId: string;
  sessionKey: string;
  config: AgentConfig;
  abortSignal?: AbortSignal;
}

// src/llm/types.ts
export interface LLMProvider {
  id: string;
  models: ModelInfo[];
  authenticate: (config: AuthConfig) => Promise<AuthResult>;
  complete: (params: CompletionParams) => Promise<CompletionResult>;
  stream: (params: CompletionParams) => AsyncIterable<StreamChunk>;
}

export interface ModelConfig {
  provider: string;
  modelId: string;
  temperature?: number;
  maxTokens?: number;
  reasoningLevel?: 'off' | 'low' | 'medium' | 'high';
}

// src/streaming/types.ts
export interface StreamingOptions {
  onContentDelta?: (delta: string) => void;
  onToolStart?: (toolCall: ToolCallStart) => void;
  onToolResult?: (result: ToolCallResult) => void;
  onComplete?: (result: AgentRunResult) => void;
  blockReplyChunking?: BlockReplyChunking;
}
```

### 2.3 支援的 LLM Providers

| Provider | 優先級 | 來源檔案 |
|----------|--------|----------|
| OpenAI | ✅ 高 | `models-config.providers.ts` |
| Anthropic | ✅ 高 | `models-config.providers.ts` |
| Google Gemini | ✅ 高 | `providers/google-shared.ts` |
| Ollama | ✅ 中 | `models-config.providers.ts` |
| AWS Bedrock | ⚠️ 低 | `providers/bedrock.ts` |
| GitHub Copilot | ⚠️ 低 | `providers/github-copilot-*.ts` |

### 2.4 外部依賴

```json
{
  "dependencies": {
    "@sinclair/typebox": "^0.34.0",
    "openai": "^4.0.0",
    "@anthropic-ai/sdk": "^0.20.0",
    "@google/generative-ai": "^0.2.0",
    "ajv": "^8.17.0",
    "eventemitter3": "^5.0.0"
  },
  "peerDependencies": {
    "@aws-sdk/client-bedrock": "^3.0.0"
  }
}
```

---

## 3. dataagent-channels 詳細規劃

### 3.1 從 openclaw 抽取的模組

| 來源路徑 (openclaw) | 目標路徑 (dataagent-channels) | 說明 |
|---------------------|-------------------------------|------|
| `src/channels/plugins/types.ts` | `src/core/types.ts` | 主要類型定義 |
| `src/channels/plugins/types.core.ts` | `src/core/types.core.ts` | 核心類型 |
| `src/channels/plugins/types.adapters.ts` | `src/adapters/index.ts` | Adapter 介面 |
| `src/channels/plugins/catalog.ts` | `src/plugins/catalog.ts` | Plugin 註冊 |
| `src/channels/plugins/load.ts` | `src/plugins/loader.ts` | 動態載入 |
| `src/channels/registry.ts` | `src/core/registry.ts` | Channel 註冊表 |
| `src/routing/resolve-route.ts` | `src/routing/resolver.ts` | 訊息路由 |
| `src/routing/session-key.ts` | `src/routing/session-key.ts` | Session Key 解析 |
| `src/routing/bindings.ts` | `src/routing/bindings.ts` | Channel-Agent 綁定 |
| `src/channels/allowlist-match.ts` | `src/utils/allowlist.ts` | 存取控制 |
| `src/channels/chat-type.ts` | `src/utils/chat-type.ts` | 聊天類型 |

### 3.2 Channel Plugin 結構

每個 Channel Plugin 包含：

```
src/plugins/telegram/
├── index.ts              # Plugin 入口 & 匯出
├── adapter.ts            # TelegramChannelAdapter
├── bot.ts                # Grammy bot 初始化
├── message-handler.ts    # 訊息處理
├── send.ts               # 發送訊息
├── format.ts             # 訊息格式化
├── types.ts              # Telegram 特定類型
├── webhook.ts            # Webhook 處理
└── utils/
    ├── targets.ts        # 目標解析
    ├── probe.ts          # 健康檢查
    └── pairing.ts        # 設備配對
```

### 3.3 關鍵類型定義

```typescript
// src/core/types.ts
export type ChatChannelId =
  | 'telegram' | 'discord' | 'slack' | 'signal'
  | 'imessage' | 'line' | 'whatsapp' | 'web_widget'
  | 'messenger' | 'instagram';

export interface ChannelPlugin {
  id: ChatChannelId;
  meta: ChannelMeta;

  // 生命週期
  onLoad?: (deps: PluginLoadDeps) => Promise<void>;
  onUnload?: () => Promise<void>;

  // Adapters
  auth?: ChannelAuthAdapter;
  setup?: ChannelSetupAdapter;
  outbound?: ChannelOutboundAdapter;
  gateway?: ChannelGatewayAdapter;
  resolver?: ChannelResolverAdapter;
  config?: ChannelConfigAdapter;
  messaging?: ChannelMessagingAdapter;
  streaming?: ChannelStreamingAdapter;
  status?: ChannelStatusAdapter;
}

export interface ChannelMeta {
  id: ChatChannelId;
  name: string;
  icon: string;
  capabilities: ChannelCapabilities;
  setupRequired: boolean;
  description?: string;
}

export interface ChannelCapabilities {
  chatTypes: Array<'dm' | 'group' | 'thread'>;
  media?: boolean;
  reactions?: boolean;
  edit?: boolean;
  reply?: boolean;
  threads?: boolean;
  polls?: boolean;
  streaming?: boolean;
}

// src/adapters/index.ts
export interface ChannelAuthAdapter {
  login(config: unknown): Promise<AuthResult>;
  logout(): Promise<void>;
  validateCredentials(creds: unknown): Promise<boolean>;
}

export interface ChannelOutboundAdapter {
  send(params: SendParams): Promise<SendResult>;
  sendMedia(params: SendMediaParams): Promise<SendResult>;
  editMessage?(params: EditParams): Promise<void>;
  deleteMessage?(params: DeleteParams): Promise<void>;
}

export interface ChannelGatewayAdapter {
  start(): Promise<void>;
  stop(): Promise<void>;
  onMessage(handler: MessageHandler): void;
  getStatus(): ChannelStatus;
}

// src/routing/types.ts
export interface RouteResolution {
  agentId: string;
  accountId: string;
  sessionKey: string;
  chatType: 'dm' | 'group';
}

export interface MessageContext {
  channelId: ChatChannelId;
  accountId: string;
  chatId: string;
  userId: string;
  groupId?: string;
  threadId?: string;
  messageId: string;
  content: MessageContent;
  timestamp: Date;
}
```

### 3.4 Channel Plugins 優先級

| Channel | 優先級 | 來源目錄 | 外部依賴 |
|---------|--------|----------|----------|
| Web Widget | ✅ 關鍵 | 新建 | `socket.io` |
| LINE | ✅ 關鍵 | `src/line/` | `@line/bot-sdk` |
| Telegram | ✅ 高 | `src/telegram/` | `grammy` |
| WhatsApp | ⚠️ 中 | `src/web/`, `src/whatsapp/` | `@whiskeysockets/baileys` |
| Discord | ⚠️ 中 | `src/discord/` | `discord.js` |
| Slack | ⚠️ 低 | `src/slack/` | `@slack/bolt` |
| Messenger | ⚠️ 低 | 新建 | Facebook Graph API |

### 3.5 Web Widget 新建規劃

```typescript
// src/plugins/web-widget/index.ts
export const webWidgetPlugin: ChannelPlugin = {
  id: 'web_widget',
  meta: {
    name: 'Web Widget',
    icon: '🌐',
    capabilities: {
      chatTypes: ['dm'],
      media: true,
      streaming: true,
    },
    setupRequired: false,
  },

  gateway: {
    // Socket.IO based real-time communication
    start: async () => { /* ... */ },
    stop: async () => { /* ... */ },
    onMessage: (handler) => { /* ... */ },
  },

  outbound: {
    send: async (params) => { /* ... */ },
    sendMedia: async (params) => { /* ... */ },
  },
};

// Widget SDK (for embedding)
export interface WidgetConfig {
  agentId: string;
  apiUrl: string;
  theme?: WidgetTheme;
  position?: 'bottom-right' | 'bottom-left';
  welcomeMessage?: string;
}

export function initWidget(config: WidgetConfig): WidgetInstance;
```

---

## 4. 開發計畫

### Phase 0: 專案初始化 (Week 1)

| 任務 | 產出 |
|------|------|
| 建立 `dataagent-agent-core` 專案結構 | TypeScript monorepo |
| 建立 `dataagent-channels` 專案結構 | TypeScript monorepo |
| 設定 build 工具 (tsup/esbuild) | 可打包發布 |
| 設定測試框架 (vitest) | 測試環境就緒 |
| 建立 CI/CD (GitHub Actions) | 自動化測試 |

### Phase 1: Agent Core 抽取 (Week 2-3)

| 任務 | 來源 | 產出 |
|------|------|------|
| LLM Provider 抽象層 | `model-*.ts` | `src/llm/` |
| OpenAI Provider | `models-config.providers.ts` | `src/llm/providers/openai.ts` |
| Anthropic Provider | `models-config.providers.ts` | `src/llm/providers/anthropic.ts` |
| Tool 系統 | `pi-tools*.ts` | `src/tools/` |
| System Prompt Builder | `system-prompt*.ts` | `src/prompt/` |
| Streaming Handler | `pi-embedded-subscribe*.ts` | `src/streaming/` |
| Agent Runner | `pi-embedded-runner/` | `src/agent/` |
| 單元測試 | - | 80%+ coverage |

### Phase 2: Channels 核心抽取 (Week 4-5)

| 任務 | 來源 | 產出 |
|------|------|------|
| Channel 類型定義 | `plugins/types*.ts` | `src/core/` |
| Adapter 介面 | `plugins/types.adapters.ts` | `src/adapters/` |
| Plugin 系統 | `plugins/catalog.ts`, `load.ts` | `src/plugins/` |
| 路由系統 | `routing/*.ts` | `src/routing/` |
| 存取控制 | `allowlist*.ts` | `src/utils/` |
| 單元測試 | - | 80%+ coverage |

### Phase 3: Priority Channels (Week 6-8)

| 任務 | 優先級 | 產出 |
|------|--------|------|
| Web Widget Plugin | ✅ 關鍵 | `src/plugins/web-widget/` |
| LINE Plugin | ✅ 關鍵 | `src/plugins/line/` |
| Telegram Plugin | ✅ 高 | `src/plugins/telegram/` |
| 整合測試 | - | E2E 測試 |

### Phase 4: 整合與發布 (Week 9)

| 任務 | 產出 |
|------|------|
| 與 DataAgent AI Agent Platform 整合測試 | 整合驗證 |
| 文件撰寫 | README, API docs |
| npm 發布 | `@dataagent/agent-core`, `@dataagent/channels` |

---

## 5. 技術決策

### Q1: TypeScript 還是 Python?

**決定: TypeScript**

理由:
1. openclaw 現有程式碼是 TypeScript
2. Channel SDKs (grammy, @line/bot-sdk) 都是 JS/TS 生態
3. 可直接複製/修改現有程式碼
4. 前端 Widget SDK 需要 TypeScript

### Q2: Monorepo 還是 Multi-repo?

**決定: 兩個獨立 Repo**

```
dataagent-agent-core/    # 獨立 repo
dataagent-channels/      # 獨立 repo (可單獨使用)
```

理由:
1. Agent Core 和 Channels 可獨立使用
2. 版本可獨立演進
3. 不同專案可選擇性安裝

### Q3: 如何處理 @mariozechner/pi-* 依賴?

**決定: 重新實作核心邏輯，不依賴 pi-* 套件**

理由:
1. pi-* 是 openclaw 特定實作
2. 減少外部依賴
3. 更好的控制權

### Q4: 如何與 DataAgent AI Agent Platform (Python) 整合?

**方案: 作為獨立服務運行**

```
┌─────────────────────────────────────────┐
│ DataAgent AI Agent Platform (Python)     │
│   FastAPI Backend                        │
└─────────────────────────────────────────┘
          │ HTTP/WebSocket
          ▼
┌─────────────────────────────────────────┐
│ Agent & Channel Service (TypeScript)     │
│   Express + Socket.IO                    │
│   - dataagent-agent-core                 │
│   - dataagent-channels                   │
└─────────────────────────────────────────┘
```

或者使用 **Bridge 模式**:
- Python 呼叫 Node.js subprocess
- 透過 stdin/stdout 或 Unix socket 通訊

---

## 6. 風險與緩解

| 風險 | 影響 | 緩解措施 |
|------|------|----------|
| pi-* 依賴難以移除 | 高 | 逐步重新實作，保持 API 相容 |
| Channel SDK 版本不相容 | 中 | 鎖定版本，建立相容性矩陣 |
| 整合複雜度高 | 中 | 先完成 Web Widget，驗證架構 |
| 測試覆蓋不足 | 中 | 設定 80% coverage 門檻 |

---

## 7. 成功指標

| 指標 | 目標 |
|------|------|
| Agent Core 可獨立執行 | ✅ |
| 支援 3+ LLM Providers | OpenAI, Anthropic, Gemini |
| 支援 3+ Channels | Web Widget, LINE, Telegram |
| 測試覆蓋率 | ≥ 80% |
| npm 發布 | @dataagent/* |
| 與 Platform 整合 | E2E 驗證通過 |

---

*此文件定義了從 openclaw 抽取 Core Agent 和 Multi-Channel Messaging 的完整計畫。執行此計畫後，DataAgent AI Agent Platform 可使用這些獨立模組，避免重複開發。*
