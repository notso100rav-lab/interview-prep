# Chapter 6: Flyway & Liquibase Migrations

> **Estimated study time:** 1 day | **Priority:** 🟡 Medium

---

## Key Concepts

- Schema versioning and migration scripts
- Flyway: file naming conventions, checksums, repair
- Liquibase: changelogs, changesets, contexts
- Rollback strategies
- Baseline migrations for existing databases
- CI/CD integration
- Zero-downtime schema changes

---

## Questions & Answers

---

### Q1 — 🟢 What is database migration tooling and why do you need it?

<details><summary>Click to reveal answer</summary>

Database migration tools manage schema changes as versioned, repeatable scripts that are tracked in the database itself. They solve the problem of keeping the database schema in sync across environments (dev, staging, prod) and across developers.

**Problems without migration tooling:**
- "Works on my machine" — schema differs between developer databases
- Manual SQL scripts applied inconsistently across environments
- No audit trail of what changed and when
- Risky deployments when schema and application code are out of sync

**How it works:**
1. You write SQL (or XML/YAML) migration scripts
2. The tool maintains a metadata table in the DB (e.g. `flyway_schema_history`)
3. On app startup, the tool checks which migrations have already run
4. Applies any unapplied migrations in order

**Flyway vs Liquibase:**

| Feature | Flyway | Liquibase |
|---------|--------|-----------|
| Script format | SQL (primary) | XML, YAML, JSON, SQL |
| Rollback | Manual undo scripts | Built-in rollback |
| Checksum validation | Yes | Yes |
| Enterprise features | Pro license | Pro license |
| Learning curve | Lower | Higher |
| Community | Large | Large |

</details>

---

### Q2 — 🟢 How does Flyway work and what is its naming convention?

<details><summary>Click to reveal answer</summary>

Flyway scans a configurable location for migration scripts and applies unapplied ones in version order.

**File naming convention:**
```
V{version}__{description}.sql
^           ^^
|           Double underscore
Version number (can use dots: V1.2.3)

Examples:
V1__create_users_table.sql
V2__add_email_index.sql
V3.1__add_phone_column.sql
V20240315_1__add_audit_columns.sql
```

**Types of scripts:**
- `V` — **Versioned**: applied once, checksum validated
- `U` — **Undo**: paired undo script for Flyway Teams
- `R` — **Repeatable**: reapplied whenever their checksum changes (e.g. views, functions)

**Spring Boot setup:**
```xml
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-core</artifactId>
</dependency>
<!-- For PostgreSQL 10+ -->
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false   # true for existing DBs
    baseline-version: "0"
    out-of-order: false          # reject migrations with lower versions than applied
    validate-on-migrate: true    # verify checksums before applying
```

```
src/main/resources/db/migration/
├── V1__create_schema.sql
├── V2__seed_reference_data.sql
├── V3__add_order_status_enum.sql
└── R__create_report_views.sql     # repeatable
```

**`flyway_schema_history` table:**
```sql
SELECT version, description, type, installed_on, success
FROM flyway_schema_history
ORDER BY installed_rank;
```

</details>

---

### Q3 — 🟡 What happens if you modify a Flyway migration that has already been applied?

<details><summary>Click to reveal answer</summary>

Flyway stores a **checksum** (CRC32) of each applied migration. If you modify a migration file that's already been applied:

```
ERROR: Validate failed:
Migration checksum mismatch for migration version 3:
→ Applied: 1234567890
→ Resolved: 9876543210
```

**What NOT to do:**
- Never modify an applied Flyway migration. Treat applied migrations as immutable.

**What to do instead:**
- Create a new versioned migration to correct the data or schema.

**When you need to fix a checksum (dev environments only):**
```bash
# Repair: update the checksum in the metadata table to match the current file
./mvnw flyway:repair

# Or programmatically
@Autowired
private Flyway flyway;

public void repairDev() {
    flyway.repair();
}
```

**In production:** `repair` should only be used if the migration itself ran successfully but the metadata is incorrect (e.g. file was reformatted without content change). Never use `repair` to cover up logic changes.

</details>

---

### Q4 — 🟡 How do you handle baseline migrations for an existing database?

<details><summary>Click to reveal answer</summary>

When you introduce Flyway to an existing database that already has a schema, you can't just apply V1 (it already exists). You **baseline** the existing schema:

```yaml
# application.yml
spring:
  flyway:
    baseline-on-migrate: true
    baseline-version: "1"        # treat everything up to v1 as already applied
    baseline-description: "Initial baseline from existing schema"
```

**Process:**
1. Export current schema as `V1__baseline.sql` (do NOT include data, just DDL)
2. Set `baseline-on-migrate: true` and `baseline-version: "1"`
3. On first run, Flyway creates `flyway_schema_history` and marks V1 as already applied (baseline)
4. All future migrations start at V2

```bash
# Export existing schema from PostgreSQL
pg_dump --schema-only --no-owner --no-acl \
  -h localhost -U myuser mydb > src/main/resources/db/migration/V1__baseline.sql
```

**After baselining:**
```yaml
# Disable baseline-on-migrate for safety (don't accidentally re-baseline)
spring:
  flyway:
    baseline-on-migrate: false
```

```java
// Programmatic baselining
@Bean
public Flyway flyway(DataSource dataSource) {
    return Flyway.configure()
        .dataSource(dataSource)
        .baselineOnMigrate(isFirstDeploy())
        .baselineVersion("1")
        .load();
}
```

</details>

---

### Q5 — 🟡 How does Liquibase differ from Flyway in its changelog format?

<details><summary>Click to reveal answer</summary>

Liquibase uses a **changelog** file that references **changesets**. Each changeset is uniquely identified by `id + author + file`, not by version number.

**XML changelog:**
```xml
<!-- db/changelog/db.changelog-master.xml -->
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.24.xsd">

    <include file="db/changelog/001-create-users.xml"/>
    <include file="db/changelog/002-create-orders.xml"/>
    <include file="db/changelog/003-add-indexes.xml"/>
</databaseChangeLog>

<!-- db/changelog/001-create-users.xml -->
<databaseChangeLog>
    <changeSet id="001" author="alice">
        <createTable tableName="users">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true"/>
            </column>
            <column name="email" type="VARCHAR(255)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="created_at" type="TIMESTAMP" defaultValueComputed="NOW()"/>
        </createTable>

        <rollback>
            <dropTable tableName="users"/>
        </rollback>
    </changeSet>

    <changeSet id="002" author="alice">
        <addColumn tableName="users">
            <column name="phone" type="VARCHAR(20)"/>
        </addColumn>
        <rollback>
            <dropColumn tableName="users" columnName="phone"/>
        </rollback>
    </changeSet>
</databaseChangeLog>
```

**YAML changelog:**
```yaml
# db/changelog/003-add-indexes.yaml
databaseChangeLog:
  - changeSet:
      id: "003"
      author: "bob"
      changes:
        - createIndex:
            indexName: idx_users_email
            tableName: users
            columns:
              - column:
                  name: email
      rollback:
        - dropIndex:
            indexName: idx_users_email
            tableName: users
```

**Spring Boot setup:**
```xml
<dependency>
  <groupId>org.liquibase</groupId>
  <artifactId>liquibase-core</artifactId>
</dependency>
```

```yaml
spring:
  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.xml
    contexts: dev,prod    # run changesets tagged for these contexts
    labels: "!test"       # exclude test-only changesets
```

**Liquibase contexts** allow environment-specific changesets:
```xml
<changeSet id="seed-dev-data" author="alice" context="dev">
    <insert tableName="products">
        <column name="name" value="Test Product"/>
        <column name="price" value="9.99"/>
    </insert>
</changeSet>
```

</details>

---

### Q6 — 🔴 How do you write zero-downtime schema migrations for a Java application?

<details><summary>Click to reveal answer</summary>

Zero-downtime migrations require the schema to be compatible with BOTH the old and new application version simultaneously, since during a rolling deployment, old and new code run against the same DB.

**Expand-Contract pattern:**

**Phase 1 — Expand (backward compatible):**
```sql
-- V5__expand_add_full_name.sql
-- Old code uses first_name + last_name
-- New code will use full_name
-- Add new column as nullable (old code doesn't write it)
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);
```

**Phase 2 — Migrate data (can run online):**
```sql
-- V6__migrate_full_name_data.sql
-- Backfill data; run in batches to avoid table lock
UPDATE users
SET full_name = first_name || ' ' || last_name
WHERE full_name IS NULL
  AND id BETWEEN 1 AND 10000;
-- Repeat with increasing ID ranges or use a background job
```

**Phase 3 — Contract (after old code is fully removed):**
```sql
-- V7__contract_remove_old_columns.sql
-- Only run this after ALL app instances run the new code
ALTER TABLE users
    ALTER COLUMN full_name SET NOT NULL,
    DROP COLUMN first_name,
    DROP COLUMN last_name;
```

**Other zero-downtime patterns:**

```sql
-- Adding a column: always nullable first
ALTER TABLE orders ADD COLUMN tracking_number VARCHAR(100);
-- Later: add NOT NULL constraint once all rows are populated
-- (in PostgreSQL 12+, if default is set, the NOT NULL can be added without rewrite)

-- Adding an index: use CONCURRENTLY to avoid table lock
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);

-- Renaming a column (via new column + trigger):
-- 1. Add new_column
-- 2. Add trigger: INSERT/UPDATE to new_column mirrors old_column
-- 3. Backfill
-- 4. Deploy new app code that uses new_column
-- 5. Drop old_column and trigger
```

**CI/CD integration with health checks:**
```yaml
# application.yml — fail fast if migration fails
spring:
  flyway:
    validate-on-migrate: true
  jpa:
    hibernate:
      ddl-auto: validate  # Hibernate validates schema against entities (don't auto-update!)
```

```java
// Make migration failure visible in health check
@Component
public class MigrationHealthIndicator implements HealthIndicator {

    @Autowired
    private Flyway flyway;

    @Override
    public Health health() {
        MigrationInfo[] pending = flyway.info().pending();
        if (pending.length > 0) {
            return Health.down()
                .withDetail("pendingMigrations", pending.length)
                .build();
        }
        return Health.up()
            .withDetail("appliedMigrations", flyway.info().applied().length)
            .build();
    }
}
```

</details>
