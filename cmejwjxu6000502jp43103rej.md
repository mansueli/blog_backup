---
title: "Building Secure API Key Management with Supabase, KSUID & PostgreSQL"
seoTitle: "Secure API Key Management with Supabase"
seoDescription: "Learn how to build secure API key management using Supabase, KSUID, and PostgreSQL for efficient and fast operations"
datePublished: Wed Aug 20 2025 11:41:38 GMT+0000 (Coordinated Universal Time)
cuid: cmejwjxu6000502jp43103rej
slug: building-secure-api-key-management-with-supabase-ksuid-and-postgresql
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1755689230616/7c637884-6f19-48b1-a110-790627c66002.png
tags: postgresql, apis, api-key, supabase

---


Managing API keys seems easy at first, but it involves many choices. You need keys that are secure, fast to check, easy to use, and simple to sort. This guide shows how to build secure API key management with [Supabase.com](http://supabase.com/). We use PostgreSQL and KSUID to meet these needs.

In this post, you will learn a simple pattern with Supabase, PostgreSQL, and KSUID (via the pg_idkit extension). By the end, you get SQL code to add to your database. You also get workflows to create, check, and list API keys. This setup is optimized for Supabase users, with tips on the Supabase CLI and JavaScript client. It helps with secure API key management in Supabase.

For more on secrets in Supabase, see our previous post on [Supabase Vault](https://supabase.com/blog/supabase-vault). If you work with PostgreSQL extensions, check out [Trusted Language Extensions for Postgres](https://supabase.com/blog/pg-tle).

---

## Overview: A Simple Way to Manage API Keys

We make API keys that look like this:

```
[KSUID][SECRET]
```

- The KSUID is a short ID that sorts by time. We use it as the row ID.
- The SECRET is a random string shown once to the user.
- In the database, we store:
  - The KSUID (as id),
  - A salted hash of the secret,
  - A random salt,
  - The last 4 characters for the dashboard,
  - The client_id (UUID) to link to a Supabase user (see [Supabase Auth docs](https://supabase.com/docs/guides/auth)),
  - And a name for the key label.

To check a key: Split it into KSUID and secret. Find the row by KSUID (fast with index). Compute the hash and compare. This is quick: one lookup and one hash.

This method keeps your Supabase API key management secure and efficient.

---

## Step 1: Install pg_idkit Extension with dbdev and Supabase CLI

The pg_idkit extension gives KSUID functions like gen_random_ksuid_microsecond(). We install it as a Trusted Language Extension (TLE) using dbdev CLI to generate a migration. Then, use Supabase CLI to apply it. This fits well with Supabase workflows for installing PostgreSQL extensions in Supabase.

### A. Set Up dbdev and Supabase CLI

If you do not have dbdev CLI, install it. Follow the [dbdev installation docs](https://supabase.github.io/dbdev/getting-started/). For Supabase CLI, see the [Supabase CLI docs](https://supabase.com/docs/guides/cli). Link your local project to your Supabase database.

Note: On Supabase, the pg_tle extension is already installed. This meets the prerequisite.

### B. Generate Migration File with dbdev

Run this command to create a migration file for pg_idkit version 0.0.4 in the extensions schema:

```bash
dbdev add -o "./supabase/migrations/" -v 0.0.4 -s extensions package -n kiwicopple@pg_idkit
```

- This makes a SQL file in your migrations folder.
- Adjust the path if your migrations folder is different.

For more details, see the [dbdev install a package docs](https://supabase.github.io/dbdev/install-a-package/).

### C. Apply the Migration with Supabase CLI

Push the migration to your Supabase database:

```bash
supabase db push
```

This applies the extension. Now you can use functions like:

```sql
SELECT gen_random_ksuid_microsecond();
```

For more on extensions, see [Supabase Database Extensions docs](https://supabase.com/docs/guides/database/extensions).

---

## Step 2: Set Up the Table Schema

Use this SQL in your Supabase SQL editor or a migration file. It creates the api_keys table with KSUID as the primary key.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE api_keys (
  id TEXT PRIMARY KEY DEFAULT gen_random_ksuid_microsecond(), -- KSUID as ID (27 chars)
  client_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  name TEXT NOT NULL,                    -- user label
  secret_hash BYTEA NOT NULL,            -- digest(secret || salt)
  salt BYTEA NOT NULL DEFAULT gen_random_bytes(16), -- auto salt
  last4 TEXT NOT NULL,                   -- last 4 chars of secret for UI
  created_at TIMESTAMPTZ DEFAULT now()
);
-- Index for fast per-user listing, ordered by KSUID (time-sortable)
CREATE INDEX api_keys_client_id_idx ON api_keys (client_id, id DESC);
```

Notes:
- KSUIDs sort by time and are 27 characters long.
- We link to auth.users for Supabase auth.
- Use the Supabase CLI to push this as a migration: supabase migration new create_api_keys_table, add the SQL, then supabase db push.

---

## Step 3: Create an API Key Server-Side

This function creates a key. It takes client_id and name, generates everything, and returns the full key once.

Add this in a migration or SQL editor:

```sql
CREATE OR REPLACE FUNCTION create_api_key(client_id UUID, key_name TEXT)
RETURNS TEXT AS $$
DECLARE
  ksuid_prefix TEXT := gen_random_ksuid_microsecond();
  secret TEXT := encode(gen_random_bytes(16), 'base32'); -- random secret
  salt_val BYTEA := gen_random_bytes(16);
  full_key TEXT := ksuid_prefix || secret;
BEGIN
  INSERT INTO api_keys (id, client_id, name, secret_hash, salt, last4)
  VALUES (
    ksuid_prefix,
    client_id,
    key_name,
    digest(secret || salt_val, 'sha256'),
    salt_val,
    right(secret, 4)
  );
  RETURN full_key; -- show this to the user once
END;
$$ LANGUAGE plpgsql;
```

The database makes the secret. Show it once in your app.

---

## Step 4: Verify API Keys with Supabase Edge Functions

To check keys, use a Supabase Edge Function. This is fast and secure. Create one with the Supabase CLI:

```bash
supabase functions new verify-api-key
```

Edit functions/verify-api-key/index.ts with Supabase JS:

```javascript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const supabase = createClient(Deno.env.get('SUPABASE_URL')!, Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!);

Deno.serve(async (req) => {
  const { full_key } = await req.json();
  const { data, error } = await supabase.rpc('verify_api_key', { full_key });

  if (error || !data) {
    return new Response(JSON.stringify({ error: 'Invalid key' }), { status: 401 });
  }

  return new Response(JSON.stringify({ client_id: data }), { status: 200 });
});
```

Deploy it: supabase functions deploy verify-api-key.

The verify_api_key function (add to DB):

```sql
CREATE OR REPLACE FUNCTION verify_api_key(full_key TEXT)
RETURNS UUID AS $$
DECLARE
  ksuid_prefix TEXT := left(full_key, 27);
  secret TEXT := substr(full_key, 28);
  stored_hash BYTEA;
  stored_salt BYTEA;
  key_owner UUID;
BEGIN
  SELECT secret_hash, salt, client_id
  INTO stored_hash, stored_salt, key_owner
  FROM api_keys
  WHERE id = ksuid_prefix;

  IF NOT FOUND THEN
    RETURN NULL;
  END IF;

  IF digest(secret || stored_salt, 'sha256') = stored_hash THEN
    RETURN key_owner; -- valid: return owner id
  ELSE
    RETURN NULL;       -- invalid
  END IF;
END;
$$ LANGUAGE plpgsql;
```

This is efficient: one index lookup and one hash. See [Supabase Functions docs](https://supabase.com/docs/guides/functions) for more.

---

## Step 5: List Keys for Your Dashboard

This function lists keys, sorted by time.

```sql
CREATE OR REPLACE FUNCTION list_api_keys(
  client_id UUID,
  limit_count INT DEFAULT 50,
  offset_count INT DEFAULT 0
)
RETURNS TABLE(id TEXT, name TEXT, last4 TEXT, created_at TIMESTAMPTZ) AS $$
BEGIN
  RETURN QUERY
  SELECT id, name, last4, created_at
  FROM api_keys
  WHERE client_id = list_api_keys.client_id
  ORDER BY id DESC
  LIMIT limit_count OFFSET offset_count;
END;
$$ LANGUAGE plpgsql;
```

Call it from your app with Supabase JS:

```javascript
const { data, error } = await supabase.rpc('list_api_keys', { client_id: user.id });
```

---

## How Your App Uses These in Supabase

### Create a Key
- App calls: `const { data } = await supabase.rpc('create_api_key', { client_id: user.id, key_name: 'My Key' });`
- Show data once to the user.

### Verify a Key
- Send full_key to your Edge Function.
- If it returns client_id, allow access.

### List Keys
- Use the list function in your dashboard.

---

## Additional Notes on Security and Choices

- You cannot get back secrets. This is safer.
- Add revocation: Include revoked_at in the table and check in verify_api_key.
- For rate limits, log usage after verify succeeds.

This setup makes secure API key management in Supabase simple and fast.

Ready to try? Start with the Supabase CLI and build your own.