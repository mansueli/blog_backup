---
title: "Tracking User Data with Fingerprint and Supabase in PostgreSQL"
datePublished: Tue Jun 13 2023 15:22:07 GMT+0000 (Coordinated Universal Time)
cuid: cliuflup8000i0ak2gnie4xos
slug: tracking-user-data-with-fingerprint-and-supabase-in-postgresql
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1686161062716/ab41e468-0068-407a-a91a-13dab6b8756f.png
tags: postgresql, tracking, fingerprint, supabase

---

In this blog post, we will dive into the fascinating world of tracking user data and explore how to accomplish this using Fingerprint-JS and [Supabase](https://supabase.com/) in PostgreSQL. Tracking user data is a crucial aspect of many web applications, allowing businesses to gain valuable insights into user behavior and enhance their services.

By combining the powerful features of Fingerprint, an open-source fingerprinting library, with the flexibility of Supabase, a modern database, we can build a robust tracking system. In this article, we will walk you through the process of setting up a tracking system using a custom function and provide a simple front-end example. So, let's get started and discover how to track user data effectively with Fingerprint-JS and Supabase!

## Prerequisites

Before we begin, you'll need to install [dbdev](https://database.dev/) and [pg\_headerkit](http://database.dev/burggraf/pg_headerkit). dbdev is a PostgreSQL package manager provided by [**database.dev**](http://database.dev). It allows publishing libraries and applications for repeatable deployment in PostgreSQL, similar to npm for JavaScript, pip for Python, or cargo for Rust. With dbdev, you can manage and discover SQL packages, making it easier to extend and enhance your PostgreSQL projects.

### Installing dbdev

You can install dbdev by running the following code in your [SQL Editor](https://app.supabase.com/project/_/sql).

```sql
create extension if not exists http with schema extensions;
create extension if not exists pg_tle;
select pgtle.uninstall_extension_if_exists('supabase-dbdev');
drop extension if exists "supabase-dbdev";
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

### Installing pg\_headerkit

pg\_headerkit is a set of functions for adding special features to your application that use PostgREST API calls to your PostgreSQL database. These functions provide capabilities at the database level, such as rate limiting, IP allow-listing, IP deny-listing, request logging, request filtering, request routing, and user allow-listing/deny-listing (Supabase-specific).

To install pg\_headerkit, follow these steps:

```sql
select dbdev.install('burggraf-pg_headerkit');
create extension "burggraf-pg_headerkit"
    version '1.0.0';
```

By installing pg\_headerkit, you will have access to a wide range of functions that can enhance the functionality and security of your application when interacting with your PostgreSQL database.

## **Creating the "track\_users" Table**

Before we can track and store user data, we need to create a table in the PostgreSQL database that will hold this information. The following SQL code creates the `track_users` table with the necessary columns.

```sql
CREATE TABLE public.track_users (
    id SERIAL PRIMARY KEY,
    version TEXT,
    visitor_id TEXT,
    confidence_score NUMERIC,
    timezone TEXT,
    hdr JSONB,
    math JSONB,
    audio JSONB,
    fonts JSONB,
    os_cpu JSONB,
    canvas JSONB,
    ip_address TEXT,
    user_agent TEXT,
    is_mobile BOOLEAN,
    is_iphone BOOLEAN,
    is_ipad BOOLEAN,
    is_android BOOLEAN,
    raw_fpjs JSONB,
    raw_pgrst JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

In this table definition, we have defined various columns to store different pieces of information about the user. For example, the `version` column stores the version of Fingerprint, the `visitor_id` column stores the unique identifier for the visitor, and the `confidence_score` column stores the confidence score associated with the fingerprint. Other columns store data related to different components and characteristics extracted from Fingerprint-JS.

The primary key column `id` ensures each record has a unique identifier and the `created_at` column stores the timestamp when the record is created.

With the `track_users` table in place, we can proceed to the next step, which is implementing the tracking function that will insert the user data into this table.

## Creating the Tracking Function

To track user data effectively, we will create a custom PostgreSQL function that leverages the power of Fingerprint-JS and the additional capabilities provided by the pg\_headerkit extension. This function will receive JSON data from Fingerprint-JS and insert it into the `track_users` table in the database.

Before we dive into the function itself, let's take a closer look at the components involved:

### **Fingerprint**

Fingerprint-JS is an open-source library that allows us to collect various unique browser and device information from users. This information, known as a fingerprint, can be used to track and identify users across multiple sessions. FingerprintJS provides a JavaScript API that collects data such as browser details, installed plugins, screen resolution, and more.

### **pg\_headerkit**

pg\_headerkit is an extension for PostgreSQL that adds special features and capabilities to your application when using PostgREST API calls to your database. This extension includes a set of functions that can be used inside PostgreSQL functions to enhance functionality and security at the database level.

Now that we have an understanding of Fingerprint-JS and pg\_headerkit, let's proceed with creating the tracking function. Here's the detailed function definition.

```sql
CREATE OR REPLACE FUNCTION public.track_user(fpjs jsonb)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public', 'extensions', 'hdr'
AS $$
DECLARE
    ip_data text;
    user_agent text;
    is_mobile boolean;
    is_iphone boolean;
    is_ipad boolean;
    is_android boolean;
BEGIN
    ip_data := hdr.ip();
    user_agent := hdr.agent();
    is_mobile := hdr.is_mobile();
    is_iphone := hdr.is_iphone();
    is_ipad := hdr.is_ipad();
    is_android := hdr.is_android();

    INSERT INTO track_users (
        version,
        visitor_id,
        confidence_score,
        timezone,
        hdr,
        math,
        audio,
        fonts,
        os_cpu,
        canvas,
        ip_address,
        user_agent,
        is_mobile,
        is_iphone,
        is_ipad,
        is_android,
        raw_fpjs,
        raw_pgrst
    )
    VALUES (
        fpjs->>'version',
        fpjs->>'visitorId',
        (fpjs->'confidence'->>'score')::DECIMAL,
        fpjs->'components'->'timezone'->>'value',
        fpjs->'components'->'hdr',
        fpjs->'components'->'math',
        fpjs->'components'->'audio',
        fpjs->'components'->'fonts',
        fpjs->'components'->'osCpu',
        fpjs->'components'->'canvas',
        ip_data,
        user_agent,
        is_mobile,
        is_iphone,
        is_ipad,
        is_android,
        fpjs,
        current_setting('request.headers', true)::jsonb
    );
    RETURN current_setting('request.headers', true)::jsonb;
END;
$$;
```

The `track_user` function is a custom PostgreSQL function designed to track user data using the Fingerprint-JS library and the pg\_headerkit extension. It receives a JSON object `fpjs` as input, which contains the data collected by Fingerprint-JS from the user's browser.

Inside the function, various variables are declared to store specific information extracted from the `hdr` (pg\_headerkit) extension, such as IP address, user agent, mobile device status, and specific device types (iPhone, iPad, Android).

The function then performs an `INSERT` operation into the `track_users` table, which was created in a previous step. The values inserted into the table are extracted from the `fpjs` JSON object and mapped to the respective columns in the table. This includes information such as the Fingerprint-JS version, visitor ID, confidence score, timezone, and various components extracted by Fingerprint-JS.

The `track_user` function acts as a bridge between Fingerprint-JS and the PostgreSQL database, allowing the collection and storage of user data for tracking and analysis purposes.

## **Front-end Tracking Example**

In this section, we will provide an example of how to create a simple HTML file with JavaScript to demonstrate how to use Fingerprint.js and Supabase to track user data. This example will show you how to integrate the tracking functionality into your front-end application.

```php
<!DOCTYPE html>
<html>
<head>
  <link rel="stylesheet" href="styles.css">
  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2.24.0/dist/umd/supabase.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@fingerprintjs/fingerprintjs@3.4.1/dist/fp.umd.min.js"></script>
</head>
<body>
  <div class="wrap">
    <button class="button" onclick="trackUserData()">Track Data</button>
  </div>
  <script>
    const SUPABASE_URL = 'https://your-supabase-url.supabase.co';
    const SUPABASE_ANON_KEY = 'your-supabase-anon-key';
    
    // Configure Supabase client
    const options = {
      db: { schema: 'public' },
      auth: {
        autoRefreshToken: false,
        persistSession: false,
        detectSessionInUrl: false,
      },
    };
    const supabase = supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY, options);
    
    async function trackUserData() {
      try {
        const fp = await FingerprintJS.load();
        const result = await fp.get();
        const visitorId = result.visitorId;
        const geometry = result.canvas;
        
        // Call the track_user stored procedure using Supabase client
        const { data: user, error } = await supabase.rpc('track_user', { fpjs: result });
        
        // Handle response or perform further actions
        if (error) {
          console.log('Error: ' + error.message);
        } else {
          console.log('User tracked successfully:', user);
        }
      } catch (error) {
        console.log('Error: ' + error);
      }
    }
    // Track user data when the page loads
    trackUserData();
  </script>
</body>
</html>
```

To try the front-end tracking example yourself, you can access the live demo on StackBlitz. Simply click on the following link to access the demo:

[**Live Demo - Front-end Tracking Example**](https://stackblitz.com/edit/web-platform-d7b6mr?devToolsHeight=33&file=index.html)

The `trackUserData()` function uses Fingerprint.js to gather user fingerprint data and then calls the `track_user` stored procedure using the Supabase client's `rpc` method. The user fingerprint data is passed as a parameter to the stored procedure.

You can further customize this function to include additional data or perform additional actions based on your specific requirements.

### Enhancement ideas

One important aspect to note is that the open-source version of Fingerprint.js outputs a confidence score ranging from `0.4` to `0.6`. To enhance the accuracy of user tracking, you can employ your own logic and utilize additional numeric values provided by PostgREST to extend the confidence score to a higher range, such as `0.6` to `0.99`. This allows you to create a more comprehensive tracking system.

If you want to leverage geographic data for tracking purposes, you can integrate PostGIS into your system. PostGIS, an extension for PostgreSQL, provides spatial data types and enables you to store and query geographic information. For example, you can compare the distance between the current user's location and their previous session data to generate a complementary confidence factor. This can further refine your tracking capabilities.

## Conclusion

By implementing the front-end tracking example using Fingerprint.js and Supabase, you can efficiently track user data in your application. The integration allows you to gather valuable information about users, such as their visitor ID and device attributes.

Additionally, Supabase distinguishes itself by not charging for API calls. This means that if you decide to use Supabase for tracking user data, it can be a cost-effective solution compared to traditional methods.

Remember that the example provided here is a starting point, and you can expand upon it to suit your specific tracking requirements and implement additional features tailored to your application's needs.

You can customize the function further to include additional data or perform additional actions based on your specific requirements. By integrating this front-end tracking example into your application, you can collect and store user data in the "track\_users" table we created earlier. This data can be further analyzed and utilized to enhance your application's functionality and user experience.