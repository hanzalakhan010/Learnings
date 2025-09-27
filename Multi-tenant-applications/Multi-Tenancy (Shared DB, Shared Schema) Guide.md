## Tenant Identification

- **Tenant Key Strategy**: `tenant_id` as [UUID]
- **Propagation Path**:
  - Request headers (JWT claims / API key)
  - Session context (Flask session)
  - Row-level filters (ORM / DB policies)

---

## Schema Design

- **Tables Impacted**:
  - **Core entities**: Every table associated with tenant should have a column `tenant_id` (foreign key).
  - **Reference tables**: Tenant table should store tenant-specific details and use `tenant_id` as the primary key.
  - **Logging / Audit**: Not implemented yet.

---

## Security & Isolation

- **Row Level Security (RLS)**:

  - Enable RLS per tenant-associated table.
  - Example for `courses` table:

    ```sql
    ALTER TABLE courses ENABLE ROW LEVEL SECURITY;

    ALTER POLICY courses_select_policy
    ON courses
    USING (tenant_id = current_setting('app.tenant_id')::uuid);
    ```

  - ⚠️ Avoid using `postgres` or `admin` role, as they bypass RLS policies.

---

## Application Layer

- **Tenant Resolution**:

  - Extract `tenant_id` from JWT middleware or session.
  - Inject tenant context into SQLAlchemy:

    ```python
    @app.before_request
    def extract_tenant_from_jwt():
        token = request.cookies.get("token", "")
        if token:
            try:
                status, payload = jwt.verifyJWT(token)
                if status:
                    g.tenant_id = payload.get("tenant_id")
            except Exception as error:
                abort(401, error)

    # ✅ 2. Set tenant context in PostgreSQL session
    @event.listens_for(db.session, "after_begin")
    def set_tenant_id(session, transaction, connection):
        if hasattr(g, "tenant_id"):
            connection.exec_driver_sql(
                "SET LOCAL app.tenant_id = %s", (g.tenant_id,)
            )
        else:
            # For cases like instructor login where tenant_id is unknown
            connection.exec_driver_sql(
                "SET LOCAL app.tenant_id = '00000000-0000-0000-0000-000000000000'"
            )

    @event.listens_for(db.session, "before_flush")
    def inject_tenant_id(session, flush_context, instances):
        for obj in session.new:  # only for new objects
            if hasattr(obj, "tenant_id") and getattr(obj, "tenant_id") is None:
                if hasattr(g, "tenant_id"):
                    setattr(obj, "tenant_id", g.tenant_id)
    ```

  - **Default/Fallback Tenant**: Use UUID `00000000-0000-0000-0000-000000000000`.
    \*\*\*example alembic migration script for applying poilicies on mutiple tables

  ```python


  from alembic import op
  import sqlalchemy as sa

  # revision identifiers, used by Alembic.

  tables = [
      "applicants",
      "applications",
      "course_enrollments",
      "course_sections",
      "course_sessions",
      "courses",
      "exams",
      "instructors",
      "students",
  ]

  def upgrade():
      for table in tables:
      op.execute(f"ALTER TABLE {table} ENABLE ROW LEVEL SECURITY;")
      for table in tables:
      op.execute(
      f"""
      CREATE POLICY {table}\_select_policy ON {table}
      FOR SELECT USING (tenant_id = current_setting('app.tenant_id')::uuid);
      """
      )
      op.execute(
      f"""
      CREATE POLICY {table}\_insert_policy ON {table}
      FOR INSERT WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);
      """
      )
      op.execute(
      f"""
      CREATE POLICY {table}\_update_policy ON {table}
      FOR UPDATE USING (tenant_id = current_setting('app.tenant_id')::uuid)
      WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);
        """
        )

  def downgrade():
      for table in tables:
      op.execute(f"DROP POLICY IF EXISTS {table}\_select_policy ON {table};")
      op.execute(f"DROP POLICY IF EXISTS {table}\_insert_policy ON {table};")
      op.execute(f"DROP POLICY IF EXISTS {table}\_update_policy ON {table};")
      op.execute(f"ALTER TABLE {table} DISABLE ROW LEVEL SECURITY;")````
  ```
---

## Performance & Scaling

* **Indexing Strategy**:

* Use composite indexes `(tenant_id, entity_id)`.
* Partitioning strategy to be explored.

* **Query Optimization**:

* [ ] Monitor cross-tenant queries.
* [ ] Guard against full-table scans.

---

## 🔄 Migration Process

* **Steps**:

1. Add `tenant_id` column.
2. Backfill existing tenant data.
3. Refactor queries with tenant scope.
4. Enable RLS policies.
5. Verify isolation & correctness.

* **Testing**:

* [ ] Unit tests with tenant separation.
* [ ] Load testing with multiple tenants.

---

## ✅ Migration Checklist

* [ ] Tenant key column added.
* [ ] ORM queries scoped.
* [ ] Row-level security enforced.
* [ ] Tenant context resolution added.
* [ ] Logging & auditing tenant-aware.
* [ ] Migration scripts created.
* [ ] Tests passed.

---

## Role Setup (Client User without Delete Permission)

```sql
CREATE ROLE lms_app WITH LOGIN PASSWORD 'password';
GRANT CONNECT ON DATABASE lms_db TO lms_app;
GRANT USAGE ON SCHEMA public TO lms_app;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO lms_app;
GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA public TO lms_app;
````

---

## Notes

(Add special considerations, edge cases, or decisions here.)
