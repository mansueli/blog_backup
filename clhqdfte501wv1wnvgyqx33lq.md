---
title: "sudo with Postgres & Supabase"
datePublished: Tue May 16 2023 14:30:39 GMT+0000 (Coordinated Universal Time)
cuid: clhqdfte501wv1wnvgyqx33lq
slug: sudo-with-postgres-supabase
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1684167316528/624ce38b-3982-4336-8cc2-daae0d560898.png
tags: postgresql, postgres, sql, supabase, superuser

---

With the [security patch](https://github.com/orgs/supabase/discussions/9314) from [Supabase](https://supabase.com/) revoking superuser access, many permissions were lost. However, it's worth noting that the Postgres user still retains significant power, and several restrictions do not apply on a per-table basis. This is particularly useful in scenarios where certain actions, such as disabling auto-vacuum during the import of a large dataset, are necessary.

To add tables to the "supabase\_realtime" publication, you can use the following command:

```sql
ALTER PUBLICATION supabase_realtime ADD TABLES IN SCHEMA public;
```

Alternatively, if you want to disable auto-vacuum for a specific table, you can execute the following statement:

```sql
ALTER TABLE my_table SET (autovacuum_enabled = off);
```

You can also loop through tables to disable this for the public schema:  
For example, to disable auto-vacuum for multiple tables in the public schema, you can use the following script.

```pgsql
DO
$$
DECLARE
row record;
BEGIN
FOR row IN SELECT tablename FROM pg_tables AS t
WHERE t.schemaname = 'public'
LOOP
-- disable auto-vacuum
EXECUTE format('ALTER TABLE %I SET (autovacuum_enabled = off);', row.tablename);
END LOOP;
END;
$$;
```

Creating a mock "sudo" function. This is not precisely `sudo` (superuser do), but rather a way to perform an action on all tables. The function `sudo` is defined as follows:

```pgsql
CREATE OR REPLACE FUNCTION sudo(_query text)
 RETURNS text 
 LANGUAGE plpgsql
 SECURITY DEFINER AS 
$$
DECLARE
 row record;
BEGIN
  FOR row IN SELECT tablename FROM pg_tables AS t
  WHERE t.schemaname = 'public'
  LOOP
  -- run query
    EXECUTE format(_query, row.tablename);
  END LOOP;
  RETURN 'success';
END;
$$;
```

If you need to work with multiple schemas, you can use the following function:

```pgsql
CREATE OR REPLACE FUNCTION public.sudo(_schema text, _query text)
 RETURNS text
 LANGUAGE plpgsql
 SECURITY DEFINER AS 
$$
DECLARE
 row record;
BEGIN
  FOR row IN SELECT tablename FROM pg_tables AS t
  WHERE t.schemaname = _schema
  LOOP
    -- run query & Concatenate schema and tablename
    EXECUTE format(_query, _schema || '.' || row.tablename);
  END LOOP;
  RETURN 'success';
END;
$$;
```

### ⚠️ This is a dangerous function, and you should not allow authenticated or anonymous users to run it ⚠️

To revoke access to this function for unauthorized users, you can use the following command:

```pgsql
REVOKE EXECUTE ON FUNCTION sudo FROM anon, authenticated;
```

With these new permissions, only the `postgres` user and the `service_role` key will be able to make calls to the function. If you fail to revoke access as required, someone could potentially drop all your tables. For example, to drop all tables, you can use the `sudo` function as follows:

```pgsql
perform sudo('DROP TABLE %I CASCADE;'::text);
```

# Taking it to the next level

While it is not possible to take advantage of DDL triggers to run certain actions that require superuser access, you can simulate triggers by utilizing a cache table and pg\_cron.

Let's consider an example of enabling Row Level Security (RLS) on new tables using an event trigger. Please note that the following example won't work in hosted Supabase environments due to the requirement of superuser access.

```pgsql
-- Example only. 
-- This will not work in hosted Supabase:
CREATE OR REPLACE FUNCTION enable_rls_on_new_table()
RETURNS event_trigger AS $$
BEGIN
    perform sudo('ALTER TABLE %I ENABLE ROW LEVEL SECURITY;');
END
$$
LANGUAGE plpgsql;

CREATE EVENT TRIGGER
on_create_table ON ddl_command_end
WHEN TAG IN ('CREATE TABLE')
EXECUTE PROCEDURE enable_rls_on_new_table();
```

To simulate event triggers, we can use pg\_cron along with a tracking table in the `event_trigger` schema:Next, we can create the `protect_new_tables` function to check for new tables and invoke the `sudo` function for enabling RLS:

```pgsql
-- Private schema to simulate triggers
CREATE schema event_trigger;
-- Tracking table
CREATE TABLE event_trigger.table_count (
  schema_name text,
  count integer
);
--Populates the tracking table with initial data:
INSERT INTO event_trigger.table_count (schema_name, count)
SELECT schemaname, count(*)
FROM pg_tables
GROUP BY schemaname;
```

Next, we can create the `protect_new_tables` function to check for new tables and invoke the `sudo` function for enabling RLS:

```pgsql
CREATE OR REPLACE FUNCTION protect_new_tables(schema_name text)
RETURNS void
LANGUAGE plpgsql
AS $$
DECLARE
  rec record;
  current_count integer;
  previous_count integer;
BEGIN
  SELECT count INTO current_count FROM pg_tables WHERE schemaname = schema_name;
  SELECT count INTO previous_count FROM table_count WHERE schema_name = schema_name;
  IF current_count > previous_count THEN
    FOR rec IN SELECT tablename FROM pg_tables WHERE schemaname = schema_name
    LOOP
      PERFORM sudo(schema_name, 'ALTER TABLE %I ENABLE ROW LEVEL SECURITY;');
    END LOOP;
    UPDATE table_count SET count = current_count WHERE schema_name = schema_name;
  END IF;
END;
$$;
```

Now, we can use `cron` to schedule calls to this function according to our needs:

```pgsql
-- Check for new tables every 10 minutes
SELECT cron.schedule('check_new_table_public_10min', '*/10 * * * *', $$ SELECT protect_new_tables('public'); $$);
-- Check for new tables every hour
SELECT cron.schedule('check_new_table_public_hourly', '0 * * * *', $$ SELECT protect_new_tables('public'); $$);
```

### Now, let's explore examples of using the `sudo()` function

Creating replication & adding all tables in the public schema to it:

```pgsql
create publication publication_name; 
perform sudo('ALTER PUBLICATION publication_name ADD TABLE %I;'::text);
```

Enabling Realtime for all tables:

```pgsql
SELECT * FROM sudo('ALTER PUBLICATION supabase_realtime ADD TABLE %I;'::text);
```

Disabling auto-vacuum:

```pgsql
SELECT * FROM sudo('ALTER TABLE %I SET (autovacuum_enabled = off);');
```

Enabling RLS for all tables:

```pgsql
SELECT * FROM sudo('ALTER TABLE %I ENABLE ROW LEVEL SECURITY;');
```

## Conclusion

In this blog post, we explored the usage of `sudo` with Postgres and Supabase, enabling powerful actions on tables while maintaining granular control. Despite the security patch from Supabase that revoked superuser access, we discovered that the Postgres user retains significant privileges, making it possible to perform certain operations that would otherwise be restricted. By leveraging the `sudo` function and utilizing schema-specific queries, we demonstrated how to disable auto-vacuum, enable Realtime and [Row Level Security](https://supabase.com/docs/guides/auth/row-level-security) (RLS) for all tables, and even simulate triggers for event-based actions.

We also discussed the importance of maintaining security when using functions like `sudo`. It is crucial to restrict access to authorized users and revoke execution privileges for unauthorized individuals to prevent potential risks and data loss. By following the recommended precautions and properly configuring permissions, we can ensure the safety and integrity of our database environment.

The combination of `sudo`, schema-specific queries, and careful access management, developers and administrators can effectively carry out advanced operations on tables within Postgres and Supabase, expanding the capabilities of their applications while maintaining security and control.