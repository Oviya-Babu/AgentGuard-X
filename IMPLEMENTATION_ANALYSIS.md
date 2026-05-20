# AgentGuard-X: Complete Implementation Analysis

**Status**: Production-grade core components implemented.
---

## 📊 PART 1: COMPLETE CODE WORKFLOW

### Overall Architecture Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AI AGENT (LangChain/CrewAI)                      │
│                              │                                           │
│                              ├─ Instantiate Agent                        │
│                              ├─ Pass handler: SecurityGatewayAsyncCallbackHandler
│                              └─ Agent executes tool                     │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  LangChain Calls         │
                    │  on_tool_start(...)      │
                    └────────────┬─────────────┘
                                 │
        ┌────────────────────────▼────────────────────────┐
        │  SecurityGatewayAsyncCallbackHandler.on_tool_start()
        │  - Extracts: agent_id, tool_name, tool_input    │
        │  - Generates: request_id, trace_id              │
        └────────────┬─────────────────────────────────────┘
                     │
        ┌────────────▼──────────────────────────────────────┐
        │  REQUEST VALIDATION PIPELINE (app/pipeline.py)   │
        │                                                   │
        │  ┌──────────────────────────────────────────┐    │
        │  │ STAGE 1: Global Rate Limit               │    │
        │  │ ├─ RedisClient.incr("global_count")      │    │
        │  │ ├─ Check against GLOBAL_RATE_LIMIT       │    │
        │  │ ├─ Lua script ensures atomicity          │    │
        │  │ └─ Fail: RateLimitException              │    │
        │  └──────────────────────────────────────────┘    │
        │                                                   │
        │  ┌──────────────────────────────────────────┐    │
        │  │ STAGE 2: JWT Validation                  │    │
        │  │ ├─ Extract from Authorization header     │    │
        │  │ ├─ Decode & verify signature (RS256)     │    │
        │  │ ├─ Check expiry and claims               │    │
        │  │ └─ Fail: JWTError                        │    │
        │  └──────────────────────────────────────────┘    │
        │                                                   │
        │  ┌──────────────────────────────────────────┐    │
        │  │ STAGE 3: Agent Session Lookup            │    │
        │  │ ├─ Redis.get("session:{agent_id}")       │    │
        │  │ ├─ Validate registration status          │    │
        │  │ ├─ Fail: RegistrationException           │    │
        │  └──────────────────────────────────────────┘    │
        │                                                   │
        │  ┌──────────────────────────────────────────┐    │
        │  │ STAGE 4: RBAC Enforcement (OPA)          │    │
        │  │ ├─ POST http://opa:8181/v1/data/rbac    │    │
        │  │ ├─ Payload: agent_id, tool_name, roles  │    │
        │  │ ├─ Fail: RBACDeniedException             │    │
        │  └──────────────────────────────────────────┘    │
        │                                                   │
        │  ┌──────────────────────────────────────────┐    │
        │  │ STAGE 5: Per-Agent Rate Limit            │    │
        │  │ ├─ Redis sliding window: agent:{id}      │    │
        │  │ ├─ Increment and check TTL-based count   │    │
        │  │ ├─ Fail: RateLimitException              │    │
        │  └──────────────────────────────────────────┘    │
        │                                                   │
        │  ┌──────────────────────────────────────────┐    │
        │  │ STAGE 6: Sequence Analysis               │    │
        │  │ ├─ Redis.lrange("seq:{agent_id}", ...)   │    │
        │  │ ├─ Check for exfil patterns              │    │
        │  │ ├─ Example: read_file → compress → post  │    │
        │  │ ├─ Fail: SequenceViolationException      │    │
        │  └──────────────────────────────────────────┘    │
        │                                                   │
        │  ┌──────────────────────────────────────────┐    │
        │  │ STAGE 7: Triage Engine Call              │    │
        │  │ ├─ POST http://triage:8001/analyze       │    │
        │  │ ├─ Payload: full request context         │    │
        │  │ ├─ Timeout: 50ms (defaults to SANDBOX)   │    │
        │  │ ├─ Response: verdict + score             │    │
        │  │ ├─ Fail: TriageBlockException or default │    │
        │  └──────────────────────────────────────────┘    │
        │                                                   │
        │  ┌──────────────────────────────────────────┐    │
        │  │ STAGE 8: Decision Aggregation            │    │
        │  │ ├─ All checks passed → ALLOW             │    │
        │  │ ├─ Triage uncertain → SANDBOX            │    │
        │  │ ├─ Any block trigger → BLOCK             │    │
        │  └──────────────────────────────────────────┘    │
        │                                                   │
        └────────────────┬──────────────────────────────────┘
                         │
                    DECISION: ALLOW / SANDBOX / BLOCK
                         │
        ┌────────────────┴──────────────────────────────┐
        │                                                │
        ▼                                                ▼
     ALLOW                                           BLOCK/SANDBOX
     │                                               │
     ├─ Tool executes normally                       ├─ Return error to agent
     ├─ Tool returns result                          ├─ Log decision
     ├─ Result passes to SANITIZER                   └─ Audit trail
     │
     ▼
   SANITIZER (app/output_sanitizer.py)
   ├─ TIER 1: Presidio PII Detection
   │  ├─ Scan for: SSN, email, credit card, API key
   │  ├─ Confidence threshold: 0.8
   │  ├─ Action: Redact as <ENTITY_TYPE_N>
   │  └─ Log: entity_type, offset, confidence (no PII value)
   │
   ├─ TIER 2: Regex Injection Scanning
   │  ├─ Patterns from injection_patterns.yaml
   │  ├─ Examples: "ignore instructions", "[SYSTEM]:"
   │  ├─ Action: Block if found
   │  └─ Log: pattern match indicator (no payload)
   │
   └─ TIER 3: Semantic Injection Detection
      ├─ Compare against known jailbreak embeddings
      ├─ Cosine similarity > threshold
      ├─ Action: Block if detected
      └─ Log: semantic_match=true

     ▼
   SANITIZED OUTPUT
   │
   └─ Return to Agent
      └─ Agent sees safe, redacted output
```

---

## 📋 PART 2:

### Component 1: Core FastAPI Application (`app/main.py`)

**Purpose**: Initialize and configure the entire gateway

**Key Responsibilities**:
1. **Startup Sequence**:
   - Import OTel setup (must be first)
   - Initialize logging with PII filter
   - Create AppState singleton
   - Initialize Redis connection pool
   - Load Presidio analyzers (pre-warm)
   - Check OPA health
   - Load YAML configurations (sequence_rules, injection_patterns)

2. **AppState Container**:
   ```python
   class AppState:
       redis: Optional[Redis] = None
       presidio_analyzer: Optional[AnalyzerEngine] = None
       presidio_anonymizer: Optional[AnonymizerEngine] = None
       degraded_components: Dict[str, bool] = {"redis": False, "opa": False}
       sequence_rules: Optional[Any] = None
       injection_patterns: Optional[Any] = None
   ```
   - Single source of truth for all global state
   - Allows graceful degradation when components fail

3. **Initialization Functions**:
   - `init_redis()`: Creates connection pool with retry logic, marks component degraded if fails
   - `init_opa_health_check()`: Probes OPA endpoint, graceful failure
   - `init_presidio()`: Pre-loads analyzers to avoid cold-start latency
   - `load_yaml_configs()`: Parses sequence_rules.yaml and injection_patterns.yaml

4. **Health & Readiness Endpoints** (TODO):
   - `/health`: Simple `{"status": "ok"}`
   - `/ready`: Check Redis + OPA connectivity, return component status

---

### Component 2: Security Pipeline (`app/pipeline.py`)

**Purpose**: 8-stage request validation with fail-closed behavior

**DataModels**:
```python
class RequestContext(BaseModel):
    """Request metadata"""
    agent_id: str
    tool_name: str
    tool_input: Dict[str, Any]
    jwt_token: str
    timestamp: float
    request_id: str
    trace_id: str

class DecisionResult(BaseModel):
    """Pipeline output"""
    verdict: Literal["ALLOW", "SANDBOX", "BLOCK"]
    score: float  # 0.0-1.0 (higher = more risky)
    reason: str   # Internal (not exposed to agent)
    decision_latency_ms: float
    trace_id: str
```

**Pipeline Stages**:

1. **Stage 1: Global Rate Limit**
   - Lua script: atomically increments `global_count`, checks against limit
   - Fails with RateLimitException if exceeded
   - Per-second limit across all agents combined
   - Key: `global_count`, TTL: 1 second

2. **Stage 2: JWT Validation**
   - Extract from `Authorization: Bearer <token>` header
   - Decode with RS256 algorithm
   - Verify: signature, expiry (iat + exp), required claims (agent_id, sub)
   - Fails with JWTError if invalid

3. **Stage 3: Agent Session Lookup**
   - Query Redis: `session:{agent_id}`
   - Expected format: JSON with registration status, roles
   - Fails with RegistrationException if not found or unregistered

4. **Stage 4: RBAC Enforcement (OPA)**
   - POST to `http://opa:8181/v1/data/rbac`
   - Payload:
     ```json
     {
       "agent_id": "agent_001",
       "tool_name": "read_file",
       "roles": ["data_analyst", "viewer"]
     }
     ```
   - OPA returns allow/deny decision
   - Fails with RBACDeniedException if denied

5. **Stage 5: Per-Agent Rate Limit**
   - Redis sliding window: `agent:{agent_id}`
   - Maintain list of recent request timestamps
   - Remove timestamps older than window (e.g., 60 seconds)
   - Check count < per-agent limit
   - Fails if exceeded

6. **Stage 6: Sequence Analysis**
   - Redis list: `seq:{agent_id}` stores recent tool calls
   - Load sequence rules from YAML
   - Check if current call completes a forbidden sequence
   - Example forbidden: [read_file, compress, http_post] = exfiltration
   - Fails with SequenceViolationException if pattern detected

7. **Stage 7: Triage Engine Call**
   - HTTP POST to external triage service (configurable URL)
   - 50ms timeout (fail-closed to SANDBOX if timeout)
   - Payload includes full context (agent, tool, request history)
   - Response validated as TriageResponse (strict Pydantic model)
   - Returns verdict (ALLOW/SANDBOX/BLOCK) + score + explanation

8. **Stage 8: Decision Aggregation**
   - Combine results from all stages
   - Priority:
     - Any BLOCK trigger → return BLOCK
     - Triage SANDBOX → return SANDBOX
     - All passed → return ALLOW

**Fail-Closed Behavior**:
- If Redis unavailable → falls back to in-memory rate limiter (stricter in distributed mode)
- If OPA unavailable → deny all (RBAC_DENIED)
- If Triage timeout → SANDBOX (never ALLOW)
- If JWT invalid → block immediately

---

### Component 3: LangChain Callback Handler (`app/callback_handler.py`)

**Purpose**: Bridge between LangChain agents and security gateway

**Key Methods**:
```python
async def on_tool_start(
    self,
    serialized: Dict[str, Any],
    input_str: str,
    **kwargs: Any
) -> None:
    """
    Called BEFORE tool execution by LangChain.
    
    Flow:
    1. Extract agent_id, tool_name from serialized input
    2. Generate request_id, trace_id
    3. Create RequestContext
    4. Call process_request() from pipeline
    5. If BLOCK/SANDBOX: raise exception (stops tool execution)
    6. If ALLOW: continue (tool executes)
    """

async def on_tool_end(
    self,
    output: str,
    **kwargs: Any
) -> None:
    """
    Called AFTER tool execution.
    
    Flow:
    1. Call sanitize_tool_output(output)
    2. Returns redacted version with PII removed
    3. Modify agent's context with sanitized output
    """
```

**Integration Example**:
```python
from app.callback_handler import SecurityGatewayAsyncCallbackHandler

# Initialize handler
handler = SecurityGatewayAsyncCallbackHandler(
    agent_id="agent_001",
    jwt_token="eyJhbGc..."
)

# Pass to agent (LangChain handles the rest)
agent = initialize_agent(
    tools=[web_search, read_file, send_email],
    llm=llm,
    callbacks=[handler],
    agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
)

# When agent.run() is called:
# - Every tool_start triggers security check
# - Every tool_end triggers sanitization
```

---

### Component 4: Output Sanitization (`app/output_sanitizer.py`)

**Purpose**: Redact PII and block injected content before agent sees output

**Three-Tier Detection**:

1. **Tier 1: Presidio PII Detection**
   - Pre-loaded AnalyzerEngine (warm start)
   - Detects:
     - US_SSN: `123-45-6789` → `<US_SSN_1>`
     - CREDIT_CARD: `4111111111111111` → `<CREDIT_CARD_1>`
     - EMAIL: `user@example.com` → `<EMAIL_1>`
     - API_KEY: patterns → `<API_KEY_1>`
     - PERSON, LOCATION, ORGANIZATION
   - Confidence threshold: 0.8 (tunable)
   - **Logging**: Only entity_type, offset, confidence (never raw PII)

2. **Tier 2: Regex Injection Scanning**
   - Patterns loaded from `injection_patterns.yaml`
   - Examples:
     - `(?i)ignore.*instructions`
     - `(?i)\[SYSTEM\]:`
     - `(?i)pretend.*no.*restrictions`
   - Action: If found, block entire output
   - **Logging**: Pattern match indicator, no payload

3. **Tier 3: Semantic Injection Detection**
   - Compare output against known jailbreak embeddings
   - Cosine similarity threshold: 0.85
   - Uses pre-computed embeddings of known attacks
   - Action: If similar to known attack, block
   - **Logging**: `semantic_match=true`, no payload

**Output Modification**:
```python
sanitized_output = await sanitize_tool_output(
    output=tool_result,
    agent_id=agent_id,
    tool_name=tool_name
)

# Output is a new copy, never modified in-place
# Original tool_result is discarded
```

**Fail-Closed Behavior**:
- If Presidio fails to initialize: reject all outputs (BLOCK)
- If sanitization raises exception: reject output (BLOCK)
- If injection detected: reject output (BLOCK)

---

### Component 5: Behavior Guard (`app/behavior_guard.py`)

**Purpose**: Detect insider threats and malicious agent patterns

**AgentProfile Tracking**:
```python
@dataclass
class AgentProfile:
    agent_id: str
    first_seen: float
    last_seen: float
    request_count: int
    tool_requests: Dict[str, int]      # {tool: count}
    hourly_requests: Dict[int, int]    # {hour: count}
    failure_count: int
    block_count: int
```

**Anomaly Signals**:
1. **High Failure Rate**: `failure_count / request_count > 0.5`
2. **Sudden Request Spike**: `requests_in_1min > baseline * 10`
3. **Unusual Tool Combinations**: Tool entropy drops (suspicious repetition)
4. **Repeated Blocking**: `block_count > 10` in short timespan
5. **Temporal Anomaly**: Requests at unusual hours for agent

**Action**: Log anomaly alert, increase triage score, may trigger SANDBOX

---

### Component 6: Advanced Hardening (`app/hardening_advanced.py`)

**Purpose**: Defend against sophisticated attacks on the gateway itself

**FailureRateGuard**:
- Tracks failure patterns per source
- If same source repeatedly triggers failures → BLOCK instead of SANDBOX
- Prevents circuit breaker abuse (intentional failures to get SANDBOX)
- Window: 60 seconds, threshold: >2 suspicious failures/sec

**GlobalLoadShedder**:
- Monitors system resource pressure (CPU, memory)
- If pressure high: start rejecting requests (100% → 0% accept rate gradually)
- Prevents cascading failure under load
- Thresholds: CPU >80%, Memory >85%

**EventLoopPressureGuard**:
- Tracks pending async tasks
- If too many pending: start shedding requests
- Prevents event loop stall (latency spike → cascade → collapse)
- Threshold: >1000 pending tasks

**DistributedConsistencyGuard**:
- In distributed mode: validates Redis is reachable
- If unreachable: switch to deny-all (safer than inconsistent allow/deny)
- Prevents split-brain scenarios where gateway nodes disagree

---

### Component 7: Middleware Stack (`app/security_middleware.py`)

**InputSizeLimitMiddleware**:
- Rejects bodies > 1MB
- Rejects headers > 8KB per header
- Prevents buffer overflow / memory exhaustion

**RequestTimeoutMiddleware**:
- Enforces 30-second timeout per request
- Prevents slowloris / hanging connection attacks

**TimingNormalizationMiddleware**:
- Adds random jitter (0-5ms) to response time
- Prevents timing side-channel attacks (e.g., guessing if agent is registered)

---

### Component 8: mTLS Validation (`app/mtls_advanced.py`)

**Purpose**: Ensure triage engine calls are from legitimate service

**Validation Checks**:
1. Certificate chain validity (expiry, issuer)
2. CN (Common Name) matches expected service identity
3. SAN (Subject Alternative Names) includes expected service
4. Optional: certificate pinning (hash verification)

**Fail-Closed**:
- If mTLS disabled but cert validation fails: reject call
- If cert untrusted: reject call
- Never accept unverified TLS connections

---

### Component 9: OTel Tracing (`app/observability/otel_setup.py`)

**Purpose**: Full request tracing for debugging and auditing

**Trace Structure**:
```
Root Span: on_tool_start (LangChain callback)
  │
  ├─ Child: Global Rate Limit Check
  ├─ Child: JWT Validation
  ├─ Child: Agent Session Lookup
  ├─ Child: OPA RBAC Call
  │  └─ Attributes: agent_id, tool_name, decision
  ├─ Child: Per-Agent Rate Limit
  ├─ Child: Sequence Analysis
  ├─ Child: Triage Engine Call
  │  └─ Attributes: triage_url, timeout_ms, response_status
  └─ Child: Decision + Sanitization
     └─ Attributes: verdict, score, decision_latency_ms
```

**Propagation**:
- Uses W3C traceparent header for cross-service tracing
- OTel SDK exports to Grafana Loki/Tempo
- Latency metrics recorded at each stage

---

