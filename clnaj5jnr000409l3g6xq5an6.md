---
title: "Easy Deployment and Rollback of PostgreSQL Functions with Supabase"
seoTitle: "Streamline PostgreSQL Functions with Supabase"
seoDescription: "Discover rapid function management in PostgreSQL with Supabase. Simplify your workflow for quick prototyping. Dive in now!"
datePublished: Tue Oct 03 2023 16:24:32 GMT+0000 (Coordinated Universal Time)
cuid: clnaj5jnr000409l3g6xq5an6
slug: streamlining-postgresql-function-management-with-supabase
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1696350043975/975d48e5-c57b-4163-a59f-368f0e03ac5c.png
tags: postgresql, web-development, databases, sql

---

In the realm of database management, version control, and deployment are crucial. Efficiently deploying and managing database functions is vital for maintaining the integrity of your data-driven applications. While database migrations, as detailed in [Supabase's migration guide](https://supabase.com/docs/reference/cli/supabase-migration), are ideal for long-term projects, there are scenarios, such as prototyping and rapid development, where you need more flexibility.

In this blog post, we'll explore an approach tailored for quick prototyping and agile development—how to easily deploy and rollback PostgreSQL functions using [Supabase](https://supabase.com). Supabase, a powerful open-source alternative to traditional database management systems, simplifies the process of deploying and managing functions in these scenarios.

If you're working on more complex workflows or long-term projects, we highly recommend referring to [Supabase's migration guide](https://supabase.com/docs/reference/cli/supabase-migration) for optimal version control and deployment practices.

## PostgreSQL and Supabase in Modern Web Applications

PostgreSQL, a robust open-source relational database management system (RDBMS), has gained popularity in web development due to its reliability, extensibility, and support for complex data types.

Supabase, an open-source platform, offers various tools and services for modern web applications. It leverages PostgreSQL as its core database engine and provides a user-friendly interface for managing data, authentication, and more.

We'll explore how Supabase complements PostgreSQL by simplifying function deployment and rollback. You can refer to the [PostgreSQL Documentation](https://www.postgresql.org/docs/) to learn more about PostgreSQL.

## Tracking Function History in PostgreSQL

When managing a PostgreSQL database, it's essential to track changes made to functions over time. This historical record allows you to review, audit, and revert to previous versions if needed.

To facilitate this, we'll create an `archive.function_history` table that stores crucial information about each function, including its name, arguments, return type, source code, and language settings.

Here's the SQL code for creating this table:

```sql
CREATE SCHEMA archive;

CREATE TABLE archive.function_history (
  schema_name text,
  function_name text,
  args text,
  return_type text,
  source_code text,
  lang_settings text,
  updated_at timestamp default now()
);
```

## Saving Function History

### The `archive.save_function_history` Function

To automate recording function changes, we'll create a PostgreSQL function called `archive.save_function_history`. This function takes parameters such as the function name, arguments, return type, source code, schema name, and language settings.

Here's the SQL code for creating the `archive.save_function_history` function:

```sql
CREATE OR REPLACE 
FUNCTION archive.save_function_history(
  function_name text,
  args text,
  return_type text,
  source_code text,
  schema_name text default 'public',
  lang_settings text default 'plpgsql'
) RETURNS void 
SET search_path = public, archive
SECURITY DEFINER
AS
$$
BEGIN
  INSERT INTO archive.function_history (
        schema_name, 
        function_name, 
        args, 
        return_type, 
        source_code, 
        lang_settings)
  VALUES (schema_name, function_name, args, return_type, source_code, lang_settings);
END;
$$
LANGUAGE plpgsql;
-- Protecting the function:
REVOKE EXECUTE ON FUNCTION 
archive.save_function_history FROM public;

REVOKE EXECUTE ON FUNCTION 
archive.save_function_history FROM anon, authenticated;
```

This function allows us to easily store a snapshot of a function each time it's modified.

## Deploying Functions from Source

### The `create_function_from_source` Function

Managing functions often involves deploying them from source code. PostgreSQL requires specific syntax for function creation, and Supabase simplifies this with the `create_function_from_source` function.

```sql
CREATE OR REPLACE FUNCTION 
create_function_from_source(
  function_text text,
  schema_name text default 'public'
) RETURNS text 
SECURITY DEFINER
AS $$
DECLARE
  function_name text;
  argument_types text;
  return_type text;
  function_source text;
  lang_settings text;
BEGIN
  -- Execute the function text to create the function
  EXECUTE function_text;

  -- Extract function name from function text
  SELECT (regexp_matches(function_text, 'create (or replace )?function (public\.)?(\w+)', 'i'))[3]
  INTO function_name;

  -- Get function details from the system catalog
  SELECT pg_get_function_result(p.oid), 
                pg_get_function_arguments(p.oid), p.prosrc, l.lanname
  INTO return_type, argument_types, function_source, lang_settings
  FROM pg_proc p
  JOIN pg_namespace n ON n.oid = p.pronamespace
  JOIN pg_language l ON l.oid = p.prolang
  WHERE n.nspname = schema_name AND p.proname = function_name;

  -- Save function history
  PERFORM archive.save_function_history(function_name, argument_types, return_type, function_text, schema_name, lang_settings);
  
  RETURN 'Function created successfully.';
EXCEPTION
  WHEN others THEN
    RAISE EXCEPTION 'Error creating function: %', sqlerrm;
END;
$$ LANGUAGE plpgsql;
-- Protecting the function:
REVOKE EXECUTE ON FUNCTION 
create_function_from_source FROM public;

REVOKE EXECUTE ON FUNCTION 
create_function_from_source FROM anon, authenticated;
```

This function takes the function's SQL source code and schema name as parameters, creating the function within the database. It's a powerful tool for dynamic function creation.

Here's an example of deploying a function using `create_function_from_source`:

```sql
SELECT create_function_from_source(
$$
-- Note that you can just paste the function below:
CREATE OR REPLACE FUNCTION public.convert_to_uuid(input_value text)
 RETURNS uuid
AS $function$
DECLARE
  hash_hex text;
BEGIN
  -- Return null if input_value is null or an empty string
  IF input_value IS NULL OR NULLIF(input_value, '') IS NULL THEN
    RETURN NULL;
  END IF;
  hash_hex := substring(encode(digest(input_value::bytea, 'sha512'), 'hex'), 1, 36);
  RETURN (left(hash_hex, 8) || '-' || right(hash_hex, 4) || '-4' || right(hash_hex, 3) || '-a' || right(hash_hex, 3) || '-' || right(hash_hex, 12))::uuid;
END;
$function$
LANGUAGE plpgsql
IMMUTABLE
SECURITY DEFINER;
-- End of the function above
$$
);
```

## Rolling Back Functions

Rolling back functions is as crucial as deploying them. Mistakes happen, and being able to revert to a previous version can save valuable time and prevent data corruption.

The `rollback_function` function comes to the rescue. It retrieves the most recent function version from the `archive.function_history` table and executes it. If no previous version exists, it gracefully handles the situation.

Here's the SQL code for creating and using the `rollback_function`:

```sql
CREATE OR REPLACE FUNCTION rollback_function(
  func_name text,
  schema_n text default 'public'
) RETURNS text 
SECURITY DEFINER
AS $$
DECLARE
  function_text text;
BEGIN
  -- Get the most recent function version from the function_history table
  SELECT source_code
  INTO function_text
  FROM archive.function_history
  WHERE function_name = func_name AND schema_name = schema_n
  ORDER BY updated_at DESC
  LIMIT 1;

  -- If no previous version is found, raise an error
  IF function_text IS NULL THEN
    RAISE EXCEPTION 'No previous version of function % found.', func_name;
  END IF;

  -- Add 'or replace' to the function text if it's not already there (case-insensitive search and replace)
  IF NOT function_text ~* 'or replace' THEN
    function_text := regexp_replace(function_text, 'create function', 'create or replace function', 'i');
  END IF;

  -- Execute the function text to create the function
  EXECUTE function_text;

  RETURN 'Function rolled back successfully.';
EXCEPTION
  WHEN others THEN
    RAISE EXCEPTION 'Error rolling back function: %', sqlerrm;
END;
$$ LANGUAGE plpgsql;

-- Protecting the function:
REVOKE EXECUTE ON FUNCTION rollback_function FROM public;
REVOKE EXECUTE ON FUNCTION rollback_function FROM anon, authenticated;

-- Example of rolling back a function
SELECT rollback_function('convert_to_uuid');
```

## Conclusion

In conclusion, efficient management of PostgreSQL functions is crucial for web application development. Supabase, with its integration with PostgreSQL and the tools we've explored, offers a streamlined approach to function deployment and rollback.

Key takeaways from this blog post include the importance of function history tracking, the creation of the `archive.function_history` table, the `archive.save_function_history` function for recording changes, and the convenience of `create_function_from_text` and

`rollback_function` for deployment and rollback.

If you found this article valuable, you might also be interested in exploring related topics:

* [Rate Limiting Supabase Requests with PostgreSQL and pg\_headerkit](https://blog.mansueli.com/rate-limiting-supabase-requests-with-postgresql-and-pgheaderkit): Learn how to implement rate limiting for Supabase requests.
    
* [Comprehensive Guide: Deploying and Debugging Custom Webhooks on Supabase & PostgreSQL](https://blog.mansueli.com/automating-webhooks-supabase-postgresql-guide): Explore techniques for deploying and debugging custom webhooks.
    

We encourage you to explore Supabase and PostgreSQL further to unlock the full potential of efficient database management.

## Additional Resources

For further information and exploration, here are some additional resources:

* [Supabase Documentation](https://supabase.com/docs)
    
* [PostgreSQL Documentation](https://www.postgresql.org/docs/)
    
* [Supabase GitHub Repository](https://github.com/supabase/supabase)
    
* [PostgreSQL Official Site](https://www.postgresql.org/)
    

Feel free to delve deeper into these resources to enhance your understanding of these powerful tools for database management.