---
title: "Building a Queue System with Supabase and PostgreSQL"
datePublished: Tue Jun 20 2023 14:15:38 GMT+0000 (Coordinated Universal Time)
cuid: clj4dbbuz00090al8cfd37m9g
slug: building-a-queue-system-with-supabase-and-postgresql
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1687116676721/1ae2f3ad-f771-44df-9fb1-5d8bf9e72316.png
tags: postgresql, queue, supabase, edge-functions

---

[Supabase](https://supabase.com/) is an open-source alternative to Firebase that offers a wide range of capabilities for building robust and scalable applications. One of the key advantages of Supabase is its integration with PostgreSQL, a powerful and reliable database management system. In this blog post, we will explore how to leverage Supabase and PostgreSQL to create a highly efficient queue system.

A queue system is crucial for handling asynchronous tasks and managing job processing. By utilizing a queue, we can ensure that tasks are processed in a controlled and organized manner, improving performance and scalability. Let's dive into the process of setting up a queue system with Supabase and PostgreSQL.

## Setting up the Queue System

To set up the queue system, we need to enable the required extensions in PostgreSQL. Specifically, we need to enable the pg\_net and pg\_cron extensions, which provide the necessary functionality for our queue system.

Please navigate to the [Extensions page](https://app.supabase.com/project/_/database/extensions) in your Supabase project and enable both the pg\_net and pg\_cron extensions.

Next, we can define the necessary table structure. The three key tables we'll be working with are job\_queue, current\_jobs, and workers. The job\_queue table stores the pending jobs, while the current\_jobs table keeps track of the jobs currently being processed. The workers table maintains records of the available worker instances.

To create these tables, execute the following SQL code:

```sql
CREATE TABLE job_queue (
    job_id serial PRIMARY KEY,
    http_verb TEXT NOT NULL CHECK (http_verb IN ('GET', 'POST', 'DELETE')),
    payload jsonb,
    status TEXT NOT NULL DEFAULT '',
    retry_count INTEGER DEFAULT 0,
    retry_limit INTEGER DEFAULT 10,
    url_path TEXT DEFAULT '', 
    content TEXT DEFAULT '',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE current_jobs (
    request_id BIGINT NOT NULL,
    job_id BIGINT NOT NULL
);

CREATE TABLE workers (
    id SERIAL PRIMARY KEY,
    locked BOOLEAN NOT NULL DEFAULT FALSE
);
-- Adding two workers:
INSERT INTO workers (locked)
VALUES (FALSE), (FALSE);
-- You can also add a worker with:
INSERT INTO workers 
DEFAULT VALUES;
-- Adding 10 workers:
do $$
begin
execute (
    select string_agg('INSERT INTO workers DEFAULT VALUES',';')
    from generate_series(1,10)
);
end; 
$$;
```

## Functions for Job Processing

To process jobs in the queue system, we must define several functions that work together. Let's discuss the purpose and functionality of each function:

* `process_current_jobs_if_unlocked`:This function checks if there is an available worker by selecting an unlocked worker from the `workers` table. If a worker is found, it locks the worker, processes the current jobs using the `process_current_jobs` function, and unlocks the worker afterward.
    
* `process_current_jobs`: This function iterates over the records in the `current_jobs` table and retrieves the response from the `pg_net` extension using the `net._http_collect_response` function. It checks the status and status code of the response and performs the necessary actions based on the result. If the job is successful, it updates the `job_queue` table with the completion status and content and deletes the corresponding record from the `current_jobs` table. If the job fails, it updates the `job_queue` table with the failure status and increases the retry count, and deletes the record from the `current_jobs` table. If the job is still in progress or not found, it raises a notice.
    
* `process_job`: This trigger function is called after a record is inserted into the `job_queue` table. It updates the status of the job to "processing" and retrieves the API key from the `vault.decrypted_secrets` table. It then calls the `request_wrapper` function to process the job by making an HTTP request to the specified URL. The request ID is inserted into the `current_jobs` table for tracking purposes.
    
* `request_wrapper`: This function is a convenience wrapper around the `pg_net` extension functions (`net.http_delete`, `net.http_post`, and `net.http_get`) for making HTTP requests. It accepts the method, URL, parameters, body, and headers as input and returns the request ID.
    
* `retry_failed_jobs`: This function is responsible for retrying failed jobs. It selects the failed jobs from the `job_queue` table based on the status and retry count. It then updates the retry count and calls the `request_wrapper` function to process the job again. The new request ID is inserted into the `current_jobs` table.
    

The trigger function `process_job()` plays a vital role in executing job processing when a new job is added to the queue. Then, there are schedule functions to update the status and response acquired from the requests.

```sql
-- 
-- Execute current jobs function if there are available workers
--
CREATE OR REPLACE FUNCTION process_current_jobs_if_unlocked()
RETURNS VOID AS $$
DECLARE
    worker RECORD;
BEGIN
    -- Find an unlocked worker
    SELECT * INTO worker FROM workers FOR UPDATE SKIP LOCKED LIMIT 1;
    IF worker IS NOT NULL THEN
        RAISE NOTICE 'Using worker_id: %', worker.id;
        -- Lock the worker (this is already done by the SELECT ... FOR UPDATE)
        
        -- Process current jobs
        PERFORM process_current_jobs();
        
        -- Unlock the worker
        UPDATE workers SET locked = FALSE WHERE id = worker.id;
    ELSE
        RAISE NOTICE 'No unlocked workers available';
    END IF;
END;
$$ LANGUAGE plpgsql;
-- 
-- Loop through records in current_jobs and get response from pg_net
--
CREATE OR REPLACE FUNCTION process_current_jobs()
RETURNS VOID 
SECURITY DEFINER
SET search_path = public, extensions, net, vault
AS $$
DECLARE
    current_job RECORD;
    response_result RECORD;
BEGIN
    FOR current_job IN SELECT * FROM current_jobs 
    FOR UPDATE SKIP LOCKED
    LOOP
        RAISE NOTICE 'Processing job_id: %, request_id: %', current_job.job_id, current_job.request_id;
        
        SELECT
            status,
            (response).status_code AS status_code,
            (response).body AS body
        INTO response_result
        FROM net._http_collect_response(current_job.request_id);

        IF response_result.status = 'SUCCESS' AND response_result.status_code BETWEEN 200 AND 299 THEN
            RAISE NOTICE 'Job completed (job_id: %)', current_job.job_id;
            
            UPDATE job_queue
            SET status = 'complete',
                content = response_result.body::TEXT
            WHERE job_id = current_job.job_id;
            
            DELETE FROM current_jobs
            WHERE request_id = current_job.request_id;
        ELSIF response_result.status = 'ERROR' THEN
            RAISE NOTICE 'Job failed (job_id: %)', current_job.job_id;
            
            UPDATE job_queue
            SET status = 'failed',
                retry_count = retry_count + 1
            WHERE job_id = current_job.job_id;
            
            DELETE FROM current_jobs
            WHERE request_id = current_job.request_id;
        ELSE
            RAISE NOTICE 'Job still in progress or not found (job_id: %)', current_job.job_id;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- 
-- Called after a record is insert in the queue
--
CREATE OR REPLACE FUNCTION process_job() 
RETURNS TRIGGER
SECURITY DEFINER
SET search_path = public, extensions, net, vault
AS $$
DECLARE
    request_id BIGINT;
    api_key TEXT;
    did_timeout BOOLEAN;
    response_message TEXT;
    response_status_code INTEGER;
BEGIN
    RAISE NOTICE 'Processing job_id: %', NEW.job_id;

    UPDATE job_queue
    SET status = 'processing'
    WHERE job_id = NEW.job_id;

    -- Get the API key
    SELECT decrypted_secret
    INTO api_key
    FROM vault.decrypted_secrets
    WHERE name = 'service_role';

    -- Call the request_wrapper to process the job
    request_id := request_wrapper(
        method := NEW.http_verb,
        url := 'https://contoso.supabase.co/functions/v1/consume_job' || COALESCE(NEW.url_path, ''),
        body := COALESCE(NEW.payload::jsonb, '{}'::jsonb),
        headers := jsonb_build_object('Authorization', 'Bearer ' || api_key, 'Content-Type', 'application/json')
    );

    INSERT INTO current_jobs (request_id, job_id)
    VALUES (request_id, NEW.job_id);

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- 
-- Convenience function around pg_net:
--
CREATE OR REPLACE FUNCTION request_wrapper(
    method TEXT,
    url TEXT,
    params JSONB DEFAULT '{}'::JSONB,
    body JSONB DEFAULT '{}'::JSONB,
    headers JSONB DEFAULT '{}'::JSONB
)
RETURNS BIGINT
SECURITY DEFINER
SET search_path = public, extensions, net
LANGUAGE plpgsql
AS $$
DECLARE
    request_id BIGINT;
    timeout INT;
BEGIN
    timeout := 3000;

    IF method = 'DELETE' THEN
        SELECT net.http_delete(
            url:=url,
            params:=params,
            headers:=headers,
            timeout_milliseconds:=timeout
        ) INTO request_id;
    ELSIF method = 'POST' THEN
        SELECT net.http_post(
            url:=url,
            body:=body,
            params:=params,
            headers:=headers,
            timeout_milliseconds:=timeout
        ) INTO request_id;
    ELSIF method = 'GET' THEN
        SELECT net.http_get(
            url:=url,
            params:=params,
            headers:=headers,
            timeout_milliseconds:=timeout
        ) INTO request_id;
    ELSE
        RAISE EXCEPTION 'Method must be DELETE, POST, or GET';
    END IF;

    RETURN request_id;
END;
$$;

-- 
-- Retrying jobs flagged as failures to increase reliability 
--
CREATE OR REPLACE FUNCTION retry_failed_jobs() 
RETURNS VOID 
SECURITY DEFINER
SET search_path = public, extensions, net, vault
AS $$
DECLARE
    r RECORD;
    request_id BIGINT;
    api_key TEXT;
    response_result net._http_response_result;
BEGIN
    RAISE NOTICE 'Retrying failed jobs';

    -- Get the API key
    SELECT decrypted_secret
    INTO api_key
    FROM vault.decrypted_secrets
    WHERE name = 'service_role';

    FOR r IN (
        SELECT * FROM job_queue
        WHERE status = 'failed' AND retry_count < retry_limit
        FOR UPDATE SKIP LOCKED
    ) LOOP
        RAISE NOTICE 'Retrying job_id: %', r.job_id;

        UPDATE job_queue
        SET retry_count = retry_count + 1
        WHERE job_id = r.job_id;

        -- Call the request_wrapper to process the job
        request_id := request_wrapper(
            method := r.http_verb,
            -- Edge function call (like AWS lambda)
            url := 'https://contoso.supabase.co/functions/v1/consume_job' || COALESCE(r.url_path, ''),
            body := COALESCE(r.payload::jsonb, '{}'::jsonb),
            headers := jsonb_build_object('Authorization', 'Bearer ' || api_key, 'Content-Type', 'application/json')
        );
        INSERT INTO current_jobs (request_id, job_id)
        VALUES (request_id, r.job_id);
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Adding the trigger to the queue table:
CREATE TRIGGER process_job_trigger
AFTER INSERT ON job_queue
FOR EACH ROW
EXECUTE FUNCTION process_job();
```

These functions work together to process jobs in the queue system, handle retries for failed jobs, and ensure the proper flow of job execution. We are also using Postgres [SKIP LOCKED](https://www.postgresql.org/docs/current/sql-select.html) feature on the following functions: `retry_failed_jobs()`, `process_current_jobs()` and `process_current_jobs_if_unlocked()`.

## Scheduling Tasks

Scheduling tasks is essential for regular job processing and retrying failed jobs. In this part, we'll explore how to schedule tasks using the `cron.schedule()` function in PostgreSQL.

### **Scheduling Retry Jobs**

To automatically retry failed jobs at specific intervals, we can use the `cron.schedule()` function. Here's an example:

```sql
SELECT cron.schedule(
    'retry_failed_jobs',
    '*/10 * * * *', 
    $$ SELECT retry_failed_jobs(); $$
);
```

In this example, the `retry_failed_jobs` function is scheduled to run every 10 minutes (`*/10 * * * *`). This function is responsible for retrying failed jobs by calling the `retry_failed_jobs()` function.

### **Scheduling Current Job Processing**

To process current jobs periodically, we can schedule the `process_current_jobs_if_unlocked` function. Here's an example:

```pgsql
SELECT cron.schedule(
    'process_current_jobs_if_unlocked_',
    '* * * * *',
    $$ SELECT process_current_jobs_if_unlocked(); $$
);
```

In this example, the `process_current_jobs_if_unlocked` function is scheduled to run every minute (`* * * * *`). This function checks if there is an unlocked worker available and processes current jobs if there is one.

### **Setting Up Sub-Minute Pooling**

For an optimal setup that distributes the workload evenly, we can use sub-minute pooling with the help of `pg_sleep()`. Here's an example:

```sql
-- Run this in PSQL to schedule pooling workers 20 seconds apart:
SET statement_timeout TO 0;
CREATE OR REPLACE FUNCTION schedule_jobs()
RETURNS VOID
AS $$
BEGIN
    -- Schedule retry job
    SELECT cron.schedule(
        'retry_failed_jobs',
        '*/10 * * * *', 
        $$ SELECT retry_failed_jobs(); $$
    );
    -- Schedule first job
    PERFORM cron.schedule(
        'process_current_jobs_if_unlocked_job_1',
        '* * * * *',
        $$ SELECT process_current_jobs_if_unlocked(); $$
    );
    -- Schedule second job with a 20-second delay
    PERFORM pg_sleep(20);
    PERFORM cron.schedule(
        'process_current_jobs_if_unlocked_job_2',
        '* * * * *',
        $$ SELECT process_current_jobs_if_unlocked(); $$
    );
    -- Schedule third job with another 20-second delay
    PERFORM pg_sleep(20);
    PERFORM cron.schedule(
        'process_current_jobs_if_unlocked_job_3',
        '* * * * *',
        $$ SELECT process_current_jobs_if_unlocked(); $$
    );
END;
$$ LANGUAGE plpgsql;
```

In this example, we define the `schedule_jobs()` function to set up the scheduling. Firstly, we set the statement timeout to 0 to avoid any timeouts during the execution. Then, we schedule the retry job and the first job to run immediately. After that, we introduce a 20-second delay using `pg_sleep(20)` before scheduling the second and third jobs. This delay helps distribute the jobs evenly across multiple workers.

By using the `cron.schedule()` function and setting up sub-minute pooling, we can ensure that failed jobs are retried periodically and parallel jobs are processed efficiently.

## Consuming the Queue with Supabase Edge Function

Supabase Edge Functions play a crucial role in serverless execution at the edge. They can be used to consume tasks from the job queue. Here's an example of an Edge Function that fetches and processes jobs from the queue:

```javascript
import { serve } from "https://deno.land/std@0.192.0/http/server.ts";

console.log("Function that will handle the tasks!");

serve(async (req) => {
  const payload = await req.text();
  const url = new URL(req.url);
  const headers = req.headers;
  const method = req.method;
  // Bundle variables in the 'data' variable as a JSON object
  const data = { payload, url, headers, method }; 
  return new Response(
    JSON.stringify(data),
    { headers: { "Content-Type": "application/json" } },
  );
});
```

## Securing the SQL Functions

Securing the SQL functions is of utmost importance to prevent unauthorized access. To revoke the execution permission on the functions from anonymous and authenticated users, you can use the following SQL code:

```sql
-- SQL code to revoke execute permission on the functions
REVOKE ALL PRIVILEGES ON FUNCTION process_current_jobs_if_unlocked
  FROM anon, authenticated;
REVOKE ALL PRIVILEGES ON FUNCTION process_job
  FROM anon, authenticated;
REVOKE ALL PRIVILEGES ON FUNCTION process_current_jobs
  FROM anon, authenticated;
REVOKE ALL PRIVILEGES ON FUNCTION request_wrapper 
  FROM anon, authenticated;
REVOKE ALL PRIVILEGES ON FUNCTION retry_failed_jobs
  FROM anon, authenticated;
```

## Conclusion

Building a queue system with Supabase and PostgreSQL offers a powerful solution for handling asynchronous tasks and managing job processing. By leveraging the table structure, functions, scheduled tasks, and edge functions, you can create a robust and scalable queue system.

In this blog post, we explored the process of setting up the queue system, which involved enabling the necessary PostgreSQL extensions, defining functions for job processing, scheduling tasks, securing the functions, and consuming the queue using Supabase Edge Functions. Note that this example takes into consideration the URL path to execute and you could expand it with similar ideas to the [RESTful service example.](https://github.com/supabase/supabase/blob/master/examples/edge-functions/supabase/functions/restful-tasks/index.ts)

With a complete understanding of both processing and consuming the jobs in the queue, you now have the knowledge to implement a highly efficient queue system in your applications. Happy queuing!