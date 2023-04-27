---
title: "Get WebFlow Collection items with Supabase"
datePublished: Tue Apr 11 2023 17:01:39 GMT+0000 (Coordinated Universal Time)
cuid: clgcif7240745gynv4i8ocw26
slug: get-webflow-collection-items-with-supabase
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1681162575993/41b53e2e-d51f-4342-8f4f-9474467b39a8.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1681162677595/4bbd801b-8393-461f-adb3-56c57f9f47ec.png
tags: postgresql, javascript, javascript-library, supabase, webflow

---

This blog post will walk you through the process of fetching data from Webflow using Supabase. Webflow provides an API that you can use to access data from collections with a sync RPC call. However, to access the API, you need an API key. This post will show you how to **store the API key securely** using [pgsodium](https://supabase.com/docs/guides/database/extensions/pgsodium) and Supabase.

Additionally, it will guide you on how to fetch data from Webflow using Supabase with code snippets. You will learn how to use the supabase-js client library to call a Supabase stored procedure that retrieves data from Webflow. Lastly, the post includes examples of the response data that you can expect from Webflow. By following the instructions in this post, you can easily fetch data from Webflow and use it in your Supabase application.

## Setting up Webflow access:

Go to your web flow [dashboard](https://webflow.com/dashboard/sites/_/integrations), then click on Integrations, then scroll down to API Access and click on "Generate new API Token":

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680550481808/7dd3b10c-f2eb-44ee-865a-9da7dad6d150.png align="center")

Then, copy the API key in a text file (for now):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680550629574/379b45c7-5090-4965-8f2a-b21c5a3d6d60.png align="center")

### Enabling extensions on Supabase:

Go to the Supabase [dashboard](https://app.supabase.com/project/_/database/extensions) and enable the following extensions:

* HTTP
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680554098598/de28ff9a-0345-4e31-8bab-0f79a427dcd4.png align="center")
    
* PGSODIUM
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680554044331/4854c482-9c60-43aa-81a0-af8a23821d40.png align="center")
    
* PG\_NET
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680554070692/4034eb89-da5b-41bc-938a-4af04efc2b14.png align="center")
    

## Storing the API key properly with pgsodium:

Please note that if you are running this in Supabase SQL Editor, then you'll need to run a command at each time eg \``CREATE SCHEMA secrets;`\`, click on `Run`, then run the command below it:

```pgsql
-- Creating the schema to store this:
CREATE SCHEMA secrets;
-- Creating the table to store the API keys:
CREATE TABLE secrets.api_keys (
   id uuid PRIMARY KEY NOT NULL DEFAULT uuid_generate_v4(),
   key_id uuid NOT NULL REFERENCES pgsodium.key(id) DEFAULT (pgsodium.create_key()).id,
   nonce bytea NOT NULL DEFAULT pgsodium.crypto_aead_det_noncegen(),
   api_secret text NOT NULL DEFAULT 'undefined'::text,
   name text NOT NULL
);
-- Setting up pg_sodium for the table:
SECURITY LABEL FOR pgsodium
  ON COLUMN secrets.api_keys.api_secret
  IS 'ENCRYPT WITH KEY COLUMN key_id NONCE nonce ASSOCIATED id';

-- Adding the secrets to the table:
INSERT INTO secrets.api_keys(name, api_secret) values ('webflowapi','06cfa959a211d3314ad0cf796105b16363d420a2dab07d922035914739a40079');
```

### **Allowing the** `postgres` **user to access the** `pgsodium`**:**

This is not necessary if you don't plan on using this from Supabase dashboard.

```pgsql
GRANT pgsodium_keyiduser TO "postgres";
GRANT USAGE ON SCHEMA pgsodium TO "postgres";
```

You can access this secret using the view that was automatically created for it:

```pgsql
select decrypted_api_secret from secrets.decrypted_api_keys where name = 'webflowapi';
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680552358207/f94e7d0a-f85c-4110-b824-b36793543768.png align="center")

## Fetching data from webflow:

To assist with requesting passing the authorization/bearer tokens, I've created a helper function that extends http\_get just for this very specific request.

```pgsql
CREATE OR REPLACE FUNCTION http_get_with_auth(url_address text, bearer text)
RETURNS TABLE (__status text, __content jsonb)
SECURITY DEFINER
SET search_path = public, extensions, secrets AS
$BODY$
DECLARE
  full_bearer text := 'Bearer ' || bearer;
BEGIN
 RETURN QUERY SELECT status::text, content::jsonb
  FROM http((
          'GET',
           url_address,
           ARRAY[http_header('Authorization',full_bearer)],
           NULL,
           NULL
        )::http_request);
END;
$BODY$
LANGUAGE plpgsql volatile;
```

Now, we take advantage of the helper function and call it from the function that fetches the data from Webflow:

```pgsql
CREATE OR REPLACE FUNCTION fetch_webflow(collection_id text, _offset bigint, _limit bigint)
RETURNS TABLE (status text, content jsonb)
SECURITY DEFINER
SET search_path = public, extensions, secrets AS
$BODY$
 DECLARE
    webflow_api text;
    url_address text;
 BEGIN
  webflow_api := (select decrypted_api_secret from secrets.decrypted_api_keys where name = 'webflowapi');
  url_address := format('https://api.webflow.com/collections/%s/items?offset=%s&limit=%s', collection_id, _offset::text, _limit::text);
  RETURN QUERY (SELECT __status, __content FROM http_get_with_auth(url_address, webflow_api));
 END;
$BODY$
LANGUAGE plpgsql volatile;
```

### Getting the CollectionID from Webflow:

Open your Webflow dashboard and go to the designer tool:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680554584892/18176585-004e-4778-a0c8-a2a95399a745.png align="center")

Then, click on the collections, select the desired collection to see the ID:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680554464548/c28a676b-2129-4df7-985b-f7034400e59f.png align="center")

It is possible to fetch collection items with Supabase:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680554868998/9c5171df-8b3d-4087-9acb-9f40217c2703.png align="center")

### Using the supabase-js client library:

```javascript
const { data: ret, error } = await supabase
  .rpc('fetch_webflow', 
    {
      collection_id: '64275ecc5c4c4c51453539df',
      _limit: 10, 
      _offset:1 
    });
if (ret[0]['status']=="200")
  console.log(ret[0]['content']);
else{
  console.log("Error: "+ret[0]['status']+" "+JSON.stringify(ret[0]['content']));
}
```

### Examples of response:

200 with results

```javascript
{
  count: 2,
  items: [
    {
      as: 'Maxime nihil molestiae rem est ipsam animi velit. Accusamus est placeat sapiente ver',
      _id: '64275ed269d01c30c9fddc2f',
      bar: 'https://wordpress.com',
      _cid: '64275ecc5c4c4c51453539df',
      name: 'Sapiente Id Quo Consequatur',
      slug: 'sapiente-id-quo-consequatur',
      _draft: false,
      _archived: false,
      'created-by': 'Person_64275647bde7a7c408785df6',
      'created-on': '2023-03-31T22:29:38.963Z',
      'updated-by': 'Person_64275647bde7a7c408785df6',
      'updated-on': '2023-03-31T22:29:38.963Z',
      'published-by': 'Person_64275647bde7a7c408785df6',
      'published-on': '2023-03-31T22:30:45.459Z'
    },
    {
      as: 'Enim cum molestiae distinctio porro nisi atque. Tenetur voluptas qui qui molestiae cum veniam. Facilis explicabo et harum ipsa dolore enim ',
      _id: '64275ed269d01c83aafddc30',
      bar: 'https://www.craigslist.org',
      _cid: '64275ecc5c4c4c51453539df',
      name: 'Nulla Minus',
      slug: 'nulla-minus',
      _draft: false,
      _archived: false,
      'created-by': 'Person_64275647bde7a7c408785df6',
      'created-on': '2023-03-31T22:29:38.963Z',
      'updated-by': 'Person_64275647bde7a7c408785df6',
      'updated-on': '2023-03-31T22:29:38.963Z',
      'published-by': 'Person_64275647bde7a7c408785df6',
      'published-on': '2023-03-31T22:30:45.459Z'
    }
  ],
  limit: 2,
  total: 10,
  offset: 1
}
```

No more entries:

```javascript
{ count: 0, items: [], limit: 2, total: 10, offset: 50 }
```

Collection ID error:

```javascript
Error: 400 {"err":"ValidationError: Provided ID is invalid","msg":"Provided ID is invalid","code":400,"name":"ValidationError","path":"/collections/64275ecc5c4c4c51453539df_/items"}
```

Congratulations for following along in this guide. If you read this here, share this post on Twitter with the emoji of your favorite fruit. 🍉