---
title: "Using Supabase-JS as a Script in Your Terminal"
datePublished: Wed May 03 2023 17:00:41 GMT+0000 (Coordinated Universal Time)
cuid: clh7y2pgj03wbfdnv2nwm2vsr
slug: using-supabase-js-as-a-script-in-your-terminal
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1683129164164/ec173d06-d7ee-42b2-91f1-6480f17bb599.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1683130496701/ab7fd99b-9cf6-4425-847b-cdb2814dd1ef.png
tags: javascript, terminal, nodejs, supabase, row-level-security

---

Supabase-JS is an isomorphic JavaScript client for [Supabase](https://supabase.com/), a platform that provides a Postgres database, authentication, storage, and serverless functions. While Supabase-JS is commonly used for interacting with your database, listening to changes, managing users, and uploading files, did you know that you can also use it as a script to run tasks from your terminal? In this blog post, we will show you how to set up the environment for using Supabase-JS as you would call a Bash or Python script. This can be useful for performing operations on your data, testing your queries, or automating workflows.

## Prerequisites

To follow along with this tutorial, you will need:

* A Supabase account and project. You can sign up for free at [https://supabase.com](https://supabase.com)
    
* Node.js and npm are installed on your machine. You can check the versions by running `node -v` and `npm -v` in your terminal.
    
* A code editor of your choice.
    

## Setting up the project

First, let's create a new folder for our project and initialize it with npm:

```bash
mkdir supabase-script
cd supabase-script
npm init -y
```

This will create a `package.json` file with some default values. Next, let's install the Supabase-JS library as a dependency:

```bash
npm install @supabase/supabase-js
```

This will add the `@supabase/supabase-js` package to our `package.json` file and download it to the `node_modules` folder.

### Setting up dotenv:

To securely store your Supabase project's URL and API key, we will use the dotenv package. This package will read a .env file in your project's root directory and load any environment variables from it. First, install the package:

```bash
npm install dotenv
```

## Setting up package.json for standalone script:

Since the goal is to create standalone scripts, you need to set the `type` as `module`. Here's an example of how to set up package.json:

```json
{
  "name": "supabase-script",
  "version": "1.0.0",
  "description": "",
  "keywords": [],
  "author": "",
  "license": "MIT",
  "dependencies": {
    "@supabase/supabase-js": "^2.21.0",
    "dotenv": "^16.0.3"
  },
  "type": "module"
}
```

## Setting up the .env file

Here we create a `.env` file where we'll add the `SUPABASE_URL` and `API KEYS`. Creating the file in bash:

```bash
touch .env
```

Open the file in your code editor:

```bash
open .env
```

Now, set up the variables in the file:

```bash
SUPABASE_URL=https://supa_nacho.supabase.co

SUPABASE_SERVICE_ROLE=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Im5vIPCfjaogZm9yIHlvdSIsInJvbGUiOiLwn6W3IEFyZSB5b3UgdHJ5aW5nIHRvIGhhY2ssIGVoP_CfpbciLCJpYXQiOjE2NzA4ODE2NjEsImV4cCI6MTk4NjQ1NzY2MX0.NpuTP4jyvMniVMVmI157fCWdat1GUCNrD9KMk4Wmr9U

SUPABASE_ANON=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6IvCfpbcgQXJlIHlvdSB0cnlpbmcgdG8gaGFjaywgZWg_8J-ltyIsInJvbGUiOiJhbm9uIiwiaWF0IjoxNjcwODgxNjYxLCJleHAiOjE5ODY0NTc2NjF9.5SlUOeIh5x26VInEUjhKpHhe7S9VguiSOEIu51M-Bpw
```

Having the service\_role and anon key is nice so that you can test locally diverse scenarios. Also, this allows you to use the [Auth Admin API](https://supabase.com/docs/reference/javascript/auth-admin-inviteuserbyemail) in quick scripts.

In the next part, we will write a simple script to connect to your Supabase database and perform a query.

## Creating the script

Now that we have the library installed, let's create a new file `service_script.js` in our project folder. This is where we will write our Supabase-JS code.

To use Supabase-JS, we need to import the `createClient` function from the library and create an instance of the client with our project URL and public API key. You can find these values in your Supabase dashboard under [Settings &gt; API](https://app.supabase.com/project/_/settings/api).

```javascript
import { createClient } from '@supabase/supabase-js'
// Load environment variables from .env file
import dotenv from 'dotenv' 
dotenv.config()
// Get Supabase URL from environment variables and log it
const SUPABASE_URL = process.env.SUPABASE_URL
console.log(SUPABASE_URL)
const SUPABASE_SERVICE =process.env.SUPABASE_SERVICE_ROLE
const SUPABASE_ANON_KEY =process.env.SUPABASE_ANON

// Using options here, in case you want to make changes to the default
const options =  {
  db: {
  schema: 'public',
  }
};
// Initializing with SERVICE_ROLE key:
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE, options); 
// Initializing with ANON key:
/*
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, options); 
*/ 
//Some test call here (a INNER JOIN that counts the related records in foreign keys): 
const {data:data , error} = await supabase
.from('teams')
.select('*, team_members!inner(count)')
//We can also apply Javascript filters on the data:
const filteredData = data.filter(team => {
  return team.team_members[0].count > 0;
});
//Display on console:
console.log(JSON.stringify(filteredData,null,2));
//Check for errors
if (error) {
  console.log("error ");
  throw error;
}
```

Now, we call the script from the terminal:

```bash
node service_script.js
```

# Testing RLS policies

This is slightly trickier to set it up, but it is extremely useful to help debug RLS and test it with the client libraries. Please note that you can also [test them in SQL](https://blog.mansueli.com/using-custom-claims-testing-rls-with-supabase) as covered before here on the blog. Let's create a logged\_script, to make API calls as an authenticated user.

```javascript
import { createClient } from '@supabase/supabase-js'
// Load environment variables from .env file
import dotenv from 'dotenv' 
dotenv.config()
// Get Supabase URL from environment variables and log it
const SUPABASE_URL = process.env.SUPABASE_URL
const SUPABASE_SERVICE =process.env.SUPABASE_SERVICE_ROLE
const SUPABASE_ANON_KEY =process.env.SUPABASE_ANON

// Using options here, with extra options for using Auth as script:
const options =  {
  db: {
    schema: 'public',
  },
  auth: {
    autoRefreshToken: false,
    persistSession: false,
    detectSessionInUrl: false
  }
};
// Initializing with ANON key:
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, options); 
//Signing into our test user:
const { data: {session} , error} =  await supabase.auth.signInWithPassword({
  email: 'rodrigo@contoso.com',
  password: '<secret pass here>',
});
const opt = {
  global: {
    headers: { Authorization: "Bearer "+session.access_token },
  },
};
//This client is logged as rodrigo@contoso.com
const supa_logged = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, opt);
//Query with logged user & RLS
const { data: user, error } = await supa_logged.from('team_members').select('*');
console.log(JSON.stringify(user));
//Check for errors
if (error) {
  console.log("error ");
  throw error;
}
```

We can call the script like in the previous example:

```bash
node logged_script.js
```

In this blog post, we learned how to use Supabase-JS in a Node.js environment to interact with our Supabase project. We also explored how to test RLS policies using a script authenticated with Supabase's API. By following the steps outlined here, you can build powerful and secure applications with Supabase and test your policies with the client libraries.