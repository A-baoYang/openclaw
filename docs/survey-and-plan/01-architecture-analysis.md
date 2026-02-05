# OpenClaw Architecture Analysis

> **Purpose:** Understand current clawdbot capabilities and how they map to the customer service platform vision

## Current Architecture Overview

The existing OpenClaw (clawdbot) codebase provides a solid foundation for building a multi-channel AI customer service platform. Here's a detailed analysis of the core systems.

---

## 1. Agent System

### 1.1 Multi-Agent Configuration

**Location:** `src/agents/agent-scope.ts`

The platform already supports multiple agents with per-agent configuration:

```typescript
type ResolvedAgentConfig = {
  name?: string;
  workspace?: string;           // Per-agent workspace directory
  agentDir?: string;
  model?: AgentEntry["model"];   // Per-agent model selection
  memorySearch?: AgentEntry["memorySearch"];
  humanDelay?: AgentEntry["humanDelay"];
  heartbeat?: AgentEntry["heartbeat"];
  identity?: AgentEntry["identity"];
  groupChat?: AgentEntry["groupChat"];
  subagents?: AgentEntry["subagents"];
  sandbox?: AgentEntry["sandbox"];
  tools?: AgentEntry["tools"];   // Per-agent tool policies
};
```

**Reusable for CS Platform:**
- Multi-agent architecture can map to "one agent per workspace/customer"
- Per-agent workspace enables isolated knowledge bases
- Per-agent tool policies allow customized API integrations

### 1.2 Session Key System

**Location:** `src/routing/session-key.ts`

Session keys use a structured format:
```
agent:{agentId}:{sessionType}:{peerId}
```

Examples:
- `agent:main:telegram:dm:user123`
- `agent:bot1:whatsapp:dm:886912345678`

**Adaptation Needed:**
- Extend to include tenant/workspace context
- Add customer profile linking
- Support cross-channel session continuity

### 1.3 Agent Execution Engine

**Location:** `src/agents/pi-embedded-runner/`

The embedded PI agent runner handles:
- Session initialization and loading
- Tool preparation and policy application
- System prompt construction
- Streaming response handling
- Context compaction for long conversations

**Return Type:**
```typescript
type EmbeddedPiRunResult = {
  payloads?: Array<{
    text?: string;
    mediaUrl?: string;
    replyToId?: string;
    isError?: boolean;
  }>;
  meta: EmbeddedPiRunMeta;
  didSendViaMessagingTool?: boolean;
  messagingToolSentTexts?: string[];
};
```

**Strengths for CS Platform:**
- Already handles streaming responses
- Supports media attachments
- Built-in error handling
- Context compaction prevents token overflow

---

## 2. Tool System

### 2.1 Tool Definition Pattern

**Location:** `src/agents/pi-tools.ts`, `src/agents/openclaw-tools.ts`

Tools are created via factory functions with extensive configuration:

```typescript
export function createOpenClawTools(options?: {
  sandboxBrowserBridgeUrl?: string;
  agentSessionKey?: string;
  config?: OpenClawConfig;
  modelProvider?: string;
  abortSignal?: AbortSignal;
  // ... 15+ more options
}): AnyAgentTool[];
```

**Tool Categories Available:**

| Category | Tools | CS Platform Use |
|----------|-------|-----------------|
| Messaging | `send`, channel-specific sends | Multi-channel responses |
| Sessions | `sessions_list`, `sessions_send` | Session management |
| Web | `web_fetch`, `web_search`, `browser` | Knowledge retrieval |
| Coding | `exec`, `read`, `write`, `edit` | (Disable for CS) |
| Media | `image`, `tts` | Image analysis, voice |
| Utilities | `memory_search`, `memory_get` | Context retrieval |

### 2.2 Tool Policy System

**Location:** `src/agents/pi-tools.policy.ts`, `src/agents/tool-policy.ts`

Multi-level policy inheritance:
```
Agent-level > Channel-level > Global > Model-default
```

**Configuration:**
- `tools.alsoAllow`: Whitelist additional tools
- `tools.deny`: Blacklist tools
- Per-agent override capability

**Adaptation for CS Platform:**
- Enable only customer-service-safe tools
- Add custom API tools dynamically
- Product catalog query tools
- Order management tools

### 2.3 Schema System

Tools use TypeBox for schema definition:
- Base types: `@sinclair/typebox`
- Provider-specific normalization
- Claude and Gemini compatibility patches

**Strengths:**
- Type-safe tool definitions
- Runtime validation
- Schema generation for documentation

---

## 3. Channel/Messaging System

### 3.1 Channel Plugin Architecture

**Location:** `src/channels/plugins/types.plugin.ts`

The channel plugin contract is comprehensive:

```typescript
type ChannelPlugin<ResolvedAccount> = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;

  // Adapter interfaces
  config: ChannelConfigAdapter<ResolvedAccount>;
  setup?: ChannelSetupAdapter;
  security?: ChannelSecurityAdapter<ResolvedAccount>;
  outbound?: ChannelOutboundAdapter;
  status?: ChannelStatusAdapter<ResolvedAccount>;
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;
  auth?: ChannelAuthAdapter;
  commands?: ChannelCommandAdapter;
  streaming?: ChannelStreamingAdapter;
  messaging?: ChannelMessagingAdapter;
  threading?: ChannelThreadingAdapter;
  actions?: ChannelMessageActionAdapter;
  heartbeat?: ChannelHeartbeatAdapter;

  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];
};
```

### 3.2 Built-in Channels

**Core Channels:** `src/telegram`, `src/discord`, `src/slack`, `src/signal`, `src/imessage`, `src/web`

**Extensions:** `extensions/msteams`, `extensions/matrix`, `extensions/zalo`, `extensions/voice-call`

| Channel | Status | CS Platform Priority |
|---------|--------|---------------------|
| Web (WhatsApp) | ✅ Core | High |
| Telegram | ✅ Core | Medium |
| Discord | ✅ Core | Low |
| Slack | ✅ Core | Medium (B2B) |
| Signal | ✅ Core | Low |
| iMessage | ✅ Core | Medium |
| MS Teams | ✅ Extension | High (B2B) |
| Matrix | ✅ Extension | Low |
| Zalo | ✅ Extension | High (Vietnam) |
| Voice Call | ✅ Extension | High (Future) |

### 3.3 Channels Needed (Not Yet Built)

| Channel | Priority | Market |
|---------|----------|--------|
| **LINE** | Critical | Japan, Taiwan, Thailand |
| **Facebook Messenger** | High | Global |
| **Instagram DM** | High | Global |
| **WeChat** | High | China |
| **Web Widget** | Critical | All |
| Threads | Medium | Global |
| Twitter/X DM | Medium | Global |

### 3.4 Message Routing

**Location:** `src/routing/resolve-route.ts`, `src/channels/dock.ts`

Routing flow:
1. Inbound message → Channel ID resolution
2. Account binding (channel + token → account)
3. Chat type detection (DM vs group)
4. Session key derivation
5. Policy-based routing (allowlist, mention-only)

**Session Key Building:**
```typescript
buildAgentPeerSessionKey({
  agentId: "main",
  channel: "telegram",
  peerKind: "dm" | "group" | "channel",
  peerId: "12345",
  dmScope: "main" | "per-peer" | "per-channel-peer"
})
```

---

## 4. Data Storage System

### 4.1 Current Storage (File-based)

**Location:** `src/config/sessions/store.ts`

```
~/.openclaw/
  agents/
    {agentId}/
      agent/
        sessions.json           # Session index
        session-{id}.jsonl      # Conversation turns
      conversations/            # Group histories
        {channel}_{groupId}.jsonl
  config.json                   # Main config
  config.env.json               # Secrets
  credentials/                  # OAuth/API keys
```

### 4.2 Session Entry Structure

```typescript
type SessionEntry = {
  sessionId: string;
  updatedAt: number;
  sessionFile?: string;

  // Chat context
  channel?: string;
  groupId?: string;
  origin?: SessionOrigin;

  // Usage tracking
  inputTokens?: number;
  outputTokens?: number;
  totalTokens?: number;
  contextTokens?: number;
  compactionCount?: number;
};
```

### 4.3 Storage Limitations for CS Platform

**Current:**
- File-based, single-machine only
- No multi-tenant support
- No customer profile storage
- No analytics aggregation

**Needed:**
- Relational database (PostgreSQL recommended)
- Multi-tenant data isolation
- Customer profile management
- Conversation analytics
- Usage metering for billing

---

## 5. Authentication/Authorization

### 5.1 Current Model Auth

**Location:** `src/agents/model-auth.ts`, `src/agents/auth-profiles.ts`

- Multiple API keys per provider
- Profile rotation on rate limits
- Failover chains

### 5.2 Missing for CS Platform

| Component | Current | Needed |
|-----------|---------|--------|
| User Auth | None | Email/OAuth login |
| Workspace Auth | None | Role-based access |
| API Key Management | CLI config | Self-service UI |
| Channel Auth | Per-account | Per-workspace |
| Rate Limiting | Basic | Per-plan limits |

---

## 6. System Prompt Architecture

**Location:** `src/agents/system-prompt.ts`

Modular prompt construction with sections:
1. Skills (workspace skill references)
2. Memory (search guidance)
3. User Identity
4. Time/Timezone
5. Safety rules
6. Reply tags
7. Messaging hints
8. Workspace context
9. Tool descriptions
10. Sandbox info
11. Approval gating
12. Runtime info

**Adaptation for CS Platform:**
- Add customer context section
- Add product catalog context
- Add knowledge base results
- Add conversation history summary
- Add escalation guidelines

---

## 7. Architectural Patterns

### 7.1 Dependency Injection
```typescript
createOpenClawCodingTools(options?: {
  config?: OpenClawConfig;
  sandbox?: SandboxContext;
  modelProvider?: string;
  abortSignal?: AbortSignal;
});
```

### 7.2 Adapter Pattern
- Channel adapters for lifecycle events
- Stateless functions with context passing

### 7.3 Factory Pattern
- `createMessageTool()`, `createBrowserTool()`
- `registerChannel({ plugin })`

### 7.4 Observer Pattern
- Event streaming via `pi-embedded-subscribe.ts`
- Message lifecycle handlers

### 7.5 Composition
- Tools composed from base + channel-specific
- Policies merged from multiple sources
- Config includes and environment substitution

---

## 8. Summary: Reusable vs. New Components

### Fully Reusable (Core Strengths)

| Component | Reuse Level | Notes |
|-----------|-------------|-------|
| Agent Execution Engine | 95% | Add tenant context |
| Tool System | 90% | Add CS-specific tools |
| Channel Plugin Architecture | 95% | Add new channels |
| System Prompt Builder | 80% | Add CS sections |
| Message Routing | 70% | Add tenant routing |
| Session Management | 60% | Migrate to DB |

### Needs Significant Extension

| Component | Effort | Priority |
|-----------|--------|----------|
| Multi-tenant Auth | High | Critical |
| Database Storage | High | Critical |
| Customer Profiles | High | Critical |
| Knowledge Base Integration | Medium | High |
| Analytics System | Medium | High |
| Billing System | Medium | High |
| Admin Dashboard | High | High |

### New Components Required

| Component | Complexity | Priority |
|-----------|------------|----------|
| User Registration/Login | Medium | Critical |
| Workspace Management | Medium | Critical |
| Knowledge Base Uploader | Medium | High |
| API Tool Builder | High | High |
| Product Catalog Connector | Medium | High |
| Human Handover System | Medium | High |
| Analytics Dashboard | High | Medium |
| Billing/Subscription | High | Medium |

---

## 9. Key Files Reference

| Component | Primary Files | LOC (Approx) |
|-----------|---------------|--------------|
| Agent Config | `src/agents/agent-scope.ts` | 180 |
| Agent Execution | `src/agents/pi-embedded-runner/run.ts` | 200 |
| Session Storage | `src/config/sessions/store.ts` | 150 |
| Tool Creation | `src/agents/pi-tools.ts` | 400 |
| Messaging Router | `src/channels/dock.ts` | 350 |
| System Prompt | `src/agents/system-prompt.ts` | 400+ |
| Plugin Registry | `src/plugins/registry.ts` | 300 |
| Channel Types | `src/channels/plugins/types.plugin.ts` | 85 |
| Config Schema | `src/config/schema.ts` | 1000+ |
| Message Tool | `src/agents/tools/message-tool.ts` | 400+ |

---

*This analysis confirms that clawdbot provides a solid 60-70% foundation for the CS platform, with the agent and tool systems being particularly strong. The main gaps are in multi-tenancy, persistent storage, and user-facing management interfaces.*
