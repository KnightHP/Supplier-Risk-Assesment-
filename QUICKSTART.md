# Apple Supplier Risk Intelligence System - Quick Start Guide

Complete setup and run instructions for the production-grade multi-agent supplier risk assessment platform.

## System Overview

This is a comprehensive system for evaluating supplier risk using:
- **4 specialized risk agents** (Financial, Operational, News/Reputation, Compliance/ESG)
- **LangGraph orchestration** for multi-agent coordination
- **FastAPI** REST service layer
- **SQLite/PostgreSQL** for data persistence
- **Background scheduling** for continuous monitoring
- **Web dashboard** for visualization and control

**Key Principle:** Decision-support only - generates explainable recommendations, never automatic blocking.

## Quick Start (5 minutes)

### Step 1: Set Up Backend

```bash
cd backend

# Create Python virtual environment
python3.11 -m venv venv

# Activate virtual environment
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Run the API server
python -m app.main
```

The backend will start on `http://localhost:8000`

**Verify it's working:**
```bash
curl http://localhost:8000/health
```

Should return:
```json
{
  "status": "healthy",
  "service": "Apple Supplier Risk Intelligence",
  "version": "1.0.0",
  "timestamp": "2025-02-25T10:30:45.123456"
}
```

### Step 2: Set Up Frontend

In a **new terminal window:**

```bash
cd frontend

# Start simple HTTP server
python -m http.server 8080
```

**Open your browser:**
```
http://localhost:8080
```

You should see the Apple Supplier Risk Intelligence dashboard.

### Step 3: Start Monitoring

The system comes with 6 pre-seeded Apple suppliers:
- Foxconn (Taiwan)
- Pegatron (Taiwan)
- TSMC (Taiwan)
- Luxshare (China)
- BOE Technology (China)
- Murata Manufacturing (Japan)

**From the dashboard:**
1. Click **"Refresh Suppliers"** to load the list
2. Click **"Run Monitor Once"** to assess all suppliers
3. Click on any supplier card → **"Evaluate Risk"** for detailed assessment

Or via API:

```bash
# Get all suppliers
curl http://localhost:8000/suppliers

# Evaluate specific supplier
curl -X POST http://localhost:8000/risk/evaluate/foxconn_001

# Get system status
curl http://localhost:8000/status

# Manually trigger monitoring
curl -X POST http://localhost:8000/monitor/run-once
```

## Complete Directory Structure

```
.
├── backend/
│   ├── app/
│   │   ├── main.py                          # FastAPI app - START HERE
│   │   ├── config.py                        # Configuration
│   │   ├── logging_config.py                # Logging setup
│   │   │
│   │   ├── orchestration/
│   │   │   └── supervisor.py                # LangGraph orchestrator
│   │   │
│   │   ├── agents/
│   │   │   ├── financial_agent.py           # Debt & revenue analysis
│   │   │   ├── operational_agent.py         # Delivery & quality
│   │   │   ├── news_agent.py                # News & reputation (LLM)
│   │   │   └── compliance_agent.py          # Sanctions & ESG
│   │   │
│   │   ├── fusion/
│   │   │   ├── risk_fusion.py               # Weighted score combining
│   │   │   └── explainability.py            # Risk explanation
│   │   │
│   │   ├── recommendation/
│   │   │   └── recommender.py               # Decision support actions
│   │   │
│   │   ├── data/
│   │   │   ├── models.py                    # SQLAlchemy models
│   │   │   ├── repository.py                # Database access
│   │   │   ├── feature_store.py             # Feature management
│   │   │   └── sample_suppliers.json
│   │   │
│   │   ├── ingestion/
│   │   │   └── supplier_loader.py           # Data import
│   │   │
│   │   ├── alerts/
│   │   │   └── alert_manager.py             # Alert generation
│   │   │
│   │   ├── scheduler/
│   │   │   └── monitor.py                   # Background monitoring
│   │   │
│   │   └── utils/
│   │       ├── retry.py                     # Retry logic
│   │       └── scoring.py                   # Scoring utilities
│   │
│   ├── requirements.txt
│   └── README.md                            # Backend documentation
│
├── frontend/
│   ├── index.html                           # Web dashboard
│   └── README.md                            # Frontend documentation
│
└── README.md                                # This file
```

## Configuration

### Backend Settings

Edit `backend/app/config.py` for:

```python
# Risk Thresholds
risk_threshold_critical = 0.85
alert_threshold = 0.6  # Alert if score exceeds this

# Risk Weights (sum must = 1.0)
weight_financial = 0.30
weight_operational = 0.25
weight_compliance = 0.25
weight_reputation = 0.20

# Database
database_url = "sqlite:///./supplier_risk.db"  # or PostgreSQL

# Scheduler
scheduler_enabled = True
scheduler_interval_seconds = 300  # Run assessment every 5 minutes

# LLM (News Agent)
mock_llm = True  # Set to False for real OpenAI/Claude
llm_provider = "openai"
llm_api_key = os.getenv("OPENAI_API_KEY")
```

### Environment Variables (Optional)

```bash
export DEBUG=true                           # Enable debug logging
export ENVIRONMENT=development              # development/production
export DATABASE_URL=sqlite:///./supplier_risk.db
export SCHEDULER_ENABLED=true
export LLM_API_KEY=sk-...                   # For real LLM
export MOCK_LLM=true                        # Use mock LLM
```

## API Usage Examples

### Supplier Management

```bash
# List all suppliers
curl http://localhost:8000/suppliers

# Get specific supplier
curl http://localhost:8000/suppliers/foxconn_001

# Add new supplier
curl -X POST http://localhost:8000/suppliers \
  -H "Content-Type: application/json" \
  -d '{
    "supplier_id": "samsung_001",
    "supplier_name": "Samsung",
    "country": "South Korea",
    "industry": "Electronics Manufacturing",
    "annual_revenue": 220000000000,
    "debt_ratio": 0.52,
    "on_time_delivery_rate": 0.97,
    "defect_rate": 0.004,
    "esg_score": 75,
    "sanctions_flag": false
  }'
```

### Risk Assessment

```bash
# Evaluate supplier risk (multi-agent)
curl -X POST http://localhost:8000/risk/evaluate/foxconn_001

# Get latest assessment
curl http://localhost:8000/risk/foxconn_001

# Get historical scores (last 30 days)
curl http://localhost:8000/risk/foxconn_001/history?days=30
```

### Alerts

```bash
# Get unacknowledged alerts
curl http://localhost:8000/alerts

# Acknowledge an alert
curl -X POST http://localhost:8000/alerts/1/acknowledge
```

### System Control

```bash
# Health check
curl http://localhost:8000/health

# System status
curl http://localhost:8000/status

# Scheduler status
curl http://localhost:8000/scheduler/status

# Manually trigger monitoring
curl -X POST http://localhost:8000/monitor/run-once
```

## Assessment Output Example

```json
{
  "supplier_id": "foxconn_001",
  "supplier_name": "Foxconn",
  "assessment": {
    "overall_risk": 0.42,
    "risk_level": "Medium",
    "confidence": 0.85
  },
  "component_scores": {
    "financial": 0.35,
    "operational": 0.30,
    "compliance": 0.48,
    "reputation": 0.45
  },
  "top_risk_drivers": [
    "Compliance and Esg (0.48)",
    "Reputation (0.45)",
    "Financial (0.35)"
  ],
  "apple_impact_narrative": "Foxconn presents moderate risk to Apple supply continuity...",
  "alerts_triggered": 0,
  "timestamp": "2025-02-25T10:35:22.456789"
}
```

## Risk Levels & Actions

| Risk Level | Score | Action | Timeframe |
|-----------|-------|--------|-----------|
| **Low** | <0.30 | Standard monitoring | Quarterly |
| **Medium** | 0.30-0.50 | Monthly review | Within 90 days |
| **High** | 0.50-0.85 | Weekly monitoring | Within 30 days |
| **Critical** | >0.85 | Immediate escalation | Immediate |

## Database

### SQLite (Default - Development)
```bash
# Located in backend/supplier_risk.db
# Auto-created on first run
```

### PostgreSQL (Production)

```bash
# Install PostgreSQL
sudo apt-get install postgresql

# Create database
createdb supplier_risk

# Update config
export DATABASE_URL="postgresql://user:password@localhost/supplier_risk"
```

## Data Flow

```
1. Supplier Data
   ↓
2. Feature Store (load metrics)
   ↓
3. Supervisor (LangGraph)
   ├── Financial Agent → financial_score
   ├── Operational Agent → operational_score
   ├── News Agent → reputation_score
   └── Compliance Agent → compliance_score
   ↓
4. Risk Fusion Engine
   ├── Weighted calculation
   ├── Risk level determination
   └── Apple impact narrative
   ↓
5. Recommendation Engine
   ├── Priority actions
   └── Remediation suggestions
   ↓
6. Alert Manager
   ├── Threshold checks
   └── Alert persistence
   ↓
7. Database Storage
   └── Risk scores, alerts, history
```

## Logging

Logs appear in console with colors:
- 🔵 **DEBUG** (gray)
- ℹ️ **INFO** (blue)
- ⚠️ **WARNING** (yellow)
- ❌ **ERROR** (red)
- 🚨 **CRITICAL** (bold red)

Also written to: `backend/logs/supplier_risk.log`

## Troubleshooting

### Backend won't start

```bash
# Check Python version
python --version  # Should be 3.11+

# Reinstall dependencies
pip install --upgrade -r requirements.txt

# Check for port conflicts
lsof -i :8000  # Should show nothing
```

### Frontend can't connect to API

```bash
# Verify backend is running
curl http://localhost:8000/health

# Check browser console (F12) for CORS errors
# CORS is enabled in backend/app/main.py

# Try accessing API directly from browser dev tools
fetch('http://localhost:8000/suppliers')
```

### No suppliers showing

```bash
# Manually seed database
python -c "
from app.data.repository import Database, SupplierRepository
from app.ingestion.supplier_loader import SupplierLoader

db = Database()
db.create_tables()
session = db.get_session()
loader = SupplierLoader(SupplierRepository(db))
loader.seed_default_apple_suppliers(session)
session.commit()
"

# Or trigger monitoring endpoint
curl -X POST http://localhost:8000/monitor/run-once
```

### Agent timeouts

Increase timeout in `backend/app/config.py`:
```python
agent_timeout_seconds = 20.0  # Increase from 10
```

### Database locked errors (SQLite)

Close other connections and ensure only one backend process:
```bash
pkill -f "python -m app.main"
python -m app.main
```

## Production Deployment

### Docker

```dockerfile
# backend/Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app/ .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
docker build -t apple-supplier-risk:latest .
docker run -p 8000:8000 -e DATABASE_URL=postgresql://... apple-supplier-risk
```

### Using PostgreSQL

```python
# In backend/app/config.py
database_url = "postgresql://user:password@db-host:5432/supplier_risk"
```

Install psycopg2:
```bash
pip install psycopg2-binary
```

### HTTPS/TLS

Deploy behind reverse proxy (nginx/Caddy):

```nginx
# nginx config
upstream api {
    server localhost:8000;
}

server {
    listen 443 ssl;
    server_name api.apple-suppliers.internal;
    
    ssl_certificate /etc/certs/cert.pem;
    ssl_certificate_key /etc/certs/key.pem;
    
    location / {
        proxy_pass http://api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Monitoring & Alerting

Integrate with Datadog/New Relic:

```python
# In app/main.py
from datadog import initialize, api

options = {
    'api_key': os.getenv('DATADOG_API_KEY'),
    'app_key': os.getenv('DATADOG_APP_KEY')
}

initialize(**options)
```

## Testing

Run manual tests:

```bash
# Test financial agent
python -c "
from app.agents.financial_agent import FinancialRiskAgent
from app.data.feature_store import SupplierFeatures

agent = FinancialRiskAgent()
features = SupplierFeatures(
    supplier_id='test',
    supplier_name='Test Supplier',
    country='USA',
    industry='Tech',
    annual_revenue=1e9,
    debt_ratio=0.5,
    on_time_delivery_rate=0.95,
    defect_rate=0.01,
    esg_score=70,
    sanctions_flag=False,
    anchor_company='Apple'
)
result = agent.evaluate(features)
print(result)
"
```

## Next Steps

1. **Define Custom Suppliers** - Add your own suppliers via API
2. **Tune Risk Weights** - Adjust component weights in config
3. **Integrate with LLM** - Set real OpenAI/Claude API keys
4. **Deploy to Production** - Use Docker, PostgreSQL, and reverse proxy
5. **Add Authentication** - Implement JWT or OAuth2
6. **Enable Alerts** - Send alerts to Slack, email, or PagerDuty

## Documentation

- **Backend:** See `backend/README.md`
- **Frontend:** See `frontend/README.md`
- **API Docs:** Open `http://localhost:8000/docs` (Swagger UI)

## Support

For issues or questions:
1. Check logs: `tail -f backend/logs/supplier_risk.log`
2. Review API docs: `http://localhost:8000/docs`
3. Check browser console: Press F12 in frontend

## License

Proprietary - Apple Internal

---

**System Version:** 1.0.0  
**Python:** 3.11+  
**Last Updated:** February 25, 2025
