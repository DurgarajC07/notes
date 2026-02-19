# 🔐 DCL Commands - Data Control Language

> DCL commands manage permissions and access control. Master GRANT, REVOKE, and user management to secure your database effectively.

---

## 📖 1. Concept Explanation

### What is DCL?

**Data Control Language (DCL)** commands control access to data and database objects through permissions and privileges.

**Core DCL Commands:**
```
DCL COMMANDS
│
├── GRANT - Give permissions
│   ├── Object privileges (SELECT, INSERT, UPDATE, DELETE)
│   ├── System privileges (CREATE TABLE, CREATE USER)
│   └── Role assignment
│
├── REVOKE - Remove permissions
│   ├── Remove object privileges
│   └── Remove role membership
│
└── Supporting Commands
    ├── CREATE USER
    ├── CREATE ROLE
    ├── ALTER USER
    └── DROP USER
```

**Permission Hierarchy:**
```
DATABASE SERVER
├── System Privileges
│   ├── CREATE DATABASE
│   ├── CREATE USER
│   └── SHUTDOWN
│
├── Database Privileges
│   ├── CREATE TABLE
│   ├── CREATE VIEW
│   └── CREATE PROCEDURE
│
└── Object Privileges
    ├── SELECT on table/view
    ├── INSERT, UPDATE, DELETE
    ├── EXECUTE on procedure
    └── REFERENCES on table
```

---

## 🧠 2. Why It Matters in Real Systems

### Security Breach from Over-Privileged User

**❌ Bad: Everyone has full access**
```sql
-- Junior developer account
GRANT ALL PRIVILEGES ON database.* TO 'developer'@'%';

-- Result: Developer can:
-- - DROP production tables ❌
-- - See customer credit cards ❌
-- - Delete all data ❌
-- - Create backdoor accounts ❌
```

**✅ Good: Principle of Least Privilege**
```sql
-- Developer: Read-only access to non-sensitive data
GRANT SELECT ON app_db.products TO 'developer'@'%';
GRANT SELECT ON app_db.orders TO 'developer'@'%';
-- No DROP, no DELETE, no sensitive tables ✅

-- Application: Only needed permissions
GRANT SELECT, INSERT, UPDATE ON app_db.* TO 'app_user'@'10.0.%';
-- No DELETE, no DROP ✅
```

---

## ⚙️ 3. Internal Working

### Permission Check Flow

```sql
-- When user executes:
SELECT * FROM customers;

-- Database checks:
-- 1. Authentication: Valid user?
-- 2. Connection: Allowed from this IP/host?
-- 3. Database access: User has USE privilege?
-- 4. Table access: User has SELECT on customers?
-- 5. Column access: (If column-level security)
-- 6. Row access: (If row-level security)
-- 
-- If all pass → Execute query
-- If any fail → ERROR 1142: Access denied
```

### Permission Storage

```sql
-- MySQL stores permissions in system tables:
mysql.user        -- Global privileges
mysql.db          -- Database-level privileges
mysql.tables_priv -- Table-level privileges
mysql.columns_priv -- Column-level privileges

-- PostgreSQL uses:
pg_authid         -- Users/roles
pg_auth_members   -- Role membership
pg_class          -- Table ACLs
pg_proc           -- Function ACLs
```

---

## ✅ 4. Best Practices

### Principle of Least Privilege

```sql
-- ✅ Application user: Only what's needed
CREATE USER 'app_prod'@'10.0.1.%' IDENTIFIED BY 'strong_password';
GRANT SELECT, INSERT, UPDATE ON app_db.users TO 'app_prod'@'10.0.1.%';
GRANT SELECT, INSERT, UPDATE ON app_db.orders TO 'app_prod'@'10.0.1.%';
-- No DELETE (prevents data loss)
-- No DROP (prevents accidental table deletion)
-- No ALL PRIVILEGES

-- ✅ Read-only analyst
CREATE USER 'analyst'@'%' IDENTIFIED BY 'password';
GRANT SELECT ON analytics_db.* TO 'analyst'@'%';
-- Can only read, cannot modify

-- ✅ Admin (use sparingly)
CREATE USER 'dba_admin'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'dba_admin'@'localhost' WITH GRANT OPTION;
-- Only from localhost (not from internet)
```

### Use Roles (Not Individual Users)

```sql
-- ❌ Bad: Grant to each user (hard to maintain)
GRANT SELECT ON app_db.* TO 'alice'@'%';
GRANT SELECT ON app_db.* TO 'bob'@'%';
GRANT SELECT ON app_db.* TO 'charlie'@'%';
-- If permissions change, must update 100+ users ❌

-- ✅ Good: Use roles
CREATE ROLE developer_role;
GRANT SELECT ON app_db.* TO developer_role;

-- Assign role to users
GRANT developer_role TO 'alice'@'%';
GRANT developer_role TO 'bob'@'%';
GRANT developer_role TO 'charlie'@'%';
-- Change once, affects all users ✅
```

### Restrict by IP/Host

```sql
-- ✅ Production app: Only from app servers
CREATE USER 'app_prod'@'10.0.1.%' IDENTIFIED BY 'password';
-- Only connections from 10.0.1.* network

-- ✅ Admin: Only from VPN
CREATE USER 'admin'@'172.16.0.%' IDENTIFIED BY 'password';
-- Only from VPN network

-- ❌ Dangerous: Allow from anywhere
CREATE USER 'admin'@'%' IDENTIFIED BY 'password';  -- ❌ Security risk
```

### Audit and Rotate Credentials

```sql
-- ✅ Check who has access
SELECT user, host, authentication_string 
FROM mysql.user;

-- ✅ Revoke unused accounts
REVOKE ALL PRIVILEGES ON *.* FROM 'old_app'@'%';
DROP USER 'old_app'@'%';

-- ✅ Rotate passwords regularly
ALTER USER 'app_prod'@'10.0.1.%' IDENTIFIED BY 'new_password';
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Using Root in Production

```sql
-- ❌ Application connects as root
DB_USER=root
DB_PASS=password
-- Root can do ANYTHING: DROP DATABASE, DELETE ALL, etc. ❌

-- ✅ Create dedicated user
CREATE USER 'app_prod'@'10.0.1.%' IDENTIFIED BY 'password';
GRANT SELECT, INSERT, UPDATE ON app_db.* TO 'app_prod'@'10.0.1.%';
```

### Mistake 2: Granting to '%' Wildcard

```sql
-- ❌ Bad: Allow from anywhere
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%';
-- Anyone from anywhere can connect ❌

-- ✅ Good: Restrict by IP
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'10.0.%';
-- Only from private network ✅
```

### Mistake 3: Not Revoking Old Users

```sql
-- ❌ Bad: Old user still has access
-- Employee leaves company, account still active ❌

-- ✅ Good: Revoke and drop
REVOKE ALL PRIVILEGES ON *.* FROM 'ex_employee'@'%';
DROP USER 'ex_employee'@'%';

-- ✅ Better: Audit regularly
SELECT user, host, 
       DATEDIFF(NOW(), password_last_changed) as days_old
FROM mysql.user
WHERE days_old > 90;
```

### Mistake 4: Hardcoding Credentials

```sql
-- ❌ Bad: Hardcoded in code
$db = new PDO('mysql:host=prod.db', 'root', 'password123');
-- Credentials in source code ❌
-- Leaked if code is compromised ❌

-- ✅ Good: Environment variables
$db = new PDO(
    'mysql:host=' . getenv('DB_HOST'),
    getenv('DB_USER'),
    getenv('DB_PASS')
);

-- ✅ Better: Secrets manager (AWS Secrets Manager, Vault)
$credentials = getSecretsManager('prod/db/credentials');
$db = new PDO(..., $credentials['user'], $credentials['password']);
```

### Mistake 5: No Audit Logging

```sql
-- ❌ Bad: No audit trail
-- Who deleted that data? Don't know! ❌

-- ✅ Good: Enable audit logs (MySQL)
SET GLOBAL general_log = 'ON';

-- ✅ Better: Audit plugin (MySQL Enterprise)
INSTALL PLUGIN audit_log SONAME 'audit_log.so';

-- ✅ PostgreSQL: Enable logging
ALTER SYSTEM SET log_statement = 'all';
SELECT pg_reload_conf();
```

---

## 🔐 6. Security Considerations

### Password Security

```sql
-- ✅ Strong password policy
ALTER USER 'user'@'%' IDENTIFIED BY 'Str0ng!P@ssw0rd#123';

-- ✅ Password expiration (MySQL)
ALTER USER 'user'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;

-- ✅ Require SSL
ALTER USER 'user'@'%' REQUIRE SSL;

-- ✅ Lock after failed attempts
ALTER USER 'user'@'%' FAILED_LOGIN_ATTEMPTS 3 PASSWORD_LOCK_TIME 1;
```

### Encryption

```sql
-- ✅ Require encrypted connections
GRANT ALL ON app_db.* TO 'user'@'%' REQUIRE SSL;

-- ✅ PostgreSQL: Force SSL
ALTER SYSTEM SET ssl = on;
-- Then in pg_hba.conf:
-- hostssl  all  all  0.0.0.0/0  md5
```

### Row-Level Security (PostgreSQL)

```sql
-- ✅ Users see only their own data
CREATE POLICY tenant_isolation ON orders
    FOR ALL
    TO app_user
    USING (tenant_id = current_setting('app.current_tenant')::int);

ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Application sets:
SET app.current_tenant = 123;
SELECT * FROM orders;  -- Only sees tenant 123's orders ✅
```

---

## 🚀 7. Performance Optimization

### Cached Permissions

```sql
-- MySQL caches permissions
-- To see effective permissions:
SHOW GRANTS FOR 'user'@'host';

-- After GRANT/REVOKE, flush cache:
FLUSH PRIVILEGES;

-- Or restart connection (automatic cache refresh)
```

### Minimize Permission Checks

```sql
-- ✅ Grant at database level (faster)
GRANT SELECT ON app_db.* TO 'user'@'%';

-- vs

-- ❌ Grant per table (slower, many permission checks)
GRANT SELECT ON app_db.table1 TO 'user'@'%';
GRANT SELECT ON app_db.table2 TO 'user'@'%';
-- ... 100 more tables
```

---

## 🧪 8. Examples

### Application User Setup

```sql
-- Production read-write application
CREATE USER 'app_prod'@'10.0.1.%' 
    IDENTIFIED BY 'strong_password'
    REQUIRE SSL
    WITH MAX_CONNECTIONS_PER_HOUR 10000
    MAX_QUERIES_PER_HOUR 1000000;

GRANT SELECT, INSERT, UPDATE ON app_db.users TO 'app_prod'@'10.0.1.%';
GRANT SELECT, INSERT, UPDATE ON app_db.orders TO 'app_prod'@'10.0.1.%';
GRANT SELECT, INSERT ON app_db.logs TO 'app_prod'@'10.0.1.%';
```

### Read-Only Replica User

```sql
-- Read replica for analytics
CREATE USER 'analytics'@'%' 
    IDENTIFIED BY 'password'
    REQUIRE SSL;

GRANT SELECT ON analytics_db.* TO 'analytics'@'%';

-- Verify
SHOW GRANTS FOR 'analytics'@'%';
```

### Role-Based Access

```sql
-- Create roles
CREATE ROLE developer_role;
CREATE ROLE analyst_role;
CREATE ROLE admin_role;

-- Grant permissions to roles
GRANT SELECT, INSERT, UPDATE ON dev_db.* TO developer_role;
GRANT SELECT ON prod_db.* TO analyst_role;
GRANT ALL PRIVILEGES ON *.* TO admin_role;

-- Assign roles to users
CREATE USER 'alice'@'%' IDENTIFIED BY 'password';
GRANT developer_role TO 'alice'@'%';

CREATE USER 'bob'@'%' IDENTIFIED BY 'password';
GRANT analyst_role TO 'bob'@'%';

-- User must activate role (MySQL 8.0+)
-- By default:
SET ROLE developer_role;

-- Or auto-activate:
ALTER USER 'alice'@'%' DEFAULT ROLE developer_role;
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: AWS RDS - IAM Authentication

```sql
-- Instead of passwords, use IAM tokens
-- 1. Enable IAM authentication on RDS
-- 2. Create user with IAM plugin
CREATE USER 'app_user' IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';
GRANT SELECT, INSERT, UPDATE ON app_db.* TO 'app_user';

-- 3. Application gets temporary token from AWS
token = get_auth_token(region='us-east-1', db_hostname='prod.xxx.rds.amazonaws.com')

-- 4. Connect with token (valid 15 minutes)
connection = mysql.connect(
    host='prod.xxx.rds.amazonaws.com',
    user='app_user',
    password=token,
    ssl={'ssl_mode': 'REQUIRED'}
)
-- No long-lived passwords! ✅
```

### Case Study 2: Multi-Tenant SaaS - Row-Level Security

```sql
-- Shopify-style multi-tenancy
CREATE POLICY tenant_isolation ON products
    FOR ALL
    TO app_user
    USING (shop_id = current_setting('app.current_shop_id')::bigint);

ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
-- ... all tables

-- Application sets tenant context
SET app.current_shop_id = 12345;

-- All queries automatically filtered
SELECT * FROM products;  -- Only shop 12345's products
INSERT INTO orders ...;  -- shop_id auto-set to 12345
```

### Case Study 3: Stripe - Database Credentials Rotation

```sql
-- Rotate every 30 days with zero downtime
-- 1. Create new user
CREATE USER 'app_prod_v2'@'10.0.%' IDENTIFIED BY 'new_password';
GRANT SELECT, INSERT, UPDATE ON payments_db.* TO 'app_prod_v2'@'10.0.%';

-- 2. Deploy new credentials to 50% of app servers
-- 3. Monitor for errors
-- 4. Deploy to remaining 50%
-- 5. Revoke old user
REVOKE ALL PRIVILEGES ON *.* FROM 'app_prod_v1'@'10.0.%';
DROP USER 'app_prod_v1'@'10.0.%';

-- Automated via HashiCorp Vault or AWS Secrets Manager
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What's the difference between GRANT and REVOKE?

**Answer:**
- **GRANT**: Gives permissions to users/roles
- **REVOKE**: Removes permissions

```sql
-- Grant
GRANT SELECT ON users TO 'analyst'@'%';

-- Revoke
REVOKE SELECT ON users FROM 'analyst'@'%';
```

---

### Q2: What is the principle of least privilege?

**Answer:** Give users **only the minimum permissions** needed to do their job.

```sql
-- ❌ Bad: Too much access
GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'%';

-- ✅ Good: Only what's needed
GRANT SELECT, INSERT, UPDATE ON app_db.orders TO 'app_user'@'%';
-- No DELETE, no DROP, no other databases
```

---

### Q3: How do you restrict database access by IP address?

**Answer:** Use the host part in user definition.

```sql
-- Only from specific IP
CREATE USER 'admin'@'192.168.1.100' IDENTIFIED BY 'password';

-- Only from subnet
CREATE USER 'app'@'10.0.%' IDENTIFIED BY 'password';

-- Only from localhost
CREATE USER 'backup'@'localhost' IDENTIFIED BY 'password';

-- From anywhere (dangerous!)
CREATE USER 'public'@'%' IDENTIFIED BY 'password';  -- ❌
```

---

### Q4: What are roles and why use them?

**Answer:** Roles are **named groups of permissions**. Benefits:
- **Easier management**: Change once, affects all users
- **Consistency**: All developers get same permissions
- **Auditing**: Clear permission structure

```sql
-- Create role
CREATE ROLE developer_role;
GRANT SELECT, INSERT, UPDATE ON app_db.* TO developer_role;

-- Assign to users
GRANT developer_role TO 'alice'@'%';
GRANT developer_role TO 'bob'@'%';

-- Change once, affects both alice and bob
GRANT DELETE ON app_db.logs TO developer_role;
```

---

### Q5: How do you audit who has access to a database?

**Answer:**

```sql
-- MySQL: Check all users
SELECT user, host, authentication_string 
FROM mysql.user;

-- Check specific user's permissions
SHOW GRANTS FOR 'app_user'@'10.0.%';

-- PostgreSQL: List users
\du

-- Check table permissions
SELECT grantee, privilege_type 
FROM information_schema.table_privileges 
WHERE table_name = 'orders';
```

---

## 🧩 11. Design Patterns

### Pattern 1: Application User Pattern

```sql
-- Separate users for different purposes
CREATE USER 'app_prod_rw'@'10.0.%' IDENTIFIED BY 'pwd';  -- Read-write
CREATE USER 'app_prod_ro'@'10.0.%' IDENTIFIED BY 'pwd';  -- Read-only
CREATE USER 'app_migration'@'10.0.%' IDENTIFIED BY 'pwd';  -- Schema changes

GRANT SELECT, INSERT, UPDATE ON app_db.* TO 'app_prod_rw'@'10.0.%';
GRANT SELECT ON app_db.* TO 'app_prod_ro'@'10.0.%';
GRANT ALL PRIVILEGES ON app_db.* TO 'app_migration'@'10.0.%';
```

### Pattern 2: Service Account Pattern

```sql
-- Each microservice has its own user
CREATE USER 'auth_service'@'10.0.%' IDENTIFIED BY 'pwd';
CREATE USER 'payment_service'@'10.0.%' IDENTIFIED BY 'pwd';
CREATE USER 'notification_service'@'10.0.%' IDENTIFIED BY 'pwd';

-- Auth service: Only users table
GRANT SELECT, INSERT, UPDATE ON app_db.users TO 'auth_service'@'10.0.%';

-- Payment service: Only payments table
GRANT SELECT, INSERT, UPDATE ON app_db.payments TO 'payment_service'@'10.0.%';
```

### Pattern 3: Time-Limited Access

```sql
-- Grant temporary access (PostgreSQL)
CREATE USER 'contractor'@'%' IDENTIFIED BY 'pwd' VALID UNTIL '2026-03-01';

-- MySQL: Use cron to revoke
-- (No native expiration)
```

---

## 📚 Summary

### Key Takeaways

1. **Principle of least privilege**: Give only needed permissions
2. **Use roles**: Easier to manage than individual grants
3. **Restrict by IP/host**: Use `'user'@'10.0.%'` not `'user'@'%'`
4. **Audit regularly**: Check who has access, revoke unused accounts
5. **No root in production**: Create dedicated application users
6. **Password security**: Strong passwords, SSL, rotation
7. **Avoid wildcards**: `@'%'` is dangerous, restrict networks
8. **Enable audit logs**: Track who does what
9. **Row-level security**: For multi-tenant applications
10. **Secrets management**: Use Vault/AWS Secrets Manager, not hardcoded passwords

---

**Next:** [04_TCL_Commands.md](04_TCL_Commands.md) - Transaction control  
**Related:** [../28_Database_Security/](../28_Database_Security/) - Advanced security

---

*Last Updated: February 19, 2026*
