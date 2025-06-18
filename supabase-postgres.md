# PostgreSQL Conventions for Supabase Projects

> Last updated: 18 Jan 2025  
> Author: Nutchanon Ninyawee  
> Version: 2.0

## Overview

This document defines PostgreSQL naming conventions and best practices for Supabase projects. It serves as a reference for both human developers and AI assistants to ensure consistent database design.

## Environment Context

### Supabase Platform
- **Current PostgreSQL Version**: 17
- **Deprecated Extensions**: 
  - timescaledb
  - plv8
  - plls
  - plcoffee
  - pgjwt
- **PostgreSQL 15 EOL**: May 2026

### Key Principles
1. **Consistency**: All database objects follow predictable naming patterns
2. **Self-documenting**: Names clearly indicate purpose and type
3. **Type safety**: Suffixes indicate data types and validation requirements
4. **AI-friendly**: Structured patterns enable automated code generation

## Naming Conventions

### Object Prefixes

All database objects must use the following prefixes:

| Object Type | Prefix | Example | Description |
|-------------|--------|---------|-------------|
| Tables | `tb_` | `tb_users` | Data storage tables |
| Views | `v_` | `v_active_users` | Virtual tables based on queries |
| Materialized Views | `mv_` | `mv_daily_stats` | Cached query results |
| Functions | `fn_` | `fn_get_balance_v1` | Stored procedures |
| Triggers | `tgr_` | `tgr_update_ts` | Automated actions |
| Indexes | `idx_` | `idx_email` | Performance optimization |
| Foreign Keys | `fk_` | `fk_order_user` | Referential constraints |
| Primary Keys | `pk_` | `pk_users` | Unique identifiers |
| Unique Constraints | `uq_` | `uq_email` | Uniqueness enforcement |
| Enum Types | `en_` | `en_weight_direction` | Fixed value sets |
| RLS Policies | `pc_` | `pc_follow_permission_user_role` | Row-level security |

### Object Suffixes

| Object Type | Suffix Pattern | Example | Use Case |
|-------------|----------------|---------|----------|
| Functions | `_v{number}` | `fn_calculate_total_v2` | Version control for functions |

### Field Suffixes

Field suffixes communicate data type, validation requirements, and units to both developers and AI systems.

#### Common Suffixes

| Suffix | Type | Example | Validation/Notes |
|--------|------|---------|------------------|
| `_dt` | date | `birth_dt` | Date only (no time) |
| `_ts` | timestamp | `login_ts` | Date + time with timezone |
| `_num` | number | `items_num` | Integer count |
| `_amt` | decimal | `total_amt` | Money values |
| `_pct` | decimal | `discount_pct` | 0-100 range |
| `_uid` | uuid | `user_uid` | UUID v4 identifier |
| `_cd` | text | `status_cd` | Enumerated codes |
| `_bool` | boolean | `active_bool` | True/false values |
| `_pn` | text | `contact_pn` | Phone number format |
| `_em` | text | `contact_em` | Email format validation |
| `_txt` | text | `description_txt` | Generic text (use sparingly) |
| `_kg` | decimal | `weight_kg` | SI unit: kilograms |
| `_path` | text | `avatar_path` | Supabase storage path |

#### Self-Explanatory Fields (No Suffix Required)
- `id`, `name`, `email`
- `created_at`, `updated_at`, `deleted_at`
- `created_by`, `updated_by`, `deleted_by`

## General Guidelines

### Naming Rules
1. **Case**: Always use `lowercase_snake_case`
2. **Tables**: Use plural forms (`tb_users` not `tb_user`)
3. **Clarity**: Prefer descriptive names over abbreviations

### Standard Table Structure

```sql
-- Required fields for all tables
CREATE TABLE tb_examples (
    id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    created_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
    -- Optional audit fields
    created_by uuid REFERENCES auth.users(id),
    updated_by uuid REFERENCES auth.users(id)
);
```

### Modern PostgreSQL Types

Leverage PostgreSQL's advanced types when appropriate:

| Use Case | Type | Example | Benefits |
|----------|------|---------|----------|
| Hierarchical data | `ltree` | Organization structure | Efficient tree queries |
| Flexible schemas | `jsonb` | User settings, API responses | Schema flexibility |
| Time intervals | `tstzrange` | Event durations | Built-in overlap checks |
| Number ranges | `int8range` | ID ranges, pagination | Range operations |

## Data Type Standards

| Purpose | PostgreSQL Type | Example Declaration | Notes |
|---------|----------------|---------------------|-------|
| Text/String | `text` | `name text NOT NULL` | Default for all strings |
| Timestamps | `timestamptz` | `created_at timestamptz` | Always with timezone |
| Money | `decimal(15,2)` | `price_amt decimal(15,2)` | Precise decimal math |
| IDs | `uuid` | `id uuid DEFAULT uuid_generate_v4()` | UUIDv4 standard |
| JSON | `jsonb` | `settings jsonb DEFAULT '{}'` | Binary JSON format |
| Enums | Custom ENUM | `CREATE TYPE en_status AS ENUM('active', 'inactive')` | Fixed value sets |

## Documentation Standards

### Table and Column Comments

All tables and non-obvious columns must have comments:

```sql
-- Table comments: Module prefix + description
COMMENT ON TABLE tb_users IS 'Core: User accounts and authentication';

-- Column comments: Include valid values for enums/codes
COMMENT ON COLUMN tb_users.status_cd IS 'User status (ACTIVE, SUSPENDED, DELETED)';
COMMENT ON COLUMN tb_users.role_cd IS 'User role (ADMIN, USER, GUEST)';

-- Type comments: Describe purpose and values
COMMENT ON TYPE en_user_role IS 'Available roles for role-based access control';
```

### Comment Format Guidelines
- Tables: `[Module]: [Description]`
- Enum columns: `[Description] ([VALUE1], [VALUE2], ...)`
- Complex columns: Include validation rules or business logic

## Supabase-Specific Considerations

### Security with Views

When using RLS (Row Level Security) with views, always use `security_invoker`:

```sql
-- Correct: RLS policies are applied based on the current user
CREATE VIEW v_user_posts WITH (security_invoker) AS 
SELECT * FROM tb_posts WHERE deleted_at IS NULL;

-- Without security_invoker, view runs with definer's privileges
```

Reference: [Supabase Discussion #901](https://github.com/orgs/supabase/discussions/901)

### Function Security

All functions must specify security context and search path:

```sql
CREATE OR REPLACE FUNCTION fn_calculate_total_v1(order_id uuid)
RETURNS decimal AS $$
BEGIN
    -- Function body
    RETURN 0;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER
SET search_path = extensions, public, pg_temp;
```

**Important**: The `search_path` setting prevents search path injection attacks.

Reference: [Supabase Issue #462](https://github.com/supabase/supabase/issues/462)

## Common Patterns

### Automatic Timestamp Updates

For tables with `updated_at` columns, create this trigger:

```sql
-- Generic function (create once)
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to each table
CREATE TRIGGER tgr_update_users_timestamp 
    BEFORE UPDATE ON tb_users 
    FOR EACH ROW 
    EXECUTE FUNCTION update_updated_at_column();
```

### Soft Delete Pattern

```sql
-- Add soft delete column
ALTER TABLE tb_users ADD COLUMN deleted_at timestamptz;

-- Create view for active records
CREATE VIEW v_active_users WITH (security_invoker) AS
SELECT * FROM tb_users WHERE deleted_at IS NULL;
```

## Quick Reference

### Creating a New Table

```sql
-- 1. Create enum if needed
CREATE TYPE en_user_status AS ENUM ('active', 'inactive', 'suspended');

-- 2. Create table with standard fields
CREATE TABLE tb_users (
    id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    email text UNIQUE NOT NULL,
    status_cd en_user_status NOT NULL DEFAULT 'active',
    created_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 3. Add comments
COMMENT ON TABLE tb_users IS 'Core: User accounts';
COMMENT ON COLUMN tb_users.status_cd IS 'User status (active, inactive, suspended)';

-- 4. Create indexes
CREATE INDEX idx_users_email ON tb_users(email);
CREATE INDEX idx_users_status ON tb_users(status_cd) WHERE deleted_at IS NULL;

-- 5. Add timestamp trigger
CREATE TRIGGER tgr_update_users_timestamp 
    BEFORE UPDATE ON tb_users 
    FOR EACH ROW 
    EXECUTE FUNCTION update_updated_at_column();

-- 6. Enable RLS
ALTER TABLE tb_users ENABLE ROW LEVEL SECURITY;
```

## AI Assistant Instructions

When creating or modifying database objects:

1. **Always follow naming conventions**: Check prefixes and suffixes
2. **Include standard fields**: id, created_at, updated_at
3. **Add comments**: Describe purpose and valid values
4. **Consider security**: Use RLS policies and secure functions
5. **Use appropriate types**: Prefer PostgreSQL's advanced types
6. **Create indexes**: For foreign keys and frequently queried columns
7. **Version functions**: Use `_v1`, `_v2` suffix pattern

## Examples for AI Context

### Pattern Recognition Examples

```sql
-- Correct naming patterns
tb_users                    -- Table: plural, with prefix
v_active_orders            -- View: descriptive, with prefix
fn_calculate_total_v1      -- Function: versioned
idx_users_email            -- Index: table_column format
en_order_status            -- Enum: descriptive type

-- Field naming with suffixes
user_uid                   -- UUID identifier
created_ts                 -- Timestamp with timezone
total_amt                  -- Money/decimal amount
status_cd                  -- Enumerated code
weight_kg                  -- Value with SI unit

-- Incorrect patterns to avoid
table_user                 -- Should be tb_users (plural)
getUserData()              -- Should be fn_get_user_data_v1
email_address              -- Should be email or email_em
timestamp                  -- Should be more specific: created_ts
```

### Migration Template

```sql
-- Migration: Add user profile table
BEGIN;

-- Create enum for profile visibility
CREATE TYPE en_profile_visibility AS ENUM ('public', 'private', 'friends');

-- Create profile table
CREATE TABLE tb_user_profiles (
    id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_uid uuid NOT NULL REFERENCES tb_users(id) ON DELETE CASCADE,
    display_name text NOT NULL,
    bio_txt text,
    avatar_path text,
    visibility_cd en_profile_visibility NOT NULL DEFAULT 'public',
    verified_bool boolean NOT NULL DEFAULT false,
    created_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Add comments
COMMENT ON TABLE tb_user_profiles IS 'User: Public profile information';
COMMENT ON COLUMN tb_user_profiles.visibility_cd IS 'Profile visibility (public, private, friends)';
COMMENT ON COLUMN tb_user_profiles.avatar_path IS 'Supabase storage path to avatar image';

-- Create indexes
CREATE UNIQUE INDEX idx_user_profiles_user ON tb_user_profiles(user_uid);
CREATE INDEX idx_user_profiles_visibility ON tb_user_profiles(visibility_cd);

-- Add timestamp trigger
CREATE TRIGGER tgr_update_user_profiles_timestamp 
    BEFORE UPDATE ON tb_user_profiles 
    FOR EACH ROW 
    EXECUTE FUNCTION update_updated_at_column();

-- Enable RLS
ALTER TABLE tb_user_profiles ENABLE ROW LEVEL SECURITY;

-- Create RLS policy
CREATE POLICY pc_user_profiles_select ON tb_user_profiles
    FOR SELECT USING (
        visibility_cd = 'public' 
        OR user_uid = auth.uid()
    );

COMMIT;
```
