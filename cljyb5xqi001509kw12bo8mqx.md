---
title: "Testing Supabase Edge Functions with Deno Test"
seoTitle: "Testing Supabase Edge Functions with Deno Test - a complete guide"
seoDescription: "Learn how to leverage Supabase Edge Functions with Deno Test to enhance your backend development. A step-by-step guide with practical examples."
datePublished: Tue Jul 11 2023 13:08:33 GMT+0000 (Coordinated Universal Time)
cuid: cljyb5xqi001509kw12bo8mqx
slug: testing-supabase-edge-functions-with-deno-test
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1688568563653/68f370fa-df30-4f9e-a68e-7363662c1a64.png
tags: unit-testing, deno, supabase, edge-functions

---

Backend development requires robust and reliable tools in today's rapidly evolving technology landscape. While Firebase has been a go-to choice for many developers, there is now an open-source alternative called Supabase that offers a PostgreSQL-based backend. [Supabase](https://supabase.com/) combines the robustness of PostgreSQL with the ease of use and scalability of Firebase, providing a compelling solution for backend development. One crucial aspect of utilizing Supabase is testing its Edge Functions, which play a vital role in extending the functionality of your application while ensuring its reliability. In this blog post, we will explore the process of setting up Supabase, delve into the world of Supabase Edge Functions, and learn how to write and test a simple `hello-world` function and run Edge Functions locally for efficient development.

### Setting up Supabase and PostgreSQL

To get started with Supabase, we must first install and set it up using the Command Line Interface (CLI). The CLI offers a convenient way to initialize and manage Supabase projects effortlessly. Let's walk through the steps to install Supabase CLI and set up a Supabase project:

1. Install Docker: Supabase relies on Docker for local development. Install Docker by following the instructions on the official [Docker website](https://docs.docker.com/desktop/install/mac-install/).
    
2. Install Supabase CLI: Open your terminal and run the following command to install the Supabase CLI:
    
    ```bash
    npm install -g supabase
    ```
    
3. Initialize a Supabase project: In your terminal, navigate to the desired directory and run the following command to initialize a new Supabase project:
    
    ```bash
    supabase init
    ```
    
    This command sets up a new Supabase project and creates the necessary configuration files.
    
4. Start the Supabase server: Once the initialization is complete, start the Supabase server using the following command:
    
    ```bash
    supabase start
    ```
    
    This command starts the Supabase server locally on your machine, allowing you to interact with the Supabase backend.
    
5. PostgreSQL as the underlying database: Supabase leverages PostgreSQL as its underlying database. This powerful and reliable database management system ensures the stability and scalability of your backend. You can interact with PostgreSQL using Supabase's intuitive APIs and tools.
    

### Overview of Supabase Edge Functions

Supabase Edge Functions are an integral part of the Supabase ecosystem, empowering developers to extend the functionality of their applications with serverless computing. These functions run at the edge of the Supabase infrastructure, ensuring optimal performance and reduced latency. Let's explore the advantages of using Supabase Edge Functions:

* Serverless computing: Supabase Edge Functions provide a serverless architecture, allowing you to focus on writing code without the need to manage server infrastructure. This results in reduced operational overhead and improved scalability.
    
* TypeScript support: Edge Functions can be written in TypeScript, a statically-typed superset of JavaScript. TypeScript brings type safety to your code, enabling early detection of potential errors and improving overall code quality.
    
* Seamless integration with Supabase client: Edge Functions seamlessly integrates with the Supabase client, enabling easy communication with your Supabase backend. This integration allows you to leverage the full power of Supabase's APIs and functionalities within your Edge Functions.
    

### Understanding the `hello-world` Edge Function with CORS

Now, let's dive into the details of the "Hello-world" Edge Function written in TypeScript and explore its functionality along with Cross-Origin Resource Sharing (CORS) implementation. We'll break down the code step by step, providing a clear understanding of each section and its interaction with the Supabase client.

Here's the code snippet for the `hello-world` Edge Function:

```javascript
// Import the serve function from the Deno standard library
import {
    serve
} from "https://deno.land/std@0.192.0/http/server.ts"

// Print a greeting message to the console
console.log("Hello from Functions!")

// Define the headers to handle CORS (Cross-Origin Resource Sharing)
const corsHeaders = {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
};

// Start the server and define the request handler
serve(async(req) => {
    // If the HTTP method is OPTIONS, return a response with CORS headers
    if (req.method === 'OPTIONS') {
        return new Response('ok', {
            headers: corsHeaders
        })
    }
    // Extract the name from the request body
    const {
        name
    } = await req.json()
    // Prepare the response data
    const data = {
        message: `Hello ${name}!`,
    }

    // Return a response with the data and headers
    return new Response(
        JSON.stringify(data), {
            headers: {
                ...corsHeaders,
                "Content-Type": "application/json"
            },
        }
    )
})
```

In this section, we'll analyze the different aspects and functionalities of the `Hello-world` Edge Function, providing insights into each segment's purpose and its seamless integration with the Supabase client.

* We import the `serve` function from the Deno standard library to create a server that listens for incoming HTTP requests.
    
* The `console.log` statement is for logging purposes and will be displayed in the console when the function is executed.
    
* We define the `corsHeaders` object to handle Cross-Origin Resource Sharing (CORS) headers, allowing requests from any origin and specifying the allowed headers.
    
* The `serve` function takes an asynchronous callback function as a parameter, which handles the incoming requests.
    
* In the callback function, we check if the request method is `OPTIONS`. If it is, we return a response with a status of 'ok' and the CORS headers.
    
* If the request method is not `OPTIONS`, we parse the JSON data from the request body and extract the `name` field.
    
* We construct a response object with a JSON payload containing a dynamic greeting message using the provided name.
    
* Finally, we return the response object along with the CORS headers.
    

Now, we need to [run the function locally](https://supabase.com/docs/guides/functions/local-development#running-edge-functions-locally) :

```bash
supabase functions serve
```

### Testing Supabase Edge Functions

Testing is a crucial step in the development process to ensure the correctness and performance of your Edge Functions. Let's discuss the importance of testing and examining the code in the `deno-test.ts` file, which demonstrates how to test the Supabase client and the `hello-world` function.

```typescript
// deno-test.ts

// Import necessary libraries and modules
import {
  assert,
  assertExists,
  assertEquals,
} from "https://deno.land/std@0.192.0/testing/asserts.ts";
import {
  createClient,
  SupabaseClient,
} from "https://esm.sh/@supabase/supabase-js@2.23.0";
import { delay } from 'https://deno.land/x/delay@v0.2.0/mod.ts';

// Setup the Supabase client configuration
const supabaseUrl = Deno.env.get("SUPABASE_URL") ?? "";
const supabaseKey = Deno.env.get("SUPABASE_ANON_KEY") ?? "";
const options = {
  auth: {
    autoRefreshToken: false,
    persistSession: false,
    detectSessionInUrl: false
  }
};

// Test the creation and functionality of the Supabase client
const testClientCreation = async () => {
  var client: SupabaseClient = createClient(supabaseUrl, supabaseKey, options);

  // Check if the Supabase URL and key are provided
  if (!supabaseUrl) throw new Error('supabaseUrl is required.')
  if (!supabaseKey) throw new Error('supabaseKey is required.')

  // Test a simple query to the database
  const { data: table_data, error: table_error } = await client.from('my_table').select('*').limit(1);
  if (table_error) {
    throw new Error('Invalid Supabase client: ' + table_error.message);
  }
  assert(table_data, "Data should be returned from the query.");
};

// Test the 'hello-world' function
const testHelloWorld = async () => {
  var client: SupabaseClient = createClient(supabaseUrl, supabaseKey, options);

  // Invoke the 'hello-world' function with a parameter
  const { data: func_data, error: func_error } = await client.functions.invoke('hello-world', {
    body: { name: 'bar' }
  });

  // Check for errors from the function invocation
  if (func_error) {
    throw new Error('Invalid response: ' + func_error.message);
  }

  // Log the response from the function
  console.log(JSON.stringify(func_data, null, 2));

  // Assert that the function returned the expected result
  assertEquals(func_data.message, 'Hello bar!');
};

// Register and run the tests
Deno.test("Client Creation Test", testClientCreation);
Deno.test("Hello-world Function Test", testHelloWorld);
```

This is a test case with two parts. The first tests the client library and check that the database is connectable and returning values from a table (`my_table`). The second part is testing the edge function and checks if the value received is expected. Here's a brief overview of the code:

* We import various testing functions from the Deno standard library, including `assert`, `assertExists`, and `assertEquals`.
    
* We import the `createClient` and `SupabaseClient` classes from the `@supabase/supabase-js` library to interact with the Supabase client.
    
* We define the necessary configuration for the Supabase client, including the Supabase URL, API key, and authentication options.
    
* The `testClientCreation` function tests the creation of a Supabase client instance and queries the database for data from a table. It asserts that data is returned from the query.
    
* The `testHelloWorld` function tests the "Hello-world" Edge Function by invoking it using the Supabase client's `functions.invoke` method. It checks if the response message matches the expected greeting.
    
* We run the tests using the `Deno.test` function, providing a descriptive name for each test case and the corresponding test function.
    

Note: Please make sure to replace the placeholders (`supabaseUrl`, `supabaseKey`, `my_table`) with the actual values relevant to your Supabase setup.

### Running Edge Functions Locally

To test and debug Edge Functions locally, you can utilize the Supabase CLI. Let's explore how to run Edge Functions locally using the Supabase CLI:

1. Ensure that the Supabase server is running by executing the following command:
    
    ```bash
    supabase start
    ```
    
2. In your terminal, use the following command to serve the Edge Functions locally:
    
    ```bash
    supabase functions serve
    ```
    
    This command starts a local server that runs your Edge Functions, allowing you to test and debug them in a development environment.
    
3. Create the environment variables file:
    
    ```bash
    # creates the file
    touch .env.local
    # adds the SUPABASE_URL secret
    echo "SUPABASE_URL=http://localhost:54321" >> .env.local
    # adds the SUPABASE_ANON_KEY secret
    echo "SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZS1kZW1vIiwicm9sZSI6ImFub24iLCJleHAiOjE5ODM4MTI5OTZ9.CRXP1A7WOeoJeXxjNni43kdQwgnWNReilDMblYTn_I0" >> .env.local
    #Alternatively, you can open in your editor:
    open .env.local
    ```
    
4. To run the tests, use the following command in your terminal:
    
    ```bash
    deno test --allow-all deno-test.ts --env-file .env.local
    ```
    
    Here's an example of what this would look like:
    
    ![Example of running tests](https://cdn.hashnode.com/res/hashnode/image/upload/v1688508225738/09b364ce-1fd7-419e-a11a-bf563527df97.png align="left")
    

### Conclusion

In this blog post, we covered the process of setting up Supabase and PostgreSQL for backend development. We explored the concept of Supabase Edge Functions and their advantages in extending the functionality of your application. We also walked through the process of writing a simple "Hello-world" function and discussed the importance of testing Supabase Edge Functions for ensuring their correctness and performance. Lastly, we learned how to run Edge Functions locally using the Supabase CLI. By incorporating Supabase and PostgreSQL into your backend stack and thoroughly testing your Edge Functions, you can maintain the reliability and functionality of your application.

To learn more about Supabase and its features, consult the official [Supabase documentation](https://supabase.com/docs). For information on Deno and its testing framework, refer to the [Deno documentation](https://deno.land/manual). Additionally, you can explore the [Supabase GitHub repository](https://github.com/supabase/supabase).