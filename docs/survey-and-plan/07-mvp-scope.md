# MVP Scope Definition (Revised)

> **Purpose:** Define minimum viable product for initial customer testing
> **Target Launch:** 2-3 months from start (accelerated by reusing domain-specific-rag-app)
> **Revision Date:** 2026-02-01

---

## 1. MVP Philosophy (Revised)

### 1.1 Core Principles

1. **功能優先** - 先完成 AI 客服核心功能，營運商務功能後補
2. **重用現有組件** - 最大化利用 domain-specific-rag-app 的 RAG 管線
3. **最小完整流程** - 一個品牌能完成渠道串接、AI 回覆、後台查看對話
4. **為擴展預留空間** - 架構支持未來加入多租戶、計費系統

### 1.2 MVP 核心目標

> **一個介面可成功進行一次渠道串接、讓 AI Agent 幫品牌回第一次訊息、有被記錄下來並顯示在後台**

### 1.3 Success Criteria

| Metric | Target |
|--------|--------|
| Time to first AI response | < 30 minutes (setup) |
| AI resolution rate | > 50% |
| Setup completion rate | > 80% |
| 單一品牌可串接渠道 | ≥ 1 |

### 1.4 Deferred to Post-MVP

| Feature | Reason | Priority |
|---------|--------|----------|
| User Registration/Login | 營運商務面，非功能核心 | Post-MVP Phase 1 |
| Workspace/Team Management | 營運商務面 | Post-MVP Phase 1 |
| Billing/Subscription | 營運商務面 | Post-MVP Phase 2 |
| Multi-tenant Isolation | 依賴 User/Workspace | Post-MVP Phase 1 |

---

## 2. MVP Feature Scope (Revised)

### 2.1 Included (Must Have)

#### Agent Configuration
| Feature | Description | Priority |
|---------|-------------|----------|
| Single Agent Config | 一個 AI 客服 Agent 配置 | ✅ |
| Agent Name/Avatar | 基本品牌化 | ✅ |
| Greeting Message | 可配置歡迎語 | ✅ |
| Fallback Behavior | AI 無法回答時的處理 | ✅ |
| System Prompt | 可自訂 Agent 人設/指令 | ✅ |

#### Knowledge Base (重用 domain-specific-rag-app)
| Feature | Description | Reuse From | Priority |
|---------|-------------|------------|----------|
| Document Upload | PDF, DOCX, TXT, MD | batch_processor.py | ✅ |
| URL Import | 網頁爬取並索引 | crawler.py, web_processor.py | ✅ |
| Manual FAQ | 手動新增 Q&A | New (simple) | ✅ |
| RAG Pipeline | 檢索增強生成 | optimized_rag_pipeline.py | ✅ |
| Hybrid Retrieval | Dense + BM25 | hybrid_retriever.py | ✅ |
| Preview Responses | 上線前測試 AI 回覆 | New (UI) | ✅ |

#### Channels
| Feature | Description | Priority |
|---------|-------------|----------|
| Web Widget | 嵌入式聊天視窗 | ✅ |
| LINE Official Account | LINE Login 連接 | ✅ |
| WhatsApp (Optional) | Cloud API 基礎整合 | ⚠️ Nice to have |

#### Conversations
| Feature | Description | Priority |
|---------|-------------|----------|
| Unified Inbox | 所有對話列表 | ✅ |
| Conversation History | 完整對話記錄 | ✅ |
| Customer Info | 基本客戶資訊 | ✅ |
| Manual Reply | 人工可隨時回覆 | ✅ |
| Basic Handover | AI → 人工轉接標記 | ✅ |

#### Analytics (Basic)
| Feature | Description | Priority |
|---------|-------------|----------|
| Conversation Count | 對話總數 | ✅ |
| Message Volume | 訊息量趨勢 | ✅ |
| AI vs Human Ratio | AI/人工回覆比例 | ✅ |
| Daily Chart | 基本日趨勢圖 | ✅ |

### 2.2 Explicitly Excluded (Post-MVP)

| Feature | Moved To | Reason |
|---------|----------|--------|
| User Registration/Login | 08-post-mvp-scope.md | 營運商務面 |
| Workspace Management | 08-post-mvp-scope.md | 營運商務面 |
| Team Invite/Roles | 08-post-mvp-scope.md | 營運商務面 |
| Billing/Subscription | 08-post-mvp-scope.md | 營運商務面 |
| Usage Metering | 08-post-mvp-scope.md | 依賴 Billing |
| SSO/SAML | 08-post-mvp-scope.md | 企業功能 |
| Product Catalog | 08-post-mvp-scope.md | 電商進階功能 |
| Custom API Tools | 08-post-mvp-scope.md | 進階功能 |
| Advanced Analytics | 08-post-mvp-scope.md | CSAT, Resolution Rate |
| Voice AI | 08-post-mvp-scope.md | 複雜整合 |

---

## 3. Reusing domain-specific-rag-app

### 3.1 Component Mapping

```
domain-specific-rag-app          →    openclaw-ai-agents-platform
─────────────────────────────────────────────────────────────────────
src/ingestion/batch_processor.py →    Knowledge Base: PDF processing
src/ingestion/parent_child_chunker.py → Knowledge Base: Chunking
src/retrieval/vector_store.py    →    Knowledge Base: Vector storage
src/retrieval/hybrid_retriever.py →   Knowledge Base: Retrieval
src/retrieval/reranker.py        →    Knowledge Base: Reranking
src/retrieval/query_expander.py  →    Knowledge Base: Query expansion
src/generation/optimized_rag_pipeline.py → Agent: RAG pipeline
src/crawling/crawler.py          →    Knowledge Base: URL import
src/crawling/web_processor.py    →    Knowledge Base: Web content
src/conversation/memory.py       →    Conversations: Context memory
src/utils.py (get_llm)           →    Agent: LLM factory
```

### 3.2 Adaptation Needed

| Component | Adaptation |
|-----------|------------|
| Vector Store | 改為 per-agent 的 collection 命名 |
| Conversation Memory | 加入 customer_id, conversation_id |
| RAG Pipeline | 加入 agent_config 注入 |
| Web Processor | 加入 agent_id metadata |

### 3.3 Shared Library Strategy

**建議：創建共享 Python 套件**

```
packages/
  dataagent-rag-core/                 # 共享 RAG 核心
    src/
      ingestion/            # 從 domain-specific-rag-app 複製
      retrieval/
      generation/
      crawling/
    pyproject.toml
```

兩個專案都依賴這個共享套件，避免程式碼重複。

---

## 4. Repository Strategy

### 4.1 建議方案：獨立專案 + 共享套件

```
~/dev/
  domain-specific-rag-app/          # 原有專案 (保持獨立)
  openclaw-ai-agents-platform/   # 新專案 (獨立開發)
    packages/
      dataagent-rag-core/                     # 從 domain-specific-rag-app 抽出
    apps/
      api/                          # Backend API
      web/                          # Frontend Dashboard
    ...
```

**理由：**
1. **關注點分離** - domain-specific-rag-app 專注護理領域，新平台是通用 SaaS
2. **獨立演進** - 兩個專案可以不同步更新
3. **共享核心** - RAG 核心抽成套件避免重複
4. **清楚邊界** - 明確哪些是共享、哪些是專案特定

### 4.2 維護策略

| 場景 | 策略 |
|------|------|
| RAG 核心改進 | 在 dataagent-rag-core 套件中改，兩邊都受益 |
| 護理特定功能 | 只在 domain-specific-rag-app |
| 客服平台功能 | 只在 openclaw-ai-agents-platform |
| Bug fixes in RAG | 修 dataagent-rag-core，發新版本 |

### 4.3 為什麼不在現有 openclaw 下開發？

| 選項 | 優點 | 缺點 |
|------|------|------|
| 在 openclaw 下開發 | 重用現有 channel plugins | 職責混亂、Python + Node 混合 |
| **新開 openclaw-ai-agents-platform** | 清楚架構、專注 SaaS | 需重建部分 channel 整合 |

**建議：新開專案**，但 channel plugins 可從 openclaw 複製/改寫。

---

## 5. Technical Architecture (MVP - Simplified)

### 5.1 System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         Frontend                             │
│  ┌──────────────┐  ┌──────────────┐                         │
│  │ Dashboard UI │  │ Widget SDK   │                         │
│  │ (Next.js)    │  │ (React)      │                         │
│  └──────────────┘  └──────────────┘                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      API Layer                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ FastAPI (Python)                                     │   │
│  │ - Agent API                                          │   │
│  │ - Knowledge Base API                                 │   │
│  │ - Conversation API                                   │   │
│  │ - Channel Webhooks                                   │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ RAG Core        │  │ Channel Service │  │ Agent Service   │
│ (dataagent-rag-core pkg)  │  │ (LINE, Widget)  │  │ (LLM + Tools)   │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Data Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ PostgreSQL   │  │ Qdrant       │  │ S3/R2        │       │
│  │ (Metadata)   │  │ (Vectors)    │  │ (Files)      │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Tech Stack (MVP - Simplified)

| Layer | Technology | Rationale |
|-------|------------|-----------|
| Frontend | Next.js 14+ (App Router) | React, 現代化 |
| UI Components | shadcn/ui + Tailwind | 快速開發 |
| **Backend** | **FastAPI (Python)** | 與 dataagent-rag-core 相容 |
| RAG Core | 重用 domain-specific-rag-app | 節省開發時間 |
| Vector DB | **Qdrant** (已有) | 無需遷移 |
| Metadata DB | PostgreSQL | 標準選擇 |
| File Storage | Cloudflare R2 | 成本效益 |

**注意：選擇 Python (FastAPI) 而非 Node.js，以便直接重用 RAG 組件。**

### 5.3 Database Schema (MVP - Simplified)

```sql
-- MVP: 無多租戶，單一品牌使用

-- Agent 配置
CREATE TABLE agents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  avatar_url VARCHAR(500),
  greeting_message TEXT,
  fallback_message TEXT,
  system_prompt TEXT,
  config JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 知識庫
CREATE TABLE knowledge_bases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID REFERENCES agents(id),
  name VARCHAR(255) NOT NULL,
  qdrant_collection VARCHAR(255), -- Qdrant collection 名稱
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 知識庫文件
CREATE TABLE kb_documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  kb_id UUID REFERENCES knowledge_bases(id),
  title VARCHAR(255),
  source_type VARCHAR(50), -- 'pdf', 'url', 'faq'
  source_path VARCHAR(500),
  status VARCHAR(50) DEFAULT 'processing',
  chunk_count INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 渠道配置
CREATE TABLE channels (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID REFERENCES agents(id),
  type VARCHAR(50) NOT NULL, -- 'web_widget', 'line'
  config JSONB NOT NULL,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 客戶
CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  channel_type VARCHAR(50),
  channel_customer_id VARCHAR(255), -- LINE user ID, etc.
  name VARCHAR(255),
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(channel_type, channel_customer_id)
);

-- 對話
CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID REFERENCES agents(id),
  customer_id UUID REFERENCES customers(id),
  channel_id UUID REFERENCES channels(id),
  status VARCHAR(50) DEFAULT 'open', -- 'open', 'handover', 'resolved'
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 訊息
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID REFERENCES conversations(id),
  sender_type VARCHAR(50) NOT NULL, -- 'customer', 'ai', 'agent'
  content TEXT,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 6. API Endpoints (MVP - Simplified)

```yaml
# Agent
GET    /api/agents                    # List agents
POST   /api/agents                    # Create agent
GET    /api/agents/:id                # Get agent
PATCH  /api/agents/:id                # Update agent

# Knowledge Base
GET    /api/agents/:id/knowledge-base # Get KB info
POST   /api/agents/:id/knowledge-base/documents  # Upload document
GET    /api/agents/:id/knowledge-base/documents  # List documents
DELETE /api/agents/:id/knowledge-base/documents/:docId
POST   /api/agents/:id/knowledge-base/test       # Test query

# Channels
GET    /api/agents/:id/channels       # List channels
POST   /api/agents/:id/channels       # Add channel
PATCH  /api/agents/:id/channels/:channelId
DELETE /api/agents/:id/channels/:channelId

# Conversations
GET    /api/conversations             # List all
GET    /api/conversations/:id         # Get conversation
POST   /api/conversations/:id/messages  # Send message (human reply)
PATCH  /api/conversations/:id/status  # Change status (handover, resolve)

# Analytics
GET    /api/analytics/overview        # Basic stats

# Webhooks (Channel inbound)
POST   /api/webhooks/line             # LINE webhook
POST   /api/webhooks/widget           # Widget messages
```

---

## 7. User Journey (MVP - Simplified)

### 7.1 Setup Flow (無登入)

```
1. 訪問 Dashboard
   └─> 直接進入設定頁面 (MVP 無需登入)

2. Agent 設定
   └─> 設定名稱、頭像、歡迎語、人設

3. 知識庫設定
   └─> 上傳 PDF/文件
   └─> 或輸入網址讓系統爬取
   └─> 或手動輸入 FAQ
   └─> 測試 AI 回覆

4. 渠道連接
   └─> Web Widget: 複製嵌入代碼
   └─> LINE: QR Code 掃描連接

5. 測試對話
   └─> 使用內建測試介面
   └─> 確認 AI 回覆正確

6. 上線
   └─> 啟用渠道
   └─> 客戶開始與 AI 對話

時間目標: < 30 分鐘
```

### 7.2 Daily Usage Flow

```
1. 開啟 Dashboard
   └─> 查看基本統計

2. 檢視對話
   └─> 查看 AI 自動處理的對話
   └─> 處理標記為 "需人工" 的對話
   └─> 必要時手動回覆

3. 優化知識庫
   └─> 查看 AI 無法回答的問題
   └─> 補充相關文件/FAQ
```

---

## 8. Development Phases (Revised v2)

> **重要更新 (2026-02-02):** 新增 Pre-Phase: 從 openclaw 抽取 Core Agent & Channels 模組
> 詳細規劃請參考: [09-openclaw-core-extraction.md](./09-openclaw-core-extraction.md)

### 8.0 Pre-Phase: Core Module Extraction (Week 1-4)

在進入 MVP 開發前，需先從 openclaw 抽取以下核心模組為獨立套件：

#### 8.0.1 dataagent-agent-core (TypeScript)

| 模組 | 來源 (openclaw) | 說明 |
|------|-----------------|------|
| Agent Runner | `src/agents/pi-embedded-runner/` | Agent 執行引擎 |
| Tool System | `src/agents/pi-tools*.ts` | Tool 工廠 + 存取控制 |
| LLM Providers | `src/agents/model-*.ts` | OpenAI, Anthropic, Gemini |
| System Prompt | `src/agents/system-prompt*.ts` | Prompt 建構器 |
| Streaming | `src/agents/pi-embedded-subscribe*.ts` | 串流回應處理 |
| Memory | `src/memory/*.ts` | Context 管理 |

**產出:** `@dataagent/agent-core` npm 套件

#### 8.0.2 dataagent-channels (TypeScript)

| 模組 | 來源 (openclaw) | 說明 |
|------|-----------------|------|
| Channel Abstraction | `src/channels/plugins/types*.ts` | 通用介面定義 |
| Plugin System | `src/channels/plugins/catalog.ts` | Plugin 註冊 & 載入 |
| Routing | `src/routing/*.ts` | 訊息路由 |
| Web Widget | 新建 | 嵌入式聊天 (Socket.IO) |
| LINE Plugin | `src/line/` | LINE Official Account |
| Telegram Plugin | `src/telegram/` | Telegram Bot |

**產出:** `@dataagent/channels` npm 套件

#### 8.0.3 Pre-Phase 時程

| Week | 任務 | 產出 |
|------|------|------|
| Week 1 | 專案初始化 + LLM Provider 抽取 | 專案結構、OpenAI/Anthropic provider |
| Week 2 | Tool System + Streaming 抽取 | Tool 工廠、串流處理器 |
| Week 3 | Channel 核心 + Web Widget | Channel 抽象層、Widget SDK |
| Week 4 | LINE + Telegram Plugin | Channel plugins、整合測試 |

**Milestone:** 兩個獨立 npm 套件可安裝使用

---

### 8.1 Phase 0: Platform Foundation (Week 5) ✅ 已完成

| Task | Output | Status |
|------|--------|--------|
| 專案結構建立 | openclaw-ai-agents-platform monorepo | ✅ |
| dataagent-rag-core 套件抽取 | 從 domain-specific-rag-app 複製核心 | 🔄 待執行 |
| PostgreSQL + Qdrant 設定 | Docker compose | ✅ |
| FastAPI 骨架 | 基本 API 結構 | ✅ |
| Next.js 骨架 | Dashboard 基本結構 | ✅ |

**Milestone:** 開發環境就緒 ✅

### 8.2 Phase 1: Knowledge Base (Week 6-7)

| Task | Output |
|------|--------|
| 文件上傳 API | S3 儲存 + 處理佇列 |
| PDF 處理整合 | 重用 batch_processor |
| URL 爬取整合 | 重用 crawler + web_processor |
| FAQ 手動輸入 | 簡單 CRUD |
| RAG Pipeline 整合 | 重用 optimized_rag_pipeline |
| KB 管理 UI | 上傳、列表、刪除 |
| 測試介面 | 在 Dashboard 測試 AI 回覆 |

**Milestone:** 可上傳文件、測試 AI 回覆

### 8.3 Phase 2: Agent & Channels (Week 8-9)

| Task | Output |
|------|--------|
| Agent 配置 API | CRUD for agent settings |
| Agent 配置 UI | 名稱、頭像、人設設定 |
| **整合 dataagent-agent-core** | Agent 執行引擎整合 |
| **整合 dataagent-channels** | Web Widget + LINE 整合 |
| Widget 安裝頁面 | 複製嵌入代碼 |
| LINE 設定 UI | 連接、狀態、啟用/停用 |

**Milestone:** 可串接 Web Widget 和 LINE

### 8.4 Phase 3: Conversations & Handover (Week 10-11)

| Task | Output |
|------|--------|
| 對話儲存 | 訊息寫入 PostgreSQL |
| 統一收件箱 UI | 對話列表 + 篩選 |
| 對話詳情 UI | 訊息歷史 + 客戶資訊 |
| 人工回覆功能 | 從 Dashboard 發送訊息 |
| 轉接標記 | AI → 人工 狀態切換 |
| 即時更新 | WebSocket 推送新訊息 |

**Milestone:** 可在後台查看對話、人工回覆

### 8.5 Phase 4: Analytics & Polish (Week 12)

| Task | Output |
|------|--------|
| 基本統計 API | 對話數、訊息量 |
| 統計 Dashboard | 簡單圖表 |
| Bug 修復 | 測試回饋處理 |
| 效能優化 | 慢查詢優化 |
| 文件撰寫 | 使用說明 |

**Milestone:** MVP 完成，可交付測試

---

## 9. Key Decisions Summary

### Q1: 用 domain-specific-rag-app 作為基礎？
**A:** Yes，將核心 RAG 組件抽成 dataagent-rag-core 套件重用。

### Q2: 維護策略？
**A:** 獨立專案 + 共享套件。RAG 核心改進放 dataagent-rag-core，兩邊都受益。

### Q3: 新開資料夾還是在 openclaw 下開發？
**A:** 新開 `openclaw-ai-agents-platform` 專案。理由：
- 職責清楚分離
- Python 為主 (與 RAG 相容) vs openclaw 是 Node.js
- 避免混亂

### Q4: User/Workspace/Billing 先往後放？
**A:** Yes，移至 08-post-mvp-scope.md。功能優先，營運商務後補。

### Q5: 為什麼要先抽取 openclaw 核心模組？
**A:** 因為：
- openclaw 已有成熟的 Agent 執行引擎、Tool 系統、多 Channel 支援
- 直接重用可節省 2-3 個月開發時間
- 抽成獨立套件後，其他專案也可使用
- TypeScript 套件可透過獨立服務或 Bridge 模式與 Python 後端整合

詳細規劃見: [09-openclaw-core-extraction.md](./09-openclaw-core-extraction.md)

---

## 10. Timeline Summary (Revised)

| Phase | Duration | Output |
|-------|----------|--------|
| **Pre-Phase: Core Extraction** | Week 1-4 | dataagent-agent-core, dataagent-channels |
| Phase 0: Foundation | Week 5 | 專案結構、環境 ✅ |
| Phase 1: Knowledge Base | Week 6-7 | KB 功能完整 |
| Phase 2: Agent & Channels | Week 8-9 | 渠道串接 |
| Phase 3: Conversations | Week 10-11 | 對話管理 |
| Phase 4: Polish | Week 12 | MVP 完成 |

**Total: 12 週 (3 個月)**

> 注意：新增 Pre-Phase 使總時程從 8 週延長至 12 週，但這是必要的基礎建設，可讓後續開發更快速且可維護。

---

## 11. Team Requirements (MVP)

| Role | Count | Responsibilities |
|------|-------|------------------|
| Full-stack Engineer | 1-2 | Backend (FastAPI) + Frontend (Next.js) |
| Founder/PM | 1 | 方向、測試、客戶開發 |

**Total:** 2-3 人

---

*This revised MVP focuses on delivering the core AI customer service functionality first, leveraging the existing domain-specific-rag-app components to accelerate development. User management and billing are deferred to post-MVP to focus on validating the core value proposition.*
