---
title: "How to Create a Pseudo MySQL Foreign Data Wrapper for Supabase with PostgreSQL and Edge Functions"
seoTitle: "Creating a Pseudo MySQL Foreign Data Wrapper for Supabase: A Guide"
seoDescription: "This comprehensive guide explores the motivation behind the wrapper, its functionality, and how it enables developers to seamlessly fetch data from MySQL."
datePublished: Tue Jun 27 2023 14:30:12 GMT+0000 (Coordinated Universal Time)
cuid: cljedx0x8000a0al1bhpi6td6
slug: mysql-foreign-data-wrapper-supabase-postgresql-edge-functions
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1687551170409/b4a3ec10-b49f-476a-ab38-a79d69a954fe.png
tags: postgresql, mysql, supabase, edge-functions, fdw

---

In this blog post, we will explore how to create a pseudo-MySQL foreign data wrapper for [Supabase](https://supabase.com/) using PostgreSQL and Supabase's Edge Functions. We'll discuss the motivation behind this wrapper and how it enables developers to fetch data from a MySQL database.

### Setting up the `service_role` key in Vault

o ensure the security and access control of our Edge Function, we need to restrict it to only accept admin/server requests. For this purpose, we'll securely store the service\_role key in the database using Vault. Vault is a popular tool for managing secrets and protecting sensitive information. By storing the service\_role key in Vault, we can ensure its confidentiality and integrity.

To set up the service\_role key in Vault, follow these steps:

1. Open the Supabase dashboard.
    
2. Go to the project settings.
    
3. Navigate to the Vault secrets configuration: [**Supabase Vault Secrets**](https://app.supabase.com/project/_/settings/vault/secrets)
    
4. Store the service\_role key securely in Vault.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687539755181/cdb24225-b36a-4658-9033-de3b06a8f167.png align="center")

### Creating the MySQL Wrapper Functions

Now, let's dive into the code and understand the logic and functionality of each function that comprises the MySQL wrapper. These functions enable developers to retrieve data from a MySQL database through the Supabase Edge Function.

The first function we'll examine is `http_post_with_auth`. This function acts as a convenient wrapper around the HTTP extension in PostgreSQL, allowing us to make authenticated requests to the Edge Function using a bearer token. It takes the URL address, POST data, and bearer token as input parameters and returns the response status and content in a table format.

Here's the code for the `http_post_with_auth` function:

```sql
--
-- Function to make HTTP POST request with authentication
--
CREATE OR REPLACE FUNCTION public.http_post_with_auth(
    url_address text, 
    post_data text, 
    bearer text
)
 RETURNS TABLE(_status text, _content jsonb)
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'extensions'
AS $function$
DECLARE
  full_bearer TEXT := 'Bearer ' || bearer;
  response RECORD;
BEGIN
  -- Make the HTTP POST request with the given URL, data, and bearer token
  SELECT status::text, content::jsonb
  INTO response
  FROM http((
          'POST',
           url_address,
           ARRAY[http_header('Authorization', full_bearer), http_header('Content-Type', 'application/json')],
           'application/json',
           coalesce(post_data, '') -- Set content to an empty string if post_data is NULL
        )::http_request);

  -- Raise an exception if the response content is NULL
  IF response.content IS NULL THEN
    RAISE EXCEPTION 'Error: Edge Function returned NULL content. Status: %', response.status;
  END IF;

  -- Return the status and content of the response
  RETURN QUERY SELECT response.status, response.content;
END;
$function$;
```

### Edge Wrapper

Next, let's discuss the `edge_wrapper` function. This function decrypts the service\_role from Vault and requests the Edge Function, passing the MySQL query as a parameter. It retrieves the API key from Vault, performs an HTTP call to the Edge Function, and returns the response as a JSON object.

Here's the code for the `edge_wrapper` function:

```sql
--
-- Wrapper function for making queries to the Edge Function
--
CREATE OR REPLACE FUNCTION public.edge_wrapper(query text)
 RETURNS jsonb
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'extensions', 'vault'
AS $function$
DECLARE
  api_key TEXT;
  response JSON;
  edge_function_url TEXT := 'https://wqazfpwdgwumetjycblf.supabase.co/functions/v1/mysql_wrapper';
BEGIN
  -- Get the API key from the vault
  SELECT decrypted_secret
  INTO api_key
  FROM vault.decrypted_secrets
  WHERE name = 'service_role';

  -- Make the HTTP call to the Edge Function
  SELECT _content::JSON
  INTO response
  FROM http_post_with_auth(
    edge_function_url,
    json_build_object('query', query)::TEXT,
    api_key
  );

  -- Return the JSON response
  RETURN response;
END;
$function$;
```

### MySQL convenience function

The `mysql()` function dynamically constructs column expressions based on the provided `columns` parameter. It loops through the array of columns, extracts the corresponding values from the JSON object, and assigns them aliases matching the column names.

```sql
--
-- Function to execute a MySQL query and return the specified columns
--
CREATE OR REPLACE FUNCTION mysql(query text, VARIADIC columns text[])
  RETURNS SETOF RECORD
  LANGUAGE plpgsql
AS $function$
DECLARE
  column_exprs text := '';
BEGIN
  -- Construct the column expressions dynamically based on the provided columns
  FOR i IN 1..array_length(columns, 1) LOOP
    IF i > 1 THEN
      column_exprs := column_exprs || ', ';
    END IF;
    column_exprs := column_exprs || format('(obj->>''%s'') AS %s', columns[i], columns[i]);
  END LOOP;

  -- Execute the dynamic query and return the result
  RETURN QUERY EXECUTE format('SELECT %s FROM jsonb_array_elements(edge_wrapper($1)) AS obj', column_exprs) USING query;
END;
$function$; 
```

You can specify the columns you want to retrieve from the JSON data by passing them as individual parameters after the `query` parameter.

### Creating the Edge Function

Before we dive into the implementation details, let's set up the necessary connection secrets in Deno using the [Supabase CLI](https://supabase.com/docs/reference/cli/introduction). These secrets will allow our Edge Function to establish a connection with the MySQL database. Open your terminal and run the following commands

```javascript
 supabase secrets set MYSQL_HOST=127.0.0.1
 supabase secrets set MYSQL_USER=db_user
 supabase secrets set MYSQL_DBNAME=database_name
 supabase secrets set MYSQL_PASSWORD=123
```

Make sure to replace the values (`127.0.0.1`, `db_user`, `database_name`, `123`) with the actual host, username, database name, and password for your MySQL setup.

The secrets above will be accessed within our Edge Function code to establish the necessary connection parameters securely.

### Edge Function Code

The code snippet below demonstrates the implementation of the Edge Function responsible for executing the query and returning the results in JSON format. Additionally, it incorporates authorization checks to ensure that only authorized requests are processed.

```javascript

import { serve } from "https://deno.land/std@0.192.0/http/server.ts";
import { Client } from "https://deno.land/x/mysql@v2.11.0/mod.ts";

// Serve the HTTP request
serve(async (req: Request) => {
  // Check if the request method is POST
  if (req.method !== "POST") {
    return new Response("Method not allowed", { status: 405 });
  }

  try {
    // Retrieve the service role key from the environment variables
    const serviceRole = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY");
    // Retrieve the token from the Authorization header
    const token = req.headers.get("Authorization")?.split(" ")[1];

    // Check if the token is missing or invalid
    if (!token) {
      return new Response("Missing authorization header", { status: 401 });
    }
    if (token !== serviceRole) {
      console.log(token + "\n" + serviceRole);
      return new Response("Not authorized", { status: 403 });
    }

    // Parse the request body and retrieve the query
    const requestBody = await req.json();
    const query = requestBody.query;

    // Check if the query is missing
    if (!query) {
      return new Response("Missing query parameter", { status: 400 });
    }

    // Retrieve the MySQL connection details from the environment variables
    const host = Deno.env.get("MYSQL_HOST");
    const user = Deno.env.get("MYSQL_USER");
    const db_name = Deno.env.get("MYSQL_DBNAME");
    const password = Deno.env.get("MYSQL_PASSWORD");

    // Connect to the MySQL database
    const client = await new Client().connect({
      hostname: host!,
      username: user!,
      db: db_name!,
      password: password!,
    });

    // Execute the query and store the response
    const response = await client.query(query);

    // Close the database connection
    await client.close();

    // Return the response as JSON
    return new Response(JSON.stringify(response), {
      headers: { "Content-Type": "application/json" },
    });
  } catch (error) {
    // Log the error and return an error response
    console.error(error);
    return new Response(JSON.stringify({ error: "An error occurred" }), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    });
  }
});
```

The code snippet above illustrates an example implementation of the Edge Function. It sets up an HTTP server using Deno's `serve()` function and listens for incoming POST requests. The function performs various checks, such as ensuring the request method is POST, verifying the authorization header, and validating the presence of the required query parameter.

Once the checks pass, the function retrieves the MySQL connection details from the previously set secrets. It establishes a connection to the MySQL database using the `Client` class from the `mysql` module. The query specified in the request body is executed, and the response is sent back as a JSON string.

### Fetching data from MySQL

To retrieve data from a MySQL database using Supabase, you have two options: fetching the data as JSON or fetching it as columns using the `mysql()` convenience function.

1. Fetching data as JSON: To retrieve data as JSON, you can use the `edge_wrapper` function. Here's an example:
    

```sql
SELECT * FROM edge_wrapper('SELECT * FROM wp_comments;');
```

This query will return the data from the `wp_comments` table as a JSON array.

1. Fetching data as columns: If you prefer to fetch data as separate columns, you can use the `mysql()` function. Here's an example:
    

```sql
SELECT * FROM mysql(
    'SELECT * FROM wp_comments;', 
    'user_id', 
    'comment_date', 
    'comment_content'
    ) 
-- Note that we need to specify the output format to get the table
AS (user_id text, comment_date text, comment_content text);
```

In this example, we fetch the `user_id`, `comment_date`, and `comment_content` columns from the `wp_comments` table. By specifying the output format and column types in the `AS` clause, we can structure the result as a table with the desired column names and data types.

You can include additional output columns by extending the list of column names in the `mysql()` function, as shown in the following example:

```sql
SELECT * FROM mysql(
    'SELECT * FROM wp_comments;', 
    'user_id', 
    'comment_date', 
    'comment_content',
    'comment_agent'
) AS (
    user_id text, 
    comment_date text, 
    comment_content text, 
    comment_agent text
);
```

In this case, we have added the `comment_agent` column to the result.

Counting rows: You can also use the `mysql()` function to perform row counting. Here's an example:

```sql
SELECT * FROM mysql(
    'SELECT COUNT(*) AS post_count FROM wp_posts;',
    'post_count'
) AS (post_count text);
```

This query retrieves the count of rows from the `wp_posts` table and assigns it to the `post_count` column.

By using the `mysql()` function in these ways, you can fetch data from MySQL databases within Supabase. In the next section, we present some more advanced examples.

## Advanced examples of querying WordPress

```sql
SELECT * FROM mysql(
    'SELECT post_title, display_name FROM wp_posts 
    INNER JOIN wp_users ON wp_posts.post_author = wp_users.ID 
    WHERE post_type = "post";',
    'post_title',
    'display_name'
) AS (post_title text, author_name text);
```

In this example, we fetch the post titles and corresponding author names from the WordPress database. The query joins the `wp_posts` and `wp_users` tables on the `post_author` and `ID` columns, respectively. We also include a condition (`WHERE post_type = "post"`) to retrieve only posts. The result will include columns `post_title` and `author_name`.

### Counting comments per post

```sql
SELECT * FROM mysql(
    'SELECT p.ID, p.post_title, COUNT(c.comment_ID) AS comment_count 
    FROM wp_posts p 
    LEFT JOIN wp_comments c ON p.ID = c.comment_post_ID 
    WHERE p.post_type = "post" 
    GROUP BY p.ID;',
    'ID',
    'post_title',
    'comment_count'
) AS (id text, post_title text, comment_count text);
```

In this example, we retrieve the post ID, post title, and the count of comments for each post in the WordPress database. The query performs a left join between the `wp_posts` and `wp_comments` tables based on the `ID` and `comment_post_ID` columns, respectively. We apply a condition to select only posts (`WHERE` [`p.post`](http://p.post)`_type = "post"`) and use the `GROUP BY` clause to aggregate the comment count per post. The result includes columns `post_id`, `post_title`, and `comment_count`.

### Fetching user roles

```sql
SELECT * FROM mysql(
    'SELECT user_login, meta_value AS role 
    FROM wp_users 
    INNER JOIN wp_usermeta ON wp_users.ID = wp_usermeta.user_id 
    WHERE meta_key = "wp_capabilities";',
    'user_login',
    'role'
) AS (username text, role text);
```

In this example, we retrieve the usernames and corresponding roles for users in the WordPress database. The query joins the `wp_users` and `wp_usermeta` tables on the `ID` and `user_id` columns, respectively. We include a condition (`WHERE meta_key = "wp_capabilities"`) to select only the user roles. The result includes columns `username` and `role`.

### Conclusion

In this blog post, we delved into the process of creating a pseudo-MySQL foreign data wrapper for Supabase using PostgreSQL and Supabase's Edge Functions. By leveraging the capabilities of Supabase and PostgreSQL, developers can seamlessly integrate their MySQL data into their Postgres database. The code examples provided in this post serve as a starting point for implementing this functionality in your projects, offering exciting possibilities for working with Supabase and PostgreSQL together.

We encourage you to explore and experiment with the concepts and techniques discussed here, tailoring them to suit your specific needs and use cases. By leveraging the combined power of Supabase and PostgreSQL, you can unlock new avenues for efficient data management and integration in your applications.