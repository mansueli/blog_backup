---
title: "Unleash Powerful Webhooks with pgwebhook in Supabase"
seoTitle: "Harness Webhooks with pgwebhook in Supabase"
seoDescription: "Enhance Supabase with pgwebhook: automatic retries, regional fallbacks, and customizable payloads for seamless integration and real-time data sync"
datePublished: Tue Jun 25 2024 18:30:05 GMT+0000 (Coordinated Universal Time)
cuid: clxuqrkwm000109l4djy9giqw
slug: unleash-powerful-webhooks-with-pgwebhook
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1719340068394/f829061a-e8f8-4d82-860e-78dab7da96fc.png
tags: postgresql, webhooks, supabase

---

In today's data-driven world, seamless integration between databases and external services is crucial for real-time data synchronization and triggering actions based on database events. While Supabase offers built-in webhooks for Postgres integration, managing webhooks can become challenging, especially when dealing with complex scenarios or GDPR compliance. This is where pgwebhook, a powerful Postgres Trusted Language Extension, comes into play. pgwebhook simplifies the integration process by offering robust features like automatic retries, regional fallbacks, and customizable payloads, making it an ideal choice for developers seeking to enhance their [Supabase](https://supabase.com) workflows.

This blog post explores the limitations of default Supabase webhooks and how pgwebhook empowers developers to overcome these challenges. We'll delve into the functionalities of pgwebhook and showcase practical use cases that demonstrate its effectiveness.

## Understanding the Need for Integration

Integrating databases with external services presents several challenges. The default approach for Postgres webhooks with pg_net might fall short when you need to process the results of a call or handle high volumes that could overwhelm the worker process. Additionally, Supabase's built-in webhooks currently lack the ability to customize the payload before sending it to external services.

## Introducing pgwebhook: A revamped Webhook experience

`pgwebhook` is more than just a webhook solution; it's a comprehensive extension designed to streamline Postgres integration with webhooks. Built with reliable technology like cURL (through [pghttp](https://github.com/pramsey/pgsql-http)), pgwebhook ensures dependable and performant HTTP requests. It surpasses traditional webhook functionalities by offering features like:

`Automatic Fallbacks`: Ensures uninterrupted operation even during edge function failures.
`Regional Fallbacks`:  *(exclusive for edge functions)* Guarantees compliance with regulations like GDPR by allowing you to restrict calls to specific regions.
Customizable Payloads: Provides flexibility in tailoring the data sent to external services.
Seamless Supabase Integration: Integrates effortlessly with the Supabase platform, making it an ideal choice for Supabase users.

## Installation and Setup

Installing `pgwebhook` is a straightforward process, whether you're a Supabase user or prefer manual installation.

**Installing Database.dev**

```sql
create extension if not exists http with schema extensions;
create extension if not exists pg_tle;
drop extension if exists "supabase-dbdev";
select pgtle.uninstall_extension_if_exists('supabase-dbdev');
select
    pgtle.install_extension(
        'supabase-dbdev',
        resp.contents ->> 'version',
        'PostgreSQL package manager',
        resp.contents ->> 'sql'
    )
from http(
    (
        'GET',
        'https://api.database.dev/rest/v1/'
        || 'package_versions?select=sql,version'
        || '&package_name=eq.supabase-dbdev'
        || '&order=version.desc'
        || '&limit=1',
        array[
            ('apiKey', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InhtdXB0cHBsZnZpaWZyYndtbXR2Iiwicm9sZSI6ImFub24iLCJpYXQiOjE2ODAxMDczNzIsImV4cCI6MTk5NTY4MzM3Mn0.z2CN0mvO2No8wSi46Gw59DFGCTJrzM0AQKsu_5k134s')::http_header
        ],
        null,
        null
    )
) x,
lateral (
    select
        ((row_to_json(x) -> 'content') #>> '{}')::json -> 0
) resp(contents);
create extension "supabase-dbdev";
select dbdev.install('supabase-dbdev');
drop extension if exists "supabase-dbdev";
create extension "supabase-dbdev";
```

Installing pgwebhook:

```sql

-- 
SELECT dbdev.install('mansueli@pgwebhook');
CREATE EXTENSION "mansueli@pgwebhook" VERSION '0.1.1';
```

You can also install it manually by running the [SQL script](https://github.com/mansueli/tle/blob/master/pgwebhook/pgwebhook--0.1.1.sql) in your database.

[https://github.com/mansueli/tle/blob/master/pgwebhook/](https://github.com/mansueli/tle/blob/master/pgwebhook/)

## Practical Usage Scenarios

Let's explore practical scenarios where `pgwebhook` shines:

### Direct Usage:

Direct calls to external services are sometimes necessary, particularly for edge functions or APIs restricted to specific regions. For instance, imagine OpenAI restricts calls to allowed regions, or for GDPR compliance, you might need to keep your Edge Function calls within specific subregions. In such cases, pgwebhook offers a solution by providing a wrapper function that sets defaults for the integration process.

#### Secure Secret Management with Vault

For best practices, consider storing secrets in Vault and fetching them with functions like:

```sql
CREATE OR REPLACE FUNCTION vault.get_anon (bearer BOOLEAN DEFAULT FALSE)
RETURNS TEXT 
AS $$
BEGIN
    IF bearer THEN
        RETURN 'Bearer ' || (SELECT decrypted_api_secret FROM secrets.decrypted_api_keys WHERE name = 'anon');
    ELSE
        RETURN (SELECT decrypted_api_secret FROM secrets.decrypted_api_keys WHERE name = 'anon');
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION vault.get_supabase_url () 
RETURNS TEXT 
AS $$
BEGIN
    RETURN (SELECT decrypted_api_secret 
            FROM secrets.decrypted_api_keys 
            WHERE name = 'supabase_url');
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE 
FUNCTION vault.get_service_role (bearer BOOLEAN DEFAULT FALSE) 
RETURNS TEXT 
AS $$
BEGIN
    IF bearer THEN
        RETURN 'Bearer ' || (SELECT decrypted_api_secret 
                             FROM secrets.decrypted_api_keys 
                             WHERE name = 'service_role');    
    ELSE
        RETURN (SELECT decrypted_api_secret 
                FROM secrets.decrypted_api_keys 
                WHERE name = 'service_role');
    END IF;
END;
$$ LANGUAGE plpgsql;
```

Following this approach, you can create a wrapper function that ensures calls originate from the specified regions:

```sql
-- Create wrapper function in the public schema
CREATE OR REPLACE FUNCTION public.euro_edge (func TEXT, data JSONB) RETURNS JSONB LANGUAGE plpgsql
AS $function$
DECLARE
    custom_headers JSONB;
    allowed_regions TEXT[] := ARRAY['eu-west-1', 
                                    'eu-west-2', 
                                    'eu-west-3', 
                                    'eu-north-1', 
                                    'eu-central-1'];
BEGIN
    -- Set headers with anon key and Content-Type
    custom_headers := jsonb_build_object('Authorization', 
                                   vault.get_anon_key(bearer := true), 
                      'Content-Type', 'application/json');
    -- Call edge_wrapper function with default values
    RETURN hook.edge_wrapper(url := ('https://supanacho.supabase.co/functions/v1/' || func), 
                             headers := custom_headers, 
                             payload := data, 
                             max_retries := 5, 
                             allowed_regions := allowed_regions);
END;
$function$;
```

### Webhooks: Powerful Automation

Webhook triggers are crucial for automating actions based on database events. However, ensuring compliance and reliability can be challenging, especially with edge cases like GDPR or service availability issues. pgwebhook addresses these challenges with features like automatic retries and regional fallbacks.

**Using Triggers on a Table**

You can use `pgwebhook` similarly to Supabase webhooks:

```sql
CREATE TRIGGER your_trigger_name
AFTER INSERT OR UPDATE OR DELETE ON your_table
FOR EACH ROW EXECUTE FUNCTION hook.webhook_trigger(
    'https://your-webhook-url.com',
    'POST',
    '{"Content-Type": "application/json"}',
    '{}',
    5000
);
```

**However, you can also get more:**  
  
`pgwebhook` goes beyond Supabase webhooks by enabling triggers that automatically handle retries and regional headers. This ensures compliance with regulations like GDPR and maintains service availability during regional outages.

#### Create a Trigger Using a PL/pgSQL Custom Handler

```sql
CREATE OR REPLACE FUNCTION hook.custom_handler_function(payload jsonb)
RETURNS jsonb AS $$
DECLARE
    new_payload jsonb;
BEGIN
    -- Modify the payload as needed
    -- Example: Add a new field to the payload
    new_payload := payload || jsonb_build_object('additional_info', 'This is extra info');

    -- Example: Remove a sensitive field
    new_payload := new_payload - 'sensitive_field';

    -- Return the modified payload
    RETURN new_payload;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER your_trigger_name
AFTER INSERT OR UPDATE OR DELETE ON your_table
FOR EACH ROW EXECUTE FUNCTION hook.webhook_trigger(
    'https://your-webhook-url.com',
    'POST',
    '{"Content-Type": "application/json"}',
    '{}',
    5000,
    'hook.custom_handler_function'
);
```

#### Create a Trigger Using a PLv8 Custom Handler

```sql
CREATE OR REPLACE FUNCTION hook.custom_handler_function_js(payload jsonb)
RETURNS jsonb AS $$
var newPayload = payload;

// Modify the payload: Add new field and remove a sensitive field
newPayload.additional_info = 'This is extra info';
delete newPayload.sensitive_field;

return newPayload;
$$ LANGUAGE plv8;

CREATE TRIGGER your_trigger_name
AFTER INSERT OR UPDATE OR DELETE ON your_table
FOR EACH ROW EXECUTE FUNCTION hook.webhook_trigger(
    'https://your-webhook-url.com',
    'POST',
    '{"Content-Type": "application/json"}',
    '{}',
    5000,
    'hook.custom_handler_function_js'
);
```
## Conclusion

`pgwebhook` is a promising new Postgres Trusted Language Extension that simplifies and streamlines webhook integration for Supabase users. Its robust feature set, including automatic retries, regional fallbacks, and customizable payloads, addresses common challenges associated with managing webhooks effectively. By leveraging `pgwebhook`, developers can enhance their Supabase workflows and ensure reliable communication between databases and external services.

## Experimentation and Community

As an actively developed project, pgwebhook welcomes contributions from the community. We encourage you to explore the possibilities of pgwebhook and share your feedback. Feel free to experiment, submit pull requests (PRs) for improvements, and open issues on GitHub to report any bugs or request new features.

We believe that pgwebhook has the potential to become a valuable asset for the Supabase developer community. Join us on GitHub (https://github.com/mansueli/tle/blob/master/pgwebhook/) to explore further and contribute to its development!