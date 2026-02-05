# Post-MVP Scope Definition

> **Purpose:** Define features deferred from MVP for future development phases
> **Created:** 2026-02-01

---

## 1. Post-MVP Philosophy

MVP 完成後，我們已驗證核心功能：
- AI Agent 可以透過知識庫回覆客戶問題
- 渠道串接正常運作 (Web Widget, LINE)
- 對話可被記錄並在後台查看

Post-MVP 階段專注於：
1. **Phase 1:** 營運商務基礎 (User, Workspace, Billing)
2. **Phase 2:** 功能擴展 (更多渠道、電商整合)
3. **Phase 3:** 企業級功能 (合規、進階分析)

---

## 2. Post-MVP Phase 1: 營運商務基礎 (4-6 週)

### 2.1 User & Authentication

| Feature | Description | Priority |
|---------|-------------|----------|
| Email Registration | Email/Password 註冊 | ✅ |
| Email Verification | 驗證郵件 | ✅ |
| Google OAuth | 一鍵 Google 登入 | ✅ |
| Login/Logout | 登入登出流程 | ✅ |
| Password Reset | 忘記密碼重設 | ✅ |
| Session Management | JWT + Refresh Token | ✅ |

**Tech Stack:**
- NextAuth.js 或 Lucia Auth
- PostgreSQL (users table)
- Email: Resend 或 SendGrid

### 2.2 Workspace & Team Management

| Feature | Description | Priority |
|---------|-------------|----------|
| Workspace Creation | 建立工作空間 | ✅ |
| Workspace Settings | 名稱、時區、語言 | ✅ |
| Team Invite | 邀請成員 (Email) | ✅ |
| Role Management | Admin, Agent, Viewer | ✅ |
| Member List | 查看/移除成員 | ✅ |
| Workspace Switch | 切換多個工作空間 | ⚠️ |

**Database Schema Addition:**
```sql
CREATE TABLE workspaces (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  owner_id UUID REFERENCES users(id),
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE workspace_members (
  workspace_id UUID REFERENCES workspaces(id),
  user_id UUID REFERENCES users(id),
  role VARCHAR(50) DEFAULT 'member',
  invited_at TIMESTAMPTZ DEFAULT NOW(),
  joined_at TIMESTAMPTZ,
  PRIMARY KEY (workspace_id, user_id)
);

-- 更新 agents, channels, conversations 加入 workspace_id
ALTER TABLE agents ADD COLUMN workspace_id UUID REFERENCES workspaces(id);
ALTER TABLE channels ADD COLUMN workspace_id UUID REFERENCES workspaces(id);
ALTER TABLE conversations ADD COLUMN workspace_id UUID REFERENCES workspaces(id);
```

### 2.3 Multi-Tenant Data Isolation

| Feature | Description | Priority |
|---------|-------------|----------|
| Row-Level Security | PostgreSQL RLS policies | ✅ |
| Workspace Context | API 層 workspace 驗證 | ✅ |
| Qdrant Collection per Workspace | 向量資料隔離 | ✅ |
| File Storage Isolation | S3 前綴隔離 | ✅ |

**Implementation:**
```sql
-- PostgreSQL RLS
ALTER TABLE agents ENABLE ROW LEVEL SECURITY;
CREATE POLICY workspace_isolation ON agents
  USING (workspace_id = current_setting('app.workspace_id')::UUID);
```

### 2.4 Billing & Subscription (Basic)

| Feature | Description | Priority |
|---------|-------------|----------|
| Free Tier | 100 resolutions/月 | ✅ |
| Usage Tracking | Resolution 計數 | ✅ |
| Usage Dashboard | 使用量顯示 | ✅ |
| Upgrade Prompt | 接近上限提醒 | ✅ |
| Stripe Integration | 付款連接 | ✅ |
| Plan Selection | Free → Starter → Pro | ✅ |
| Invoice History | 發票記錄 | ⚠️ |

**Pricing Tiers (Initial):**

| Plan | Price | Resolutions | Channels | Users |
|------|-------|-------------|----------|-------|
| Free | $0 | 100/月 | 1 | 1 |
| Starter | $49/月 | 500/月 | 3 | 3 |
| Pro | $149/月 | 2,000/月 | All | 10 |

**Database Schema Addition:**
```sql
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY,
  workspace_id UUID REFERENCES workspaces(id),
  plan VARCHAR(50) DEFAULT 'free',
  stripe_customer_id VARCHAR(255),
  stripe_subscription_id VARCHAR(255),
  status VARCHAR(50) DEFAULT 'active',
  current_period_start TIMESTAMPTZ,
  current_period_end TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE usage_records (
  id UUID PRIMARY KEY,
  workspace_id UUID REFERENCES workspaces(id),
  type VARCHAR(50), -- 'resolution', 'message'
  count INTEGER DEFAULT 1,
  recorded_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 3. Post-MVP Phase 2: 功能擴展 (6-8 週)

### 3.1 Additional Channels

| Channel | Priority | Effort | Market |
|---------|----------|--------|--------|
| Facebook Messenger | ✅ High | 2 週 | 全球 |
| Instagram DM | ✅ High | 2 週 | 全球 |
| Telegram | ⚠️ Medium | 1 週 | 俄語/東歐 |
| WeChat | ⚠️ Medium | 3 週 | 中國 |
| Zalo | ⚠️ Medium | 2 週 | 越南 |

**Implementation per Channel:**
1. OAuth/API 連接
2. Webhook 處理
3. 訊息發送
4. Rich Message 支援 (按鈕、卡片)
5. 設定 UI

### 3.2 E-commerce Integrations

| Integration | Features | Priority |
|-------------|----------|----------|
| Shopify | 商品同步、訂單查詢 | ✅ High |
| WooCommerce | 商品同步、訂單查詢 | ⚠️ Medium |
| Custom API | 通用 REST 連接 | ⚠️ Medium |

**Shopify Integration:**
```
1. Shopify App 註冊
2. OAuth 安裝流程
3. Product sync (Webhook + Polling)
4. Order lookup API
5. Agent Tool: "Check order status"
6. Agent Tool: "Search products"
```

### 3.3 Product Catalog System

| Feature | Description | Priority |
|---------|-------------|----------|
| CSV Import | 上傳商品 CSV | ✅ |
| Product Search Tool | AI 可搜尋商品 | ✅ |
| Inventory Query | 庫存查詢 | ⚠️ |
| Price Lookup | 價格查詢 | ⚠️ |
| Product Recommendations | AI 推薦商品 | ⚠️ |

### 3.4 Custom API Tools

| Feature | Description | Priority |
|---------|-------------|----------|
| API Spec Import | 上傳 OpenAPI/Swagger | ✅ |
| Visual API Builder | 視覺化 API 配置 | ✅ |
| Auth Config | API Key, OAuth, JWT | ✅ |
| Dynamic Tool Gen | 自動生成 Agent Tool | ✅ |
| Test Sandbox | 測試 API 呼叫 | ✅ |

**Use Cases:**
- 查詢訂單狀態 (ERP/CRM API)
- 查詢會員資料
- 查詢庫存
- 下訂單/取消訂單

### 3.5 Advanced Analytics

| Feature | Description | Priority |
|---------|-------------|----------|
| CSAT Collection | 對話後滿意度調查 | ✅ |
| AI CSAT (Inferred) | 根據對話推測滿意度 | ⚠️ |
| Resolution Rate | AI/人工解決率 | ✅ |
| Response Time | 平均回覆時間 | ✅ |
| Knowledge Gaps | 未解答問題分析 | ✅ |
| Custom Reports | 自訂報表 | ⚠️ |
| Data Export | CSV/API 匯出 | ✅ |

---

## 4. Post-MVP Phase 3: 企業級功能 (8-12 週)

### 4.1 Enterprise Authentication

| Feature | Description | Priority |
|---------|-------------|----------|
| SAML SSO | 企業 IdP 整合 | ⚠️ |
| SCIM Provisioning | 自動用戶同步 | ⚠️ |
| MFA | 多因子驗證 | ⚠️ |
| IP Allowlist | IP 白名單 | ⚠️ |

### 4.2 Compliance & Security

| Feature | Description | Priority |
|---------|-------------|----------|
| SOC 2 Type II | 合規認證 | ⚠️ |
| GDPR Tools | 資料匯出、刪除 | ✅ |
| Data Retention | 可配置保留期限 | ✅ |
| Audit Logs | 操作記錄 | ✅ |
| PII Masking | 敏感資料遮蔽 | ⚠️ |
| Encryption at Rest | 資料加密 | ⚠️ |

### 4.3 Advanced Human Handover

| Feature | Description | Priority |
|---------|-------------|----------|
| Agent Queue | 排隊等候 | ✅ |
| Skill-Based Routing | 技能路由 | ⚠️ |
| Agent Availability | 上線狀態管理 | ✅ |
| Canned Responses | 快速回覆範本 | ✅ |
| AI Suggested Replies | AI 建議回覆 | ⚠️ |
| Transfer Notes | 轉接備註 | ✅ |

### 4.4 Customer Data Platform (CDP)

| Feature | Description | Priority |
|---------|-------------|----------|
| Unified Customer Profile | 跨渠道合併 | ✅ |
| Custom Fields | 自訂屬性 | ✅ |
| Tags & Segments | 標籤與分群 | ⚠️ |
| Customer Timeline | 互動時間軸 | ⚠️ |
| External Profile Sync | CRM 同步 | ⚠️ |

### 4.5 Voice AI (Future)

| Feature | Description | Priority |
|---------|-------------|----------|
| Inbound Voice | 接聽電話 | Low |
| Speech-to-Text | 語音轉文字 | Low |
| Text-to-Speech | 文字轉語音 | Low |
| Voice Agent | 語音 AI 客服 | Low |
| IVR Integration | 整合現有 IVR | Low |

---

## 5. Feature Priority Matrix

### By Business Value vs Implementation Effort

```
High Value
    │
    │  ★ User/Auth        ★ Shopify          ★ CSAT
    │  ★ Workspace        ★ Custom API
    │  ★ Billing          ★ FB/IG Channels
    │                                          ★ CDP
    │                     ★ Advanced          ★ SOC 2
    │                       Analytics
    │                                          ★ Voice AI
    │                     ★ SSO/SAML
    └──────────────────────────────────────────────────
                    Low Effort              High Effort
```

### Recommended Order

1. **Post-MVP Phase 1 (Must Have):**
   - User & Auth
   - Workspace
   - Basic Billing

2. **Post-MVP Phase 2 (Should Have):**
   - FB/IG Channels
   - Shopify Integration
   - Basic CSAT
   - Custom API Tools

3. **Post-MVP Phase 3 (Nice to Have):**
   - Advanced Analytics
   - CDP
   - Enterprise Security

4. **Future (Later):**
   - Voice AI
   - Full Compliance Suite

---

## 6. Timeline Overview

| Phase | Duration | Key Deliverables |
|-------|----------|------------------|
| **MVP** | 8 週 | 核心 AI 客服功能 |
| **Post-MVP 1** | 4-6 週 | User, Workspace, Billing |
| **Post-MVP 2** | 6-8 週 | 更多渠道、電商整合 |
| **Post-MVP 3** | 8-12 週 | 企業級功能 |

**Total to Full Platform:** 6-9 個月

---

## 7. Milestones

### M1: MVP Complete (Week 8)
- [x] AI Agent 配置
- [x] 知識庫上傳
- [x] Web Widget + LINE
- [x] 對話管理後台

### M2: Multi-Tenant SaaS (Week 14)
- [ ] 用戶註冊登入
- [ ] 工作空間管理
- [ ] 基礎計費

### M3: E-commerce Ready (Week 22)
- [ ] FB/IG 渠道
- [ ] Shopify 整合
- [ ] 商品目錄

### M4: Enterprise Ready (Week 34)
- [ ] CSAT 追蹤
- [ ] 進階分析
- [ ] 合規功能

---

## 8. Dependencies

```
MVP
 │
 ├──> User & Auth ──> Workspace ──> Multi-tenant Isolation
 │                        │
 │                        └──> Billing (requires workspace)
 │
 ├──> Additional Channels (can be parallel)
 │
 ├──> E-commerce Integrations (can be parallel)
 │
 └──> Analytics (requires conversation data)
           │
           └──> Enterprise Features (requires analytics)
```

---

## 9. Risk Mitigation

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| User/Auth 複雜度 | Medium | High | 使用成熟套件 (NextAuth) |
| Billing 整合問題 | Medium | Medium | Stripe 官方 SDK |
| 渠道 API 變更 | Medium | High | 抽象層設計 |
| 合規要求變更 | Low | High | 持續追蹤法規 |

---

## 10. Success Metrics by Phase

### Post-MVP Phase 1
| Metric | Target |
|--------|--------|
| 註冊用戶 | 500 |
| 付費轉換 | 5% |
| Churn Rate | < 10%/月 |

### Post-MVP Phase 2
| Metric | Target |
|--------|--------|
| 付費用戶 | 50 |
| MRR | $5,000 |
| 渠道覆蓋 | 5+ |

### Post-MVP Phase 3
| Metric | Target |
|--------|--------|
| 企業客戶 | 5 |
| MRR | $20,000 |
| CSAT | > 85% |

---

*This post-MVP scope provides a clear roadmap from MVP to a full-featured enterprise AI customer service platform. The phased approach ensures we validate core functionality before investing in operational infrastructure.*
