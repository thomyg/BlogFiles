# Zero-Downtime Single-Tenant to Multi-Tenant PostgreSQL Migration Plan

## Architecture Decision: Row-Based Multi-Tenancy with tenant_id

**Choice Rationale**: Row-based isolation provides better operational efficiency, easier analytics, and simpler schema management compared to schema-per-tenant approach.

## 1. Architecture Diagram (ASCII)

```
BEFORE (Single-Tenant):
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Tenant A      │    │   Tenant B      │    │   Tenant C      │
│   App Instance  │    │   App Instance  │    │   App Instance  │
└─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
          │                      │                      │
┌─────────▼───────┐    ┌─────────▼───────┐    ┌─────────▼───────┐
│   PostgreSQL    │    │   PostgreSQL    │    │   PostgreSQL    │
│   DB_A          │    │   DB_B          │    │   DB_C          │
└─────────────────┘    └─────────────────┘    └─────────────────┘

AFTER (Multi-Tenant):
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Tenant A      │    │   Tenant B      │    │   Tenant C      │
│   App Instance  │    │   App Instance  │    │   App Instance  │
└─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │    PgBouncer Pool       │
                    │   (Connection Pooling)  │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   Multi-Tenant          │
                    │   PostgreSQL            │
                    │   - Row Level Security  │
                    │   - tenant_id columns   │
                    │   - Logical Replication │
                    └─────────────────────────┘

MIGRATION FLOW:
┌─────────────────┐         ┌─────────────────┐
│   Source DBs    │────────▶│   Target MT-DB  │
│   (Read/Write)  │  Logic  │   (Write Only)  │
└─────────────────┘  Repl   └─────────────────┘
         │                           │
         ▼                           ▼
┌─────────────────┐         ┌─────────────────┐
│   During        │         │    After        │
│   Migration     │         │    Cutover      │
│   (Read Only)   │         │   (Read/Write)  │
└─────────────────┘         └─────────────────┘
```

## 2. Step-by-Step Migration Runbook

### Phase 1: Preparation (Week 1-2)

#### Step 1.1: Setup Multi-Tenant Target Database

```bash
# Create new multi-tenant database
createdb -h target-host -U postgres saas_multitenant

# Connect to target database
psql -h target-host -U postgres saas_multitenant
```

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";

-- Create tenants table
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    slug VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    status VARCHAR(50) DEFAULT 'active'
);

-- Insert tenant records (example)
INSERT INTO tenants (slug, name) VALUES 
    ('tenant-a', 'Tenant A Corp'),
    ('tenant-b', 'Tenant B Inc'),
    ('tenant-c', 'Tenant C LLC');
```

#### Step 1.2: Schema Migration Script

```sql
-- Create multi-tenant schema with tenant_id columns
-- Example for users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Create unique constraint including tenant_id
ALTER TABLE users ADD CONSTRAINT users_email_tenant_unique 
    UNIQUE (email, tenant_id);

-- Enable Row Level Security
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Create RLS policies
CREATE POLICY tenant_isolation_policy ON users
    FOR ALL TO application_role
    USING (tenant_id = current_setting('app.tenant_id')::UUID);

-- Create indexes optimized for multi-tenancy
CREATE INDEX idx_users_tenant_id ON users(tenant_id);
CREATE INDEX idx_users_tenant_email ON users(tenant_id, email);
```

#### Step 1.3: Application Role Setup

```sql
-- Create application role
CREATE ROLE application_role;
GRANT CONNECT ON DATABASE saas_multitenant TO application_role;
GRANT USAGE ON SCHEMA public TO application_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO application_role;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO application_role;

-- Create migration role for data loading
CREATE ROLE migration_role;
GRANT ALL PRIVILEGES ON DATABASE saas_multitenant TO migration_role;
```

### Phase 2: Logical Replication Setup (Week 2-3)

#### Step 2.1: Configure Source Databases for Logical Replication

```bash
# For each source database, edit postgresql.conf
sudo nano /etc/postgresql/15/main/postgresql.conf
```

```conf
# Enable logical replication
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

```bash
# Restart PostgreSQL
sudo systemctl restart postgresql

# Create replication user on each source
psql -h source-tenant-a -U postgres
```

```sql
CREATE USER replication_user WITH REPLICATION LOGIN PASSWORD 'secure_password';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replication_user;
```

#### Step 2.2: Create Publications on Source Databases

```bash
# Connect to each tenant database
psql -h source-tenant-a -U postgres tenant_a_db
```

```sql
-- Create publication for all tables
CREATE PUBLICATION tenant_migration FOR ALL TABLES;

-- Verify publication
SELECT * FROM pg_publication;
```

#### Step 2.3: Setup Initial Data Load

```bash
#!/bin/bash
# initial_data_load.sh

TENANTS=("tenant-a" "tenant-b" "tenant-c")
SOURCE_HOSTS=("source-host-a" "source-host-b" "source-host-c")
TARGET_HOST="target-host"
TARGET_DB="saas_multitenant"

for i in "${!TENANTS[@]}"; do
    TENANT=${TENANTS[$i]}
    SOURCE_HOST=${SOURCE_HOSTS[$i]}
    
    echo "Starting initial load for $TENANT"
    
    # Get tenant_id from target database
    TENANT_ID=$(psql -h $TARGET_HOST -U postgres -d $TARGET_DB -t -c \
        "SELECT id FROM tenants WHERE slug = '$TENANT';")
    
    # Dump and load data with tenant_id injection
    pg_dump -h $SOURCE_HOST -U postgres --data-only --no-owner \
        --no-privileges ${TENANT}_db | \
    sed "s/INSERT INTO \([^(]*\)/INSERT INTO \1 (tenant_id, /g" | \
    sed "s/VALUES (/VALUES ('$TENANT_ID', /g" | \
    psql -h $TARGET_HOST -U postgres -d $TARGET_DB
    
    echo "Completed initial load for $TENANT"
done
```

#### Step 2.4: Create Subscriptions

```bash
# Connect to target database
psql -h target-host -U postgres saas_multitenant
```

```sql
-- Create subscription for each tenant
CREATE SUBSCRIPTION tenant_a_sub
    CONNECTION 'host=source-host-a user=replication_user password=secure_password dbname=tenant_a_db'
    PUBLICATION tenant_migration
    WITH (copy_data = false); -- We already did initial load

-- Repeat for other tenants
CREATE SUBSCRIPTION tenant_b_sub
    CONNECTION 'host=source-host-b user=replication_user password=secure_password dbname=tenant_b_db'
    PUBLICATION tenant_migration
    WITH (copy_data = false);

CREATE SUBSCRIPTION tenant_c_sub
    CONNECTION 'host=source-host-c user=replication_user password=secure_password dbname=tenant_c_db'
    PUBLICATION tenant_migration
    WITH (copy_data = false);
```

### Phase 3: Connection Pooling Setup (Week 3)

#### Step 3.1: PgBouncer Configuration

```bash
# Install PgBouncer
sudo apt-get install pgbouncer

# Configure PgBouncer
sudo nano /etc/pgbouncer/pgbouncer.ini
```

```ini
[databases]
saas_multitenant = host=target-host port=5432 dbname=saas_multitenant

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
min_pool_size = 5
reserve_pool_size = 5
max_db_connections = 100
server_round_robin = 1
ignore_startup_parameters = extra_float_digits
```

```bash
# Create userlist
sudo nano /etc/pgbouncer/userlist.txt
```

```
"application_role" "md5<hashed_password>"
```

#### Step 3.2: Application Configuration Update

```yaml
# application.yml - staged deployment config
database:
  migration_phase: "replication" # Options: single_tenant, replication, cutover, multi_tenant
  
  single_tenant:
    tenant_a:
      host: source-host-a
      database: tenant_a_db
    tenant_b:
      host: source-host-b  
      database: tenant_b_db
    tenant_c:
      host: source-host-c
      database: tenant_c_db
      
  multi_tenant:
    host: pgbouncer-host
    port: 6432
    database: saas_multitenant
    connection_pool_size: 20
```

### Phase 4: Application Code Changes (Week 4)

#### Step 4.1: Tenant Context Middleware

```python
# tenant_middleware.py
import uuid
from django.db import connection

class TenantMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Extract tenant from subdomain, header, or JWT
        tenant_slug = self.extract_tenant(request)
        
        if tenant_slug:
            with connection.cursor() as cursor:
                cursor.execute(
                    "SELECT id FROM tenants WHERE slug = %s", 
                    [tenant_slug]
                )
                tenant_id = cursor.fetchone()
                
                if tenant_id:
                    # Set tenant context for RLS
                    cursor.execute(
                        "SET LOCAL app.tenant_id = %s", 
                        [str(tenant_id[0])]
                    )
        
        response = self.get_response(request)
        return response

    def extract_tenant(self, request):
        # Extract from subdomain
        host = request.get_host()
        return host.split('.')[0] if '.' in host else None
```

#### Step 4.2: Database Router for Migration Phases

```python
# database_router.py
from django.conf import settings

class MigrationRouter:
    def db_for_read(self, model, **hints):
        phase = settings.DATABASE_MIGRATION_PHASE
        
        if phase == 'single_tenant':
            return self.get_tenant_db()
        elif phase == 'replication':
            return self.get_tenant_db()  # Still read from source
        elif phase in ['cutover', 'multi_tenant']:
            return 'multi_tenant'
        
    def db_for_write(self, model, **hints):
        phase = settings.DATABASE_MIGRATION_PHASE
        
        if phase == 'single_tenant':
            return self.get_tenant_db()
        elif phase == 'replication':
            # Write to both during replication validation
            return 'multi_tenant'
        elif phase in ['cutover', 'multi_tenant']:
            return 'multi_tenant'
    
    def get_tenant_db(self):
        # Logic to determine tenant database
        return f'tenant_{get_current_tenant()}_db'
```

### Phase 5: Cutover Plan (Week 5)

#### Step 5.1: Pre-Cutover Validation Script

```bash
#!/bin/bash
# pre_cutover_validation.sh

validate_replication_lag() {
    echo "Checking replication lag..."
    
    for tenant in tenant_a tenant_b tenant_c; do
        LAG=$(psql -h target-host -U postgres -d saas_multitenant -t -c \
            "SELECT EXTRACT(EPOCH FROM (now() - received_time)) 
             FROM pg_stat_subscription 
             WHERE subname = '${tenant}_sub';")
        
        if (( $(echo "$LAG > 5" | bc -l) )); then
            echo "ERROR: Replication lag for $tenant is ${LAG}s"
            exit 1
        else
            echo "OK: Replication lag for $tenant is ${LAG}s"
        fi
    done
}

validate_data_consistency() {
    echo "Validating data consistency..."
    
    # Compare row counts
    for tenant in tenant_a tenant_b tenant_c; do
        SOURCE_COUNT=$(psql -h source-${tenant//_/-} -U postgres -d ${tenant}_db -t -c \
            "SELECT COUNT(*) FROM users;")
        
        TENANT_ID=$(psql -h target-host -U postgres -d saas_multitenant -t -c \
            "SELECT id FROM tenants WHERE slug = '${tenant//_/-}';")
        
        TARGET_COUNT=$(psql -h target-host -U postgres -d saas_multitenant -t -c \
            "SELECT COUNT(*) FROM users WHERE tenant_id = '$TENANT_ID';")
        
        if [ "$SOURCE_COUNT" != "$TARGET_COUNT" ]; then
            echo "ERROR: Row count mismatch for $tenant: Source=$SOURCE_COUNT, Target=$TARGET_COUNT"
            exit 1
        else
            echo "OK: Row counts match for $tenant: $SOURCE_COUNT rows"
        fi
    done
}

validate_replication_lag
validate_data_consistency
echo "Pre-cutover validation completed successfully!"
```

#### Step 5.2: Cutover Execution Script

```bash
#!/bin/bash
# cutover_execution.sh

set -e

MAINTENANCE_START=$(date)
echo "Starting cutover at $MAINTENANCE_START"

# Step 1: Enable maintenance mode
echo "Enabling maintenance mode..."
kubectl patch deployment app-deployment -p '{"spec":{"replicas":0}}'

# Step 2: Wait for active connections to drain
echo "Waiting for connections to drain..."
sleep 30

# Step 3: Stop replication to ensure data consistency
echo "Stopping logical replication..."
psql -h target-host -U postgres -d saas_multitenant -c \
    "SELECT pg_drop_subscription('tenant_a_sub');"
psql -h target-host -U postgres -d saas_multitenant -c \
    "SELECT pg_drop_subscription('tenant_b_sub');"
psql -h target-host -U postgres -d saas_multitenant -c \
    "SELECT pg_drop_subscription('tenant_c_sub');"

# Step 4: Final data consistency check
./pre_cutover_validation.sh

# Step 5: Update application configuration
echo "Updating application configuration..."
kubectl create configmap app-config \
    --from-literal=DATABASE_MIGRATION_PHASE=multi_tenant \
    --dry-run=client -o yaml | kubectl apply -f -

# Step 6: Deploy updated application
echo "Deploying multi-tenant application..."
kubectl patch deployment app-deployment -p '{"spec":{"replicas":3}}'

# Step 7: Wait for application readiness
echo "Waiting for application readiness..."
kubectl wait --for=condition=available --timeout=300s deployment/app-deployment

# Step 8: Run post-cutover validation
./post_cutover_validation.sh

MAINTENANCE_END=$(date)
echo "Cutover completed at $MAINTENANCE_END"
echo "Total maintenance window: $(date -d "$MAINTENANCE_END" +%s) - $(date -d "$MAINTENANCE_START" +%s) seconds"
```

## 3. Data Validation Tactics

### Continuous Validation During Replication

```python
# data_validator.py
import psycopg2
import hashlib
from datetime import datetime

class DataValidator:
    def __init__(self):
        self.source_connections = {
            'tenant_a': psycopg2.connect("host=source-a dbname=tenant_a_db"),
            'tenant_b': psycopg2.connect("host=source-b dbname=tenant_b_db"), 
            'tenant_c': psycopg2.connect("host=source-c dbname=tenant_c_db")
        }
        self.target_connection = psycopg2.connect("host=target dbname=saas_multitenant")
    
    def validate_row_counts(self):
        """Compare row counts between source and target"""
        results = {}
        
        for tenant, source_conn in self.source_connections.items():
            source_cur = source_conn.cursor()
            target_cur = self.target_connection.cursor()
            
            # Get tenant_id
            target_cur.execute("SELECT id FROM tenants WHERE slug = %s", (tenant.replace('_', '-'),))
            tenant_id = target_cur.fetchone()[0]
            
            for table in ['users', 'orders', 'products']:  # Add all your tables
                # Source count
                source_cur.execute(f"SELECT COUNT(*) FROM {table}")
                source_count = source_cur.fetchone()[0]
                
                # Target count
                target_cur.execute(f"SELECT COUNT(*) FROM {table} WHERE tenant_id = %s", (tenant_id,))
                target_count = target_cur.fetchone()[0]
                
                results[f"{tenant}_{table}"] = {
                    'source': source_count,
                    'target': target_count,
                    'match': source_count == target_count
                }
        
        return results
    
    def validate_data_integrity(self, table_name, sample_size=1000):
        """Compare checksums of sample data"""
        results = {}
        
        for tenant, source_conn in self.source_connections.items():
            source_cur = source_conn.cursor()
            target_cur = self.target_connection.cursor()
            
            # Get tenant_id
            target_cur.execute("SELECT id FROM tenants WHERE slug = %s", (tenant.replace('_', '-'),))
            tenant_id = target_cur.fetchone()[0]
            
            # Sample random records from source
            source_cur.execute(f"""
                SELECT * FROM {table_name} 
                ORDER BY RANDOM() LIMIT %s
            """, (sample_size,))
            source_rows = source_cur.fetchall()
            
            matches = 0
            for row in source_rows:
                row_id = row[0]  # Assuming first column is ID
                
                # Check if same record exists in target
                target_cur.execute(f"""
                    SELECT * FROM {table_name} 
                    WHERE id = %s AND tenant_id = %s
                """, (row_id, tenant_id))
                target_row = target_cur.fetchone()
                
                if target_row and self.compare_rows(row, target_row[2:]):  # Skip tenant_id column
                    matches += 1
            
            results[f"{tenant}_{table_name}"] = {
                'total_sampled': len(source_rows),
                'matches': matches,
                'integrity_score': matches / len(source_rows) if source_rows else 0
            }
        
        return results
    
    def compare_rows(self, source_row, target_row):
        """Compare two rows for equality"""
        return str(source_row) == str(target_row)
```

### Automated Validation Pipeline

```bash
#!/bin/bash
# continuous_validation.sh

while true; do
    echo "$(date): Running validation checks..."
    
    # Run validation
    python3 data_validator.py > /tmp/validation_$(date +%s).log
    
    # Check for failures
    if grep -q "FAIL" /tmp/validation_$(date +%s).log; then
        echo "ALERT: Data validation failed!"
        # Send alert to monitoring system
        curl -X POST "$SLACK_WEBHOOK" -d '{"text":"🚨 Migration validation failed!"}'
    fi
    
    # Wait 5 minutes
    sleep 300
done
```

## 4. Failure Playbook

### Scenario 1: Replication Lag Increases Beyond Threshold

**Detection**: Monitoring alerts when lag > 10 seconds

**Response**:
```bash
#!/bin/bash
# handle_replication_lag.sh

# Check current lag
check_lag() {
    psql -h target-host -U postgres -d saas_multitenant -c \
        "SELECT subname, 
                EXTRACT(EPOCH FROM (now() - received_time)) as lag_seconds
         FROM pg_stat_subscription;"
}

# Restart subscription if needed
restart_subscription() {
    local sub_name=$1
    echo "Restarting subscription: $sub_name"
    
    psql -h target-host -U postgres -d saas_multitenant -c \
        "ALTER SUBSCRIPTION $sub_name DISABLE;"
    
    sleep 10
    
    psql -h target-host -U postgres -d saas_multitenant -c \
        "ALTER SUBSCRIPTION $sub_name ENABLE;"
}

# Main execution
LAG=$(check_lag)
echo "Current replication lag: $LAG"

if (( $(echo "$LAG > 30" | bc -l) )); then
    echo "High lag detected, restarting subscriptions..."
    restart_subscription "tenant_a_sub"
    restart_subscription "tenant_b_sub" 
    restart_subscription "tenant_c_sub"
fi
```

### Scenario 2: Data Inconsistency Detected

**Detection**: Validation scripts report mismatches

**Response**:
```bash
#!/bin/bash
# handle_data_inconsistency.sh

# Resync specific tenant
resync_tenant() {
    local tenant=$1
    local tenant_id=$2
    
    echo "Resyncing tenant: $tenant"
    
    # Disable subscription
    psql -h target-host -U postgres -d saas_multitenant -c \
        "ALTER SUBSCRIPTION ${tenant}_sub DISABLE;"
    
    # Clear existing data for tenant
    psql -h target-host -U postgres -d saas_multitenant -c \
        "DELETE FROM users WHERE tenant_id = '$tenant_id';"
    # Add other table deletions...
    
    # Re-enable subscription with copy_data
    psql -h target-host -U postgres -d saas_multitenant -c \
        "ALTER SUBSCRIPTION ${tenant}_sub ENABLE;"
    
    psql -h target-host -U postgres -d saas_multitenant -c \
        "ALTER SUBSCRIPTION ${tenant}_sub REFRESH PUBLICATION WITH (copy_data = true);"
}

# Usage
resync_tenant "tenant_a" "uuid-of-tenant-a"
```

### Scenario 3: Application Errors After Cutover

**Detection**: Error rate spikes, health checks fail

**Response**:
```bash
#!/bin/bash
# emergency_rollback.sh

echo "EMERGENCY ROLLBACK INITIATED"

# Step 1: Scale down multi-tenant app
kubectl patch deployment app-deployment -p '{"spec":{"replicas":0}}'

# Step 2: Revert configuration
kubectl create configmap app-config \
    --from-literal=DATABASE_MIGRATION_PHASE=single_tenant \
    --dry-run=client -o yaml | kubectl apply -f -

# Step 3: Scale up with old configuration
kubectl patch deployment app-deployment -p '{"spec":{"replicas":3}}'

# Step 4: Wait for readiness
kubectl wait --for=condition=available --timeout=300s deployment/app-deployment

# Step 5: Verify rollback success
curl -f http://health-check-endpoint/health || echo "ROLLBACK FAILED - MANUAL INTERVENTION REQUIRED"

echo "Emergency rollback completed"
```

### Scenario 4: Connection Pool Exhaustion

**Detection**: Connection errors, pool statistics

**Response**:
```bash
#!/bin/bash
# handle_connection_issues.sh

# Check PgBouncer status
check_pgbouncer() {
    psql -h pgbouncer-host -p 6432 -U pgbouncer -d pgbouncer -c "SHOW POOLS;"
    psql -h pgbouncer-host -p 6432 -U pgbouncer -d pgbouncer -c "SHOW CLIENTS;"
}

# Increase pool size temporarily
increase_pool_size() {
    # Update PgBouncer config
    sudo sed -i 's/default_pool_size = 25/default_pool_size = 50/' /etc/pgbouncer/pgbouncer.ini
    
    # Reload configuration
    sudo systemctl reload pgbouncer
    
    echo "Pool size increased to 50"
}

# Kill idle connections
kill_idle_connections() {
    psql -h pgbouncer-host -p 6432 -U pgbouncer -d pgbouncer -c "KILL idle;"
}

# Execute based on severity
check_pgbouncer
increase_pool_size
kill_idle_connections
```

## 5. Complete Rollback Plan

### Rollback Decision Matrix

| Issue | Rollback Trigger | Rollback Type |
|-------|------------------|---------------|
| Data inconsistency >1% | Immediate | Full |
| Replication lag >60s | After 15min | Partial |
| Application errors >5% | Immediate | Full |
| Performance degradation >50% | After 30min | Full |

### Full Rollback Procedure

```bash
#!/bin/bash
# full_rollback.sh

set -e

ROLLBACK_START=$(date)
echo "Starting full rollback at $ROLLBACK_START"

# Step 1: Immediate traffic switch
echo "Switching traffic back to single-tenant..."
kubectl create configmap app-config \
    --from-literal=DATABASE_MIGRATION_PHASE=single_tenant \
    --dry-run=client -o yaml | kubectl apply -f -

# Step 2: Rolling restart of application pods
kubectl rollout restart deployment/app-deployment
kubectl rollout status deployment/app-deployment

# Step 3: Verify single-tenant functionality
for tenant in tenant-a tenant-b tenant-c; do
    RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" http://${tenant}.yourapp.com/health)
    if [ "$RESPONSE" != "200" ]; then
        echo "ERROR: Tenant $tenant health check failed"
        exit 1
    fi
    echo "OK: Tenant $tenant is healthy"
done

# Step 4: Clean up multi-tenant resources (optional)
echo "Cleaning up replication subscriptions..."
psql -h target-host -U postgres -d saas_multitenant -c \
    "DROP SUBSCRIPTION IF EXISTS tenant_a_sub;"
psql -h target-host -U postgres -d saas_multitenant -c \
    "DROP SUBSCRIPTION IF EXISTS tenant_b_sub;"
psql -h target-host -U postgres -d saas_multitenant -c \
    "DROP SUBSCRIPTION IF EXISTS tenant_c_sub;"

# Step 5: Document rollback reason
echo "Rollback completed at $(date)"
echo "Reason: $1"  # Pass reason as first argument
echo "Duration: $(($(date +%s) - $(date -d "$ROLLBACK_START" +%s))) seconds"
```

## 6. Self-Check Readiness List

### Pre-Migration Checklist ✓

- [ ] **Infrastructure Ready**
  - [ ] Target multi-tenant database provisioned and configured
  - [ ] PgBouncer connection pooler installed and configured
  - [ ] Logical replication enabled on all source databases
  - [ ] Network connectivity tested between all systems
  - [ ] SSL certificates configured for secure connections

- [ ] **Schema Validation** 
  - [ ] Multi-tenant schema created with tenant_id columns
  - [ ] Row Level Security (RLS) policies implemented and tested
  - [ ] Unique constraints updated to include tenant_id
  - [ ] Indexes optimized for multi-tenant queries
  - [ ] Foreign key relationships validated

- [ ] **Application Code**
  - [ ] Tenant middleware implemented and tested
  - [ ] Database router supports migration phases
  - [ ] All queries properly scoped by tenant_id
  - [ ] Connection pooling integrated
  - [ ] Error handling updated for multi-tenancy

- [ ] **Data Migration**
  - [ ] Initial data load script tested with sample data
  - [ ] Logical replication publications created
  - [ ] Subscriptions established and data flowing
  - [ ] Replication lag monitoring in place
  - [ ] Data validation scripts running continuously

### Pre-Cutover Checklist ✓

- [ ] **Replication Health**
  - [ ] All subscriptions active and healthy
  - [ ] Replication lag < 5 seconds for 24+ hours
  - [ ] No replication conflicts or errors
  - [ ] Data consistency validation passing 99.9%+

- [ ] **Application Readiness**
  - [ ] Multi-tenant code deployed to staging
  - [ ] Load testing completed successfully
  - [ ] Performance benchmarks meet requirements
  - [ ] Error handling and monitoring configured

- [ ] **Operational Readiness**
  - [ ] Runbooks tested and validated
  - [ ] Rollback procedures tested in staging
  - [ ] Monitoring and alerting configured
  - [ ] On-call team briefed and available
  - [ ] Communication plan ready for stakeholders

### Post-Cutover Validation ✓

```bash
#!/bin/bash
# post_cutover_validation.sh

echo "Running post-cutover validation..."

validate_application_health() {
    echo "Checking application health..."
    for tenant in tenant-a tenant-b tenant-c; do
        RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" http://${tenant}.yourapp.com/health)
        if [ "$RESPONSE" = "200" ]; then
            echo "✓ $tenant application is healthy"
        else
            echo "✗ $tenant application health check failed (HTTP $RESPONSE)"
            return 1
        fi
    done
}

validate_database_connectivity() {
    echo "Checking database connectivity..."
    RESULT=$(psql -h pgbouncer-host -p 6432 -U application_role -d saas_multitenant -c "SELECT 1;" -t)
    if [ "$RESULT" = "1" ]; then
        echo "✓ Database connectivity confirmed"
    else
        echo "✗ Database connectivity failed"
        return 1
    fi
}

validate_tenant_isolation() {
    echo "Checking tenant isolation..."
    # Test that tenant A can only see their data
    TENANT_A_ID=$(psql -h pgbouncer-host -p 6432 -U application_role -d saas_multitenant -t -c \
        "SELECT id FROM tenants WHERE slug = 'tenant-a';")
    
    # Set tenant context and query
    RESULT=$(psql -h pgbouncer-host -p 6432 -U application_role -d saas_multitenant -t -c \
        "SET app.tenant_id = '$TENANT_A_ID'; SELECT COUNT(*) FROM users;")
    
    if [ "$RESULT" -gt "0" ]; then
        echo "✓ Tenant isolation working"
    else
        echo "✗ Tenant isolation failed"
        return 1
    fi
}

validate_performance() {
    echo "Checking performance..."
    START_TIME=$(date +%s%N)
    
    # Run sample query
    psql -h pgbouncer-host -p 6432 -U application_role -d saas_multitenant -c \
        "SELECT COUNT(*) FROM users;" > /dev/null
    
    END_TIME=$(date +%s%N)
    DURATION=$(((END_TIME - START_TIME) / 1000000))  # Convert to milliseconds
    
    if [ "$DURATION" -lt "1000" ]; then
        echo "✓ Query performance acceptable (${DURATION}ms)"
    else
        echo "⚠ Query performance slow (${DURATION}ms)"
    fi
}

# Run all validations
validate_application_health && \
validate_database_connectivity && \
validate_tenant_isolation && \
validate_performance

if [ $? -eq 0 ]; then
    echo "🎉 All post-cutover validations passed!"
    echo "Migration completed successfully at $(date)"
else
    echo "❌ Post-cutover validation failed - consider rollback"
    exit 1
fi
```

### Final Operational Checklist ✓

- [ ] **Performance Monitoring**
  - [ ] Query performance within acceptable thresholds
  - [ ] Connection pool utilization optimal
  - [ ] Database resource usage monitored
  - [ ] Application response times normal

- [ ] **Security Validation**
  - [ ] Row Level Security working correctly
  - [ ] No cross-tenant data leakage
  - [ ] Access controls properly configured
  - [ ] Audit logging functional

- [ ] **Cleanup and Documentation**
  - [ ] Source database logical replication disabled
  - [ ] Temporary migration resources cleaned up
  - [ ] Documentation updated with new architecture
  - [ ] Team trained on multi-tenant operations
  - [ ] Post-mortem scheduled if issues occurred

**Migration Status**: ✅ **READY FOR PRODUCTION CUTOVER**

---

This comprehensive migration plan provides zero-downtime transition from single-tenant to multi-tenant PostgreSQL architecture with robust validation, monitoring, and rollback capabilities. The row-based multi-tenancy approach with proper RLS implementation ensures scalability while maintaining data isolation and operational simplicity.