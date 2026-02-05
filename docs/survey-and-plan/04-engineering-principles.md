# Engineering Design Principles

> **Purpose:** Establish architectural principles to avoid future major refactoring

---

## 1. Multi-Tenancy Architecture

### 1.1 Tenant Isolation Strategy

**Recommendation:** Shared database with row-level security (RLS)

| Strategy | Pros | Cons | Recommendation |
|----------|------|------|----------------|
| Separate databases | Complete isolation | High ops cost, complex | ❌ Not recommended |
| Schema per tenant | Good isolation | Migration complexity | ⚠️ Enterprise only |
| **Row-level security** | Efficient, scalable | Careful query design | ✅ Recommended |

**Implementation:**
```sql
-- Every table includes workspace_id
CREATE TABLE conversations (
  id UUID PRIMARY KEY,
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  -- ... other columns
);

-- Row-level security policy
CREATE POLICY workspace_isolation ON conversations
  USING (workspace_id = current_setting('app.workspace_id')::UUID);
```

### 1.2 Workspace ID Propagation

**Principle:** Workspace context must flow through all layers.

```typescript
// Every request handler receives workspace context
interface RequestContext {
  workspaceId: string;
  userId: string;
  permissions: Permission[];
}

// Context propagated through dependency injection
function createAgentRunner(ctx: RequestContext) {
  return new AgentRunner({
    workspaceId: ctx.workspaceId,
    // ... other deps
  });
}
```

### 1.3 Data Isolation Checklist

- [ ] All database queries filter by `workspace_id`
- [ ] File storage uses workspace-prefixed paths
- [ ] Cache keys include workspace context
- [ ] Logs include workspace ID for debugging
- [ ] Background jobs carry workspace context

---

## 2. Database Design

### 2.1 Database Choice

**Recommendation:** PostgreSQL with pgvector extension

**Rationale:**
- Mature, reliable, well-supported
- pgvector for embeddings (avoid separate vector DB initially)
- JSONB for flexible schema evolution
- Row-level security for tenant isolation
- Strong TypeScript tooling (Prisma, Drizzle)

### 2.2 Schema Evolution Strategy

**Principle:** Design for additive changes.

```typescript
// Use JSONB for extensible metadata
interface Conversation {
  id: string;
  workspaceId: string;
  channel: string;
  // Core fields are typed columns

  metadata: Record<string, unknown>;
  // Extensible fields in JSONB
}
```

**Migration Rules:**
1. Never delete columns in production; deprecate first
2. Add new columns as nullable or with defaults
3. Use feature flags for schema-dependent features
4. Maintain backwards compatibility for 2 versions

### 2.3 Key Entities

```
workspaces
  ├── users (many-to-many via workspace_members)
  ├── agents
  │   ├── agent_configs
  │   ├── tools
  │   └── knowledge_bases
  ├── channels
  │   └── channel_configs
  ├── customers
  │   ├── customer_profiles
  │   └── customer_sessions
  ├── conversations
  │   └── messages
  ├── analytics_events
  └── billing
      ├── subscriptions
      └── usage_records
```

---

## 3. API Design

### 3.1 API-First Development

**Principle:** Define API contracts before implementation.

**Tools:**
- OpenAPI 3.0 for REST APIs
- TypeScript types generated from OpenAPI
- Contract testing in CI/CD

### 3.2 REST API Conventions

```
POST   /api/v1/workspaces                    # Create workspace
GET    /api/v1/workspaces/:id                # Get workspace
PATCH  /api/v1/workspaces/:id                # Update workspace
DELETE /api/v1/workspaces/:id                # Delete workspace

GET    /api/v1/workspaces/:id/agents         # List agents
POST   /api/v1/workspaces/:id/agents         # Create agent
GET    /api/v1/workspaces/:id/agents/:agentId # Get agent
```

**Versioning:**
- URL path versioning (`/api/v1/`, `/api/v2/`)
- Maintain backwards compatibility within major version
- Deprecation notices 6 months before removal

### 3.3 Real-time API

**Use Cases:**
- Live conversation updates
- Agent typing indicators
- Human agent handover notifications

**Recommendation:** WebSocket with fallback to SSE

```typescript
// WebSocket events
type WSEvent =
  | { type: 'message.new'; data: Message }
  | { type: 'message.update'; data: Partial<Message> }
  | { type: 'agent.typing'; data: { agentId: string } }
  | { type: 'handover.requested'; data: HandoverRequest };
```

---

## 4. Agent System Extensions

### 4.1 Preserve Existing Patterns

**Principle:** Extend, don't replace, the existing agent system.

```typescript
// Existing pattern (keep)
const tools = createOpenClawTools({
  config,
  modelProvider,
  abortSignal,
});

// Extended pattern (add)
const csTools = createCSTools({
  workspaceId,
  knowledgeBase,
  productCatalog,
  customerProfile,
});

// Compose
const allTools = [...tools, ...csTools];
```

### 4.2 Context Injection

**Principle:** Use system prompt sections for dynamic context.

```typescript
interface CSSystemPromptContext {
  customer?: CustomerProfile;
  recentOrders?: Order[];
  knowledgeResults?: KBResult[];
  productMatches?: Product[];
  escalationGuidelines?: string;
}

function buildCSSystemPrompt(ctx: CSSystemPromptContext): string {
  return [
    buildCustomerSection(ctx.customer),
    buildKnowledgeSection(ctx.knowledgeResults),
    buildProductSection(ctx.productMatches),
    buildEscalationSection(ctx.escalationGuidelines),
  ].filter(Boolean).join('\n\n');
}
```

### 4.3 Tool Composition Pattern

**Principle:** Tools should be composable and workspace-aware.

```typescript
interface CSToolFactory {
  createKnowledgeQueryTool(kb: KnowledgeBase): AgentTool;
  createProductSearchTool(catalog: ProductCatalog): AgentTool;
  createOrderLookupTool(api: OrderAPI): AgentTool;
  createHandoverTool(handoverService: HandoverService): AgentTool;
}
```

---

## 5. Channel Abstraction

### 5.1 Unified Message Format

**Principle:** Normalize all channel messages to internal format.

```typescript
interface UnifiedMessage {
  id: string;
  channelId: ChannelType;
  channelMessageId: string;

  // Sender
  senderId: string;
  senderType: 'customer' | 'agent' | 'system';

  // Content (normalized)
  content: {
    text?: string;
    attachments?: Attachment[];
    buttons?: Button[];
    metadata?: Record<string, unknown>;
  };

  // Threading
  replyToId?: string;
  threadId?: string;

  // Timestamps
  createdAt: Date;
  receivedAt: Date;
}
```

### 5.2 Channel Plugin Contract

**Principle:** Keep existing plugin interface, extend for CS features.

```typescript
interface CSChannelExtension {
  // Customer identification
  extractCustomerId?(message: InboundMessage): string;

  // Rich message support
  supportsButtons?: boolean;
  supportsCarousels?: boolean;
  supportsQuickReplies?: boolean;

  // Typing indicators
  sendTypingIndicator?(conversationId: string): Promise<void>;

  // Read receipts
  sendReadReceipt?(messageId: string): Promise<void>;
}
```

### 5.3 New Channel Checklist

When adding a new channel:
- [ ] Implement base `ChannelPlugin` interface
- [ ] Implement `CSChannelExtension` for CS features
- [ ] Add OAuth/credential management
- [ ] Add rate limiting handling
- [ ] Add message retry logic
- [ ] Add channel-specific tests
- [ ] Add setup wizard UI

---

## 6. Knowledge Base Architecture

### 6.1 Document Processing Pipeline

```
Upload → Parse → Chunk → Embed → Store → Index
                                    ↓
                              Vector DB (pgvector)
```

### 6.2 RAG Pipeline Design

**Principle:** Separate retrieval from generation.

```typescript
interface RAGPipeline {
  // 1. Query understanding
  analyzeQuery(query: string): QueryIntent;

  // 2. Retrieval
  retrieve(query: string, options: RetrieveOptions): Document[];

  // 3. Reranking (optional)
  rerank(query: string, docs: Document[]): Document[];

  // 4. Context building
  buildContext(docs: Document[]): string;

  // 5. Prompt augmentation
  augmentPrompt(prompt: string, context: string): string;
}
```

### 6.3 Embedding Strategy

**Recommendation:** Start with OpenAI embeddings, abstract for flexibility.

```typescript
interface EmbeddingProvider {
  embed(text: string): Promise<number[]>;
  embedBatch(texts: string[]): Promise<number[][]>;
  dimensions: number;
}

// Implementations
class OpenAIEmbedding implements EmbeddingProvider { }
class CohereEmbedding implements EmbeddingProvider { }
class LocalEmbedding implements EmbeddingProvider { }
```

---

## 7. Event-Driven Architecture

### 7.1 Event Bus

**Principle:** Use events for cross-module communication.

```typescript
type CSEvent =
  | { type: 'conversation.started'; data: Conversation }
  | { type: 'message.received'; data: Message }
  | { type: 'message.sent'; data: Message }
  | { type: 'handover.requested'; data: HandoverRequest }
  | { type: 'handover.completed'; data: HandoverResult }
  | { type: 'customer.updated'; data: Customer }
  | { type: 'usage.recorded'; data: UsageRecord };

interface EventBus {
  emit(event: CSEvent): void;
  on(type: string, handler: (event: CSEvent) => void): void;
}
```

### 7.2 Event Handlers

**Principle:** Side effects via event handlers, not inline.

```typescript
// Analytics handler
eventBus.on('message.sent', (event) => {
  analyticsService.recordMessage(event.data);
});

// Billing handler
eventBus.on('message.sent', (event) => {
  billingService.incrementUsage(event.data.workspaceId);
});

// Webhook handler
eventBus.on('handover.requested', (event) => {
  webhookService.notify(event.data);
});
```

---

## 8. Scalability Considerations

### 8.1 Stateless Services

**Principle:** Keep services stateless for horizontal scaling.

- Session state in database/Redis
- No in-memory caches (use Redis)
- Background jobs via queue (BullMQ/Temporal)

### 8.2 Database Scaling Path

1. **Start:** Single PostgreSQL instance
2. **Scale reads:** Read replicas
3. **Scale writes:** Connection pooling (PgBouncer)
4. **Future:** Sharding by workspace_id

### 8.3 Caching Strategy

```typescript
// Cache hierarchy
interface CacheStrategy {
  // L1: Request-scoped (in-memory, per-request)
  requestCache: Map<string, unknown>;

  // L2: Process-scoped (in-memory, TTL)
  processCache: LRUCache<string, unknown>;

  // L3: Shared (Redis, longer TTL)
  sharedCache: RedisClient;
}

// Cache key conventions
const cacheKey = `ws:${workspaceId}:kb:${kbId}:doc:${docId}`;
```

---

## 9. Security Principles

### 9.1 Defense in Depth

1. **Network:** API gateway, WAF, rate limiting
2. **Authentication:** JWT with short expiry, refresh tokens
3. **Authorization:** RBAC with workspace context
4. **Data:** Encryption at rest, TLS in transit
5. **Logging:** Audit logs, PII masking

### 9.2 API Security Checklist

- [ ] All endpoints require authentication
- [ ] Workspace ID verified against user permissions
- [ ] Input validation on all parameters
- [ ] Rate limiting per user/workspace
- [ ] CORS configured for allowed origins
- [ ] Security headers (CSP, HSTS, etc.)

### 9.3 LLM-Specific Security

- [ ] Prompt injection detection
- [ ] Output filtering for PII
- [ ] Tool execution sandboxing
- [ ] API credential encryption

---

## 10. Testing Strategy

### 10.1 Test Pyramid

```
           /\
          /  \        E2E (10%)
         /----\
        /      \      Integration (30%)
       /--------\
      /          \    Unit (60%)
     --------------
```

### 10.2 Test Requirements by Module

| Module | Unit | Integration | E2E |
|--------|------|-------------|-----|
| Agent execution | ✅ | ✅ | ✅ |
| Tool system | ✅ | ✅ | |
| Channel adapters | ✅ | ✅ | ✅ |
| RAG pipeline | ✅ | ✅ | |
| API endpoints | ✅ | ✅ | ✅ |
| UI components | ✅ | | ✅ |

### 10.3 Mock Strategy

```typescript
// Mock external services
const mockLLM = new MockLLMProvider();
const mockEmbedding = new MockEmbeddingProvider();
const mockChannel = new MockChannelAdapter();

// Use dependency injection
const agent = createAgent({
  llm: process.env.USE_MOCKS ? mockLLM : realLLM,
  embedding: process.env.USE_MOCKS ? mockEmbedding : realEmbedding,
});
```

---

## 11. Monitoring & Observability

### 11.1 Metrics to Track

**Business Metrics:**
- Conversations per workspace
- Resolution rate
- Response time (p50, p95, p99)
- Human handover rate
- Customer satisfaction

**Technical Metrics:**
- API latency
- Error rates
- Queue depth
- Database query time
- LLM token usage

### 11.2 Logging Standards

```typescript
// Structured logging
logger.info('Message processed', {
  workspaceId,
  conversationId,
  messageId,
  channel,
  durationMs,
  tokenUsage,
});
```

### 11.3 Tracing

**Recommendation:** OpenTelemetry for distributed tracing

```typescript
// Trace conversation flow
const span = tracer.startSpan('processMessage');
span.setAttribute('workspace_id', workspaceId);
span.setAttribute('channel', channel);
// ... processing
span.end();
```

---

## 12. Summary Checklist

### Before Starting Development

- [ ] Database schema designed with multi-tenancy
- [ ] API contracts defined (OpenAPI)
- [ ] Authentication/authorization design complete
- [ ] Event types defined
- [ ] Monitoring/logging standards established

### Before Each Feature

- [ ] Feature designed with workspace isolation
- [ ] API endpoints follow conventions
- [ ] Events defined for side effects
- [ ] Tests planned (unit, integration, E2E)
- [ ] Security considerations documented

### Before Production

- [ ] Load testing completed
- [ ] Security audit performed
- [ ] Monitoring dashboards created
- [ ] Runbooks written
- [ ] Disaster recovery tested

---

*Following these principles will ensure the platform can scale, evolve, and maintain without major architectural rewrites.*
