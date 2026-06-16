# System Architecture & Design Documentation

## High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          WEB DASHBOARD                           │
│                    (Frontend - HTML/CSS/JS)                       │
│  - Supplier grid view                                              │
│  - Risk assessment modal                                           │
│  - Alerts panel                                                    │
│  - System status monitoring                                        │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                        HTTP/REST │
                                 │
┌────────────────────────────────▼────────────────────────────────┐
│                        FASTAPI SERVICE                           │
│  (backend/app/main.py)                                            │
│                                                                   │
│  ├─ Health & Status Endpoints                                     │
│  ├─ Supplier Management (CRUD)                                    │
│  ├─ Risk Assessment API                                           │
│  ├─ Alert Management                                              │
│  └─ Monitoring & Scheduling                                       │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                      ┌──────────┴──────────┐
                      │                     │
         ┌────────────▼────────────┐    ┌──▼──────────────────────┐
         │ ORCHESTRATION LAYER     │    │ DATA PERSISTENCE LAYER  │
         │ (LangGraph Supervisor)  │    │ (SQLAlchemy + SQLite)   │
         │                         │    │                         │
         │ ┌─────────────────────┐ │    │ ┌───────────────────┐   │
         │ │ Financial Agent     │ │    │ │ Supplier Table    │   │
         │ └─────────────────────┘ │    │ ├───────────────────┤   │
         │ ┌─────────────────────┐ │    │ │ RiskScore Table   │   │
         │ │ Operational Agent   │ │    │ ├───────────────────┤   │
         │ └─────────────────────┘ │    │ │ Alert Table       │   │
         │ ┌─────────────────────┐ │    │ └───────────────────┘   │
         │ │ News/Reputation Ag. │ │    │                         │
         │ └─────────────────────┘ │    │ supplier_risk.db        │
         │ ┌─────────────────────┐ │    │ (SQLite)                │
         │ │ Compliance/ESG Ag.  │ │    │                         │
         │ └─────────────────────┘ │    │ OR postgresql://...     │
         │         ↓               │    └──────────────────────────┘
         │ ┌─────────────────────┐ │
         │ │ Risk Fusion Engine  │ │
         │ │ (Weighted Scoring)  │ │
         │ └─────────────────────┘ │
         │         ↓               │
         │ ┌─────────────────────┐ │
         │ │ Recommendation Eng. │ │
         │ └─────────────────────┘ │
         │         ↓               │
         │ ┌─────────────────────┐ │
         │ │ Alert Manager       │ │
         │ └─────────────────────┘ │
         └────────────────────────┘
                                 │
            ┌────────────────────┼────────────────────┬────────┐
            │                    │                    │        │
     ┌──────▼──────┐      ┌──────▼──────┐    ┌──────▼──────┐  │
     │   Feature   │      │ Scheduler   │    │Explainability│  │
     │    Store    │      │  (Monitor)  │    │   Engine     │  │
     └─────────────┘      └─────────────┘    └──────────────┘  │
                                                           ┌─────▼──┐
                                                           │Logging │
                                                           └────────┘
```

## Detailed Component Architecture

### 1. Frontend Layer (browser-based)
- **Technology:** HTML5, CSS3, Vanilla JavaScript
- **Responsibilities:**
  - Display supplier metrics and risk assessments
  - Trigger risk evaluations on-demand
  - Show alerts and system status
  - Provide interactive dashboard

### 2. API Layer (FastAPI)
- **Framework:** FastAPI 0.104+
- **Port:** 8000 (default)
- **Responsibilities:**
  - REST endpoints for all operations
  - Request/response validation with Pydantic
  - CORS middleware for frontend communication
  - Async request handling
  - Startup/shutdown lifecycle management

**Key Endpoints:**
```
GET    /health                              # Health check
GET    /status                              # System status
POST   /suppliers                           # Create supplier
GET    /suppliers                           # List suppliers
GET    /suppliers/{id}                      # Get supplier
POST   /risk/evaluate/{supplier_id}        # Evaluate risk
GET    /risk/{supplier_id}                 # Get latest score
GET    /risk/{supplier_id}/history         # Historical scores
GET    /alerts                              # Get alerts
POST   /alerts/{id}/acknowledge            # Acknowledge alert
POST   /monitor/run-once                   # Trigger monitoring
GET    /scheduler/status                   # Scheduler status
```

### 3. Orchestration Layer (LangGraph)
- **Technology:** LangGraph 0.0.25+
- **Pattern:** State machine graph
- **Responsibilities:**
  - Coordinate multi-agent workflow
  - Manage agent execution order
  - Handle partial failures gracefully
  - Aggregate agent outputs

**Graph Structure:**
```
START
  ├─ FinancialAgent
  ├─ OperationalAgent
  ├─ NewsAgent
  ├─ ComplianceAgent
  ├─ RiskFusionEngine
  ├─ RecommendationEngine
  └─ END

State Flow: Supplier → Agents → Fusion → Recommendations → Alerts → DB
```

### 4. Risk Agents (Specialized Evaluators)

#### 4a. Financial Risk Agent
```python
Inputs:
  - debt_ratio: float
  - annual_revenue: float

Outputs:
  - financial_score: 0.0-1.0
  - debt_ratio_component: float
  - revenue_component: float
  - confidence: float
  - explanation: str

Logic:
  - debt_risk = normalize(debt_ratio, 0-2.0)
  - revenue_risk = invert(normalize(revenue, 0-10B))
  - score = 0.6 × debt_risk + 0.4 × revenue_risk
```

#### 4b. Operational Risk Agent
```python
Inputs:
  - on_time_delivery_rate: float (0-1)
  - defect_rate: float (0-1)

Outputs:
  - operational_score: 0.0-1.0
  - delivery_component: float
  - quality_component: float
  - confidence: float
  - explanation: str

Logic:
  - delivery_risk = invert(on_time_rate)
  - quality_risk = normalize(defect_rate, 0-0.1)
  - score = 0.55 × delivery_risk + 0.45 × quality_risk
```

#### 4c. News & Reputation Agent
```python
Inputs:
  - supplier_name: str
  - anchor_company: str (Apple)

Process:
  1. Fetch news via NewsAPI (or mock)
  2. Use LLM to extract risk signals
  3. Analyze sentiment

Outputs:
  - reputation_score: 0.0-1.0
  - sentiment: "positive", "neutral", "negative"
  - news_count: int
  - evidence_snippets: list[str]
  - confidence: float

LLM Integration:
  - Supports: OpenAI, Claude, or mock
  - Prompt pattern: "{name} Apple supplier lawsuit OR labor OR violation"
```

#### 4d. Compliance & ESG Agent
```python
Inputs:
  - sanctions_flag: bool
  - esg_score: float (0-100)
  - country: str

Outputs:
  - compliance_score: 0.0-1.0
  - sanctions_risk: float
  - esg_component: float
  - regional_risk: float
  - violations: list[str]
  - confidence: float

Regional Risk Mapping:
  China: 0.30
  Taiwan: 0.25
  Russia: 0.50
  Iran: 0.60
  Others: 0.10
```

### 5. Risk Fusion Engine
```
Formula:
  overall_risk = 
    0.30 × financial +
    0.25 × operational +
    0.25 × compliance +
    0.20 × reputation

Outputs:
  - overall_risk: float (0-1)
  - risk_level: "Low" | "Medium" | "High" | "Critical"
  - confidence: weighted_avg(component_confidences)
  - top_risk_drivers: list[str] (top 3)
  - apple_impact_narrative: str (Apple-specific)

Risk Level Thresholds:
  Low:       0.0 - 0.30
  Medium:    0.30 - 0.50
  High:      0.50 - 0.85
  Critical:  0.85 - 1.0
```

### 6. Recommendation Engine
```
Decision-Support Rules:
  
  CRITICAL (score > 0.85):
    → Escalate immediately
    → Emergency audit required
    → Activate contingency suppliers
    → Daily monitoring
  
  HIGH (0.50-0.85):
    → Schedule audit within 30 days
    → Weekly business reviews
    → Develop remediation plan
    → Consider order reduction
  
  MEDIUM (0.30-0.50):
    → Schedule audit within 90 days
    → Monthly monitoring
    → Maintain standard processes
  
  LOW (< 0.30):
    → Standard monitoring
    → Quarterly reviews
```

### 7. Alert Manager
```
Trigger Conditions:
  1. Threshold Breach
     - If overall_risk > alert_threshold (default 0.60)
     - Create "threshold_breach" alert
  
  2. Compliance Violation
     - If compliance_score > 0.75
     - Create "compliance_violation" alert
  
  3. Risk Spike
     - If |current - previous| > 0.20
     - Create "risk_spike" alert
  
  4. Critical Risk
     - If risk_level == "Critical"
     - Create "critical_risk" alert

Severity Mapping:
  Critical → Critical severity
  High → High severity
  Medium → Medium severity
  Low → Low severity
```

### 8. Background Scheduler
```
Technology: APScheduler (BackgroundScheduler)

Job:
  Every N seconds (configurable, default 300):
    1. Load all suppliers
    2. For each supplier:
       - Get features
       - Run orchestration
       - Store risk score
       - Process alerts
    3. Log completion/errors

Error Handling:
  - Failures don't stop other suppliers
  - Detailed logging of failures
  - Graceful shutdown support
  - Configurable max instances (prevent overlap)
```

### 9. Data Layer (SQLAlchemy + SQLite/PostgreSQL)

#### Supplier Table
```sql
CREATE TABLE suppliers (
  id INTEGER PRIMARY KEY,
  supplier_id VARCHAR(100) UNIQUE NOT NULL,
  supplier_name VARCHAR(255) NOT NULL,
  country VARCHAR(100),
  industry VARCHAR(255),
  annual_revenue FLOAT,
  debt_ratio FLOAT,
  on_time_delivery_rate FLOAT,
  defect_rate FLOAT,
  esg_score FLOAT,
  sanctions_flag BOOLEAN,
  anchor_company VARCHAR(100),  -- "Apple"
  created_at DATETIME,
  updated_at DATETIME
);
```

#### RiskScore Table
```sql
CREATE TABLE risk_scores (
  id INTEGER PRIMARY KEY,
  supplier_id VARCHAR(100),
  supplier_name VARCHAR(255),
  financial_score FLOAT,
  operational_score FLOAT,
  compliance_score FLOAT,
  reputation_score FLOAT,
  overall_risk FLOAT,
  risk_level VARCHAR(50),       -- "Low"|"Medium"|"High"|"Critical"
  confidence FLOAT,
  top_risk_drivers VARCHAR(1000), -- JSON string
  anchor_company VARCHAR(100),
  timestamp DATETIME
);
-- Indexes: (supplier_id, timestamp), (anchor_company), (timestamp)
```

#### Alert Table
```sql
CREATE TABLE alerts (
  id INTEGER PRIMARY KEY,
  supplier_id VARCHAR(100),
  supplier_name VARCHAR(255),
  alert_type VARCHAR(100),     -- "threshold_breach"|"compliance"|"spike"
  severity VARCHAR(50),         -- "low"|"medium"|"high"|"critical"
  message VARCHAR(1000),
  risk_score_value FLOAT,
  previous_risk_score FLOAT,
  anchor_company VARCHAR(100),
  acknowledged BOOLEAN,
  created_at DATETIME
);
-- Indexes: (supplier_id, created_at), (anchor_company), (acknowledged)
```

### 10. Feature Store
```
Purpose: Centralized feature loading and caching

Operations:
  - get_supplier_features(supplier_id)
    → Returns SupplierFeatures dataclass
  
  - get_all_supplier_features(anchor_company)
    → Returns list of SupplierFeatures
  
  - get_previous_risk_score(supplier_id)
    → Returns float (latest overall_risk)

Caching Strategy:
  - In-memory during request (session-scoped)
  - Database queries for persistence
  - No distributed cache (SQLite limitation)
```

## Data Flow Sequence

```
1. USER REQUEST
   ├─ Frontend: Click "Evaluate Risk"
   └─ Backend: POST /risk/evaluate/{supplier_id}

2. DATA LOADING
   ├─ Repository: Load supplier from DB
   └─ FeatureStore: Convert to features

3. ORCHESTRATION
   ├─ Supervisor.evaluate(features)
   └─ LangGraph executes state machine:
      
      State = OrchestratorState(supplier_features)
      
      Node 1: FinancialAgent.evaluate()
         → State.financial_output = FinancialRiskOutput
      
      Node 2: OperationalAgent.evaluate()
         → State.operational_output = OperationalRiskOutput
      
      Node 3: NewsAgent.evaluate()
         → State.news_output = NewsRiskOutput
      
      Node 4: ComplianceAgent.evaluate()
         → State.compliance_output = ComplianceRiskOutput
      
      Node 5: RiskFusionEngine.fuse()
         → State.fused_assessment = FusedRiskAssessment
      
      Node 6: RecommendationEngine.generate()
         → State.recommendations = RecommendationPackage

4. ALERT GENERATION
   ├─ AlertManager.process_assessment()
   ├─ Check: threshold_breach?
   ├─ Check: compliance_violation?
   ├─ Check: risk_spike?
   └─ Store alerts in DB

5. PERSISTENCE
   ├─ Store RiskScore record
   ├─ Store Alert records
   └─ Update timestamps

6. RESPONSE
   ├─ ExplainabilityEngine.generate_detailed_report()
   └─ Return JSON to frontend

7. DISPLAY
   ├─ Frontend: Show modal
   ├─ Display component scores
   ├─ Show top risk drivers
   └─ Display recommendations
```

## Key Design Patterns

### 1. Multi-Agent Pattern
- Decoupled agents evaluate different risk dimensions
- No inter-agent dependencies
- Graceful degradation on single agent failure

### 2. State Machine Pattern (LangGraph)
- Deterministic workflow execution
- Clear state transitions
- Testable and debuggable

### 3. Repository Pattern
- Data access abstraction
- Supports SQLite → PostgreSQL migration
- Type-safe with SQLAlchemy

### 4. Strategy Pattern
- LLM selection (mock vs real)
- Database selection (SQLite vs PostgreSQL)
- Alert reporters (logging vs external)

### 5. Decorator Pattern
- @retry for resilience
- @async_retry for async operations

### 6. Observer Pattern
- Scheduler triggers monitoring
- Alerts published to observers

## Extensibility Points

### Add New Agent
1. Create `app/agents/new_agent.py`
2. Implement `evaluate(features) → OutputDataclass`
3. Add to supervisor graph
4. Include output in risk fusion

### Add New Risk Factor
1. Add field to Supplier model
2. Update agent(s) to use field
3. Adjust risk fusion weights

### Add New Alert Trigger
1. Add condition in `alert_manager.process_assessment()`
2. Define alert_type and severity
3. Create AlertTrigger and persist

### Integrate Real LLM
1. Set LLM_API_KEY environment variable
2. Set mock_llm = False in config
3. Implement actual LLM call in news_agent

### Multi-OEM Support
1. Suppliers already tagged with anchor_company
2. Add filter/query parameter `?anchor_company=Tesla`
3. No code changes needed - configuration only

## Performance Characteristics

### Single Assessment
- **Total Time:** 5-10 seconds (typical)
- Financial Agent: 500ms
- Operational Agent: 500ms
- News Agent: 3-5s (LLM latency)
- Compliance Agent: 500ms
- Fusion & Recommendations: 500ms

### Monitoring Cycle (All Suppliers)
- 6 Suppliers × 8s = ~48s overhead
- Parallel agents could reduce to ~8s per supplier
- Typically runs in background every 5 minutes

### Database Operations
- Query: <10ms
- Insert: 10-50ms
- Update: 10-50ms

## Security Considerations

### 1. Input Validation
- Pydantic models for all inputs
- SQLAlchemy parameterized queries
- No SQL injection risk

### 2. Authentication (Future)
- Add JWT/OAuth2 to protected endpoints
- API keys for service-to-service

### 3. Data Sensitivity
- Sanitize supplier names in logs
- Never log financial details
- Encrypt sensitive fields at rest

### 4. API Security
- CORS configured
- Rate limiting (recommended)
- HTTP-only cookies for tokens

## Monitoring & Observability

### Logging
```python
# Colored console output
# File logging to logs/supplier_risk.log
# Rotation: 10MB max, 5 backups

Log Levels:
  DEBUG   - Agent internals, metrics
  INFO    - Assessments, alerts
  WARNING - Failures, retries
  ERROR   - Exceptions
  CRITICAL - System failures
```

### Metrics to Track
- Assessment runtime per agent
- Assessment completeness (% agents succeeded)
- Alert trigger rates
- Database query times
- Scheduler execution frequency

### Health Checks
```
GET /health → Returns service status
GET /status → Returns system metrics
GET /scheduler/status → Returns monitoring status
```

## Disaster Recovery

### Database Backup
```bash
# SQLite
cp supplier_risk.db supplier_risk.db.backup.$(date +%s)

# PostgreSQL
pg_dump supplier_risk > backup.sql
```

### Supplier Reseeding
```bash
DELETE FROM suppliers;
# Restart server to trigger seed_default_apple_suppliers()
```

### Past Assessments
```sql
SELECT * FROM risk_scores
WHERE timestamp >= DATE_SUB(NOW(), INTERVAL 30 DAY)
ORDER BY timestamp DESC;
```

---

**Architecture Version:** 1.0  
**Last Updated:** February 25, 2025
