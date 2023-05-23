---
title: "Dynamically Adding Columns with Supabase and Postgres"
datePublished: Tue May 23 2023 14:20:41 GMT+0000 (Coordinated Universal Time)
cuid: cli0d5z7l04p69ynv1qzj2il5
slug: dynamically-adding-columns-with-supabase-and-postgres
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1684339995857/5d802b5d-69c9-48e0-b0cf-9bc9d75fa74a.png
tags: postgresql, javascript, sql, supabase, plpgsql

---

[Supabase](https://supabase.com/) is an open-source Backend-as-a-Service (BaaS) that provides a user-friendly interface and a set of tools for building applications with PostgreSQL. In this blog post, we will explore a custom function that allows you to dynamically add columns to a table in a PostgreSQL database using Supabase. We will delve into the code of the function and provide a step-by-step explanation of how it works.

### Creating the Function

The function we will be discussing is called `add_column`. It is defined in the public schema. Let's break down the function's signature and purpose:

```pgsql
CREATE OR REPLACE FUNCTION public.add_column(
  schema_name_in text DEFAULT 'public',
  table_name_in text,
  column_name_in text,
  type_in text DEFAULT 'text',
  is_array boolean DEFAULT false
) RETURNS text
LANGUAGE plpgsql
SECURITY DEFINER
AS $function$
```

The `add_column` function takes five parameters:

* `schema_name_in`: The name of the schema where the table resides (default: '*public*').
    
* `table_name_in`: The name of the table to which the column should be added.
    
* `column_name_in`: The name of the column to be added.
    
* `type_in`: The data type of the new column (default: '*text*').
    
* `is_array`: A boolean value indicating whether the new column should be an array (default: *false*).
    

### Checking the Type

The function starts by checking if the specified data type exists among the common PostgreSQL types. It does this by executing a query that checks for the existence of the type in the `pg_type` system catalog table:

```pgsql
DECLARE
  _type_exists BOOLEAN;
BEGIN
  SELECT EXISTS (SELECT 1 FROM pg_type WHERE typname = type_in)
  INTO _type_exists;

  IF NOT _type_exists THEN
    RETURN 'Invalid type: ' || type_in;
  END IF;
```

If the type does not exist, the function returns an error message indicating that the type is invalid. Otherwise, it proceeds to add the column.

### Adding the Column

To [add the column](https://supabase.com/docs/guides/database/tables#columns) dynamically, the function utilizes the `EXECUTE` statement with a formatted SQL string. If the `is_array` parameter is set to `true`, the function executes the following statement:

```pgsql
EXECUTE format('ALTER TABLE %I.%I ADD COLUMN %I %I[];', schema_name_in, table_name_in, column_name_in, type_in);
```

This statement uses the `ALTER TABLE` command to add the column specified by `column_name_in` and `type_in[]` (if `is_array` is true) to the table specified by `schema_name_in` and `table_name_in`.

If the `is_array` parameter is `false`, the function executes the following statement instead:

```pgsql
EXECUTE format('ALTER TABLE %I.%I ADD COLUMN %I %I;', schema_name_in, table_name_in, column_name_in, type_in);
```

This statement adds the column without an array type.

### Handling Exceptions

The function includes an exception block to catch any errors that may occur during execution. If an error is encountered, the function returns an error message that includes the specific error details:

```pgsql
EXCEPTION
  WHEN OTHERS THEN
    RETURN 'Error: ' || SQLERRM;
```

### Full function

The full function code provides a reusable custom function that allows you to dynamically add columns to a table in a PostgreSQL database using Supabase. It takes several parameters such as `schema_name_in`, `table_name_in`, `column_name_in`, `type_in`, and `is_array` to define the schema, table, column name, column type, and whether the column should be an array or not.

```pgsql
CREATE OR REPLACE FUNCTION public.add_column(schema_name_in text DEFAULT 'public', table_name_in text, column_name_in text, type_in text DEFAULT 'text', is_array boolean DEFAULT false)
RETURNS text
LANGUAGE plpgsql
SECURITY DEFINER
AS $function$
DECLARE
  _type_exists BOOLEAN;
BEGIN
  -- Check if the type exists among common PostgreSQL types
  SELECT EXISTS (SELECT 1 FROM pg_type WHERE typname = type_in)
  INTO _type_exists;

  IF NOT _type_exists THEN
    RETURN 'Invalid type: ' || type_in;
  END IF;

  -- Add the column to the specified schema and table
  IF is_array THEN
    EXECUTE format('ALTER TABLE %I.%I ADD COLUMN %I %I[];', schema_name_in, table_name_in, column_name_in, type_in);
  ELSE
    EXECUTE format('ALTER TABLE %I.%I ADD COLUMN %I %I;', schema_name_in, table_name_in, column_name_in, type_in);
  END IF;

  RETURN 'DONE';
EXCEPTION
  WHEN OTHERS THEN
    RETURN 'Error: ' || SQLERRM;
END;
$function$
```

### Creating new columns from the client library:

To create new columns from your application using the Supabase client library, you can utilize the [rpc method](https://supabase.com/docs/reference/javascript/rpc). The `rpc` method allows you to invoke a stored procedure or function on the PostgreSQL database. In this case, you call the `add_column` function with the required parameters: `schema_name_in`, `table_name_in`, `column_name_in`, and optionally `type_in` and `is_array`.

By providing the necessary parameters, you can dynamically add a new column to the specified table. The `data` variable will contain the result of the function execution, and the `error` variable will capture any errors that might occur.

This approach gives you flexibility in adding columns on the fly from your application, adapting the database structure as needed.

```javascript
const supabase = createClient(SUPABASE_URL, SUPABASE_KEY, options);  const {data:data , error} = await supabase
.rpc('add_column', {
    schema_name_in: 'public', //optional defaults to public
    table_name_in: 'my_table',//required name of the table
    column_name_in: text,     //required: name for the column
    type_in: 'NUMERIC'        //optional: defaults to text
    is_array: false           //optional: defaults to false
    });
```

### Limiting this function to the `service_role` key:

Normally, it is not desired to allow authenticated and anon users to edit the table. You can limit this with the following query:

```pgsql
REVOKE EXECUTE ON FUNCTION add_column FROM anon, authenticated;
```

Explanation: To restrict the execution of the `add_column` function to only authorized users, you can use the `REVOKE` statement. In this case, the `REVOKE` statement revokes the `EXECUTE` privilege on the `add_column` function from both anonymous (`anon`) and authenticated users (`authenticated`).

Limiting the function execution to the `service_role` key, you ensure that only the authorized service or application has permission to execute the function and add columns to the table.

### Conclusion

In this blog post, we explored the implementation of a custom function called `add_column` that allows for dynamic column addition to a table in a PostgreSQL database using Supabase. By examining the function's code, understanding its parameters, and following its implementation steps, we gained valuable insights into how to extend the schema of a table on the fly.

The ability to dynamically add columns provides significant value in scenarios where the data structure needs to evolve dynamically, enabling greater flexibility and adaptability in application development. With Supabase and PostgreSQL, developers can leverage the power of the `add_column` custom function to seamlessly incorporate new columns into their tables as their application requirements evolve.

By following best practices such as limiting the execution of the function to authorized users, developers can ensure the security and integrity of their database operations.