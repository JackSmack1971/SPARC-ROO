# SPARC Data Modeling Standards
## Schema Design Patterns for Modular, Scalable, and Secure Data Systems

### Core Philosophy

Data architecture in SPARC methodology follows the principle of **modular data excellence**: each data module should be self-contained, well-bounded, and capable of independent evolution while maintaining strong consistency and integration capabilities.

**The 20-Table Rule**: No single module should contain more than 20 tables. This constraint forces proper domain decomposition and prevents monolithic data designs that become unmaintainable.

**Data as a Product**: Treat each data module as a product with clear contracts, SLAs, and consumer interfaces. Data modules should expose well-defined APIs rather than direct table access.

**Security by Design**: Every data model must incorporate privacy, access control, and audit capabilities from the foundation level.

### The SPARC Data Module Pattern

#### Module Boundary Definition

Each data module represents a **bounded context** with clear domain responsibilities:

```
🏗️ Data Module Architecture:
┌─────────────────────────────────────────┐
│  Data Module: User Management (<20 tables) │
├─────────────────────────────────────────┤
│  Core Tables (3-5):                    │
│  • users (primary entity)              │
│  • user_profiles (extended attributes)  │
│  • user_sessions (transient data)      │
├─────────────────────────────────────────┤
│  Relationship Tables (2-4):            │
│  • user_roles (many-to-many)           │
│  • user_permissions (access control)   │
├─────────────────────────────────────────┤
│  Audit Tables (2-3):                   │
│  • user_audit_log (change tracking)    │
│  • user_access_log (security audit)    │
├─────────────────────────────────────────┤
│  Configuration Tables (1-3):           │
│  • user_preferences (settings)         │
│  • user_notifications (communication)  │
├─────────────────────────────────────────┤
│  Integration Tables (2-4):             │
│  • user_external_ids (system mapping)  │
│  • user_sync_status (integration state)│
└─────────────────────────────────────────┘
```

#### Module Interface Contracts

**Data Access Layer (DAL) Pattern**:
- Each module exposes a clean API through views or stored procedures
- Direct table access is restricted to the module's domain services
- Cross-module access occurs only through defined integration points
- All access is logged and can be monitored

**Module Integration Pattern**:
```sql
-- Example: Integration between User and Order modules
-- Users module exposes user_public_view
-- Orders module references users through user_external_ids only

CREATE VIEW user_public_view AS
SELECT 
    user_id,
    email_hash,  -- Privacy: hashed email for identification
    status,
    created_at,
    last_active_at
FROM users u
WHERE u.status = 'active'
  AND u.privacy_level <= current_user_privacy_clearance();
```

### Schema Design Patterns

#### 1. Entity Design Patterns

**Primary Entity Table Structure**:
```sql
-- Standard SPARC entity table template
CREATE TABLE users (
    -- Primary Key: UUID for global uniqueness
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Business Keys: Natural identifiers
    email VARCHAR(320) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE,
    
    -- Core Attributes: Essential business data
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    status user_status_enum NOT NULL DEFAULT 'pending',
    
    -- Privacy Controls: GDPR/CCPA compliance
    privacy_level privacy_level_enum NOT NULL DEFAULT 'standard',
    data_retention_policy VARCHAR(50) NOT NULL DEFAULT 'standard',
    consent_marketing BOOLEAN DEFAULT FALSE,
    consent_analytics BOOLEAN DEFAULT FALSE,
    
    -- Security Attributes
    password_hash VARCHAR(255),  -- Never store plain passwords
    two_factor_enabled BOOLEAN DEFAULT FALSE,
    account_locked_until TIMESTAMPTZ,
    failed_login_attempts INTEGER DEFAULT 0,
    
    -- Temporal Tracking: Essential for audit and debugging
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID REFERENCES users(user_id),
    updated_by UUID REFERENCES users(user_id),
    
    -- Soft Delete: Maintain referential integrity
    deleted_at TIMESTAMPTZ,
    deleted_by UUID REFERENCES users(user_id),
    
    -- Row Version: Optimistic concurrency control
    version_number INTEGER NOT NULL DEFAULT 1,
    
    -- Constraints
    CONSTRAINT users_email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    CONSTRAINT users_names_not_empty CHECK (
        LENGTH(TRIM(first_name)) > 0 AND 
        LENGTH(TRIM(last_name)) > 0
    ),
    CONSTRAINT users_temporal_logic CHECK (
        (deleted_at IS NULL AND deleted_by IS NULL) OR 
        (deleted_at IS NOT NULL AND deleted_by IS NOT NULL)
    )
);
```

**Extended Attributes Pattern** (for flexible schema evolution):
```sql
-- Separate table for attributes that may evolve
CREATE TABLE user_profiles (
    profile_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    
    -- Flexible attributes using JSONB for PostgreSQL
    profile_data JSONB NOT NULL DEFAULT '{}',
    
    -- Indexed commonly-queried attributes
    date_of_birth DATE,
    timezone VARCHAR(50),
    locale VARCHAR(10),
    
    -- Standard temporal tracking
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    version_number INTEGER NOT NULL DEFAULT 1,
    
    -- Constraints
    CONSTRAINT user_profiles_one_per_user UNIQUE (user_id),
    CONSTRAINT user_profiles_valid_json CHECK (jsonb_typeof(profile_data) = 'object')
);

-- Index for JSONB queries
CREATE INDEX idx_user_profiles_data_gin ON user_profiles USING GIN (profile_data);
```

#### 2. Relationship Design Patterns

**Many-to-Many with Audit Pattern**:
```sql
-- Relationship table with full audit capability
CREATE TABLE user_roles (
    user_role_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id),
    role_id UUID NOT NULL REFERENCES roles(role_id),
    
    -- Relationship attributes
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    assigned_by UUID NOT NULL REFERENCES users(user_id),
    expires_at TIMESTAMPTZ,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    
    -- Audit trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL REFERENCES users(user_id),
    updated_by UUID REFERENCES users(user_id),
    
    -- Prevent duplicate active assignments
    CONSTRAINT user_roles_unique_active UNIQUE (user_id, role_id, is_active) 
        DEFERRABLE INITIALLY DEFERRED,
    
    -- Business logic constraints
    CONSTRAINT user_roles_expiry_logic CHECK (
        expires_at IS NULL OR expires_at > assigned_at
    )
);
```

#### 3. Temporal and Audit Patterns

**Complete Audit Log Pattern**:
```sql
-- Comprehensive audit logging for regulatory compliance
CREATE TABLE user_audit_log (
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- What was changed
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    operation_type audit_operation_enum NOT NULL, -- INSERT, UPDATE, DELETE
    
    -- Change details
    old_values JSONB,
    new_values JSONB,
    changed_columns TEXT[],
    
    -- Context information
    user_id UUID REFERENCES users(user_id),
    session_id UUID,
    ip_address INET,
    user_agent TEXT,
    request_id UUID,
    
    -- Temporal information
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- Compliance tracking
    retention_until TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '7 years',
    compliance_tags TEXT[] DEFAULT '{}',
    
    -- Performance optimization
    partition_date DATE NOT NULL DEFAULT CURRENT_DATE
);

-- Partition by month for performance
CREATE INDEX idx_user_audit_log_partition ON user_audit_log (partition_date, table_name);
CREATE INDEX idx_user_audit_log_record ON user_audit_log (table_name, record_id, occurred_at);
```

#### 4. Security and Privacy Patterns

**Row-Level Security Pattern**:
```sql
-- Enable RLS for data protection
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their own data unless they have admin role
CREATE POLICY user_self_access ON users
    FOR ALL
    TO application_role
    USING (
        user_id = current_setting('app.current_user_id')::UUID
        OR has_role('admin')
        OR has_role('user_manager')
    );

-- Policy: Soft-deleted records are hidden unless explicitly requested
CREATE POLICY user_active_records ON users
    FOR SELECT
    TO application_role
    USING (
        deleted_at IS NULL 
        OR current_setting('app.include_deleted', true) = 'true'
    );
```

**Data Classification Pattern**:
```sql
-- Data sensitivity classification
CREATE TYPE data_classification AS ENUM (
    'public',       -- No restrictions
    'internal',     -- Company-only access
    'confidential', -- Role-based access required
    'restricted',   -- Explicit authorization required
    'top_secret'    -- Highest security clearance
);

-- Apply classification to sensitive columns
ALTER TABLE users ADD COLUMN data_class data_classification DEFAULT 'internal';
```

### Integration and Interoperability Standards

#### Cross-Module Communication Patterns

**Event-Driven Integration**:
```sql
-- Outbox pattern for reliable event publishing
CREATE TABLE user_events_outbox (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Event identification
    event_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    aggregate_type VARCHAR(100) NOT NULL,
    
    -- Event payload
    event_data JSONB NOT NULL,
    event_version INTEGER NOT NULL DEFAULT 1,
    
    -- Publishing control
    published_at TIMESTAMPTZ,
    published_by VARCHAR(100),
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    
    -- Temporal tracking
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- Processing status
    status event_status_enum NOT NULL DEFAULT 'pending',
    error_message TEXT,
    
    -- Correlation
    correlation_id UUID,
    causation_id UUID
);
```

**External System Integration Pattern**:
```sql
-- External system identifier mapping
CREATE TABLE user_external_mappings (
    mapping_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id),
    
    -- External system identification
    external_system VARCHAR(100) NOT NULL,
    external_id VARCHAR(500) NOT NULL,
    external_type VARCHAR(100) NOT NULL DEFAULT 'primary',
    
    -- Mapping metadata
    confidence_level DECIMAL(3,2) DEFAULT 1.0,
    verification_status verification_enum DEFAULT 'unverified',
    last_synchronized_at TIMESTAMPTZ,
    
    -- Standard tracking
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Unique constraint for active mappings
    CONSTRAINT user_external_unique_active 
        UNIQUE (external_system, external_id, is_active) 
        DEFERRABLE INITIALLY DEFERRED
);
```

### Performance and Scalability Patterns

#### Indexing Strategy

**Standard Index Patterns**:
```sql
-- Primary lookup indexes (B-tree for equality and range queries)
CREATE INDEX idx_users_email ON users (email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_status ON users (status) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_created_at ON users (created_at) WHERE deleted_at IS NULL;

-- Composite indexes for common query patterns
CREATE INDEX idx_users_status_created ON users (status, created_at) 
    WHERE deleted_at IS NULL;

-- Partial indexes for specific use cases
CREATE INDEX idx_users_locked_accounts ON users (account_locked_until)
    WHERE account_locked_until IS NOT NULL AND account_locked_until > NOW();

-- GIN indexes for JSONB and array columns
CREATE INDEX idx_user_profiles_data ON user_profiles USING GIN (profile_data);
CREATE INDEX idx_users_tags ON users USING GIN (tags) WHERE tags IS NOT NULL;
```

#### Partitioning Strategy

**Time-based Partitioning for Audit Tables**:
```sql
-- Partition audit table by month
CREATE TABLE user_audit_log_y2024m01 PARTITION OF user_audit_log
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Automated partition management
CREATE OR REPLACE FUNCTION maintain_audit_partitions()
RETURNS void AS $$
DECLARE
    partition_date DATE;
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    -- Create partitions for next 3 months
    FOR i IN 0..2 LOOP
        partition_date := date_trunc('month', CURRENT_DATE + (i || ' months')::INTERVAL);
        partition_name := 'user_audit_log_y' || 
                         EXTRACT(year FROM partition_date) || 'm' ||
                         LPAD(EXTRACT(month FROM partition_date)::TEXT, 2, '0');
        
        start_date := partition_date;
        end_date := partition_date + INTERVAL '1 month';
        
        -- Create partition if it doesn't exist
        EXECUTE format('
            CREATE TABLE IF NOT EXISTS %I PARTITION OF user_audit_log
            FOR VALUES FROM (%L) TO (%L)',
            partition_name, start_date, end_date);
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### Data Quality and Validation Standards

#### Constraint Patterns

**Business Rule Constraints**:
```sql
-- Email domain validation for business rules
ALTER TABLE users ADD CONSTRAINT users_email_domain
    CHECK (
        -- Allow personal emails for customers
        email ~* '@(gmail|yahoo|hotmail|outlook)\.com$'
        OR
        -- Require corporate emails for employees
        (user_type = 'employee' AND email ~* '@company\.com$')
    );

-- Temporal business logic
ALTER TABLE user_subscriptions ADD CONSTRAINT subscription_dates_logical
    CHECK (
        start_date <= end_date
        AND start_date >= created_at::DATE
        AND (canceled_at IS NULL OR canceled_at >= start_date)
    );
```

#### Data Validation Functions

**Standardized Validation**:
```sql
-- Email validation function
CREATE OR REPLACE FUNCTION is_valid_email(email_address TEXT)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN email_address ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
           AND LENGTH(email_address) <= 320
           AND LENGTH(split_part(email_address, '@', 1)) <= 64;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Phone number standardization
CREATE OR REPLACE FUNCTION standardize_phone(phone_input TEXT)
RETURNS TEXT AS $$
BEGIN
    -- Remove all non-digits
    phone_input := regexp_replace(phone_input, '[^0-9]', '', 'g');
    
    -- Handle US numbers
    IF LENGTH(phone_input) = 10 THEN
        RETURN '+1' || phone_input;
    ELSIF LENGTH(phone_input) = 11 AND LEFT(phone_input, 1) = '1' THEN
        RETURN '+' || phone_input;
    END IF;
    
    -- Return null for invalid formats
    RETURN NULL;
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```

### Module Documentation Standards

#### Required Documentation per Module

**Schema Documentation Template**:
```markdown
# Data Module: User Management

## Overview
Manages user accounts, authentication, profiles, and basic user lifecycle.

## Table Inventory (15/20 tables used)
- **Core Tables (4)**: users, user_profiles, user_sessions, user_preferences
- **Security Tables (3)**: user_credentials, user_access_tokens, user_security_log
- **Relationship Tables (3)**: user_roles, user_groups, user_permissions
- **Audit Tables (2)**: user_audit_log, user_access_log
- **Integration Tables (3)**: user_external_mappings, user_sync_status, user_events_outbox

## Integration Points
- **Exposes**: user_public_view, user_authentication_api
- **Consumes**: role_definitions (from Access Control module)
- **Events Published**: user.created, user.updated, user.deleted, user.authenticated
- **Events Consumed**: role.assigned, permission.granted

## Business Rules
- Users must verify email before account activation
- Password must meet complexity requirements (defined in user_security_policies)
- Account locks after 5 failed login attempts
- GDPR compliance: user data retained for 7 years after account deletion

## Performance Characteristics
- Target: <50ms for authentication queries
- Partitioned audit tables by month
- Read replicas for reporting queries
- Connection pooling required for high concurrency

## Security Controls
- Row-level security enabled on all user tables
- All passwords hashed with bcrypt (min 12 rounds)
- PII encrypted at rest using column-level encryption
- Audit logging for all data access and modifications
```

### Migration and Evolution Standards

#### Schema Migration Pattern

**Versioned Migration Strategy**:
```sql
-- Migration script template: V001__Create_user_module.sql
-- SPARC Migration Header
-- Module: User Management
-- Version: 1.0.0
-- Author: SPARC Data Architect
-- Dependencies: None
-- Rollback: V001_rollback__Drop_user_module.sql

BEGIN;

-- Create enums first (they can't be rolled back easily)
CREATE TYPE user_status AS ENUM ('pending', 'active', 'suspended', 'deleted');
CREATE TYPE privacy_level AS ENUM ('minimal', 'standard', 'enhanced', 'maximum');

-- Create tables with full constraints
-- (Table creation code here)

-- Create indexes
-- (Index creation code here)

-- Insert reference data
INSERT INTO user_security_policies (policy_name, policy_value) VALUES 
    ('password_min_length', '12'),
    ('password_complexity_required', 'true'),
    ('account_lockout_attempts', '5'),
    ('session_timeout_minutes', '30');

-- Grant appropriate permissions
GRANT SELECT, INSERT, UPDATE ON users TO application_role;
GRANT SELECT ON user_public_view TO read_only_role;

COMMIT;
```

**Rollback Strategy**:
```sql
-- V001_rollback__Drop_user_module.sql
-- SPARC Rollback Header
-- WARNING: This will destroy all user data
-- Ensure proper backup before execution

BEGIN;

-- Drop in reverse order of creation
DROP TABLE IF EXISTS user_audit_log;
DROP TABLE IF EXISTS user_external_mappings;
DROP TABLE IF EXISTS user_roles;
DROP TABLE IF EXISTS user_profiles;
DROP TABLE IF EXISTS users;

-- Drop types
DROP TYPE IF EXISTS privacy_level;
DROP TYPE IF EXISTS user_status;

COMMIT;
```

### Quality Assurance Checklist

#### Pre-Implementation Review

✅ **Module Design Review**
- [ ] Module contains ≤20 tables
- [ ] All tables have proper primary keys (UUID recommended)
- [ ] Temporal tracking implemented on all business tables
- [ ] Audit logging designed for compliance requirements
- [ ] Row-level security policies defined
- [ ] Integration points clearly documented

✅ **Security Review**
- [ ] No sensitive data stored in plain text
- [ ] All PII properly classified and protected
- [ ] Access control policies implemented
- [ ] Audit trail covers all data access and modifications
- [ ] Data retention policies defined and implemented

✅ **Performance Review**
- [ ] Indexing strategy covers all query patterns
- [ ] Partitioning implemented for large tables
- [ ] Query performance targets defined
- [ ] Connection pooling and caching strategies planned

✅ **Integration Review**
- [ ] Module boundaries respect domain model
- [ ] Integration contracts clearly defined
- [ ] Event publishing/consuming patterns implemented
- [ ] External system mappings properly designed

✅ **Documentation Completeness**
- [ ] Schema documentation complete
- [ ] Business rules documented
- [ ] Integration points documented
- [ ] Performance characteristics documented
- [ ] Security controls documented
- [ ] Migration strategy documented

#### Post-Implementation Validation

✅ **Functional Validation**
- [ ] All business rules enforced by constraints
- [ ] CRUD operations work correctly
- [ ] Audit logging captures all changes
- [ ] Security policies function as designed
- [ ] Integration points tested

✅ **Performance Validation**
- [ ] Query performance meets targets
- [ ] Index usage optimized
- [ ] Connection pooling configured
- [ ] Partitioning working correctly

✅ **Security Validation**
- [ ] Access control policies tested
- [ ] PII protection verified
- [ ] Audit trails complete and accurate
- [ ] Data encryption working correctly

These standards ensure that every data module in your SPARC system maintains the highest levels of quality, security, and maintainability while supporting the modular architecture principles that make SPARC systems so effective.
