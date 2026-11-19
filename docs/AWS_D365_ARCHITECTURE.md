# Production Architecture: AWS + D365 OData Incremental ETL

## 🏗️ **Simple Architecture Overview**

```
┌──────────────────────────────────────────────────────────────────┐
│                    STEP 1: DATA SOURCE                           │
│                                                                  │
│                     DYNAMICS 365                                 │
│                  (Your ERP System)                               │
│                                                                  │
│     Sales Orders    Inventory     Customers                      │
│         └───────────┴──────────────┘                            │
│                     │                                            │
│                 OData API                                        │
│   https://your-d365.com/data/SalesTableEntities                 │
└─────────────────────┬────────────────────────────────────────────┘
                      │
                      │ 🔒 HTTPS + OAuth 2.0
                      │
┌─────────────────────▼────────────────────────────────────────────┐
│                    STEP 2: DATA SYNC (AWS Lambda)                │
│                                                                  │
│   ┌──────────────────────────────────────────────────┐         │
│   │  Lambda Function (Runs Every 15 Minutes)         │         │
│   │                                                  │         │
│   │  1. Check: "What's the last time I synced?"     │         │
│   │     (Read from DynamoDB)                        │         │
│   │                                                  │         │
│   │  2. Ask D365: "Give me records changed after    │         │
│   │     that time"                                  │         │
│   │     (OData: $filter=ModifiedDateTime > X)       │         │
│   │                                                  │         │
│   │  3. Save new/updated records to PostgreSQL      │         │
│   │                                                  │         │
│   │  4. Remember: "I synced up to time Y"           │         │
│   │     (Write to DynamoDB)                         │         │
│   └──────────────────────────────────────────────────┘         │
│                                                                  │
│   Triggered by: EventBridge (Clock)                             │
│   Secrets from: Secrets Manager (Passwords)                     │
└─────────────────────┬────────────────────────────────────────────┘
                      │
                      │ Writes to
                      │
┌─────────────────────▼────────────────────────────────────────────┐
│                    STEP 3: DATA STORAGE (PostgreSQL)             │
│                                                                  │
│   ┌──────────────────────────────────────────────────┐         │
│   │  AWS RDS PostgreSQL Database                     │         │
│   │                                                  │         │
│   │  Tables:                                         │         │
│   │  • sales_orders                                  │         │
│   │  • inventory_transactions                        │         │
│   │                                                  │         │
│   │  Views (for your app):                           │         │
│   │  • v_sold (filtered + joined data)               │         │
│   │  • v_orders (filtered order headers)             │         │
│   └──────────────────────────────────────────────────┘         │
│                                                                  │
│   Features: Auto-backups, Multi-AZ (no downtime)                │
└─────────────────────┬────────────────────────────────────────────┘
                      │
                      │ Queries from
                      │
┌─────────────────────▼────────────────────────────────────────────┐
│                    STEP 4: YOUR APPLICATION                      │
│                                                                  │
│   ┌──────────────────────────────────────────────────┐         │
│   │  FastAPI Backend (Your Current App)              │         │
│   │                                                  │         │
│   │  • Text-to-SQL (LLM generates queries)           │         │
│   │  • Forecasting (Prophet/ETS models)              │         │
│   │  • Reads from PostgreSQL                         │         │
│   └──────────────────────────────────────────────────┘         │
│                                                                  │
│   Deployed on: ECS Fargate (serverless containers)              │
└─────────────────────┬────────────────────────────────────────────┘
                      │
                      │ HTTPS
                      │
┌─────────────────────▼────────────────────────────────────────────┐
│                    STEP 5: USERS ACCESS                          │
│                                                                  │
│   ┌──────────────────────────────────────────────────┐         │
│   │  Frontend (HTML + Chart.js)                      │         │
│   │                                                  │         │
│   │  • Chat interface                                │         │
│   │  • Charts and tables                             │         │
│   └──────────────────────────────────────────────────┘         │
│                                                                  │
│   Hosted on: S3 + CloudFront (fast, global CDN)                 │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📊 **What Happens Every 15 Minutes (The ETL Process)**

```
⏰ EventBridge Timer Triggers Lambda
        ↓
┌───────────────────────────────────────────────────────────────┐
│ Lambda Wakes Up and Says:                                     │
│                                                               │
│ 1. "Let me check DynamoDB: when did I last sync?"            │
│    → Answer: "2026-10-06 10:30:00"                           │
│                                                               │
│ 2. "Let me ask D365: give me ALL records changed after that" │
│    → D365 returns: 234 new/modified records                  │
│                                                               │
│ 3. "Let me save these 234 records to PostgreSQL"            │
│    → PostgreSQL: "Saved! Some were updates, some were new"   │
│                                                               │
│ 4. "Let me remember: I synced up to 2026-10-06 10:45:00"    │
│    → DynamoDB: "Got it, saved!"                              │
│                                                               │
│ 5. "I'm done, going back to sleep until next trigger"        │
└───────────────────────────────────────────────────────────────┘
        ↓
⏰ 15 minutes later... Lambda wakes up again and repeats
```

---

## 🎯 **Why This Architecture?**

### **Problem with Current Setup:**
❌ You manually copy Excel files  
❌ Data gets stale quickly  
❌ No real-time updates  
❌ Can't scale to multiple users  

### **Solution with AWS + D365:**
✅ **Automatic sync** every 15 minutes (or real-time if needed)  
✅ **Incremental** - only pulls changed data (fast!)  
✅ **Scalable** - handles 100+ concurrent users  
✅ **Reliable** - auto-backups, no data loss  
✅ **Secure** - encrypted, OAuth, private network  

---

## 💡 **Key Components (Simple Explanation)**

| Component | What It Does | Why You Need It |
|-----------|--------------|-----------------|
| **D365 OData API** | Window into your ERP data | Lets us query D365 without direct DB access |
| **Lambda** | Python script that runs every 15 min | Pulls new data from D365 automatically |
| **DynamoDB** | Tiny database (2 KB) | Remembers "last sync time" so we don't re-fetch old data |
| **RDS PostgreSQL** | Main production database | Stores all your sales/inventory data for querying |
| **Secrets Manager** | Password vault | Stores D365/DB passwords securely (not in code) |
| **EventBridge** | Alarm clock | Wakes up Lambda every 15 minutes |
| **ECS Fargate** | Container for your FastAPI app | Runs your current backend in the cloud |
| **S3 + CloudFront** | Website hosting | Serves your frontend HTML/JS globally |

---

## 🔄 **Incremental Sync (How It Works)**

### **Traditional Full Sync (BAD):**
```
Every 15 minutes:
  1. Read ALL 100,000 sales orders from D365
  2. Delete all rows in PostgreSQL
  3. Insert all 100,000 rows again
  
Time: 10+ minutes
Network: 500 MB transferred
❌ Slow, wasteful, blocks database
```

### **Incremental Sync (GOOD):**
```
Every 15 minutes:
  1. Ask D365: "Give me ONLY records changed since 10:30"
  2. D365 returns: 50 records (not 100,000!)
  3. Update/insert those 50 rows in PostgreSQL
  
Time: 5 seconds
Network: 100 KB transferred
✅ Fast, efficient, no blocking
```

**How D365 Knows What Changed:**
- Every table in D365 has a `ModifiedDateTime` field
- When any record is created/updated, D365 updates this timestamp
- Lambda queries: `$filter=ModifiedDateTime gt 2026-10-06T10:30:00Z`
- Only records modified after 10:30 are returned

---

## 🏗️ **Data Flow (End-to-End)**

```
USER ASKS: "What are top 5 items in 2025?"
     ↓
FRONTEND sends question to BACKEND
     ↓
BACKEND (FastAPI):
  • LLM generates SQL: SELECT item_number, SUM(sold_qty)...
  • Executes SQL on PostgreSQL
  • PostgreSQL queries v_sold view
     ↓
POSTGRESQL returns: [Item1: 1000, Item2: 800, ...]
     ↓
BACKEND formats as chart data
     ↓
FRONTEND displays bar chart
     ↓
USER sees answer in 2 seconds

---

Meanwhile (in the background, every 15 min):
  LAMBDA syncs latest data from D365 → PostgreSQL
  (User never notices, data is always fresh!)
```

---

## 💰 **Total AWS Cost Estimate**

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| **RDS PostgreSQL** | db.t4g.medium, 100 GB, Multi-AZ | $120 |
| **Lambda** | 4 runs/hour × 24 × 30 × 1 sec @ 1024 MB | $5 |
| **EventBridge** | Scheduled rule | $0 (free) |
| **DynamoDB** | On-demand (low volume) | $0.50 |
| **Secrets Manager** | 4 secrets | $2 |
| **CloudWatch** | Logs + alarms | $5 |
| **S3 + Glacier** | ETL archive (optional) | $1 |
| **ECS Fargate / App Runner** | 0.25 vCPU, 0.5 GB RAM | $35 |
| **CloudFront + S3** | Frontend hosting | $5 |
| **Data Transfer** | Outbound data | $10 |
| **Total** | | **~$183/month** |

---

## 🔄 **Incremental ETL Strategy**

### **Option 1: Timestamp-Based (Recommended)**

D365 tracks `ModifiedDateTime` and `RecVersion` on every table. Use these for incremental sync:

```sql
-- OData query for incremental sync
GET /data/SalesTableEntities
  ?$filter=ModifiedDateTime gt 2026-10-06T10:30:00Z
  &$orderby=ModifiedDateTime asc
  &$top=5000
```

**Benefits:**
- ✅ Simple to implement
- ✅ Captures updates (not just inserts)
- ✅ Standard D365 feature

**Considerations:**
- ⚠️ Soft deletes not captured (need separate query)
- ⚠️ Large backfills can be slow

### **Option 2: Change Tracking (Advanced)**

If D365 has Change Tracking enabled:

```sql
-- Query for changes since last sync
GET /data/SalesTableEntities
  ?$filter=RecVersion gt 5637144576
  &$orderby=RecVersion asc
```

**Benefits:**
- ✅ Captures deletes
- ✅ More efficient than timestamp

**Considerations:**
- ⚠️ Requires D365 change tracking setup
- ⚠️ More complex logic

### **Option 3: Full Sync + CDC in PostgreSQL**

Initial full load, then use PostgreSQL triggers to track changes:

**Benefits:**
- ✅ No dependency on D365 change tracking
- ✅ Works with any source

**Considerations:**
- ⚠️ Slower initial load
- ⚠️ More storage for CDC tables

---

## 🐍 **Lambda Function Code (Incremental ETL)**

### **File: `lambda/d365_etl_handler.py`**

```python
"""
AWS Lambda function for D365 OData incremental ETL.

Triggered by: EventBridge (every 15 minutes)
Reads from: D365 OData API
Writes to: RDS PostgreSQL
State tracking: DynamoDB
"""

import json
import os
from datetime import datetime, timezone
import boto3
import requests
import psycopg2
from psycopg2.extras import execute_batch
from msal import ConfidentialClientApplication

# AWS clients
secrets_client = boto3.client('secretsmanager')
dynamodb = boto3.resource('dynamodb')
s3_client = boto3.client('s3')

# Environment variables
D365_URL = os.environ['D365_URL']  # e.g., https://your-d365.operations.dynamics.com
RDS_HOST = os.environ['RDS_HOST']
RDS_DATABASE = os.environ['RDS_DATABASE']
RDS_USER = os.environ['RDS_USER']
STATE_TABLE = os.environ['DYNAMODB_STATE_TABLE']  # e.g., etl_state
ARCHIVE_BUCKET = os.environ.get('S3_ARCHIVE_BUCKET')  # Optional


def get_secret(secret_name):
    """Retrieve secret from AWS Secrets Manager."""
    response = secrets_client.get_secret_value(SecretId=secret_name)
    return response['SecretString']


def get_d365_token():
    """Authenticate to D365 using OAuth 2.0 (Client Credentials)."""
    client_id = get_secret('d365/oauth/client-id')
    client_secret = get_secret('d365/oauth/client-secret')
    tenant_id = get_secret('d365/oauth/tenant-id')
    
    authority = f"https://login.microsoftonline.com/{tenant_id}"
    app = ConfidentialClientApplication(
        client_id,
        authority=authority,
        client_credential=client_secret,
    )
    
    scopes = [f"{D365_URL}/.default"]
    result = app.acquire_token_for_client(scopes=scopes)
    
    if "access_token" not in result:
        raise Exception(f"Failed to get token: {result.get('error_description')}")
    
    return result["access_token"]


def get_last_sync_state(entity_name):
    """Get last successful sync timestamp from DynamoDB."""
    table = dynamodb.Table(STATE_TABLE)
    
    try:
        response = table.get_item(Key={'entity': entity_name})
        if 'Item' in response:
            return response['Item']
        else:
            # First sync - start from 1 month ago
            return {
                'entity': entity_name,
                'last_sync_time': (datetime.now(timezone.utc) - timedelta(days=30)).isoformat(),
                'last_rec_version': 0,
                'records_synced': 0,
            }
    except Exception as e:
        print(f"Error reading DynamoDB state: {e}")
        raise


def update_sync_state(entity_name, last_sync_time, last_rec_version, records_synced):
    """Update sync state in DynamoDB."""
    table = dynamodb.Table(STATE_TABLE)
    
    table.put_item(Item={
        'entity': entity_name,
        'last_sync_time': last_sync_time,
        'last_rec_version': last_rec_version,
        'records_synced': records_synced,
        'last_updated': datetime.now(timezone.utc).isoformat(),
    })


def fetch_odata_incremental(token, entity_name, last_sync_time, batch_size=5000):
    """
    Fetch data from D365 OData API with incremental filter.
    
    Uses ModifiedDateTime for incremental sync.
    """
    url = f"{D365_URL}/data/{entity_name}"
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/json",
        "OData-MaxVersion": "4.0",
    }
    
    # OData query: get records modified after last sync
    params = {
        "$filter": f"ModifiedDateTime gt {last_sync_time}",
        "$orderby": "ModifiedDateTime asc",
        "$top": batch_size,
    }
    
    all_records = []
    next_url = url
    
    while next_url:
        print(f"Fetching: {next_url}")
        response = requests.get(next_url, headers=headers, params=params if next_url == url else None, timeout=60)
        response.raise_for_status()
        
        data = response.json()
        records = data.get('value', [])
        all_records.extend(records)
        
        print(f"Fetched {len(records)} records (total: {len(all_records)})")
        
        # Archive raw response to S3 (optional, for audit/replay)
        if ARCHIVE_BUCKET:
            archive_key = f"{datetime.now().strftime('%Y/%m/%d/%H')}/{entity_name}_{datetime.now().timestamp()}.json"
            s3_client.put_object(
                Bucket=ARCHIVE_BUCKET,
                Key=archive_key,
                Body=json.dumps(data),
                ContentType='application/json',
            )
        
        # Check for next link (pagination)
        next_url = data.get('@odata.nextLink')
        params = None  # Only use params on first request
    
    return all_records


def upsert_to_postgres(records, table_name, primary_key, conn):
    """
    Upsert records to PostgreSQL using ON CONFLICT.
    
    Tracks D365 metadata (ModifiedDateTime, RecVersion) for audit.
    """
    if not records:
        print(f"No records to upsert for {table_name}")
        return
    
    cursor = conn.cursor()
    
    # Example for sales_orders table
    if table_name == 'sales_orders':
        insert_sql = """
            INSERT INTO sales_orders (
                sales_order_number, customer_account, customer_name, order_type,
                invoice_account, channel, status, release_status, do_not_process,
                sales_taker, site, warehouse, invoice_date,
                d365_modified_datetime, d365_rec_version, etl_sync_timestamp
            ) VALUES (
                %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, NOW()
            )
            ON CONFLICT (sales_order_number) DO UPDATE SET
                customer_account = EXCLUDED.customer_account,
                customer_name = EXCLUDED.customer_name,
                order_type = EXCLUDED.order_type,
                invoice_account = EXCLUDED.invoice_account,
                channel = EXCLUDED.channel,
                status = EXCLUDED.status,
                release_status = EXCLUDED.release_status,
                do_not_process = EXCLUDED.do_not_process,
                sales_taker = EXCLUDED.sales_taker,
                site = EXCLUDED.site,
                warehouse = EXCLUDED.warehouse,
                invoice_date = EXCLUDED.invoice_date,
                d365_modified_datetime = EXCLUDED.d365_modified_datetime,
                d365_rec_version = EXCLUDED.d365_rec_version,
                etl_sync_timestamp = NOW()
        """
        
        data = [
            (
                r.get('SalesOrderNumber'),
                r.get('CustomerAccount'),
                r.get('CustomerName'),
                r.get('OrderType'),
                r.get('InvoiceAccount'),
                r.get('Channel'),
                r.get('Status'),
                r.get('ReleaseStatus'),
                r.get('DoNotProcess'),
                r.get('SalesTaker'),
                r.get('Site'),
                r.get('Warehouse'),
                r.get('InvoiceDate'),
                r.get('ModifiedDateTime'),
                r.get('RecVersion'),
            )
            for r in records
        ]
        
        execute_batch(cursor, insert_sql, data, page_size=1000)
    
    elif table_name == 'inventory_transactions':
        insert_sql = """
            INSERT INTO inventory_transactions (
                transaction_id, number, item_number, physical_date, reference,
                quantity, cost_amount, site, warehouse, unit,
                d365_modified_datetime, d365_rec_version, etl_sync_timestamp
            ) VALUES (
                %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, NOW()
            )
            ON CONFLICT (transaction_id) DO UPDATE SET
                number = EXCLUDED.number,
                item_number = EXCLUDED.item_number,
                physical_date = EXCLUDED.physical_date,
                reference = EXCLUDED.reference,
                quantity = EXCLUDED.quantity,
                cost_amount = EXCLUDED.cost_amount,
                site = EXCLUDED.site,
                warehouse = EXCLUDED.warehouse,
                unit = EXCLUDED.unit,
                d365_modified_datetime = EXCLUDED.d365_modified_datetime,
                d365_rec_version = EXCLUDED.d365_rec_version,
                etl_sync_timestamp = NOW()
        """
        
        data = [
            (
                f"{r.get('Number')}_{r.get('ItemNumber')}_{r.get('PhysicalDate')}",  # Composite key
                r.get('Number'),
                r.get('ItemNumber'),
                r.get('PhysicalDate'),
                r.get('Reference'),
                abs(float(r.get('Quantity', 0))),  # Convert to positive
                abs(float(r.get('CostAmount', 0))),  # Convert to positive
                r.get('Site'),
                r.get('Warehouse'),
                r.get('Unit'),
                r.get('ModifiedDateTime'),
                r.get('RecVersion'),
            )
            for r in records
        ]
        
        execute_batch(cursor, insert_sql, data, page_size=1000)
    
    conn.commit()
    cursor.close()
    print(f"✅ Upserted {len(records)} records to {table_name}")


def sync_entity(entity_config, token, pg_conn):
    """
    Sync a single entity (table) from D365 to PostgreSQL.
    
    Args:
        entity_config: Dict with 'odata_name', 'table_name', 'primary_key'
        token: D365 OAuth token
        pg_conn: PostgreSQL connection
    """
    odata_name = entity_config['odata_name']
    table_name = entity_config['table_name']
    primary_key = entity_config['primary_key']
    
    print(f"\n{'='*60}")
    print(f"Syncing entity: {odata_name} → {table_name}")
    print(f"{'='*60}")
    
    # Get last sync state
    state = get_last_sync_state(odata_name)
    last_sync_time = state['last_sync_time']
    print(f"Last sync: {last_sync_time}")
    
    # Fetch incremental data
    records = fetch_odata_incremental(token, odata_name, last_sync_time)
    print(f"Fetched {len(records)} new/modified records")
    
    if not records:
        print("No new records to sync")
        return
    
    # Upsert to PostgreSQL
    upsert_to_postgres(records, table_name, primary_key, pg_conn)
    
    # Update sync state
    latest_modified = max(r['ModifiedDateTime'] for r in records)
    latest_rec_version = max(r.get('RecVersion', 0) for r in records)
    
    update_sync_state(
        entity_name=odata_name,
        last_sync_time=latest_modified,
        last_rec_version=latest_rec_version,
        records_synced=len(records),
    )
    
    print(f"✅ Sync complete. New last_sync_time: {latest_modified}")


def lambda_handler(event, context):
    """
    Main Lambda handler - orchestrates ETL for all entities.
    """
    print("🚀 Starting D365 ETL job...")
    print(f"Event: {json.dumps(event)}")
    
    start_time = datetime.now()
    
    try:
        # 1. Get D365 OAuth token
        print("\n1️⃣ Authenticating to D365...")
        token = get_d365_token()
        print("✅ Token acquired")
        
        # 2. Connect to PostgreSQL
        print("\n2️⃣ Connecting to PostgreSQL...")
        pg_password = get_secret('rds/postgres/password')
        pg_conn = psycopg2.connect(
            host=RDS_HOST,
            database=RDS_DATABASE,
            user=RDS_USER,
            password=pg_password,
            sslmode='require',
            connect_timeout=10,
        )
        print("✅ PostgreSQL connected")
        
        # 3. Sync entities
        entities = [
            {
                'odata_name': 'SalesTableEntities',
                'table_name': 'sales_orders',
                'primary_key': 'sales_order_number',
            },
            {
                'odata_name': 'InventTransEntities',
                'table_name': 'inventory_transactions',
                'primary_key': 'transaction_id',
            },
        ]
        
        for entity_config in entities:
            sync_entity(entity_config, token, pg_conn)
        
        # 4. Close connection
        pg_conn.close()
        
        # 5. Success
        duration = (datetime.now() - start_time).total_seconds()
        print(f"\n✅ ETL job complete in {duration:.1f}s")
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'status': 'success',
                'duration_seconds': duration,
                'entities_synced': len(entities),
            })
        }
        
    except Exception as e:
        print(f"\n❌ ETL job failed: {e}")
        import traceback
        traceback.print_exc()
        
        return {
            'statusCode': 500,
            'body': json.dumps({
                'status': 'error',
                'error': str(e),
            })
        }
```

---

## 🗄️ **PostgreSQL Schema (with ETL metadata)**

```sql
-- Create tables with ETL tracking columns

CREATE TABLE sales_orders (
    -- Business columns (from D365)
    sales_order_number VARCHAR(20) PRIMARY KEY,
    customer_account VARCHAR(20),
    customer_name VARCHAR(100),
    order_type VARCHAR(20),
    invoice_account VARCHAR(20),
    channel VARCHAR(20),
    status VARCHAR(20),
    release_status VARCHAR(20),
    do_not_process VARCHAR(5),
    sales_taker VARCHAR(50),
    site VARCHAR(10),
    warehouse VARCHAR(10),
    invoice_date DATE,
    
    -- ETL metadata columns (for tracking)
    d365_modified_datetime TIMESTAMP,
    d365_rec_version BIGINT,
    etl_sync_timestamp TIMESTAMP DEFAULT NOW(),
    etl_is_deleted BOOLEAN DEFAULT FALSE,
    
    -- Indexes for performance
    INDEX idx_sales_modified (d365_modified_datetime),
    INDEX idx_sales_invoice_date (invoice_date),
    INDEX idx_sales_customer (customer_account)
);

CREATE TABLE inventory_transactions (
    -- Business columns
    transaction_id VARCHAR(100) PRIMARY KEY,  -- Composite: number_item_date
    number VARCHAR(20),  -- Links to sales_order_number
    item_number VARCHAR(20),
    physical_date DATE,
    reference VARCHAR(20),
    quantity DOUBLE PRECISION,
    cost_amount DOUBLE PRECISION,
    site VARCHAR(10),
    warehouse VARCHAR(10),
    unit VARCHAR(10),
    
    -- ETL metadata
    d365_modified_datetime TIMESTAMP,
    d365_rec_version BIGINT,
    etl_sync_timestamp TIMESTAMP DEFAULT NOW(),
    etl_is_deleted BOOLEAN DEFAULT FALSE,
    
    -- Indexes
    INDEX idx_inventory_modified (d365_modified_datetime),
    INDEX idx_inventory_physical_date (physical_date),
    INDEX idx_inventory_item (item_number),
    INDEX idx_inventory_number (number)
);

-- Create views (same as DuckDB views, but for PostgreSQL)
CREATE OR REPLACE VIEW v_sold AS
SELECT 
    it.item_number,
    it.item_number AS product_number,
    it.physical_date AS sale_date,
    it.number AS sales_order_number,
    it.quantity AS sold_qty,
    it.unit,
    it.cost_amount,
    it.site,
    it.warehouse,
    so.customer_account,
    so.customer_name,
    so.invoice_account,
    so.channel,
    so.sales_taker
FROM inventory_transactions it
INNER JOIN sales_orders so ON it.number = so.sales_order_number
WHERE it.reference = 'Sales order'
  AND it.quantity > 0
  AND so.order_type = 'Sales order'
  AND so.release_status = 'Open'
  AND so.do_not_process = 'No'
  AND it.etl_is_deleted = FALSE
  AND so.etl_is_deleted = FALSE;

CREATE OR REPLACE VIEW v_orders AS
SELECT 
    sales_order_number,
    customer_account,
    customer_name,
    order_type,
    invoice_account,
    channel,
    status,
    release_status,
    do_not_process,
    sales_taker,
    site,
    warehouse,
    invoice_date
FROM sales_orders
WHERE order_type = 'Sales order'
  AND release_status = 'Open'
  AND do_not_process = 'No'
  AND etl_is_deleted = FALSE;
```

---

## 📦 **DynamoDB Schema**

```json
{
  "TableName": "etl_state",
  "KeySchema": [
    { "AttributeName": "entity", "KeyType": "HASH" }
  ],
  "AttributeDefinitions": [
    { "AttributeName": "entity", "AttributeType": "S" }
  ],
  "BillingMode": "PAY_PER_REQUEST",
  "ProvisionedThroughput": null
}
```

**Sample items:**
```json
{
  "entity": "SalesTableEntities",
  "last_sync_time": "2026-10-06T10:30:00Z",
  "last_rec_version": 5637144576,
  "records_synced": 1234,
  "last_updated": "2026-10-06T10:45:00Z"
}
```

---

## 🚀 **Deployment Steps**

### **1. Create RDS PostgreSQL**

```bash
# Using AWS CLI
aws rds create-db-instance \
  --db-instance-identifier sales-intelligence-prod \
  --db-instance-class db.t4g.medium \
  --engine postgres \
  --engine-version 16.2 \
  --master-username dbadmin \
  --master-user-password 'YOUR_STRONG_PASSWORD' \
  --allocated-storage 100 \
  --storage-type gp3 \
  --storage-encrypted \
  --multi-az \
  --backup-retention-period 7 \
  --vpc-security-group-ids sg-xxxxx \
  --db-subnet-group-name my-db-subnet-group \
  --publicly-accessible false
```

### **2. Create DynamoDB Table**

```bash
aws dynamodb create-table \
  --table-name etl_state \
  --attribute-definitions AttributeName=entity,AttributeType=S \
  --key-schema AttributeName=entity,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### **3. Store Secrets in Secrets Manager**

```bash
aws secretsmanager create-secret \
  --name d365/oauth/client-id \
  --secret-string "YOUR_CLIENT_ID"

aws secretsmanager create-secret \
  --name d365/oauth/client-secret \
  --secret-string "YOUR_CLIENT_SECRET"

aws secretsmanager create-secret \
  --name d365/oauth/tenant-id \
  --secret-string "YOUR_TENANT_ID"

aws secretsmanager create-secret \
  --name rds/postgres/password \
  --secret-string "YOUR_DB_PASSWORD"
```

### **4. Deploy Lambda Function**

```bash
# Package Lambda (with dependencies)
cd lambda
pip install -r requirements.txt -t .
zip -r function.zip .

# Create Lambda function
aws lambda create-function \
  --function-name d365-etl-handler \
  --runtime python3.11 \
  --handler d365_etl_handler.lambda_handler \
  --role arn:aws:iam::ACCOUNT_ID:role/lambda-execution-role \
  --zip-file fileb://function.zip \
  --timeout 900 \
  --memory-size 1024 \
  --environment Variables="{
    D365_URL=https://your-d365.operations.dynamics.com,
    RDS_HOST=sales-intelligence-prod.xxxxx.rds.amazonaws.com,
    RDS_DATABASE=sales_intelligence,
    RDS_USER=dbadmin,
    DYNAMODB_STATE_TABLE=etl_state,
    S3_ARCHIVE_BUCKET=d365-etl-archive
  }"
```

### **5. Create EventBridge Schedule**

```bash
aws events put-rule \
  --name d365-etl-schedule \
  --schedule-expression "rate(15 minutes)"

aws events put-targets \
  --rule d365-etl-schedule \
  --targets "Id"="1","Arn"="arn:aws:lambda:REGION:ACCOUNT_ID:function:d365-etl-handler"

# Grant permission for EventBridge to invoke Lambda
aws lambda add-permission \
  --function-name d365-etl-handler \
  --statement-id AllowEventBridgeInvoke \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:REGION:ACCOUNT_ID:rule/d365-etl-schedule
```

---

## 🎯 **Key Benefits of This Architecture**

1. ✅ **Incremental Sync** - Only pulls changed records (not full reload every time)
2. ✅ **Scalable** - Lambda auto-scales, RDS can scale up as needed
3. ✅ **Reliable** - Multi-AZ RDS, automated backups, CloudWatch alarms
4. ✅ **Secure** - Secrets Manager, VPC, encryption at rest + in transit
5. ✅ **Auditable** - S3 archive, ETL metadata tracking, CloudWatch logs
6. ✅ **Cost-effective** - ~$183/month for production-grade setup
7. ✅ **Zero downtime** - Lambda runs every 15 min without affecting app
8. ✅ **Handles deletes** - Can track soft deletes via `etl_is_deleted` flag

---

Want me to:
1. Create the complete Lambda deployment package?
2. Generate Terraform/CloudFormation templates for infrastructure?
3. Write the initial data migration script (full load)?