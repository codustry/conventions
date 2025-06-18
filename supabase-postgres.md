
# Supabase-Postgres Right Understanding And Conventions 
> update 18 Jun 2025, Nutchanon Ninyawee

## Right Understanding
supabase latest version is postgres 17 
no longer include the following Postgres extensions
timescaledb
plv8
plls
plcoffee
pgjwt
 “end of life” of Postgres 15 in approximately 1 year (May 2026)



## Object Prefixes

- Tables: `tb_` (tb_users)
- Views: `v_` (v_active_users)
- Materialized Views: `mv_` (mv_daily_stats)
- Functions: `fn_` (fn_get_balance_v1)
- Triggers: `tgr_` (tgr_update_ts)
- Indexes: `idx_` (idx_email)
- Foreign Keys: `fk_` (fk_order_user)
- Primary Keys: `pk_` (pk_users)
- Unique Constraints: `uq_` (uq_email)
- Enum: `en_` (en_weight_direction)
- Policy: `pc_` (pc_follow_permission_user_role)

## Object Suffixes
- Versions
  - Functions: `_v1` 


## Field Suffixes
meant to reveal the type-concept of the field with for human.
- it can explicitly imply validation. normal value range. unit of measurement. for downstream development like frontend.
- some fields might omit suffixes if the type is self-explanatory. e.g. `id` `name` `email` `created_at` `updated_at` `deleted_at` `created_by` `updated_by` `deleted_by` 
  
- `_dt`: dates (created_dt)
- `_ts`: timestamps (login_ts)
- `_num`: numbers (items_num)
- `_amt`: money (total_amt)
- `_pct`: percentages (discount_pct)
- `_uid`: UUID identifiers (user_uid)
- `_cd`: codes (status_cd, role_cd)
- `_bool`: boolean (active_bool)
- `_pn`: phone numbers (phone_number_pn)
- `_em`: email
- `_txt`: text (name_txt) use as last resort since this infer zero idea of how to validate string
- `_kg`: weight in kg SI unit
- `_path`: path usually means storage path for supabase storage

## Tradition

1. Use lowercase_snake_case
2. Tables plural (tb_users not tb_user)
3. Recommend fields:
   - id (uuid PRIMARY KEY DEFAULT uuid_generate_v4())
4. User metadata for audit
   - created_dt (timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP)
   - updated_dt (timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP)
5. Use modern Types if the biz logic allows
   - Use `ltree` for hierarchical data e.g. organization structure, product categories
   - Use `jsonb` for flexible data like api responses, user settings
   - Use [range](https://www.postgresql.org/docs/16/rangetypes.html) e.g 
    - `int8range` 
    - `tstzrange` instead of `start_dt` and `end_dt` for time intervals

## DB Types

- Default string type: `text`
- Timestamps always use `timestamptz`
- Money: `decimal`
- IDs: `uuid` with UUIDv4 
- JSON: `jsonb`
- Enums: Use PostgreSQL ENUM types for fixed values


## Comments

```sql
COMMENT ON TABLE tb_users IS 'Core: User accounts';
COMMENT ON COLUMN tb_users.status_cd IS 'Status (ACTIVE,SUSPENDED)';
COMMENT ON TYPE app_role IS 'Available roles for role-based access control';
```
## Supabase Caveats

[Using authentication policies with views](mdc:whale/whale/whale/whale/whale/whale/whale/whale/whale/whale/whale/https:/github.com/orgs/supabase/discussions/901)

```sql
CREATE VIEW sorted_titles WITH (security_invoker) as SELECT title FROM posts ORDER BY title;
```

## Function
all function must be created with the following at the end:
```sql
$$ LANGUAGE PLPGSQL SECURITY DEFINER
SET search_path = extensions, public, pg_temp;
```
ref https://github.com/supabase/supabase/issues/462

## Common Triggers

### Update timestamp trigger
For tables with `updated_at` timestamp:
```sql
CREATE TRIGGER update_[table_name]_updated_at 
    BEFORE UPDATE ON public.[table_name] 
    FOR EACH ROW 
    EXECUTE FUNCTION update_updated_at_column();
```

Example:
```sql
CREATE TRIGGER update_line_patient_mapping_updated_at 
    BEFORE UPDATE ON public.tb_line_patient_mapping 
    FOR EACH ROW 
    EXECUTE FUNCTION update_updated_at_column();
```
