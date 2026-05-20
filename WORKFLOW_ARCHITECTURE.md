# AgentGuard-X: Visual Workflow Architecture

## Complete Request Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AI AGENT (LangChain/CrewAI)                           │
│                                                                               │
│  agent.run("Search weather, then send email")                               │
│                                                                               │
│  LangChain internally:                                                       │
│  └─ Detects first tool call: web_search(query="weather")                   │
│     └─ Fires: on_tool_start() callback                                     │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │ SecurityGatewayAsync       │
                    │ CallbackHandler            │
                    │                            │
                    │ .on_tool_start()           │
                    │ ├─ Extract: agent_id       │
                    │ ├─ Extract: tool_name      │
                    │ ├─ Extract: input          │
                    │ └─ Generate: request_id    │
                    └─────────────┬──────────────┘
                                  │
        ┌─────────────────────────▼────────────────────────┐
        │                                                   │
        │  REQUEST VALIDATION PIPELINE (app/pipeline.py)   │
        │                                                   │
        │  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    │
        │  ┃ STAGE 1: Global Rate Limit (Lua atomic)  ┃    │
        │  ┃                                           ┃    │
        │  ┃ Redis SCRIPT LOAD rate_limiter.lua       ┃    │
        │  ┃ EVALSHA <sha> 1 global_count <limit>     ┃    │
        │  ┃                                           ┃    │
        │  ┃ ✓ Pass: increment count, check < limit   ┃    │
        │  ┃ ✗ Fail: RateLimitException (block)        ┃    │
        │  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    │
        │                                                   │
        │  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    │
        │  ┃ STAGE 2: JWT Validation (RS256)          ┃    │
        │  ┃                                           ┃    │
        │  ┃ Extract: Authorization: Bearer <token>   ┃    │
        │  ┃ Decode: jwt.decode(token, secret,       ┃    │
        │  ┃         algorithms=['RS256'])             ┃    │
        │  ┃ Verify: expiry, claims (agent_id, sub)   ┃    │
        │  ┃                                           ┃    │
        │  ┃ ✓ Pass: claims extracted                 ┃    │
        │  ┃ ✗ Fail: JWTError (block)                  ┃    │
        │  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    │
        │                                                   │
        │  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    │
        │  ┃ STAGE 3: Agent Session Lookup            ┃    │
        │  ┃                                           ┃    │
        │  ┃ Redis.GET session:agent_001              ┃    │
        │  ┃ Expected: {"status": "registered",       ┃    │
        │  ┃           "roles": ["viewer", "analyst"]} ┃    │
        │  ┃                                           ┃    │
        │  ┃ ✓ Pass: session exists, registered       ┃    │
        │  ┃ ✗ Fail: RegistrationException            ┃    │
        │  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    │
        │                                                   │
        │  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    │
        │  ┃ STAGE 4: RBAC Enforcement (OPA)          ┃    │
        │  ┃                                           ┃    │
        │  ┃ POST http://opa:8181/v1/data/rbac        ┃    │
        │  ┃ Payload: {                               ┃    │
        │  ┃   "agent_id": "agent_001",               ┃    │
        │  ┃   "tool_name": "web_search",             ┃    │
        │  ┃   "roles": ["viewer", "analyst"]         ┃    │
        │  ┃ }                                         ┃    │
        │  ┃                                           ┃    │
        │  ┃ OPA evaluates: tool_rbac.rego            ┃    │
        │  ┃ Returns: {"allow": true/false}           ┃    │
        │  ┃                                           ┃    │
        │  ┃ ✓ Pass: OPA allows                        ┃    │
        │  ┃ ✗ Fail: RBACDeniedException              ┃    │
        │  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    │
        │                                                   │
        │  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    │
        │  ┃ STAGE 5: Per-Agent Rate Limit            ┃    │
        │  ┃                                           ┃    │
        │  ┃ Redis.LPUSH agent:agent_001 <timestamp>  ┃    │
        │  ┃ Redis.LTRIM agent:agent_001 0 <limit>    ┃    │
        │  ┃ Count actual items vs limit               ┃    │
        │  ┃                                           ┃    │
        │  ┃ ✓ Pass: count < per-agent limit          ┃    │
        │  ┃ ✗ Fail: RateLimitException               ┃    │
        │  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    │
        │                                                   │
        │  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    │
        │  ┃ STAGE 6: Sequence Analysis               ┃    │
        │  ┃                                           ┃    │
        │  ┃ Redis.LRANGE seq:agent_001 0 -1          ┃    │
        │  ┃ Returns: ["read_file", "compress"]       ┃    │
        │  ┃ Current call: "http_post"                ┃    │
        │  ┃                                           ┃    │
        │  ┃ Load sequence_rules.yaml:                 ┃    │
        │  ┃ forbidden_sequences: [                    ┃    │
        │  ┃   ["read_file", "compress", "http_post"] ┃    │
        │  ┃ ]                                         ┃    │
        │  ┃                                           ┃    │
        │  ┃ ✓ Pass: pattern not in forbidden list    ┃    │
        │  ┃ ✗ Fail: SequenceViolationException       ┃    │
        │  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    │
        │                                                   │
        │  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    │
        │  ┃ STAGE 7: Triage Engine Call              ┃    │
        │  ┃                                           ┃    │
        │  ┃ POST http://triage:8001/analyze          ┃    │
        │  ┃ Timeout: 50ms (fail-closed to SANDBOX)   ┃    │
        │  ┃ Payload: full request context            ┃    │
        │  ┃                                           ┃    │
        │  ┃ Response (must be valid TriageResponse):  ┃    │
        │  ┃ {                                         ┃    │
        │  ┃   "verdict": "ALLOW|SANDBOX|BLOCK",      ┃    │
        │  ┃   "score": 0.15,                          ┃    │
        │  ┃   "explanation": "clean_request",        ┃    │
        │  ┃   "request_id": "<id>"                    ┃    │
        │  ┃ }                                         ┃    │
        │  ┃                                           ┃    │
        │  ┃ ✓ Pass: verdict extracted                ┃    │
        │  ┃ ✗ Fail/Timeout: default to SANDBOX       ┃    │
        │  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    │
        │                                                   │
        │  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    │
        │  ┃ STAGE 8: Decision Aggregation            ┃    │
        │  ┃                                           ┃    │
        │  ┃ Combine all stage results:                ┃    │
        │  ┃ - Any BLOCK? → return BLOCK              ┃    │
        │  ┃ - Any SANDBOX from Triage? → SANDBOX    ┃    │
        │  ┃ - All ALLOW? → return ALLOW              ┃    │
        │  ┃                                           ┃    │
        │  ┃ Create DecisionResult:                    ┃    │
        │  ┃ {                                         ┃    │
        │  ┃   "verdict": "ALLOW|SANDBOX|BLOCK",      ┃    │
        │  ┃   "score": 0.15,                          ┃    │
        │  ┃   "reason": "clean_request",             ┃    │
        │  ┃   "decision_latency_ms": 8.3,            ┃    │
        │  ┃   "trace_id": "<id>"                      ┃    │
        │  ┃ }                                         ┃    │
        │  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    │
        │                                                   │
        └─────────────────────┬────────────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │ DECISION RESULT    │
                    │                    │
                    │ verdict: ALLOW     │
                    │ latency: 8.3ms     │
                    │ reason: "clean"    │
                    └─────────┬──────────┘
                              │
                ┌─────────────┴──────────────┐
                │                            │
                ▼                            ▼
           IF ALLOW                    IF BLOCK/SANDBOX
           │                           │
           ├─ Return to callback       ├─ Raise exception
           ├─ Tool executes            ├─ Tool blocked
           ├─ Tool returns result      ├─ No execution
           │                           │
           ▼                           ▼
        ┌──────────────┐           ┌──────────────┐
        │ SANITIZER    │           │ ERROR RETURN │
        │              │           │              │
        │ Input:       │           │ Agent sees:  │
        │ {            │           │ {            │
        │   "status":  │           │   "error":   │
        │   "found",   │           │   "blocked"  │
        │   "ssn":     │           │ }            │
        │   "789-01"   │           │              │
        │ }            │           └──────────────┘
        │              │
        │ TIER 1:      │
        │ Presidio     │
        │ - Scan for   │
        │   PII        │
        │ - Redact:    │
        │   789-01 →   │
        │   <US_SSN_1> │
        │              │
        │ TIER 2:      │
        │ Regex        │
        │ - Check      │
        │   injection  │
        │   patterns   │
        │              │
        │ TIER 3:      │
        │ Semantic     │
        │ - Check      │
        │   jailbreak  │
        │   similarity │
        │              │
        │ Output:      │
        │ {            │
        │   "status":  │
        │   "found",   │
        │   "ssn":     │
        │ "<US_SSN_1>" │
        │ }            │
        └──────────────┘
              │
              ▼
        ┌──────────────────────┐
        │ Safe Output to Agent │
        │                      │
        │ No raw PII values    │
        │ No injection payloads│
        │ No sensitive data    │
        └──────────────────────┘
```

---

## Component Interaction Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                    FASTAPI APPLICATION                         │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Middleware Stack (Innermost → Outermost):                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ InputSizeLimitMiddleware (checks body/header sizes)     │  │
│  │ ↑                                                        │  │
│  │ GlobalLoadSheddingMiddleware (load shedding)            │  │
│  │ ↑                                                        │  │
│  │ RequestTimeoutMiddleware (30s timeout)                  │  │
│  │ ↑                                                        │  │
│  │ TimingSidechannelMitigationMiddleware (jitter)          │  │
│  │ (Outermost)                                             │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          ↓                                     │
│                                                                 │
│  Endpoints:                                                    │
│  ├─ GET /health                                               │
│  ├─ GET /ready                                                │
│  ├─ GET /metrics/security                                     │
│  ├─ GET /test                                                 │
│  └─ POST /execute → process_request() → return decision      │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│                    EXTERNAL DEPENDENCIES                       │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Redis (localhost:6379)                                       │
│  ├─ Rate limit counters (global, per-agent)                  │
│  ├─ Agent sessions (registration status, roles)              │
│  ├─ Sequence history (tool call sequences)                   │
│  └─ Replay protection cache (prevent replays)                │
│                                                                 │
│  OPA Policy Engine (localhost:8181)                           │
│  ├─ RBAC evaluation (agents, tools, permissions)             │
│  └─ Policies: tool_rbac.rego                                 │
│                                                                 │
│  Triage Engine (localhost:8001)                               │
│  ├─ Behavioral analysis                                       │
│  ├─ Anomaly detection                                         │
│  └─ Returns: verdict (ALLOW/SANDBOX/BLOCK) + score           │
│                                                                 │
│  Presidio (Local Python Library)                              │
│  ├─ Pre-loaded at startup                                     │
│  ├─ PII detection (SSN, email, credit card, API key)         │
│  └─ Output sanitization                                       │
│                                                                 │
│  OpenTelemetry Collector                                      │
│  ├─ OTLP exporter (localhost:4317)                           │
│  ├─ Traces → Grafana Tempo                                    │
│  └─ Metrics → Grafana                                         │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│                   INTERNAL COMPONENTS                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AppState (Singleton)                                         │
│  ├─ redis: Redis connection pool                             │
│  ├─ presidio_analyzer: AnalyzerEngine                         │
│  ├─ presidio_anonymizer: AnonymizerEngine                     │
│  ├─ degraded_components: flags (redis, opa)                 │
│  ├─ sequence_rules: YAML config                              │
│  └─ injection_patterns: YAML config                          │
│                                                                 │
│  Security Pipeline (app/pipeline.py)                          │
│  ├─ 8-stage validation system                                 │
│  ├─ Fail-closed defaults (SANDBOX on errors)                 │
│  └─ Returns: DecisionResult (verdict, score, reason)          │
│                                                                 │
│  Output Sanitizer (app/output_sanitizer.py)                   │
│  ├─ Presidio PII detection (Tier 1)                           │
│  ├─ Regex injection scanning (Tier 2)                         │
│  ├─ Semantic injection detection (Tier 3)                     │
│  └─ Returns: sanitized output or blocks                       │
│                                                                 │
│  Behavior Guard (app/behavior_guard.py)                       │
│  ├─ Tracks agent profiles                                     │
│  ├─ Detects anomalies (high failure rate, etc)               │
│  └─ Increases triage score if suspicious                      │
│                                                                 │
│  Advanced Hardening Guards:                                   │
│  ├─ FailureRateGuard (circuit breaker abuse detection)       │
│  ├─ GlobalLoadShedder (resource exhaustion)                   │
│  ├─ EventLoopPressureGuard (async task limits)               │
│  └─ DistributedConsistencyGuard (split-brain prevention)     │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## Security Pipeline Flow (Detailed)

```
REQUEST ARRIVES
      ↓
MIDDLEWARE CHECK
├─ Size check (body < 1MB, headers < 8KB)
├─ Timeout guard (30s timeout set)
├─ Load shedding check (reject if system pressure high)
└─ Timing jitter (add 0-5ms random delay)
      ↓
EXTRACT REQUEST DATA
├─ From body: agent_id, tool_name, token, payload
├─ From headers: Authorization (JWT), x-request-id
└─ Generate: request_id, trace_id, timestamp
      ↓
PIPELINE STAGE 1: GLOBAL RATE LIMIT
├─ Redis Lua script: atomic increment + check
├─ Key: global_count, Limit: 5000 req/sec
└─ Result: [PASS] → continue, [FAIL] → RateLimitException
      ↓
PIPELINE STAGE 2: JWT VALIDATION
├─ Extract token from Authorization header
├─ Decode with RS256 algorithm
├─ Verify signature, expiry, required claims
└─ Result: [PASS] → extracted claims, [FAIL] → JWTError
      ↓
PIPELINE STAGE 3: AGENT SESSION LOOKUP
├─ Query Redis: session:{agent_id}
├─ Check: exists and registered status = true
└─ Result: [PASS] → session found, [FAIL] → RegistrationException
      ↓
PIPELINE STAGE 4: RBAC ENFORCEMENT
├─ Extract agent roles from session
├─ POST to OPA /v1/data/rbac with (agent_id, tool_name, roles)
├─ OPA evaluates tool_rbac.rego policy
└─ Result: [PASS] → OPA allow=true, [FAIL] → RBACDeniedException
      ↓
PIPELINE STAGE 5: PER-AGENT RATE LIMIT
├─ Redis sliding window: agent:{agent_id}
├─ Add current timestamp, remove old entries outside window
├─ Check: count < per-agent limit (e.g., 100 req/min)
└─ Result: [PASS] → under limit, [FAIL] → RateLimitException
      ↓
PIPELINE STAGE 6: SEQUENCE ANALYSIS
├─ Query Redis: seq:{agent_id} (recent tool calls)
├─ Append current tool_name to sequence
├─ Check if sequence matches any forbidden pattern
├─ Example forbidden: [read_file, compress, http_post]
└─ Result: [PASS] → no pattern match, [FAIL] → SequenceViolationException
      ↓
PIPELINE STAGE 7: TRIAGE ENGINE CALL
├─ Prepare payload with full request context
├─ HTTP POST to triage service (50ms timeout)
├─ Response must be valid TriageResponse (Pydantic validated)
├─ Extract: verdict (ALLOW/SANDBOX/BLOCK), score, explanation
└─ Result: [PASS] → verdict extracted, [FAIL/TIMEOUT] → default SANDBOX
      ↓
PIPELINE STAGE 8: DECISION AGGREGATION
├─ Check all results:
│  ├─ Any BLOCK trigger? → DECISION = BLOCK
│  ├─ Any SANDBOX from Triage? → DECISION = SANDBOX
│  └─ All ALLOW? → DECISION = ALLOW
├─ Calculate decision_latency_ms (now - start time)
└─ Return: DecisionResult { verdict, score, reason, latency, trace_id }
      ↓
DECISION HANDLING
├─ If BLOCK or SANDBOX:
│  ├─ Log decision with audit trail
│  ├─ Raise exception (stops tool execution)
│  └─ Return error to LangChain callback
│
└─ If ALLOW:
   ├─ Log decision with audit trail
   ├─ Return to LangChain callback
   ├─ Tool executes normally
   ├─ Tool returns result
   └─ Go to SANITIZATION
         ↓
    SANITIZATION PIPELINE
    ├─ TIER 1: Presidio PII Detection
    │  ├─ Scan output for PII entities
    │  ├─ Redact with confidence > 0.8
    │  ├─ Log: entity_type, offset, confidence (no raw values)
    │  └─ Example: "SSN: 789-01-2345" → "SSN: <US_SSN_1>"
    │
    ├─ TIER 2: Regex Injection Scanning
    │  ├─ Apply patterns from injection_patterns.yaml
    │  ├─ Check for jailbreak indicators
    │  ├─ If found: block entire output
    │  └─ Log: pattern_matched=true (no payload)
    │
    └─ TIER 3: Semantic Injection Detection
       ├─ Compare against jailbreak embeddings
       ├─ If cosine_similarity > 0.85: block
       └─ Log: semantic_match=true (no payload)
         ↓
    SANITIZED OUTPUT
    ├─ Return modified output to LangChain callback
    ├─ Agent receives: safe, redacted version
    └─ No raw PII or injection payloads visible to agent
```

---

## Audit Trail Example

```
Request: web_search(query="current weather")
Agent: agent_001
Timestamp: 2024-05-20T09:15:23.123Z

AUDIT LOG ENTRY:
{
  "timestamp": "2024-05-20T09:15:23.123Z",
  "request_id": "req_abc123def456",
  "trace_id": "trace_xyz789uvw",
  "agent_id": "hash_of_agent_001",  (hashed for privacy)
  
  "security_decision": {
    "verdict": "ALLOW",
    "score": 0.15,
    "reason": "clean_request",
    "decision_latency_ms": 8.3
  },
  
  "pipeline_stages": {
    "global_rate_limit": "PASS",
    "jwt_validation": "PASS",
    "session_lookup": "PASS",
    "rbac_check": "PASS",
    "per_agent_rate_limit": "PASS",
    "sequence_analysis": "PASS",
    "triage_engine": {
      "status": "PASS",
      "triage_score": 0.15,
      "triage_latency_ms": 15.2
    }
  },
  
  "agent_context": {
    "tool_name": "web_search",
    "tool_input": "query=current weather",  (redacted/hashed)
    "roles": ["analyst", "viewer"]
  },
  
  "system_context": {
    "redis_available": true,
    "opa_available": true,
    "memory_pressure": "low",
    "cpu_pressure": "low"
  },
  
  "sanitization": {
    "applied": false,
    "reason": "no_pii_detected"
  }
}

# CRITICAL: NO RAW SENSITIVE DATA IN LOGS
# - Passwords: NOT logged
# - API keys: NOT logged
# - PII: NOT logged (only entity_type logged)
# - Injection payloads: NOT logged
```

---

## Performance Targets

```
GATEWAY LATENCY (Per Stage)
┌─────────────────────────────────────────┐
│ Stage 1 (Global Rate Limit)   : ~0.2ms  │  ← Redis Lua
│ Stage 2 (JWT Validation)      : ~0.5ms  │  ← Crypto
│ Stage 3 (Session Lookup)      : ~0.3ms  │  ← Redis
│ Stage 4 (RBAC/OPA)            : ~3.0ms  │  ← HTTP request
│ Stage 5 (Per-Agent Rate Limit): ~0.3ms  │  ← Redis
│ Stage 6 (Sequence Analysis)   : ~0.5ms  │  ← Memory/Redis
│ Stage 7 (Triage Engine)       : ~3.5ms  │  ← HTTP request
│ Stage 8 (Decision + Logging)  : ~0.2ms  │  ← CPU
├─────────────────────────────────────────┤
│ TOTAL GATEWAY LATENCY p95     : < 10ms  │  ← TARGET
└─────────────────────────────────────────┘

SANITIZATION LATENCY
┌─────────────────────────────────────────┐
│ Presidio (warm start)          : ~15ms  │  ← First call
│ Presidio (cached)              : ~5ms   │  ← Subsequent
│ Regex Injection Scanning       : ~2ms   │  ← Pattern match
│ Semantic Detection             : ~10ms  │  ← Embedding
├─────────────────────────────────────────┤
│ TOTAL SANITIZATION p95        : < 50ms │  ← TARGET
└─────────────────────────────────────────┘

FULL PIPELINE (including tool execution)
┌─────────────────────────────────────────┐
│ Security Pipeline              : < 10ms │
│ Tool Execution (varies)        : ~100ms │ (depends on tool)
│ Output Sanitization            : < 50ms │
├─────────────────────────────────────────┤
│ FULL PIPELINE p95              : < 60ms │  ← TARGET
└─────────────────────────────────────────┘
```

---

## Failure Mode Handling

```
Failure Scenario → Default Behavior → Recovery

1. Redis Unavailable
   └─ Default: SANDBOX verdict (safe)
      └─ Recovery: In-memory rate limiter active
         └─ When Redis returns: auto-resume normal mode

2. OPA Unavailable
   └─ Default: RBAC_DENIED (deny-all, safe)
      └─ Recovery: Cache last-known policies
         └─ When OPA returns: resume policy evaluation

3. Triage Timeout (>50ms)
   └─ Default: SANDBOX verdict (safe)
      └─ Recovery: Retry on next request
         └─ Circuit breaker opens after N failures

4. Presidio Error (PII scan fails)
   └─ Default: Block output (safe)
      └─ Recovery: Skip PII detection, only regex
         └─ Restart Presidio on next tool output

5. System Overload (high CPU/memory)
   └─ Default: Load shedding (reject % of requests)
      └─ Recovery: Start accepting more as load decreases
         └─ Automatic backoff algorithm

All failures are non-silent:
- Always logged with timestamps
- Metrics incremented (redis_failure_total, etc)
- Alerts triggered in monitoring
- Admin notified if degraded > 1 minute
```

---

## Success Criteria Verification

```
LATENCY METRICS
☐ gateway_overhead p95 < 10ms
  └─ Run: locust -f scripts/load_test.py
     Verify: Response time p95

☐ full_pipeline p95 < 60ms
  └─ Run: trace full request in Grafana Tempo
     Verify: Root span duration

ACCURACY METRICS
☐ Rate limit: 20 pass / 30 blocked (exact)
  └─ Run: test_concurrency.py --runs 20 --concurrent 50
     Verify: Same split in all 20 runs

☐ Injection detection: 100% blocked
  └─ Run: test_security_owasp.py
     Verify: All 5 injection variants blocked

☐ PII redaction: correct coverage
  └─ Run: test_pii_sanitization.py
     Verify: SSN → <US_SSN_N>, no raw values

RELIABILITY METRICS
☐ Zero PII in logs
  └─ Run: load test 1000 requests
     Run: grep -r "<PII_VALUE>" logs/
     Verify: Zero matches

☐ Availability (Redis down)
  └─ Run: docker-compose pause redis
     Run: make request
     Verify: SANDBOX verdict (no 500 error)

☐ Recovery (after restart)
  └─ Run: docker-compose unpause redis
     Wait: 5 seconds
     Run: make request
     Verify: ALLOW verdict (normal operation)

All tests passing = Production Ready ✅
```

---

This completes the visual architecture documentation!
