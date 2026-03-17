# Agentic Gateway - Architecture V2 (Performance-First)

**Status:** Idea Phase  
**Version:** 2.0  
**Last Updated:** March 17, 2026

---

## 🎯 Design Principles

### **1. Performance First - Minimize Latency**
- Target: <10ms added latency for fast-path requests
- LLM reasoning ONLY when necessary (not every request)
- Intelligent tiering: Fast path → Smart path → Deep reasoning
- Aggressive caching and pre-computation

### **2. Standards-Based - Google A2A Protocol**
- Use Google's Agent-to-Agent (A2A) protocol specification
- Interoperable with other A2A gateways
- Standard message formats and authentication

### **3. Leverage Existing Security Solutions**
- Plugin architecture for market-leading security tools
- Don't reinvent: WAF, IDS/IPS, DLP, SIEM
- Integration points for Palo Alto, Imperva, Splunk, etc.

---

## 🏗️ High-Performance Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EXTERNAL AGENTS                                  │
│          (Google A2A Protocol Compliant)                            │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           │ Google A2A Protocol Messages
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   AGENTIC GATEWAY CLUSTER                           │
│                   (Multi-Tier Processing)                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │              TIER 1: FAST PATH (<5ms latency)                 │ │
│  │              (90% of requests handled here)                   │ │
│  │                                                               │ │
│  │  ┌─────────────────────────────────────────────────────────┐ │ │
│  │  │  L1 Cache (Redis)                                       │ │ │
│  │  │  - Cached DID verifications                             │ │ │
│  │  │  - Recent policy decisions                              │ │ │
│  │  │  - Session tokens (TTL: 5 min)                          │ │ │
│  │  └─────────────────────────────────────────────────────────┘ │ │
│  │                          ↓                                    │ │
│  │  ┌─────────────────────────────────────────────────────────┐ │ │
│  │  │  Rules Engine (Deterministic, in-memory)                │ │ │
│  │  │  - Simple RBAC: if (role == 'admin') → ALLOW           │ │ │
│  │  │  - Pattern matching: /api/public/* → ALLOW             │ │ │
│  │  │  - Rate limit counters (local + distributed)            │ │ │
│  │  └─────────────────────────────────────────────────────────┘ │ │
│  │                          ↓                                    │ │
│  │  Decision: ALLOW (cache for next request) → Backend          │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                           │                                         │
│                           │ (If cache miss or complex policy)       │
│                           ▼                                         │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │          TIER 2: SMART PATH (<50ms latency)                   │ │
│  │          (9% of requests - policy evaluation)                 │ │
│  │                                                               │ │
│  │  ┌─────────────────────────────────────────────────────────┐ │ │
│  │  │  Policy Engine (OPA - Open Policy Agent)                │ │ │
│  │  │  - ABAC (Attribute-Based Access Control)                │ │ │
│  │  │  - Complex rules (if A AND B OR C...)                   │ │ │
│  │  │  - JSON/YAML policy definitions                         │ │ │
│  │  └─────────────────────────────────────────────────────────┘ │ │
│  │                          ↓                                    │ │
│  │  ┌─────────────────────────────────────────────────────────┐ │ │
│  │  │  Blockchain Verification (cached reads)                 │ │ │
│  │  │  - DID resolution (IPFS + local cache)                  │ │ │
│  │  │  - Credential verification (batch verify)               │ │ │
│  │  │  - Smart contract queries (read-only)                   │ │ │
│  │  └─────────────────────────────────────────────────────────┘ │ │
│  │                          ↓                                    │ │
│  │  Decision: Cache result → Backend                             │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                           │                                         │
│                           │ (If ambiguous or novel request)         │
│                           ▼                                         │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │       TIER 3: INTELLIGENT PATH (<500ms latency)               │ │
│  │       (1% of requests - LLM reasoning)                        │ │
│  │                                                               │ │
│  │  ┌─────────────────────────────────────────────────────────┐ │ │
│  │  │  🧠 LLM Agent (Claude Haiku / GPT-4 Mini)               │ │ │
│  │  │  - Natural language interpretation                      │ │ │
│  │  │  - Context-aware decisions                              │ │ │
│  │  │  - Novel policy reasoning                               │ │ │
│  │  │  - Emergency exceptions                                 │ │ │
│  │  └─────────────────────────────────────────────────────────┘ │ │
│  │                          ↓                                    │ │
│  │  ┌─────────────────────────────────────────────────────────┐ │ │
│  │  │  Decision Explainer                                     │ │ │
│  │  │  - Log reasoning to audit trail                         │ │ │
│  │  │  - Generate new policy rule (for next time)             │ │ │
│  │  │  - Cache decision pattern                               │ │ │
│  │  └─────────────────────────────────────────────────────────┘ │ │
│  │                          ↓                                    │ │
│  │  Decision: Promote to Tier 2 policy → Backend                │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │              SECURITY PLUGINS (Parallel)                      │ │
│  │              (All tiers leverage these)                       │ │
│  │                                                               │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │ │
│  │  │ WAF Plugin  │  │ IDS/IPS     │  │ DLP Plugin  │          │ │
│  │  │ (Imperva/   │  │ (Snort/     │  │ (Symantec/  │          │ │
│  │  │  ModSec)    │  │  Suricata)  │  │  Forcepoint)│          │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘          │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │ │
│  │  │ Prompt Inj. │  │ Rate Limiter│  │ Audit Log   │          │ │
│  │  │ Detector    │  │ (Token bkt) │  │ (SIEM)      │          │ │
│  │  │ (Lakera AI) │  │             │  │ (Splunk)    │          │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘          │ │
│  │                                                               │ │
│  │  All plugins run asynchronously (non-blocking where possible)│ │
│  │  Results aggregated: if ANY deny → DENY                      │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    BACKEND RESOURCES                                │
│  (Internal Agents, REST APIs, MCP Tools, Databases)                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📡 Google A2A Protocol Integration

### **What is Google A2A Protocol?**

Google's Agent-to-Agent (A2A) protocol is a standard for secure, structured communication between AI agents. Key components:

1. **Message Format** - JSON-based with standard fields
2. **Authentication** - OAuth 2.0 + Agent DIDs
3. **Capabilities Discovery** - Agents advertise what they can do
4. **Versioning** - Protocol version negotiation

### **A2A Message Structure**

```json
{
  "a2a_version": "1.0",
  "message_id": "msg_7f8e9d6c5b4a3d2f",
  "timestamp": "2026-03-17T14:30:00Z",
  "from": {
    "agent_id": "did:web:external-agent.example.com",
    "capabilities": ["data_retrieval", "analysis"],
    "oauth_token": "eyJhbGciOiJSUzI1NiIs..."
  },
  "to": {
    "agent_id": "did:web:gateway.enterprise.com",
    "endpoint": "https://gateway.enterprise.com/a2a"
  },
  "conversation_id": "conv_abc123",
  "parent_message_id": null,
  "intent": {
    "action": "request",
    "resource": "customer_data",
    "parameters": {
      "customer_id": "12345",
      "fields": ["name", "email", "phone"]
    }
  },
  "context": {
    "purpose": "customer_support",
    "urgency": "normal",
    "compliance": ["GDPR", "CCPA"]
  },
  "signature": "..."
}
```

### **Gateway Response (A2A Format)**

```json
{
  "a2a_version": "1.0",
  "message_id": "msg_response_123",
  "timestamp": "2026-03-17T14:30:00.250Z",
  "from": {
    "agent_id": "did:web:gateway.enterprise.com"
  },
  "to": {
    "agent_id": "did:web:external-agent.example.com"
  },
  "conversation_id": "conv_abc123",
  "parent_message_id": "msg_7f8e9d6c5b4a3d2f",
  "response": {
    "status": "success",
    "decision": "ALLOW",
    "tier_used": "fast_path",
    "latency_ms": 4.2,
    "data": {
      "customer_id": "12345",
      "name": "John Doe",
      "email": "john@example.com",
      "phone": "+1-555-0123"
    },
    "audit": {
      "logged": true,
      "blockchain_tx": "0x1a2b3c...",
      "policy_applied": "customer_data_read_policy_v2"
    }
  },
  "signature": "..."
}
```

### **Capabilities Discovery**

```json
{
  "a2a_version": "1.0",
  "message_id": "msg_capabilities_req",
  "from": {
    "agent_id": "did:web:external-agent.example.com"
  },
  "to": {
    "agent_id": "did:web:gateway.enterprise.com"
  },
  "intent": {
    "action": "discover_capabilities"
  }
}
```

**Gateway Response:**
```json
{
  "a2a_version": "1.0",
  "response": {
    "status": "success",
    "capabilities": {
      "supported_resources": [
        "customer_data",
        "order_history",
        "product_catalog"
      ],
      "authentication_methods": [
        "oauth2",
        "did_verification",
        "api_key"
      ],
      "rate_limits": {
        "requests_per_minute": 100,
        "burst_capacity": 20
      },
      "compliance": ["GDPR", "HIPAA", "SOC2"],
      "latency_tiers": {
        "fast_path": "<10ms",
        "smart_path": "<50ms",
        "intelligent_path": "<500ms"
      }
    }
  }
}
```

---

## ⚡ Performance Optimizations

### **1. Multi-Tier Request Classification**

```typescript
class RequestClassifier {
  classify(request: A2AMessage): Tier {
    // Tier 1: Fast path (cached or simple RBAC)
    if (this.isInCache(request)) {
      return Tier.FAST_PATH;
    }
    
    if (this.isSimpleRBAC(request)) {
      // e.g., admin role → always allow
      return Tier.FAST_PATH;
    }
    
    if (this.isPublicEndpoint(request)) {
      // e.g., /api/public/* → always allow
      return Tier.FAST_PATH;
    }
    
    // Tier 2: Smart path (policy engine, blockchain verify)
    if (this.hasComplexPolicy(request)) {
      return Tier.SMART_PATH;
    }
    
    if (this.requiresBlockchainVerify(request)) {
      return Tier.SMART_PATH;
    }
    
    // Tier 3: Intelligent path (LLM reasoning)
    if (this.isAmbiguous(request)) {
      return Tier.INTELLIGENT_PATH;
    }
    
    if (this.isNovelRequest(request)) {
      return Tier.INTELLIGENT_PATH;
    }
    
    // Default: Smart path (safe choice)
    return Tier.SMART_PATH;
  }
}
```

### **2. Aggressive Caching Strategy**

```typescript
interface CacheStrategy {
  // L1: In-memory (sub-millisecond)
  l1Cache: Map<string, CachedDecision>;  // TTL: 30 seconds
  
  // L2: Redis (1-2ms)
  l2Cache: RedisClient;  // TTL: 5 minutes
  
  // L3: DID document cache (IPFS local node)
  l3Cache: IPFSNode;  // TTL: 24 hours
}

class CachedDecision {
  decision: 'ALLOW' | 'DENY';
  policy_version: string;
  expires_at: number;
  cache_key: string;  // hash(agent_did + resource + action)
}
```

### **3. Parallel Security Plugin Execution**

```typescript
class SecurityPluginManager {
  async evaluateRequest(request: A2AMessage): Promise<SecurityDecision> {
    // Run all plugins in parallel (non-blocking)
    const results = await Promise.all([
      this.wafPlugin.check(request),           // <5ms
      this.idsPlugin.check(request),           // <10ms
      this.dlpPlugin.check(request),           // <15ms
      this.promptInjectionDetector(request),   // <8ms
      this.rateLimiter.check(request),         // <2ms (Redis)
    ]);
    
    // Aggregate: ANY deny → DENY
    const anyDeny = results.some(r => r.decision === 'DENY');
    
    return {
      decision: anyDeny ? 'DENY' : 'ALLOW',
      plugins_evaluated: results.length,
      total_latency_ms: Math.max(...results.map(r => r.latency_ms)),
      reasons: results.filter(r => r.decision === 'DENY').map(r => r.reason)
    };
  }
}
```

### **4. Smart Blockchain Integration**

**Problem:** Blockchain reads are slow (100-500ms)  
**Solution:** Aggressive caching + batch operations

```typescript
class BlockchainConnector {
  private didCache: Map<string, DIDDocument> = new Map();
  private credentialCache: Map<string, VerifiableCredential> = new Map();
  
  async verifyAgentDID(did: string): Promise<boolean> {
    // Check cache first
    if (this.didCache.has(did)) {
      const cached = this.didCache.get(did);
      if (cached.expires_at > Date.now()) {
        return true;  // Cache hit: <1ms
      }
    }
    
    // Cache miss: Fetch from blockchain
    const didDoc = await this.fetchDIDFromBlockchain(did);  // 100-200ms
    
    // Cache for next time (TTL: 24 hours)
    this.didCache.set(did, {
      ...didDoc,
      expires_at: Date.now() + (24 * 60 * 60 * 1000)
    });
    
    return didDoc.active;
  }
  
  // Batch verify (amortize blockchain cost)
  async batchVerifyCredentials(
    credentials: VerifiableCredential[]
  ): Promise<Map<string, boolean>> {
    // Single smart contract call for multiple credentials
    const results = await this.smartContract.batchVerify(
      credentials.map(c => c.id)
    );  // 150ms for 100 credentials vs 15,000ms one-by-one
    
    return new Map(results);
  }
}
```

---

## 🔌 Security Plugin Architecture

### **Plugin Interface**

```typescript
interface SecurityPlugin {
  name: string;
  version: string;
  priority: number;  // Lower = runs first
  
  // Main evaluation method
  evaluate(request: A2AMessage): Promise<PluginDecision>;
  
  // Optional: Pre-process (async, non-blocking)
  preProcess?(request: A2AMessage): Promise<void>;
  
  // Optional: Post-process (async, non-blocking)
  postProcess?(response: A2AMessage): Promise<void>;
  
  // Configuration
  configure(config: PluginConfig): void;
}

interface PluginDecision {
  decision: 'ALLOW' | 'DENY' | 'DEFER';
  confidence: number;  // 0.0 - 1.0
  latency_ms: number;
  reason?: string;
  metadata?: Record<string, any>;
}
```

### **Built-in Plugins**

#### **1. WAF Plugin (Imperva / ModSecurity Integration)**

```typescript
class ImpervaWAFPlugin implements SecurityPlugin {
  name = 'imperva_waf';
  version = '1.0.0';
  priority = 1;  // High priority (runs first)
  
  private impervaClient: ImpervaAPI;
  
  async evaluate(request: A2AMessage): Promise<PluginDecision> {
    const startTime = Date.now();
    
    // Send to Imperva Cloud WAF
    const result = await this.impervaClient.inspect({
      method: request.intent.action,
      path: request.intent.resource,
      headers: request.context,
      body: JSON.stringify(request.intent.parameters)
    });
    
    return {
      decision: result.blocked ? 'DENY' : 'ALLOW',
      confidence: 1.0,
      latency_ms: Date.now() - startTime,
      reason: result.blocked ? result.rule_triggered : undefined,
      metadata: {
        waf_score: result.threat_score,
        rules_matched: result.rules_matched
      }
    };
  }
}
```

#### **2. IDS/IPS Plugin (Snort / Suricata Integration)**

```typescript
class SnortIDSPlugin implements SecurityPlugin {
  name = 'snort_ids';
  version = '1.0.0';
  priority = 2;
  
  private snortEngine: SnortEngine;
  
  async evaluate(request: A2AMessage): Promise<PluginDecision> {
    const startTime = Date.now();
    
    // Convert A2A message to network packet format
    const packet = this.convertToPacket(request);
    
    // Run through Snort rules engine
    const alerts = await this.snortEngine.analyze(packet);
    
    // Check for high-severity alerts
    const criticalAlerts = alerts.filter(a => a.severity >= 3);
    
    return {
      decision: criticalAlerts.length > 0 ? 'DENY' : 'ALLOW',
      confidence: 0.9,
      latency_ms: Date.now() - startTime,
      reason: criticalAlerts.length > 0 
        ? `IDS detected: ${criticalAlerts[0].signature}` 
        : undefined,
      metadata: {
        alerts: alerts.length,
        critical: criticalAlerts.length
      }
    };
  }
}
```

#### **3. DLP Plugin (Symantec / Forcepoint Integration)**

```typescript
class SymantecDLPPlugin implements SecurityPlugin {
  name = 'symantec_dlp';
  version = '1.0.0';
  priority = 3;
  
  private dlpClient: SymantecDLPAPI;
  
  async evaluate(request: A2AMessage): Promise<PluginDecision> {
    const startTime = Date.now();
    
    // Extract data from request
    const dataToScan = JSON.stringify(request.intent.parameters);
    
    // Scan for sensitive data (PII, PCI, PHI)
    const scanResult = await this.dlpClient.scan({
      content: dataToScan,
      policies: ['pii_detection', 'pci_dss', 'hipaa']
    });
    
    return {
      decision: scanResult.violations.length > 0 ? 'DENY' : 'ALLOW',
      confidence: 0.95,
      latency_ms: Date.now() - startTime,
      reason: scanResult.violations.length > 0
        ? `DLP violation: ${scanResult.violations[0].policy}`
        : undefined,
      metadata: {
        violations: scanResult.violations,
        data_types_found: scanResult.data_types
      }
    };
  }
}
```

#### **4. Prompt Injection Detector (Lakera AI / Custom)**

```typescript
class LakeraPromptGuardPlugin implements SecurityPlugin {
  name = 'lakera_prompt_guard';
  version = '1.0.0';
  priority = 4;
  
  private lakeraClient: LakeraAPI;
  
  async evaluate(request: A2AMessage): Promise<PluginDecision> {
    const startTime = Date.now();
    
    // Extract natural language parts
    const textToScan = [
      request.intent.action,
      JSON.stringify(request.intent.parameters),
      request.context.purpose || ''
    ].join(' ');
    
    // Lakera Prompt Guard API
    const result = await this.lakeraClient.detect({
      prompt: textToScan,
      categories: ['jailbreak', 'injection', 'prompt_leakage']
    });
    
    return {
      decision: result.flagged ? 'DENY' : 'ALLOW',
      confidence: result.confidence,
      latency_ms: Date.now() - startTime,
      reason: result.flagged 
        ? `Prompt injection detected: ${result.category}`
        : undefined,
      metadata: {
        categories_detected: result.categories,
        confidence_score: result.confidence
      }
    };
  }
}
```

#### **5. Rate Limiter (Token Bucket Algorithm)**

```typescript
class RateLimiterPlugin implements SecurityPlugin {
  name = 'rate_limiter';
  version = '1.0.0';
  priority = 5;
  
  private redis: RedisClient;
  
  async evaluate(request: A2AMessage): Promise<PluginDecision> {
    const startTime = Date.now();
    const agentId = request.from.agent_id;
    
    // Token bucket algorithm (Redis + Lua script)
    const allowed = await this.redis.eval(`
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local rate = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])
      
      local tokens = redis.call('get', key)
      if not tokens then
        tokens = capacity
      end
      
      tokens = math.min(capacity, tokens + (now - last_refill) * rate)
      
      if tokens >= 1 then
        redis.call('set', key, tokens - 1)
        return 1  -- ALLOW
      else
        return 0  -- DENY (rate limit exceeded)
      end
    `, [agentId], [100, 1.67, Date.now()]);  // 100 requests/min
    
    return {
      decision: allowed ? 'ALLOW' : 'DENY',
      confidence: 1.0,
      latency_ms: Date.now() - startTime,
      reason: allowed ? undefined : 'Rate limit exceeded',
      metadata: {
        rate_limit: '100 req/min',
        burst_capacity: 20
      }
    };
  }
}
```

#### **6. SIEM Integration (Splunk / QRadar)**

```typescript
class SplunkAuditPlugin implements SecurityPlugin {
  name = 'splunk_audit';
  version = '1.0.0';
  priority = 100;  // Low priority (post-decision logging)
  
  private splunkHEC: SplunkHECClient;
  
  async evaluate(request: A2AMessage): Promise<PluginDecision> {
    // This plugin doesn't block - always DEFER
    return {
      decision: 'DEFER',  // Don't influence decision
      confidence: 1.0,
      latency_ms: 0
    };
  }
  
  async postProcess(response: A2AMessage): Promise<void> {
    // Log to Splunk (async, non-blocking)
    await this.splunkHEC.send({
      sourcetype: 'agentic_gateway:audit',
      event: {
        timestamp: response.timestamp,
        agent_from: response.to.agent_id,
        agent_to: response.from.agent_id,
        decision: response.response.decision,
        tier_used: response.response.tier_used,
        latency_ms: response.response.latency_ms,
        policy_applied: response.response.audit.policy_applied,
        blockchain_tx: response.response.audit.blockchain_tx
      }
    });
  }
}
```

---

## 📊 Performance Benchmarks (Target)

| Tier | Requests | Latency Target | Success Rate |
|------|----------|----------------|--------------|
| **Fast Path** | 90% | <10ms | 99.9% |
| **Smart Path** | 9% | <50ms | 99.5% |
| **Intelligent Path** | 1% | <500ms | 99% |

**Total Gateway Overhead:**
- P50: 8ms
- P95: 45ms
- P99: 350ms
- P99.9: 1200ms (complex LLM reasoning)

---

## 🔧 Implementation Strategy

### **Phase 1: Fast Path (Week 1-2)**
- Redis cache layer
- Simple RBAC rules engine
- Rate limiting (Redis + Lua)
- Google A2A protocol parser

### **Phase 2: Smart Path (Week 3-4)**
- OPA (Open Policy Agent) integration
- Blockchain DID verification (cached)
- Plugin architecture foundation

### **Phase 3: Security Plugins (Week 5-6)**
- WAF plugin (ModSecurity)
- Rate limiter plugin
- Audit logger plugin (Splunk)

### **Phase 4: Intelligent Path (Week 7-8)**
- LLM integration (Claude Haiku for low latency)
- Decision caching & promotion to Tier 2
- Performance tuning

### **Phase 5: Production Hardening (Week 9-10)**
- Load testing (target: 100K req/s)
- Security audit
- Documentation & deployment guides

---

**Status:** 🎯 Idea Phase - Performance-First Architecture  
**Next:** Validate Google A2A protocol compatibility, select security plugins

