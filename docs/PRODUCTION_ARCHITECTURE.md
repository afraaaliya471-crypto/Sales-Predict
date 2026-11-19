# Production Architecture: OData + Secure Database

## 🏗️ Production-Grade Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DYNAMICS 365 (D365)                                 │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                    │
│  │ Sales Orders │  │ Inventory    │  │ Customers    │                    │
│  │ (SalesTable) │  │ (InventTrans)│  │ (CustTable)  │                    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                    │
│         │                  │                  │                            │
│         └──────────────────┴──────────────────┘                            │
│                            │                                               │
│                    ┌───────▼───────┐                                       │
│                    │  OData API    │  (Standard D365 endpoint)             │
│                    │  /data/...    │                                       │
│                    └───────┬───────┘                                       │
└────────────────────────────┼───────────────────────────────────────────────┘
                             │ HTTPS + OAuth 2.0 (Azure AD)
                             │ Encrypted channel
                             │
┌────────────────────────────▼───────────────────────────────────────────────┐
│                    DATA INGESTION LAYER (Python)                           │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  ETL Service (Apache Airflow / Prefect / Azure Data Factory)       │  │
│  │                                                                     │  │
│  │  • Authenticates with D365 (OAuth 2.0 + Service Principal)         │  │
│  │  • Pulls data via OData queries                                    │  │
│  │  • Applies business filters (Sales Order, Open, No DNP, etc.)     │  │
│  │  • Transforms and validates data                                   │  │
│  │  • Loads to production database                                    │  │
│  │                                                                     │  │
│  │  Schedule: Every 15 min / hourly / daily (configurable)            │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬───────────────────────────────────────────────┘
                             │
                             │ Writes to
                             │
┌────────────────────────────▼───────────────────────────────────────────────┐
│                    PRODUCTION DATABASE                                     │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │  OPTION 1: PostgreSQL (Recommended for most cases)                  │ │
│  │  • Open-source, enterprise-grade                                    │ │
│  │  • ACID compliant, JSONB support                                    │ │
│  │  • Excellent performance for OLAP/OLTP hybrid                       │ │
│  │  • Azure Database for PostgreSQL / AWS RDS / Self-hosted           │ │
│  │  • Cost: ~$50-200/month (managed service)                           │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │  OPTION 2: SQL Server (If already in Microsoft ecosystem)           │ │
│  │  • Native D365 integration                                          │ │
│  │  • Azure SQL Database / SQL Managed Instance                        │ │
│  │  • Best for large enterprises with Microsoft licensing              │ │
│  │  • Cost: ~$100-500/month (managed service)                          │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │  OPTION 3: Snowflake (For analytics-heavy workloads)                │ │
│  │  • Cloud data warehouse, columnar storage                           │ │
│  │  • Scales compute/storage independently                             │ │
│  │  • Best for forecasting on 100M+ rows                               │ │
│  │  • Cost: Pay-per-use (can be expensive)                             │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                            │
│  Schema:                                                                  │
│  • sales_orders (cleaned, indexed)                                       │
│  • inventory_transactions (cleaned, indexed)                             │
│  • customers (dimension table)                                           │
│  • forecast_cache (precomputed forecasts)                                │
└────────────────────────────┬───────────────────────────────────────────────┘
                             │
                             │ Queries via
                             │
┌────────────────────────────▼───────────────────────────────────────────────┐
│                    BACKEND API (FastAPI)                                   │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │  Application Server (Current: src/backend/main.py)                   │ │
│  │                                                                      │ │
│  │  • Authentication & Authorization (JWT/OAuth)                        │ │
│  │  • Rate limiting (Redis)                                             │ │
│  │  • Text-to-SQL generation (LLM)                                      │ │
│  │  • Query execution (parameterized, SQL injection safe)               │ │
│  │  • Forecasting service (Prophet/ETS)                                 │ │
│  │  • Response caching (Redis)                                          │ │
│  │                                                                      │ │
│  │  Deployment: Docker + Kubernetes / Azure App Service                 │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬───────────────────────────────────────────────┘
                             │ HTTPS + API Key / JWT
                             │
┌────────────────────────────▼───────────────────────────────────────────────┐
│                    FRONTEND (Web App)                                      │
│                                                                            │
│  • Current: index.html + Chart.js                                         │
│  • Production: React/Vue + Vite                                           │
│  • Deploy: Azure Static Web Apps / Vercel / Cloudflare Pages             │
│  • CDN: CloudFlare / Azure CDN for global delivery                        │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔐 Security Layers

### 1. **D365 OData Connection Security**
```python
# OAuth 2.0 with Service Principal (Azure AD)
import msal
import requests

def get_d365_access_token():
    """Authenticate to D365 using Azure AD."""
    
    authority = f"https://login.microsoftonline.com/{TENANT_ID}"
    app = msal.ConfidentialClientApplication(
        CLIENT_ID,
        authority=authority,
        client_credential=CLIENT_SECRET,  # Store in Azure Key Vault
    )
    
    scopes = [f"{D365_URL}/.default"]
    result = app.acquire_token_for_client(scopes=scopes)
    return result["access_token"]

def fetch_sales_orders(access_token):
    """Pull sales orders via OData."""
    
    url = f"{D365_URL}/data/SalesTableEntities"
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json",
    }
    
    # OData query with filters
    params = {
        "$filter": "SalesType eq Microsoft.Dynamics.DataEntities.SalesOrder and ReleaseStatus eq 'Open'",
        "$select": "SalesOrderNumber,CustomerAccount,CustomerName,InvoiceDate,Site,Warehouse",
        "$top": 5000,
    }
    
    response = requests.get(url, headers=headers, params=params, timeout=30)
    response.raise_for_status()
    return response.json()["value"]
```

**Security Checklist:**
- ✅ OAuth 2.0 with Service Principal (not username/password)
- ✅ Secrets in Azure Key Vault / AWS Secrets Manager
- ✅ TLS 1.2+ for all connections
- ✅ IP whitelisting on D365 firewall
- ✅ Read-only access to OData (never write)

---

### 2. **Database Security**

**PostgreSQL Production Setup:**
```sql
-- Create read-only user for API
CREATE USER sales_api_readonly WITH PASSWORD 'strong_password_here';
GRANT CONNECT ON DATABASE sales_intelligence TO sales_api_readonly;
GRANT USAGE ON SCHEMA public TO sales_api_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO sales_api_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO sales_api_readonly;

-- Create write user for ETL only
CREATE USER etl_writer WITH PASSWORD 'strong_password_here';
GRANT CONNECT ON DATABASE sales_intelligence TO etl_writer;
GRANT USAGE, CREATE ON SCHEMA public TO etl_writer;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO etl_writer;

-- Enable SSL connections only
ALTER SYSTEM SET ssl = 'on';
ALTER SYSTEM SET ssl_cert_file = '/path/to/server.crt';
ALTER SYSTEM SET ssl_key_file = '/path/to/server.key';

-- Audit logging
ALTER SYSTEM SET log_connections = 'on';
ALTER SYSTEM SET log_disconnections = 'on';
ALTER SYSTEM SET log_statement = 'all';
```

**Security Checklist:**
- ✅ Separate read/write users (principle of least privilege)
- ✅ SSL/TLS for all connections
- ✅ Private subnet (not publicly accessible)
- ✅ Firewall rules (only API server can connect)
- ✅ Automated backups (Azure Backup / AWS RDS snapshots)
- ✅ Encryption at rest (Azure Storage Service Encryption / AWS KMS)

---

### 3. **API Security**

**FastAPI with Authentication:**
```python
# backend/main.py
from fastapi import FastAPI, Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt

app = FastAPI()
security = HTTPBearer()

def verify_token(credentials: HTTPAuthorizationCredentials = Security(security)):
    """Verify JWT token for API access."""
    try:
        token = credentials.credentials
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return payload  # Contains user_id, roles, etc.
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.post("/api/query")
async def query_endpoint(request: QueryRequest, user=Depends(verify_token)):
    """Protected endpoint requiring authentication."""
    
    # Rate limiting (using Redis)
    if await is_rate_limited(user["user_id"]):
        raise HTTPException(status_code=429, detail="Too many requests")
    
    # Audit logging
    await log_query(user["user_id"], request.question)
    
    # Process query
    result = await answer_question(request.question, request.model)
    return result
```

**Security Checklist:**
- ✅ JWT/OAuth authentication
- ✅ Rate limiting (Redis)
- ✅ SQL injection prevention (parameterized queries)
- ✅ CORS configuration (whitelist domains)
- ✅ Request size limits
- ✅ Audit logging (who asked what, when)

---

## 📊 Database Comparison

| Feature | **PostgreSQL** | **SQL Server** | **DuckDB** (Current) | **Snowflake** |
|---------|---------------|---------------|---------------------|--------------|
| **Best For** | General production | Microsoft ecosystem | Development/testing | Analytics at scale |
| **Concurrent Users** | 100s-1000s | 100s-1000s | 1-10 | 1000s+ |
| **Data Size** | 100GB - 10TB+ | 100GB - 10TB+ | <10GB | Unlimited |
| **ACID Compliance** | ✅ Yes | ✅ Yes | ✅ Yes (single process) | ✅ Yes |
| **Cost (Monthly)** | $50-$200 | $100-$500 | $0 (embedded) | $200-$2000+ |
| **Security** | ✅ Enterprise | ✅ Enterprise | ⚠️ File-based | ✅ Enterprise |
| **Backup/Recovery** | ✅ Automated | ✅ Automated | ❌ Manual | ✅ Automated |
| **High Availability** | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |
| **OData Integration** | Good (via Python) | ✅ Native | Good (via Python) | Good (via Python) |
| **Forecasting Speed** | Fast (indexes) | Fast (indexes) | ✅ Fastest (columnar) | ✅ Fastest (distributed) |

---

## 🎯 **Recommended Production Stack**

### **For Your Use Case (Small-Medium Business):**

```
┌─────────────────────────────────────────────────────────────┐
│  RECOMMENDED: PostgreSQL + Azure                            │
│                                                             │
│  • Dynamics 365 (OData)                                     │
│       ↓                                                     │
│  • Azure Data Factory (ETL, every 15 min)                  │
│       ↓                                                     │
│  • Azure Database for PostgreSQL (Flexible Server)         │
│       ↓                                                     │
│  • Azure App Service (FastAPI backend)                     │
│       ↓                                                     │
│  • Azure Static Web Apps (Frontend)                        │
│                                                             │
│  Total Cost: ~$150-300/month                                │
│  Security: ✅✅✅ Enterprise-grade                            │
└─────────────────────────────────────────────────────────────┘
```

**Why PostgreSQL?**
1. ✅ **Open-source** (no Microsoft licensing costs)
2. ✅ **ACID compliant** (data integrity guaranteed)
3. ✅ **Proven at scale** (Instagram, Uber, Reddit use it)
4. ✅ **Excellent Python support** (psycopg2, SQLAlchemy)
5. ✅ **Managed Azure service** (auto-backups, patching, monitoring)
6. ✅ **JSON support** (JSONB for storing LLM prompts/responses)
7. ✅ **Cost-effective** (~$50/month for small DB)

---

## 🔄 ETL Pipeline (OData → PostgreSQL)

### **Option 1: Azure Data Factory (No-code)**
```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   D365       │ OData │   Azure      │  SQL  │  PostgreSQL  │
│  (Source)    ├──────►│ Data Factory ├──────►│  (Sink)      │
└──────────────┘       └──────────────┘       └──────────────┘
                       Schedule: Every 15 min
                       Transforms: Filter, Clean
```

### **Option 2: Python Script (Full control)**
```python
# scripts/etl_d365_to_postgres.py
import schedule
import time
from datetime import datetime
import psycopg2
import requests
import msal

def etl_job():
    """Pull from D365 OData and load to PostgreSQL."""
    
    print(f"[{datetime.now()}] Starting ETL job...")
    
    # 1. Authenticate with D365
    token = get_d365_access_token()
    
    # 2. Fetch sales orders
    sales_orders = fetch_sales_orders(token)
    inventory_trans = fetch_inventory_transactions(token)
    
    # 3. Apply business filters
    sales_filtered = [
        order for order in sales_orders
        if order["OrderType"] == "Sales order"
        and order["ReleaseStatus"] == "Open"
        and order["DoNotProcess"] == "No"
    ]
    
    inventory_filtered = [
        trans for trans in inventory_trans
        if trans["Reference"] == "Sales order"
        and trans["Issue"] == "Sold"
        and trans["Quantity"] < 0  # Negative = sold
    ]
    
    # 4. Load to PostgreSQL
    conn = psycopg2.connect(
        host=PG_HOST,
        database=PG_DATABASE,
        user=PG_USER,
        password=PG_PASSWORD,
        sslmode="require",
    )
    
    cursor = conn.cursor()
    
    # Upsert sales orders
    for order in sales_filtered:
        cursor.execute("""
            INSERT INTO sales_orders (
                sales_order_number, customer_account, customer_name,
                invoice_date, site, warehouse, channel, sales_taker
            ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (sales_order_number) DO UPDATE SET
                customer_name = EXCLUDED.customer_name,
                invoice_date = EXCLUDED.invoice_date
        """, (
            order["SalesOrderNumber"],
            order["CustomerAccount"],
            order["CustomerName"],
            order["InvoiceDate"],
            order["Site"],
            order["Warehouse"],
            order.get("Channel"),
            order.get("SalesTaker"),
        ))
    
    # Upsert inventory transactions
    for trans in inventory_filtered:
        cursor.execute("""
            INSERT INTO inventory_transactions (
                number, item_number, physical_date, reference,
                quantity, cost_amount, site, warehouse, unit
            ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (transaction_id) DO NOTHING
        """, (
            trans["Number"],
            trans["ItemNumber"],
            trans["PhysicalDate"],
            trans["Reference"],
            abs(trans["Quantity"]),  # Convert to positive
            abs(trans["CostAmount"]),  # Convert to positive
            trans["Site"],
            trans["Warehouse"],
            trans["Unit"],
        ))
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"[{datetime.now()}] ETL complete: {len(sales_filtered)} orders, {len(inventory_filtered)} transactions")

# Schedule: Every 15 minutes
schedule.every(15).minutes.do(etl_job)

# Run immediately on startup
etl_job()

# Keep running
while True:
    schedule.run_pending()
    time.sleep(60)
```

**Deploy as:**
- Azure Container Instance (serverless container)
- Azure Functions (scheduled trigger)
- Docker on VM (self-managed)

---

## 🚀 Migration Path (DuckDB → PostgreSQL)

### **Step 1: Export Current Data**
```python
# One-time migration script
import duckdb
import psycopg2

# Export from DuckDB
duckdb_conn = duckdb.connect("sales_inventory.duckdb")
sales_df = duckdb_conn.execute("SELECT * FROM sales_order").df()
inventory_df = duckdb_conn.execute("SELECT * FROM inventory_transaction").df()

# Import to PostgreSQL
pg_conn = psycopg2.connect("postgresql://user:pass@host/db")
sales_df.to_sql("sales_orders", pg_conn, if_exists="replace", index=False)
inventory_df.to_sql("inventory_transactions", pg_conn, if_exists="replace", index=False)
```

### **Step 2: Update Connection String**
```python
# src/core/text_to_sql.py
# OLD:
# DB_PATH = "sales_inventory.duckdb"
# conn = duckdb.connect(DB_PATH)

# NEW:
import psycopg2
DATABASE_URL = os.getenv("DATABASE_URL")  # From environment variable
conn = psycopg2.connect(DATABASE_URL, sslmode="require")
```

### **Step 3: Keep DuckDB for Local Dev**
```python
# Use environment variable to switch
if os.getenv("ENV") == "production":
    conn = psycopg2.connect(DATABASE_URL)
else:
    conn = duckdb.connect("sales_inventory.duckdb")  # Local testing
```

---

## 📋 Production Checklist

### **Before Launch:**
- [ ] Set up Azure/AWS account
- [ ] Provision PostgreSQL database (Azure Database for PostgreSQL)
- [ ] Configure firewall rules (whitelist API server IP)
- [ ] Create database users (read-only for API, write for ETL)
- [ ] Enable SSL/TLS on database
- [ ] Set up OData authentication (Azure AD Service Principal)
- [ ] Store secrets in Key Vault (not in code)
- [ ] Deploy ETL pipeline (Azure Data Factory or Python script)
- [ ] Test ETL end-to-end (D365 → PostgreSQL)
- [ ] Deploy FastAPI backend (Azure App Service / Container)
- [ ] Deploy frontend (Azure Static Web Apps)
- [ ] Set up monitoring (Azure Monitor / Application Insights)
- [ ] Configure automated backups (daily, 30-day retention)
- [ ] Set up alerts (database down, ETL failures, high latency)
- [ ] Load testing (can it handle 100 concurrent users?)
- [ ] Security audit (penetration testing)
- [ ] Document runbooks (how to restart, how to rollback)

---

## 💰 Cost Estimate (Azure)

| Service | Tier | Monthly Cost |
|---------|------|--------------|
| Azure Database for PostgreSQL | Flexible Server (2 vCores, 8GB RAM) | $70 |
| Azure App Service | Basic B1 (1 vCore, 1.75GB RAM) | $55 |
| Azure Static Web Apps | Standard | $10 |
| Azure Data Factory | 10K activities/month | $15 |
| Azure Key Vault | 10K operations/month | $3 |
| Azure Monitor | 5GB logs/month | $20 |
| **Total** | | **~$173/month** |

**Scale-up costs:**
- 10x traffic: ~$300/month (scale App Service + DB)
- 100x traffic: ~$1,000/month (multiple instances + load balancer)

---

## ✅ Why This Stack?

1. **Secure**: OAuth 2.0, SSL/TLS, private networks, audit logs
2. **Reliable**: Managed services, 99.9% SLA, automated backups
3. **Scalable**: Can grow from 10 to 10,000 users
4. **Cost-effective**: ~$150/month for small business
5. **Low maintenance**: Azure manages patching, backups, monitoring
6. **Native D365 integration**: OData authentication via Azure AD
7. **Production-proven**: PostgreSQL powers Reddit, Instagram, Uber

---

**Next Steps:** Want me to:
1. Generate the ETL script for your specific D365 OData endpoints?
2. Create Azure deployment ARM templates?
3. Write the PostgreSQL migration script?
