# 🗄️ Database Schema Documentation

## 📚 Зміст
1. [Database Overview](#database-overview)
2. [Schema Design](#schema-design)
3. [Tables](#tables)
4. [Indexes](#indexes)
5. [Migrations](#migrations)
6. [Backup Strategy](#backup-strategy)

## 🏗 Database Overview

### Technology Stack
- **Database:** PostgreSQL 14+
- **ORM:** GORM (Go)
- **Migration Tool:** golang-migrate
- **Backup:** pg_dump + Google Cloud Storage

### Architecture Principles
- Normalized schema (3NF)
- UUID for primary keys
- Soft deletes for critical data
- Audit trails for sensitive operations
- Optimized indexes for common queries

## 📊 Schema Design

```sql
-- Database creation
CREATE DATABASE finance_dashboard
    WITH
    OWNER = postgres
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.utf8'
    LC_CTYPE = 'en_US.utf8'
    TABLESPACE = pg_default
    CONNECTION LIMIT = -1;

-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm"; -- For text search
```

## 📑 Tables

### Users Table

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    locale VARCHAR(10) DEFAULT 'en_US',
    timezone VARCHAR(50) DEFAULT 'UTC',
    is_active BOOLEAN DEFAULT true,
    email_verified BOOLEAN DEFAULT false,
    email_verified_at TIMESTAMP,
    last_login_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP
);

-- Indexes
CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_created_at ON users(created_at);
```

### Categories Table

```sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(20) NOT NULL CHECK (type IN ('income', 'expense', 'both')),
    color VARCHAR(7), -- HEX color code
    icon VARCHAR(50),
    parent_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    is_system BOOLEAN DEFAULT false, -- System default categories
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(user_id, name)
);

-- Indexes
CREATE INDEX idx_categories_user_id ON categories(user_id);
CREATE INDEX idx_categories_type ON categories(type);
CREATE INDEX idx_categories_parent_id ON categories(parent_id);

-- Default categories
INSERT INTO categories (name, type, is_system, color, icon) VALUES
    ('Salary', 'income', true, '#4CAF50', 'money'),
    ('Freelance', 'income', true, '#8BC34A', 'work'),
    ('Investment', 'income', true, '#00BCD4', 'trending_up'),
    ('Food', 'expense', true, '#FF5722', 'restaurant'),
    ('Transport', 'expense', true, '#9C27B0', 'directions_car'),
    ('Housing', 'expense', true, '#3F51B5', 'home'),
    ('Utilities', 'expense', true, '#009688', 'bolt'),
    ('Healthcare', 'expense', true, '#F44336', 'local_hospital'),
    ('Entertainment', 'expense', true, '#FF9800', 'movie'),
    ('Shopping', 'expense', true, '#E91E63', 'shopping_cart');
```

### Transactions Table

```sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    amount DECIMAL(15, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    type VARCHAR(20) NOT NULL CHECK (type IN ('income', 'expense', 'transfer')),
    description TEXT,
    date DATE NOT NULL,
    is_recurring BOOLEAN DEFAULT false,
    recurring_id UUID REFERENCES recurring_transactions(id) ON DELETE SET NULL,
    tags TEXT[], -- Array of tags
    location JSONB, -- {"lat": 0.0, "lng": 0.0, "name": "Location"}
    receipt_url VARCHAR(500),
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP
);

-- Indexes
CREATE INDEX idx_transactions_user_id ON transactions(user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_transactions_category_id ON transactions(category_id);
CREATE INDEX idx_transactions_date ON transactions(date DESC);
CREATE INDEX idx_transactions_type ON transactions(type);
CREATE INDEX idx_transactions_amount ON transactions(amount);
CREATE INDEX idx_transactions_user_date ON transactions(user_id, date DESC) WHERE deleted_at IS NULL;
CREATE INDEX idx_transactions_tags ON transactions USING GIN(tags);
```

### Recurring Transactions Table

```sql
CREATE TABLE recurring_transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    amount DECIMAL(15, 2) NOT NULL,
    type VARCHAR(20) NOT NULL CHECK (type IN ('income', 'expense')),
    description TEXT,
    frequency VARCHAR(20) NOT NULL CHECK (frequency IN ('daily', 'weekly', 'monthly', 'yearly')),
    interval_value INTEGER DEFAULT 1, -- Every N days/weeks/months
    day_of_week INTEGER, -- 0-6 for weekly
    day_of_month INTEGER, -- 1-31 for monthly
    month_of_year INTEGER, -- 1-12 for yearly
    start_date DATE NOT NULL,
    end_date DATE,
    next_date DATE NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_recurring_user_id ON recurring_transactions(user_id);
CREATE INDEX idx_recurring_next_date ON recurring_transactions(next_date) WHERE is_active = true;
```

### Budgets Table

```sql
CREATE TABLE budgets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    period VARCHAR(20) NOT NULL CHECK (period IN ('daily', 'weekly', 'monthly', 'yearly')),
    start_date DATE NOT NULL,
    end_date DATE,
    alert_threshold DECIMAL(5, 2) DEFAULT 80.00, -- Alert at 80% by default
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_budgets_user_id ON budgets(user_id);
CREATE INDEX idx_budgets_category_id ON budgets(category_id);
CREATE INDEX idx_budgets_period ON budgets(period);
CREATE INDEX idx_budgets_active ON budgets(is_active) WHERE is_active = true;
```

### Goals Table

```sql
CREATE TABLE goals (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    target_amount DECIMAL(15, 2) NOT NULL,
    current_amount DECIMAL(15, 2) DEFAULT 0,
    target_date DATE,
    category VARCHAR(50), -- vacation, emergency, purchase, etc.
    priority INTEGER DEFAULT 1,
    is_achieved BOOLEAN DEFAULT false,
    achieved_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_goals_user_id ON goals(user_id);
CREATE INDEX idx_goals_is_achieved ON goals(is_achieved);
CREATE INDEX idx_goals_priority ON goals(priority);
```

### Attachments Table

```sql
CREATE TABLE attachments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    transaction_id UUID REFERENCES transactions(id) ON DELETE CASCADE,
    file_name VARCHAR(255) NOT NULL,
    file_type VARCHAR(50),
    file_size INTEGER,
    storage_path VARCHAR(500) NOT NULL,
    uploaded_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_attachments_transaction_id ON attachments(transaction_id);
```

### Audit Log Table

```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL CHECK (action IN ('CREATE', 'UPDATE', 'DELETE')),
    old_values JSONB,
    new_values JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at DESC);
```

### Sessions Table

```sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) UNIQUE NOT NULL,
    refresh_token_hash VARCHAR(255) UNIQUE,
    ip_address INET,
    user_agent TEXT,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_activity_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_token_hash ON sessions(token_hash);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
```

## 🔍 Indexes

### Performance Indexes

```sql
-- Composite indexes for common queries
CREATE INDEX idx_transactions_user_month ON transactions(user_id, date_trunc('month', date))
    WHERE deleted_at IS NULL;

CREATE INDEX idx_transactions_user_category_date ON transactions(user_id, category_id, date DESC)
    WHERE deleted_at IS NULL;

-- Full text search
CREATE INDEX idx_transactions_description_search ON transactions
    USING gin(to_tsvector('english', description));

-- JSONB indexes
CREATE INDEX idx_transactions_location ON transactions USING gin(location);
```

## 📦 Views

### Monthly Summary View

```sql
CREATE VIEW monthly_summary AS
SELECT
    user_id,
    date_trunc('month', date) as month,
    SUM(CASE WHEN type = 'income' THEN amount ELSE 0 END) as total_income,
    SUM(CASE WHEN type = 'expense' THEN amount ELSE 0 END) as total_expense,
    SUM(CASE WHEN type = 'income' THEN amount ELSE -amount END) as balance,
    COUNT(CASE WHEN type = 'income' THEN 1 END) as income_count,
    COUNT(CASE WHEN type = 'expense' THEN 1 END) as expense_count
FROM transactions
WHERE deleted_at IS NULL
GROUP BY user_id, month;
```

### Category Summary View

```sql
CREATE VIEW category_summary AS
SELECT
    t.user_id,
    c.id as category_id,
    c.name as category_name,
    c.type as category_type,
    date_trunc('month', t.date) as month,
    SUM(t.amount) as total_amount,
    COUNT(t.id) as transaction_count,
    AVG(t.amount) as avg_amount
FROM transactions t
JOIN categories c ON t.category_id = c.id
WHERE t.deleted_at IS NULL
GROUP BY t.user_id, c.id, c.name, c.type, month;
```

## 🔄 Migrations

### Migration Files Structure

```
migrations/
├── 000001_create_users_table.up.sql
├── 000001_create_users_table.down.sql
├── 000002_create_categories_table.up.sql
├── 000002_create_categories_table.down.sql
├── 000003_create_transactions_table.up.sql
├── 000003_create_transactions_table.down.sql
└── ...
```

### Example Migration

```sql
-- 000004_add_budget_alerts.up.sql
CREATE TABLE budget_alerts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    budget_id UUID NOT NULL REFERENCES budgets(id) ON DELETE CASCADE,
    triggered_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    percentage_used DECIMAL(5, 2) NOT NULL,
    notified BOOLEAN DEFAULT false
);

CREATE INDEX idx_budget_alerts_budget_id ON budget_alerts(budget_id);

-- 000004_add_budget_alerts.down.sql
DROP TABLE IF EXISTS budget_alerts;
```

### Running Migrations

```bash
# Install migrate tool
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Create migration
migrate create -ext sql -dir migrations -seq create_users_table

# Run migrations up
migrate -path migrations -database "postgresql://user:pass@localhost:5432/finance_dashboard?sslmode=disable" up

# Rollback last migration
migrate -path migrations -database "postgresql://user:pass@localhost:5432/finance_dashboard?sslmode=disable" down 1

# Force version
migrate -path migrations -database "postgresql://user:pass@localhost:5432/finance_dashboard?sslmode=disable" force VERSION
```

## 🛡️ Database Functions

### Calculate Balance Function

```sql
CREATE OR REPLACE FUNCTION calculate_balance(
    p_user_id UUID,
    p_start_date DATE DEFAULT NULL,
    p_end_date DATE DEFAULT NULL
) RETURNS TABLE(
    income DECIMAL,
    expense DECIMAL,
    balance DECIMAL
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        COALESCE(SUM(CASE WHEN type = 'income' THEN amount ELSE 0 END), 0) as income,
        COALESCE(SUM(CASE WHEN type = 'expense' THEN amount ELSE 0 END), 0) as expense,
        COALESCE(SUM(CASE WHEN type = 'income' THEN amount ELSE -amount END), 0) as balance
    FROM transactions
    WHERE user_id = p_user_id
        AND deleted_at IS NULL
        AND (p_start_date IS NULL OR date >= p_start_date)
        AND (p_end_date IS NULL OR date <= p_end_date);
END;
$$ LANGUAGE plpgsql;
```

### Update Updated_at Trigger

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply trigger to all tables
CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_transactions_updated_at BEFORE UPDATE ON transactions
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ... repeat for other tables
```

### Audit Log Trigger

```sql
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_logs(entity_type, entity_id, action, new_values, user_id)
        VALUES (TG_TABLE_NAME, NEW.id, 'CREATE', to_jsonb(NEW), NEW.user_id);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_logs(entity_type, entity_id, action, old_values, new_values, user_id)
        VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), NEW.user_id);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_logs(entity_type, entity_id, action, old_values, user_id)
        VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', to_jsonb(OLD), OLD.user_id);
        RETURN OLD;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Apply to sensitive tables
CREATE TRIGGER audit_transactions AFTER INSERT OR UPDATE OR DELETE ON transactions
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
```

## 💾 Backup Strategy

### Automated Backup Script

```bash
#!/bin/bash
# backup.sh

# Configuration
DB_NAME="finance_dashboard"
DB_USER="postgres"
BACKUP_DIR="/backups"
GCS_BUCKET="gs://finance-dashboard-backups"
RETENTION_DAYS=30

# Create backup
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/backup_${TIMESTAMP}.sql.gz"

# Dump database
pg_dump -U ${DB_USER} -d ${DB_NAME} | gzip > ${BACKUP_FILE}

# Upload to Google Cloud Storage
gsutil cp ${BACKUP_FILE} ${GCS_BUCKET}/

# Clean old local backups
find ${BACKUP_DIR} -name "backup_*.sql.gz" -mtime +7 -delete

# Clean old GCS backups
gsutil ls ${GCS_BUCKET}/ | while read file; do
    FILE_DATE=$(echo $file | grep -oP '\d{8}')
    if [[ $(date -d "${FILE_DATE}" +%s) -lt $(date -d "${RETENTION_DAYS} days ago" +%s) ]]; then
        gsutil rm $file
    fi
done

echo "Backup completed: ${BACKUP_FILE}"
```

### Restore Procedure

```bash
# Restore from backup
gunzip < backup_20240115_120000.sql.gz | psql -U postgres -d finance_dashboard

# Point-in-time recovery
pg_restore -U postgres -d finance_dashboard -t transactions backup.dump
```

## 🔧 Maintenance

### Regular Maintenance Tasks

```sql
-- Vacuum and analyze (weekly)
VACUUM ANALYZE;

-- Reindex (monthly)
REINDEX DATABASE finance_dashboard;

-- Update statistics
ANALYZE;

-- Check for unused indexes
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Check table sizes
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

## 🔐 Security

### Row Level Security

```sql
-- Enable RLS
ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;

-- Create policy
CREATE POLICY transactions_isolation ON transactions
    FOR ALL
    USING (user_id = current_setting('app.current_user_id')::UUID);

-- Set user context in application
SET LOCAL app.current_user_id = 'user-uuid-here';
```

### Encryption

```sql
-- Encrypt sensitive data
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Example: Encrypt user data
UPDATE users
SET password_hash = crypt('password', gen_salt('bf', 8))
WHERE id = 'user-id';

-- Verify password
SELECT id FROM users
WHERE email = 'user@example.com'
AND password_hash = crypt('password', password_hash);
```

## 📈 Performance Monitoring

### Slow Query Log

```sql
-- Find slow queries
SELECT
    query,
    mean_exec_time,
    calls,
    total_exec_time,
    min_exec_time,
    max_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 100
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Missing indexes
SELECT
    schemaname,
    tablename,
    attname,
    n_distinct,
    most_common_vals
FROM pg_stats
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
    AND n_distinct > 100
    AND tablename||'.'||attname NOT IN (
        SELECT tablename||'.'||column_name
        FROM information_schema.constraint_column_usage
    )
ORDER BY n_distinct DESC;
```

## 🚀 Connection Pooling

### PgBouncer Configuration

```ini
[databases]
finance_dashboard = host=localhost port=5432 dbname=finance_dashboard

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3
server_lifetime = 3600
server_idle_timeout = 600
```